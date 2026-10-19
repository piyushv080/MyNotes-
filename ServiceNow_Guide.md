# ServiceNow Revision Guide (Automation / Integration Engineer)

Scope: platform basics, Service Catalog, Flow Designer, REST/Table API, OAuth, CMDB and IRE, GlideRecord scripting, Script Includes, IntegrationHub, MID Server, plus the pieces that connect ServiceNow to Ansible/AAP.

How to use: do each section on your PDI. Type the scripts into **Scripts - Background** (System Definition > Scripts - Background) or the relevant record form and run them. You already hold CSA and CAD, so this guide skips basic navigation and focuses on what you are asked in design reviews and integration interviews.

> Release note: menu names and some UI labels shift between releases (Xanadu, Yokohama, Zurich...). The concepts and APIs below are stable. Verify exact labels on your PDI.

---

## 0. Mental model

- ServiceNow is a **single-platform, table-based** system. Everything is a record in a table; tables extend other tables (`incident` extends `task`, `cmdb_ci_server` extends `cmdb_ci`).
- **Server-side** JavaScript (Rhino-based, ES5 by default; ES12 is available per-script with the setting on newer releases) runs in business rules, script includes, flow actions, scheduled jobs. **Client-side** runs in the browser (client scripts, UI policies).
- Automation entry points: **Catalog item → Flow → Actions (REST/IntegrationHub/script) → Records**, or **inbound REST → Table API / Scripted REST → Records**.
- Enterprise lens: ACLs, update sets/scoped apps, audit (`sys_audit`, journal fields), change control, MID Server for private networks, no hardcoded credentials.

---

## 1. Platform fundamentals you must be fluent in

| Concept | What to know |
|---|---|
| Tables and dictionary | `sys_db_object`, `sys_dictionary`, extending tables, `sys_id` (32-char GUID), reference fields, choice lists |
| Task table | `task` parent of incident, change_request, problem, sc_req_item, sc_task; shared fields `number`, `state`, `assigned_to`, `assignment_group` |
| Business Rule | Server script on insert/update/delete/query. `before`, `after`, `async`, `display`. Prefer `before` for field changes (no extra update) |
| Client Script / UI Policy | Browser-side form logic. Prefer UI Policy (no code) over Client Script |
| ACL | Row/field-level security. Evaluated server-side; scripts run with the user’s rights except in `GlideRecord` (does not enforce ACL, use `GlideRecordSecure`) |
| Scheduled Job | Time-based server script |
| Update Set vs Scoped App | Update sets move config between instances; scoped apps (`x_snc_*`) isolate code and can be versioned in Git (Source Control integration) |
| System Properties | `gs.getProperty('x_myapp.endpoint')` for config instead of hardcoding |
| Import Set / Transform Map | Staged data load, field mapping, `onBefore/onAfter` scripts, coalesce for upserts |

Debug tools: **Scripts - Background**, **System Logs** (`gs.info/warn/error`), **Script Debugger**, **Session Debug**, **Flow execution context**, **Transaction log**, **Outbound HTTP Requests log** (`sys_outbound_http_log`), **REST API Explorer**.

---

## 2. Service Catalog

### Building blocks

```
Catalog (sc_catalog)
└─ Category
     └─ Catalog Item (sc_cat_item)
         ├─ Variables (item_option_new): single line text, select box, reference, checkbox, ...
         ├─ Variable Set (reusable)
         ├─ Catalog UI Policy / Catalog Client Script
         ├─ Fulfillment: Flow (preferred) / Workflow (legacy) / Execution Plan
         └─ Request (sc_request) → Requested Item (sc_req_item, "RITM") → Catalog Task (sc_task)
```

### Example item: "Request Enterprise App Server"

Variables on the item:

| Name | Type | Notes |
|---|---|---|
| `server_name` | Single Line Text | Mandatory, validation regex `^[a-z0-9-]{3,30}$` (use a Catalog Client Script or variable validation regex) |
| `env` | Select Box | choices: dev, test, prod |
| `requested_by` | Reference → `sys_user` | Default: current user via default value `javascript:gs.getUserID()` |

Catalog Client Script (type **onSubmit** on the item) for validation:

```javascript
function onSubmit() {
    var name = g_form.getValue('server_name');
    if (!/^[a-z0-9-]{3,30}$/.test(name)) {
        g_form.showFieldMsg('server_name', 'Use 3-30 lowercase letters, digits or hyphens', 'error');
        return false;               // block submission
    }
    return true;
}
```

Catalog UI Policy examples (no code): show `approval_justification` only when `env == prod`.

### Reading variables in server scripts

```javascript
// In a business rule on sc_req_item or a flow script step
var serverName = current.variables.server_name.toString();
var env        = current.variables.env.toString();
var requester  = current.variables.requested_by.getDisplayValue();
```

Notes:
- Variables live in `sc_item_option`/`sc_item_option_mtom`; access with `current.variables.<name>` on RITM, or `GlideRecord` on `sc_item_option_mtom` for bulk use.
- **Record producers** create records in any table from a catalog form (e.g. create an incident); catalog items create RITMs.
- **Order guides** bundle multiple items; **Variable sets** reuse variable groups.
- **Approvals**: approval rules (`sysapproval_approver`), Flow Designer *Ask for Approval* action, or Task approvals.
- Entitlements (user criteria) control who sees an item.

---

## 3. Flow Designer

### Structure

```
Flow
├─ Trigger: Service Catalog | Record (created/updated) | Scheduled | REST API (inbound) | Application...
├─ Actions (OOTB or custom): Create Record, Update Record, Look Up Records, Ask for Approval, Wait for condition, Send email, ...
├─ Flow logic: If / Else, For Each, Do Until, Parallel, Wait For, Call Subflow, Try/Catch ("Error handling")
├─ Data pills: outputs of earlier steps (Trigger > Requested Item > Variables > server_name)
└─ Subflows (reusable, parameterised), Custom Actions (own inputs/outputs/script/REST steps)
```

### Example flow: approval then call Ansible/AAP

1. **Trigger**: Service Catalog, item = *Request Enterprise App Server*.
2. **Ask for Approval**: approver = manager (`Requested for > Manager`), rule = approve.
3. **If** approval = *Approved*:
   - **Custom Action / REST step** → POST to AAP launch API (see section 8).
   - **Update Record** RITM: state = Work in progress, add work note with job id.
   - **Wait For Condition**: RITM field `u_job_status` = `successful` (or poll the AAP job with a Do Until loop).
4. **Else** (rejected): **Update Record** RITM state = Closed Incomplete.
5. **Error handler** (flow-level): create an incident with the error message.

Key facts:
- Flows run as **system user or the flow owner** depending on run-as settings. Know this for ACL questions.
- **Flow Designer vs Workflow Editor**: Flow Designer is the current standard; legacy Workflow is maintained but not for new work.
- **Execution details** (Operations > Flow Designer > Executions) show per-step inputs/outputs, the first place to debug.
- **Subflow** = callable from flows/other subflows with typed inputs/outputs. **Custom action** = a reusable step packaged with script or IntegrationHub steps.
- **Script step** inside a custom action:

```javascript
(function execute(inputs, outputs) {
    // inputs.server_name, inputs.env  → defined in the action's Inputs tab
    outputs.payload = JSON.stringify({
        extra_vars: { server_name: inputs.server_name, env: inputs.env }
    });
})(inputs, outputs);
```

- Flow **Error handling** (Try/Catch-like): each action can be configured to *Continue on error* with the error status available as a data pill, or use the flow-level handler.
- Concurrency note: **Wait for condition** and long-running flows keep a context; use `Wait For Duration` or asynchronous callbacks instead of busy polling where possible.

---

## 4. REST and the Table API

### Inbound: Table API endpoints

Base: `https://<instance>.service-now.com/api/now/table/<table>`

| Operation | Method | Path |
|---|---|---|
| List | GET | `/api/now/table/incident?sysparm_query=active=true^priority=1&sysparm_limit=10` |
| Read | GET | `/api/now/table/incident/{sys_id}` |
| Create | POST | `/api/now/table/incident` |
| Partial update | PATCH | `/api/now/table/incident/{sys_id}` |
| Replace | PUT | `/api/now/table/incident/{sys_id}` |
| Delete | DELETE | `/api/now/table/incident/{sys_id}` |

Useful query parameters:

| Parameter | Use |
|---|---|
| `sysparm_query` | Encoded query: `active=true^caller_id=<sys_id>^ORDERBYDESCsys_created_on` |
| `sysparm_fields` | Return only these fields (smaller payload) |
| `sysparm_limit`, `sysparm_offset` | Pagination (also `Link` header with next/prev) |
| `sysparm_display_value` | `true` (display), `false` (raw), `all` (both) |
| `sysparm_exclude_reference_link` | Drop the reference link objects |
| `sysparm_input_display_value` | Treat input values as display values |

### curl examples

```bash
INSTANCE=https://devXXXXX.service-now.com

# Create incident (basic auth only for dev; use OAuth in real use)
curl -s -X POST "$INSTANCE/api/now/table/incident" \
  -u "$SN_USER:$SN_PASS" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"short_description":"Deployment failed on web01","impact":"2","urgency":"2","caller_id":"admin"}'

# Update by sys_id (state 2 = In Progress; verify numeric choice values on your instance)
curl -s -X PATCH "$INSTANCE/api/now/table/incident/$SYS_ID" \
  -u "$SN_USER:$SN_PASS" -H "Content-Type: application/json" \
  -d '{"state":"2","work_notes":"Investigating"}'

# Query with fields and limit
curl -s -G "$INSTANCE/api/now/table/incident" -u "$SN_USER:$SN_PASS" \
  --data-urlencode "sysparm_query=active=true^priority=1" \
  --data-urlencode "sysparm_fields=number,short_description,state" \
  --data-urlencode "sysparm_limit=5"
```

Response shape: `{"result": {...}}` for a single record, `{"result": [...]}` for lists. Errors: `{"error": {"message": "...", "detail": "..."}, "status": "failure"}` with HTTP 400/401/403/404.

### REST API Explorer

System Web Services > REST API Explorer: choose namespace/API/version/table, fill parameters, click **Send**, then use the language icons (Python, curl, JavaScript, Ruby) to copy generated code. Use it to learn request shapes quickly.

### Other inbound APIs worth knowing

| API | Use |
|---|---|
| **Table API** | Generic CRUD on any table, respects ACLs |
| **Aggregate API** | `/api/now/stats/<table>`, counts/sums without pulling rows |
| **Import Set API** | `/api/now/import/<staging_table>`, goes through transform maps (best for bulk/ETL) |
| **Attachment API** | Upload/download files |
| **Scripted REST API** | Your own endpoint with custom logic (below) |
| **CMDB Instance API** | `/api/now/cmdb/instance/<class>`, create/update CIs with relations |
| **Identification and Reconciliation API** | `/api/now/identifyreconcile`, create/update CIs through the IRE (preferred for CMDB writes) |

### Scripted REST API (custom endpoint)

Create in System Web Services > Scripted REST APIs. Resource: `POST /api/x_myapp/provision`.

```javascript
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {
    var body = request.body.data;                 // already-parsed JSON
    if (!body.server_name) {
        response.setStatus(400);
        return { error: 'server_name is required' };
    }

    var gr = new GlideRecord('sc_req_item');
    if (!gr.get(body.ritm_sys_id)) {
        response.setStatus(404);
        return { error: 'RITM not found' };
    }

    gr.setValue('work_notes', 'Provisioning callback: ' + body.status);
    gr.update();

    response.setStatus(200);
    return { result: 'ok', number: gr.getValue('number') };
})(request, response);
```

Set the resource to **Requires authentication** and add an **ACL/role** requirement. Never leave public.

### Outbound REST (ServiceNow calls others)

Methods: **REST Message** (System Web Services > Outbound > REST Message, reusable, stores endpoint/auth/headers/HTTP methods), **RESTMessageV2** (scripted), or **Flow Designer REST step** (IntegrationHub).

```javascript
// Outbound via a saved REST Message named 'AAP Controller', method 'launch_job'
try {
    var r = new sn_ws.RESTMessageV2('AAP Controller', 'launch_job');
    r.setStringParameterNoEscape('template_id', '42');
    r.setRequestBody(JSON.stringify({ extra_vars: { server_name: 'app01', env: 'dev' } }));
    r.setHttpTimeout(30000);
    // r.setMIDServer('mid-prod-01');            // use a MID Server for private endpoints
    var resp = r.execute();
    var status = resp.getStatusCode();
    var body = resp.getBody();
    if (status !== 201) {
        gs.error('AAP launch failed: ' + status + ' ' + body);
    } else {
        var job = JSON.parse(body);
        gs.info('AAP job id ' + job.job);
    }
} catch (e) {
    gs.error('Outbound REST error: ' + e.message);
}
```

Always: set timeouts, check status, log, avoid secrets in scripts (use Connection & Credential aliases).

---

## 5. Authentication and OAuth

### Options

| Method | Use |
|---|---|
| Basic auth | Quick tests only; username/password on every call |
| OAuth 2.0 (inbound) | ServiceNow as the resource server; clients get tokens |
| OAuth 2.0 (outbound) | ServiceNow calls external APIs; profile + credential record |
| API key / Token (inbound, newer releases) | Simple token per integration with Inbound Authentication Profile |
| mTLS, SAML, SSO | User auth and high-security integrations |

### Setting up inbound OAuth (client credentials / password grant)

1. **System OAuth > Application Registry** → New → **Create an OAuth API endpoint for external clients**.
   - Name, Client ID (auto), Client Secret (auto or set), Redirect URL (for auth-code), Token lifespan (access: 1800 s default; refresh: 8640000 s default).
2. Pick/assign an **integration user** with minimal roles (e.g. `itil` only, or a custom role). Never `admin`.
3. Request a token. The legacy endpoint uses the **password grant**:

```bash
# Password grant (legacy, still the common ServiceNow pattern)
curl -s -X POST "$INSTANCE/oauth_token.do" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=$CLIENT_ID" -d "client_secret=$CLIENT_SECRET" \
  -d "username=$SN_USER" --data-urlencode "password=$SN_PASS"
# → {"access_token":"...","refresh_token":"...","scope":"useraccount","token_type":"Bearer","expires_in":1799}

# Refresh
curl -s -X POST "$INSTANCE/oauth_token.do" \
  -d "grant_type=refresh_token" -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET" -d "refresh_token=$REFRESH"

# Use the token
curl -s "$INSTANCE/api/now/table/incident?sysparm_limit=1" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -H "Accept: application/json"
```

Newer releases also support the **client credentials** grant (`grant_type=client_credentials`) when enabled on the endpoint. Check the Application Registry "Client credentials" option and the instance release notes.

### Python helper (token + Table API call)

```python
import os
import time
import requests

INSTANCE = os.environ["SN_INSTANCE"]  # https://devXXXXX.service-now.com


def get_token() -> dict:
    r = requests.post(
        f"{INSTANCE}/oauth_token.do",
        data={
            "grant_type": "password",
            "client_id": os.environ["SN_CLIENT_ID"],
            "client_secret": os.environ["SN_CLIENT_SECRET"],
            "username": os.environ["SN_USER"],
            "password": os.environ["SN_PASS"],
        },
        timeout=30,
    )
    r.raise_for_status()
    return r.json()


def create_incident(token: str, short_description: str) -> dict:
    r = requests.post(
        f"{INSTANCE}/api/now/table/incident",
        headers={"Authorization": f"Bearer {token}", "Accept": "application/json"},
        json={"short_description": short_description, "impact": "2", "urgency": "2"},
        timeout=30,
    )
    r.raise_for_status()
    return r.json()["result"]


if __name__ == "__main__":
    tok = get_token()["access_token"]
    inc = create_incident(tok, "Created by script")
    print(inc["number"], inc["sys_id"])
```

### Outbound OAuth

Create **Connections & Credentials > Credentials > OAuth 2.0 Credential** (or OAuth Entity + Profile), reference it in the REST Message/HTTP connection. ServiceNow stores tokens and refreshes them for you. Use **Connection & Credential Aliases** so the same flow works in dev and prod with different credentials.

Security checklist: least-privilege integration user, rotate secrets, short token lifespan, IP allow-listing where possible, secrets in credential records (encrypted) not in scripts or properties, monitor `oauth_credential` and `sys_audit`.

---

## 6. CMDB and Identification & Reconciliation (IRE)

### CMDB basics

- Core table `cmdb_ci` with class hierarchy: `cmdb_ci` → `cmdb_ci_hardware` → `cmdb_ci_computer` → `cmdb_ci_server` → `cmdb_ci_linux_server`, `cmdb_ci_win_server`; apps (`cmdb_ci_appl`), services, cloud objects (`cmdb_ci_vm_instance`, `cmdb_ci_aws_datacenter`, ...).
- **Relationships**: `cmdb_rel_ci` (parent, child, type like *Runs on::Runs*, *Depends on::Used by*).
- **CSDM** (Common Service Data Model) defines how Business Services, Application Services, and Technical Services link to CIs.
- Health: **CMDB Health Dashboard** with Completeness, Correctness, Compliance. Data should come from authoritative sources (Discovery, Service Graph connectors), not manual edits.

### Why IRE exists

Many sources (Discovery, SCCM, AWS, Ansible, manual) all report the same server. Without control you get **duplicate CIs**. IRE solves it with:

1. **Identification (identifier rules)**: decides *which CI this payload refers to*. Rules are ordered **identifier entries** on a class with attributes, e.g. for servers: (1) `serial_number` + `serial_number_type`, (2) `name` + `sys_class_name`, (3) `ip_address` via lookup tables (`cmdb_ci_network_adapter`). The first rule that matches wins; no match → create; multiple matches → error (duplicate).
2. **Reconciliation (data source precedence)**: decides *which source may update which attribute*. Each attribute can have **reconciliation rules**: e.g. `ip_address` can be written by Discovery (priority 100), not by Ansible (priority 50). Lower-priority sources cannot overwrite higher-priority data.
3. **Dependent / related items**: payload can include related items (e.g. NIC, disk) in the same call.
4. **Stale / Duplicate remediation, Class migration**: operations that handle changed classification.

### Writing to CMDB the right way

Use the **IRE API**, not direct `cmdb_ci` table inserts:

```bash
curl -s -X POST "$INSTANCE/api/now/identifyreconcile?sysparm_data_source=Ansible" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "items": [{
      "className": "cmdb_ci_linux_server",
      "values": {
        "name": "app01",
        "ip_address": "10.0.1.15",
        "serial_number": "i-0abc123def4567890",
        "serial_number_type": "provider",
        "os": "Ubuntu 22.04"
      }
    }]
  }'
```

- Always pass a **data source** (`sysparm_data_source`, name of a registered source in `sys_choice` for `discovery_source`) so reconciliation rules apply.
- Response includes `sysId`, `identificationAttempts`, and `incompleteSteps`/`errorCount`. Handle partial failures.
- Variants: `/identifyreconcile/{createOrUpdateCI}`, `/identifyreconcile/identifyCI` (dry-run style lookup), `/api/now/cmdb/instance/<class>` (CMDB Instance API, which also uses IRE under the hood).

In server scripts:

```javascript
var payload = {
    items: [{
        className: 'cmdb_ci_linux_server',
        values: { name: 'app01', ip_address: '10.0.1.15',
                  serial_number: 'i-0abc123', serial_number_type: 'provider' }
    }]
};
var out = sn_cmdb.IdentificationEngine.createOrUpdateCI('Ansible', JSON.stringify(payload));
gs.info(out);
```

(`sn_cmdb.IdentificationEngine` and `sn_cmdb.IdentificationEngineScriptable` are the server-side APIs; check your release docs for exact method signatures.)

### From Ansible

`servicenow.itsm.configuration_item` module uses the Table API/CMDB instance logic. For strict IRE behaviour in enterprise (data-source precedence), call the IRE endpoint with the `servicenow.itsm.api` module or `ansible.builtin.uri`:

```yaml
- name: Register EC2 as CI via IRE
  ansible.builtin.uri:
    url: "{{ sn_instance }}/api/now/identifyreconcile?sysparm_data_source=Ansible"
    method: POST
    headers:
      Authorization: "Bearer {{ sn_token }}"
    body_format: json
    body:
      items:
        - className: cmdb_ci_linux_server
          values:
            name: "{{ server_name }}"
            ip_address: "{{ ec2.instances[0].private_ip_address }}"
            serial_number: "{{ ec2.instances[0].instance_id }}"
            serial_number_type: provider
    status_code: [200, 201]
  no_log: true
```

### Interview answer (30 seconds)

“Without IRE, every integration inserts its own CI and you get duplicates. IRE uses identifier rules to decide whether an incoming record matches an existing CI (serial number, then name+class, then network adapter), and reconciliation rules with data source priorities to decide which source may update which attribute. So Ansible can create or enrich a CI, but cannot overwrite attributes owned by Discovery. We always write through the IRE API with a registered data source.”

---

## 7. GlideRecord scripting

### The four Glide classes you must know

| Class | Use | ACL enforced? |
|---|---|---|
| `GlideRecord` | CRUD on any table (server-side) | **No** (runs with system rights unless you check) |
| `GlideRecordSecure` | Same API, but enforces ACLs (`canRead/canWrite`) | **Yes** |
| `GlideAggregate` | `COUNT/SUM/AVG/MIN/MAX` and `GROUP BY` in SQL (fast) | No |
| `GlideAjax` | Client script → calls a Script Include (client-callable) | Depends on script include |

Other useful: `GlideDateTime`, `GlideDuration`, `GlideSystem` (`gs`), `GlideElement`, `GlideFilter`, `GlideEncrypter`.

### GlideRecord patterns

```javascript
// READ many
var gr = new GlideRecord('incident');
gr.addQuery('active', true);
gr.addQuery('priority', '<=', 2);
gr.addEncodedQuery('assignment_group.name=Network^ORDERBYDESCsys_created_on');
gr.setLimit(100);
gr.query();
while (gr.next()) {
    gs.info(gr.getValue('number') + ' ' + gr.getDisplayValue('assigned_to'));
}

// READ one by sys_id
var inc = new GlideRecord('incident');
if (inc.get('SYS_ID_HERE')) {
    gs.info(inc.short_description);
}

// READ one by field
if (inc.get('number', 'INC0010001')) { /* ... */ }

// CREATE
var n = new GlideRecord('incident');
n.initialize();
n.setValue('short_description', 'Created by script');
n.setValue('impact', 2);
n.setValue('urgency', 2);
var newId = n.insert();                 // returns sys_id or null

// UPDATE many (setWorkflow(false) skips business rules; use with care)
var u = new GlideRecord('incident');
u.addQuery('state', 1);
u.query();
while (u.next()) {
    u.setValue('work_notes', 'Bulk updated');
    u.update();
}

// UPDATE many efficiently: updateMultiple (no per-record scripts/BRs)
var m = new GlideRecord('incident');
m.addQuery('state', 7);
m.setValue('active', false);
m.updateMultiple();

// DELETE (careful)
var d = new GlideRecord('u_scratch');
d.addQuery('sys_created_on', '<', gs.daysAgoStart(30));
d.deleteMultiple();
```

### GlideRecord rules that avoid real incidents

- Use **`getValue()` / `setValue()`** and `getDisplayValue()` instead of dot-walking for robust strings. `gr.field` returns a GlideElement object, not a string; compare with `.toString()`.
- **Limit queries** (`setLimit`) and always add conditions; an unfiltered `query()` on `task` can pull millions of rows.
- **Never query inside a loop** if a single query with `IN` or a join works.
- **Check `gr.next()` / `gr.hasNext()`** and `gr.isValidRecord()`.
- Use **`GlideAggregate`** for counts instead of looping:

```javascript
var ga = new GlideAggregate('incident');
ga.addQuery('active', true);
ga.addAggregate('COUNT', 'priority');
ga.groupBy('priority');
ga.query();
while (ga.next()) {
    gs.info('P' + ga.getValue('priority') + ': ' + ga.getAggregate('COUNT', 'priority'));
}
```

- **`GlideRecordSecure`** when you run on behalf of a user and ACLs must apply:

```javascript
var gr = new GlideRecordSecure('incident');
gr.query();                    // returns only rows the current user can read
while (gr.next()) { /* ... */ }
```

- In **business rules**: `current` / `previous` are the record objects; avoid `current.update()` in a `before` rule (infinite loop risk); use `current.setAbortAction(true)` to block.
- **`setWorkflow(false)`** and **`autoSysFields(false)`** suppress business rules / system fields, which bypass audit and logic. Use only in controlled migrations.
- **Encoded queries**: build them in the list view filter → right-click → *Copy query* → `addEncodedQuery(...)`.
- **Date handling**:

```javascript
var gdt = new GlideDateTime();
gdt.addDaysUTC(-7);                       // 7 days ago, UTC
var gr = new GlideRecord('incident');
gr.addQuery('sys_created_on', '>=', gdt);
```

### Business rule example

```javascript
// Table: incident | When: before | Insert/Update | Condition: current.priority.changes() && current.priority == 1
(function executeRule(current, previous /*null when async*/) {
    if (current.assignment_group.nil()) {
        current.assignment_group = gs.getProperty('x_myapp.p1_default_group');
        gs.addInfoMessage('P1 incident auto-assigned to the default group.');
    }
})(current, previous);
```

---

## 8. Script Includes

Reusable server-side classes. Create in System Definition > Script Includes. **Name must equal the class name.**

### Standard class pattern

```javascript
// Name: ServerRequestUtil | Accessible from: All application scopes (or This application scope only)
var ServerRequestUtil = Class.create();
ServerRequestUtil.prototype = {
    initialize: function (ritmSysId) {
        this.ritm = new GlideRecord('sc_req_item');
        this.valid = this.ritm.get(ritmSysId);
    },

    isValid: function () {
        return this.valid;
    },

    getPayload: function () {
        return {
            extra_vars: {
                server_name: this.ritm.variables.server_name.toString(),
                env: this.ritm.variables.env.toString(),
                requested_by: this.ritm.variables.requested_by.getDisplayValue(),
                ritm: this.ritm.getValue('number')
            }
        };
    },

    launchAnsible: function (templateId) {
        if (!this.valid) {
            return { ok: false, error: 'RITM not found' };
        }
        try {
            var r = new sn_ws.RESTMessageV2('AAP Controller', 'launch_job');
            r.setStringParameterNoEscape('template_id', templateId);
            r.setRequestBody(JSON.stringify(this.getPayload()));
            r.setHttpTimeout(30000);
            var resp = r.execute();
            if (resp.getStatusCode() !== 201) {
                gs.error('[ServerRequestUtil] AAP launch ' + resp.getStatusCode() + ': ' + resp.getBody());
                return { ok: false, error: 'AAP returned ' + resp.getStatusCode() };
            }
            return { ok: true, jobId: JSON.parse(resp.getBody()).job };
        } catch (e) {
            gs.error('[ServerRequestUtil] ' + e.message);
            return { ok: false, error: e.message };
        }
    },

    type: 'ServerRequestUtil'
};
```

Calling it:

```javascript
var util = new ServerRequestUtil(current.getUniqueValue());
var result = util.launchAnsible('42');
if (!result.ok) { current.work_notes = 'AAP launch failed: ' + result.error; }
```

### Client-callable Script Include (GlideAjax)

```javascript
// Script Include: ServerCheckAjax | Client callable: true
var ServerCheckAjax = Class.create();
ServerCheckAjax.prototype = Object.extendsObject(global.AbstractAjaxProcessor, {
    nameExists: function () {
        var name = this.getParameter('sysparm_server');
        var gr = new GlideRecord('cmdb_ci_server');
        gr.addQuery('name', name);
        gr.setLimit(1);
        gr.query();
        return gr.hasNext() ? 'true' : 'false';
    },
    type: 'ServerCheckAjax'
});
```

```javascript
// Catalog Client Script (onChange of server_name)
function onChange(control, oldValue, newValue, isLoading) {
    if (isLoading || newValue === '') return;
    var ga = new GlideAjax('ServerCheckAjax');
    ga.addParam('sysparm_name', 'nameExists');   // method to call (reserved parameter)
    ga.addParam('sysparm_server', newValue);     // your data
    ga.getXMLAnswer(function (answer) {
        if (answer === 'true') {
            g_form.showFieldMsg('server_name', 'A server with this name already exists', 'error');
        }
    });
}
```

> Careful: `sysparm_name` is the **method name** in GlideAjax. Use another parameter name (e.g. `sysparm_server`) for your data:

```javascript
ga.addParam('sysparm_name', 'nameExists');       // method
ga.addParam('sysparm_server', newValue);         // data → this.getParameter('sysparm_server')
```

### Good practice

- Keep business logic in Script Includes; keep Business Rules, Flow script steps and UI actions thin (just call the include).
- Namespace in scoped apps (`x_myapp.ServerRequestUtil`), set **Accessible from** correctly.
- Return structured results and log errors with a prefix; never swallow exceptions.
- Unit-test with **ATF (Automated Test Framework)**: server script steps.
- Avoid `GlideRecord` in client scripts; use GlideAjax or `g_form.getReference` sparingly.

---

## 9. IntegrationHub

### What it is

IntegrationHub is the **Flow Designer extension** for integrations: pre-built **spokes** (action packages for Slack, AWS, Azure, Jira, Microsoft Teams, Ansible Tower/AAP, ServiceNow-to-ServiceNow, Active Directory, etc.), the **REST/SOAP/JDBC/PowerShell/SSH steps** for custom actions, **Connection & Credential Aliases**, and **ETL** capabilities. Licensing is tiered (Starter included with some SKUs; Standard/Professional/Enterprise add capabilities and transaction packs). Always confirm entitlements before designing around a spoke.

### Spoke vs raw REST message

| Use a spoke / IntegrationHub action when… | Use a raw REST Message/Scripted call when… |
|---|---|
| Spoke exists and covers the operation (e.g. *Ansible Tower spoke > Launch Job Template*) | No spoke exists, or you need an endpoint the spoke does not cover |
| You want no-code, reusable, testable, versioned actions in flows | You need highly custom logic or are outside Flow Designer |
| Credentials via aliases are managed centrally | Licence does not include IntegrationHub |
| Others in the team build flows without scripting | Fine-grained control over retries, headers, streaming |

### Custom action with a REST step

In Flow Designer > Actions > New:
- **Inputs**: `template_id` (String), `extra_vars` (String/JSON).
- **Step 1: REST**: Connection = Alias *AAP Controller*, Base URL from alias, Resource path `/api/controller/v2/job_templates/{{template_id}}/launch/`, Method POST, Body `{"extra_vars": {{extra_vars}}}`, Headers `Content-Type: application/json`.
- **Step 2: Script** (parse response, set outputs):

```javascript
(function execute(inputs, outputs) {
    var body = JSON.parse(inputs.response_body);
    outputs.job_id = String(body.job || '');
    outputs.status = inputs.status_code;
})(inputs, outputs);
```
- **Outputs**: `job_id`, `status_code`, `error_message`.
- **Error handling** within the action: *Error Evaluation* step to mark failure if `status_code` is not 201.

### Connection & Credential Alias

Create an alias (e.g. `AAP Controller`), a **Connection** record (URL, timeout, MID Server option) and a **Credential** (OAuth/basic/API key). Switch credential per environment without touching flows. This is the standard “no secrets in flows” answer.

### Polling vs callback for long jobs

- **Polling**: Do Until loop with a REST step `GET /api/controller/v2/jobs/{id}/` + `Wait` of 30 s until `status` ∈ {successful, failed, canceled}. Simple, but consumes flow time/transactions.
- **Callback**: AAP workflow’s last node (or the playbook) PATCHes the RITM via Table API / Scripted REST; the flow uses **Wait For Condition**/**Wait for Event** on that record update. Efficient; needs inbound access to ServiceNow (it already is a cloud endpoint, so this is usually easy).

---

## 10. MID Server (read-only understanding)

A **MID (Management, Instrumentation, and Integration) Server** is a Java service you install **inside your network**. It opens an **outbound HTTPS** connection to the instance and polls an **ECC queue** (`ecc_queue`) for work; results go back through the same queue. Therefore no inbound firewall hole is needed.

Used for: **Discovery**, **Service Mapping**, **Orchestration/IntegrationHub steps against private systems** (PowerShell, SSH, JDBC, REST to on-prem), **LDAP**, **Event Management connectors**.

Key points:
- Redundancy: **MID Server clusters** (failover/load-balanced) and capabilities/IP ranges to route work.
- Security: dedicated service account, cert validation, signed JAR verification, minimal OS privileges.
- Monitor: *MID Server > Servers* (status Up/Down, version), ECC queue backlog, MID logs.
- In REST Messages/Flow steps: select **Use MID Server** (specific, or auto-select by capability).
- Pattern: **Cloud instance → ECC queue → MID Server → on-prem AAP/Jenkins/DB**.

Interview one-liner: “MID Server is the secure bridge from ServiceNow cloud to private networks. It makes only outbound connections and executes jobs from the ECC queue, so we do not expose internal AAP to the internet.”

---

## 11. Connecting ServiceNow and Ansible (your core scenario)

### Direction A: ServiceNow → Ansible (request fulfilment)

```
Catalog item  →  Flow (approval)  →  REST/IntegrationHub step
   → AAP /api/controller/v2/job_templates/<id>/launch/  (extra_vars from RITM variables)
   → AAP job (provision + configure)
   → callback / poll → RITM updated (work notes, CMDB CI, close)
```

Private AAP? Use a **MID Server**, or expose through a reverse proxy/ngrok for labs.

### Direction B: Ansible → ServiceNow (record keeping)

Using `servicenow.itsm`:

```yaml
- name: Open change before work
  servicenow.itsm.change_request:
    instance: { host: "{{ sn_host }}", client_id: "{{ sn_cid }}", client_secret: "{{ sn_cs }}",
                username: "{{ sn_user }}", password: "{{ sn_pw }}" }
    type: standard
    state: new
    short_description: "Provision {{ server_name }}"
  register: cr

- name: Close change after success
  servicenow.itsm.change_request:
    number: "{{ cr.record.number }}"
    state: closed
    close_code: successful
    close_notes: "Provisioned by AAP job {{ tower_job_id | default('local') }}"

- name: Append execution log to work notes (GxP evidence)
  servicenow.itsm.api:
    resource: change_request
    action: patch
    sys_id: "{{ cr.record.sys_id }}"
    data:
      work_notes: "{{ lookup('ansible.builtin.file', 'run.log') | truncate(30000) }}"
```

### Direction C: Event-Driven (self-heal)

Monitoring alert → EDA rulebook → Ansible creates **P1 incident** → remediation playbook → **update/close incident** with resolution notes.

---

## 12. Performance, security and governance checklist

- ACLs: integration user has least privilege; test with **Impersonate** and **ACL debugging**.
- Avoid hardcoded sys_ids across instances. Use system properties or lookups by name/number.
- Move config with **update sets / scoped app + Git**, not by recreating by hand. Know *collision* and *skipped* update handling during commit.
- Rate limits: **REST API Rate Limit Rules** protect the instance; handle HTTP 429 with backoff in clients.
- Pagination for big reads (`sysparm_limit` + `sysparm_offset`, or the `Link` header).
- Use **async business rules** or flows for slow work (external calls) so users do not wait.
- Audit: journal fields (`work_notes`, `comments`), `sys_audit`, `syslog_transaction`, flow context logs, `sys_outbound_http_log`.
- **GxP/compliance**: every automated change must link to a change/RITM, include who/what/when, and be reproducible. Store evidence in work notes or attachments (Attachment API).
- Test: **ATF** for catalog item submission, flow execution and script includes before promoting.

---

## 13. Interview quick-fire

1. **GlideRecord vs GlideRecordSecure vs GlideAggregate?** GlideRecord is CRUD without ACL enforcement; GlideRecordSecure is the same but honours ACLs; GlideAggregate runs SQL aggregates (count/sum/group by) without loading rows.
2. **How do you stop duplicate CIs?** Write through the IRE: identifier rules match existing CIs, reconciliation rules with data-source priorities control which source updates which attribute.
3. **How do you secure an inbound integration?** Dedicated integration user with minimal role, OAuth with short-lived tokens, ACLs on the table/Scripted REST, IP restrictions/rate limits, no basic auth in prod.
4. **Flow vs Workflow vs Business Rule?** Flow Designer is the current low-code orchestration for multi-step/approval/integration; Workflow is legacy; Business Rules are record-level server logic that run on DB operations (keep them small and call Script Includes).
5. **How do you call an on-prem system from ServiceNow?** MID Server (outbound-only connection, ECC queue), through a REST Message/IntegrationHub step with the MID selected.
6. **Spoke vs REST message?** Use the spoke where it exists (reusable, alias-managed credentials, supported); use raw REST for gaps or when IntegrationHub is not licensed.
7. **How do you handle long-running external jobs?** Callback to ServiceNow on completion, or poll with Do Until and wait; always set a timeout and an error path that opens an incident.
8. **How do you move changes between instances?** Update sets for simple cases, scoped apps with Source Control (Git) for managed releases, with ATF before promotion.
9. **How do you audit automation (GxP)?** Link every run to a change/RITM, write execution logs to work notes/attachments, keep AAP job history and Git PR approvals, review `sys_audit`.
10. **What is the ECC queue?** The table (`ecc_queue`) used to exchange messages between the instance and MID Servers (outbound input/output records).

---

## 14. Revision checklist (do on your PDI)

- [ ] Build catalog item "Request Enterprise App Server" with variables, a catalog client script and a UI policy
- [ ] Flow Designer: approval → REST step → update RITM; add an error path that creates an incident
- [ ] Build a custom action with a REST step and typed outputs
- [ ] Register an OAuth endpoint in Application Registry; fetch a token with curl; call the Table API with it
- [ ] POST then PATCH an incident through REST API Explorer; copy the generated Python snippet
- [ ] Python script that creates an incident with an OAuth bearer token (project task P4)
- [ ] Write a Script Include class (`ServerRequestUtil`) and call it from a business rule and a flow script step
- [ ] Use `GlideRecord`, `GlideRecordSecure` and `GlideAggregate` on the same table and compare results
- [ ] Create a Scripted REST API with proper status codes and role requirement
- [ ] Post a CI through the IRE API twice with the same serial number and confirm it updates instead of duplicating
- [ ] Read the identifier rules for `cmdb_ci_server` and the reconciliation rules for one attribute
- [ ] Explain MID Server out loud (outbound-only, ECC queue) and sketch the cloud-to-on-prem path
- [ ] Wire a RITM to launch an AAP job and receive a callback (ngrok for labs)
