# Event-Driven Ansible (EDA) Revision Guide

Scope: concepts (rulebook, source, condition, action), installation, writing rulebooks, event sources (webhook, alertmanager, kafka, file, range), conditions and the event data model, actions (`run_playbook`, `run_job_template`, `run_workflow_template`, `post_event`, `set_fact`, `debug`, `shell`), simulating alerts with curl, the self-heal chain with ServiceNow incidents, decision environments, EDA in AAP (activations), reliability/security, troubleshooting.

How to use: do sections 2-6 on your laptop with `ansible-rulebook` (no AAP needed). Build the self-heal chain in section 8 as the capstone of your portfolio project.

> Version note: `ansible-rulebook` and the `ansible.eda` collection evolve quickly (new sources, action names, throttle options). Run `ansible-rulebook --version` and `ansible-doc -t ... ` or the docs for your installed version before relying on a detail here.

---

## 0. Mental model

Traditional Ansible is **pull/trigger-based**: a human, schedule or pipeline starts a playbook. EDA is **event-based**: it listens to event streams and decides what to run.

```
Event source  →  Rulebook (rules: condition → action)  →  Action (playbook / job template / event)
(monitoring,        "if this happened,                     "do this"
  webhook, Kafka)     and it matches..."
```

| Term | Meaning |
|---|---|
| **Rulebook** | YAML file with one or more *rulesets*; the EDA equivalent of a playbook |
| **Ruleset** | A named group of sources + rules (`- name: ... hosts: all sources: [...] rules: [...]`) |
| **Source** | A plugin that produces events (webhook, alertmanager, kafka, url_check, file...) |
| **Event** | A JSON-like dictionary. Source payloads appear under `event.*` (e.g. `event.payload`) plus `event.meta` |
| **Rule** | `name` + `condition` + `action(s)` |
| **Condition** | Expression evaluated against events; supports comparison, boolean logic, `in`, regex, `is defined`, multiple events, `all`/`any`, time windows |
| **Action** | What to do when the condition matches |
| **Facts** | Persisted knowledge in the rule engine (`set_fact`) used for stateful rules |
| **Decision Environment (DE)** | Container image that runs the rule engine, with `ansible-rulebook`, `ansible.eda` plugins, JDK (Drools), and Python deps |
| **Rulebook Activation** | In AAP, a running instance of a rulebook (a managed, restartable process) |
| **Event stream** (AAP 2.5) | Platform-managed endpoint that receives events (e.g. webhooks) and routes them to activations |

Engine facts worth knowing: the rule engine is based on **Drools** (Java), so the DE image includes a JDK; rules are **evaluated continuously**; events are matched in memory (no persistence by default).

Use cases: auto-remediation (service down → restart), ticket enrichment, scaling responses, security responses (isolate a host on an alert), ITSM-linked operations (incident → fix → close), compliance drift reactions.

---

## 1. Installation (local lab)

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install ansible-core ansible-rulebook ansible-runner
ansible-galaxy collection install ansible.eda servicenow.itsm
# JDK is required by the rules engine (Drools); example on macOS:
brew install openjdk@17
export JAVA_HOME="$(/usr/libexec/java_home -v 17)"
export LD_LIBRARY_PATH="$JAVA_HOME/lib/server"      # Linux only if the JVM lib is not found
ansible-rulebook --version
ansible-doc -t eda_source ansible.eda.webhook 2>/dev/null || ansible-galaxy collection list | grep eda
```

Alternatively use `ansible-navigator`/containers with a decision environment (section 9), which avoids local Java setup.

Project layout:

```
eda-project/
├── rulebooks/
│   ├── webhook_down.yml
│   └── selfheal.yml
├── playbooks/
│   ├── restart_service.yml
│   └── close_incident.yml
├── inventories/
│   └── hosts.yml
├── vars/
│   └── eda_vars.yml
├── decision-environment/
│   ├── decision-environment.yml
│   └── requirements.yml
└── README.md
```

---

## 2. Your first rulebook

```yaml
# rulebooks/webhook_down.yml
---
- name: Listen for monitoring alerts
  hosts: all
  sources:
    - ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000
  rules:
    - name: Host reported down
      condition: event.payload.status == "down"
      action:
        debug:
          msg: "Host {{ event.payload.host }} is down"
```

Run it:

```bash
ansible-rulebook --rulebook rulebooks/webhook_down.yml -i inventories/hosts.yml --verbose
# --debug for more, --print-events to dump every event, --vars/--env-vars for variables
```

Structure rules:
- A rulebook is a **list of rulesets**; each ruleset has `name`, `hosts`, `sources`, `rules` (and optional `gather_facts`, `default_events_ttl`).
- `hosts: all` is conventional; the inventory passed at runtime is used by `run_playbook`.
- Indentation matters: `source` plugin name is the key (`ansible.eda.webhook`), its parameters nest below.

---

## 3. Sources

### Built-in/common sources (`ansible.eda` collection)

| Source | Use | Key parameters |
|---|---|---|
| `ansible.eda.webhook` | HTTP POST receiver; best for lab, ServiceNow, GitHub, generic tools | `host`, `port`, `token`, `certfile`/`keyfile`, `hmac_secret`, `hmac_algo`, `hmac_header` |
| `ansible.eda.alertmanager` | Prometheus Alertmanager webhook receiver | `host`, `port`, `data_alerts_path`, `data_host_path` |
| `ansible.eda.kafka` | Consume a Kafka topic | `host`, `port`, `topic`, `group_id`, `offset` |
| `ansible.eda.url_check` | Poll URLs and emit status events | `urls`, `delay` |
| `ansible.eda.file` / `file_watch` | React to file contents/changes | `files` / `path`, `recursive` |
| `ansible.eda.range` | Emit N numbered events (testing) | `limit` |
| `ansible.eda.generic` | Emit events from a payload in the rulebook (testing) | `payload`, `loop_count`, `time_between_events` |
| `ansible.eda.tick` | Emit events on an interval (heartbeat/polling) | `delay` |
| `ansible.eda.aws_sqs_queue`, `aws_cloudtrail` | AWS events | `queue_name`, region, creds |
| `ansible.eda.azure_service_bus` | Azure events | connection string, queue |
| `ansible.eda.pg_listener` | PostgreSQL LISTEN/NOTIFY | DSN, channels |

Vendor collections provide more (Dynatrace, Zabbix, Cisco, Splunk, ServiceNow-related, etc.). List what you have:

```bash
ansible-galaxy collection list | grep -i eda
ls $(python -c "import ansible_collections.ansible.eda as m, os; print(os.path.dirname(m.__file__))")/extensions/eda/plugins/event_source
```

### Securing a webhook source

```yaml
sources:
  - ansible.eda.webhook:
      host: 0.0.0.0
      port: 5000
      token: "{{ webhook_token }}"            # clients must send header: Authorization: Bearer <token>
      # or HMAC signature check:
      # hmac_secret: "{{ webhook_hmac }}"
      # hmac_algo: sha256
      # hmac_header: X-Hub-Signature-256
      # certfile / keyfile for TLS
```

Pass variables safely at runtime: `ansible-rulebook ... --vars vars/eda_vars.yml` (vault-protected) or `--env-vars WEBHOOK_TOKEN` and reference as `{{ WEBHOOK_TOKEN }}`... check your version's syntax for env/vars usage.

### Writing a custom source plugin (when no plugin exists)

```python
# extensions/eda/plugins/event_source/snow_poll.py
import asyncio
from typing import Any

DOCUMENTATION = r"""
short_description: Poll ServiceNow incidents
options:
  delay: {description: Seconds between polls, type: int, default: 30}
"""


async def main(queue: asyncio.Queue, args: dict[str, Any]) -> None:
    delay = args.get("delay", 30)
    while True:
        # call the Table API here (use aiohttp or run requests in a thread)
        await queue.put({"incident": {"number": "INC0010001", "priority": "1"}})
        await asyncio.sleep(delay)
```

The entry point is `async def main(queue, args)` and events are `dict`s put on the queue.

---

## 4. The event data model and conditions

### Event structure

```json
{
  "payload": { "status": "down", "host": "web01", "severity": "critical" },
  "meta": {
    "endpoint": "endpoint",
    "headers": { "Host": "localhost:5000", "Content-Type": "application/json" },
    "received_at": "2025-01-01T10:00:00Z",
    "uuid": "..."
  }
}
```

You access it as `event.payload.status`, `event.meta.headers`. Different sources shape the payload differently (alertmanager has `event.alert.labels...`, `event.alert.annotations...`). **Always inspect with `--print-events` first.**

### Condition syntax

```yaml
# comparison
condition: event.payload.cpu > 90
condition: event.payload.status == "down"
condition: event.payload.status != "ok"

# boolean logic
condition: event.payload.status == "down" and event.payload.severity == "critical"
condition: event.payload.env == "prod" or event.payload.env == "test"
condition: not (event.payload.env == "dev")

# membership
condition: event.payload.host in ["web01", "web02"]
condition: event.payload.service in vars.monitored_services     # check your version for vars usage in conditions
condition: "'timeout' in event.payload.message"                  # substring (string contains)

# regex
condition: event.payload.host is match("web[0-9]+", ignorecase=true)
condition: event.payload.message is search("disk .* full")
condition: event.payload.host is regex("^web")

# existence and null
condition: event.payload.host is defined
condition: event.payload.host is not defined
condition: event.payload.owner is none

# selectattr/select on lists of events or payload arrays
condition: event.payload.tags is select("==", "critical")
```

### Multiple events and correlation

```yaml
# ALL: both conditions must be satisfied by (possibly different) events, with matching values
condition:
  all:
    - event.cpu.host == "web01"           # first event
    - event.net.host == "web01"           # second event

# join facts across events using the same value
condition:
  all:
    - event.alert.host == event.metric.host

# ANY: either (with `events` referencing the matched one)
condition:
  any:
    - event.payload.status == "down"
    - event.payload.error_rate > 50

# NOT: absence of an event within a window (e.g. no heartbeat)
condition:
  not_all:
    - event.heartbeat.host == "web01"
  timeout: 5 minutes
```

(Exact keywords such as `not_all`/`timeout`, plus `events.first`, `events.last` and sliding windows, vary by version. Check `ansible-rulebook` docs → "Conditions" and the `ansible.eda` examples.)

### Time and counting: `throttle`

```yaml
rules:
  - name: Restart web (throttled)
    condition: event.payload.status == "down"
    throttle:
      group_by_attributes:
        - event.payload.host
      once_within: 15 minutes              # at most one action per host per 15 min
    action:
      run_playbook:
        name: playbooks/restart_service.yml
```

`throttle` prevents **remediation storms** (a flapping alert triggering 50 restarts). It is one of the most important production features. Also available: `once_after`, `accumulate_within` (version dependent).

Rule/ruleset options: `default_events_ttl` (how long unmatched events stay available for correlation), `match_multiple_events`.

---

## 5. Actions

| Action | Purpose |
|---|---|
| `run_playbook` | Run a local playbook through `ansible-runner` (rulebook CLI or activation with project) |
| `run_module` | Run a single module |
| `run_job_template` | Launch an **AAP/Controller job template** |
| `run_workflow_template` | Launch an **AAP workflow template** |
| `set_fact` | Store a persistent fact; can enable stateful rules |
| `post_event` | Post a new event into the ruleset (chaining rules) |
| `retract_fact` | Remove a fact |
| `print_event`, `debug` | Show event/message (development) |
| `shell` | Run a shell command (use sparingly; prefer playbooks) |
| `none` | Do nothing (often used for tests) |

### run_playbook

```yaml
action:
  run_playbook:
    name: playbooks/restart_service.yml
    extra_vars:
      target_host: "{{ event.payload.host }}"
      service_name: "{{ event.payload.service | default('nginx') }}"
    verbosity: 1
    post_events: true            # playbook's `set_stats` data come back as new events
    # ignore_playbook_failure: true
```

### run_job_template (AAP)

```yaml
action:
  run_job_template:
    name: "Restart Service"
    organization: "Default"
    job_args:
      extra_vars:
        target_host: "{{ event.payload.host }}"
        incident_number: "{{ event.payload.incident | default('') }}"
      limit: "{{ event.payload.host }}"
```

Run with controller connection details:

```bash
ansible-rulebook --rulebook rulebooks/selfheal.yml -i inventories/hosts.yml \
  --controller-url https://aap.example.com --controller-token "$AAP_TOKEN" \
  --controller-ssl-verify /path/ca.pem
```

(Inside AAP, the activation provides this connection automatically via a credential.)

### Several actions per rule, ordered

```yaml
action:
  run_playbook:
    name: playbooks/restart_service.yml
# for multiple actions use `actions:` (list) instead of `action:`
actions:
  - debug:
      msg: "Remediating {{ event.payload.host }}"
  - run_playbook:
      name: playbooks/restart_service.yml
  - post_event:
      event:
        remediation: { host: "{{ event.payload.host }}", status: attempted }
```

### Chaining rules with post_event and facts

```yaml
rules:
  - name: Detect failure
    condition: event.payload.status == "down"
    action:
      post_event:
        event:
          incident_candidate: { host: "{{ event.payload.host }}" }

  - name: Handle candidate
    condition: event.incident_candidate is defined
    action:
      run_playbook:
        name: playbooks/open_incident.yml
```

---

## 6. Simulating alerts (webhook)

Terminal 1: run the rulebook.

```bash
ansible-rulebook --rulebook rulebooks/webhook_down.yml -i inventories/hosts.yml --verbose --print-events
```

Terminal 2: send events.

```bash
# matches the rule
curl -X POST http://localhost:5000/endpoint \
  -H "Content-Type: application/json" \
  -d '{"status":"down","host":"web01"}'

# does not match
curl -X POST http://localhost:5000/endpoint -H "Content-Type: application/json" \
  -d '{"status":"up","host":"web01"}'

# with token auth
curl -X POST http://localhost:5000/endpoint \
  -H "Authorization: Bearer $WEBHOOK_TOKEN" -H "Content-Type: application/json" \
  -d '{"status":"down","host":"web01","severity":"critical"}'

# flapping test: fire 10 events quickly and watch throttle in action
for i in $(seq 1 10); do
  curl -s -X POST http://localhost:5000/endpoint -H "Content-Type: application/json" \
    -d '{"status":"down","host":"web01"}' >/dev/null
done
```

Notes:
- Default path is any path (`/endpoint`, `/`, ...); the path appears in `event.meta.endpoint`.
- Use `ansible.eda.generic`/`range` sources for **unit-testing rules** without HTTP.
- Test rulebook logic in CI with `generic` source + `--debug`, or assert on logs. Keep rulebook conditions simple and push logic into playbooks, where Molecule and lint can test it.

```yaml
# rulebooks/test_rules.yml: deterministic test events, no network
- name: Rule tests
  hosts: all
  sources:
    - ansible.eda.generic:
        payload:
          - { status: down, host: web01, severity: critical }
          - { status: up, host: web02 }
        loop_count: 1
  rules:
    - name: Only critical down
      condition: event.status == "down" and event.severity == "critical"
      action:
        debug:
          msg: "MATCHED {{ event.host }}"
```

(Generic-source events appear directly under `event.*`, not `event.payload.*`. This difference is a classic mistake. Print events to see the real shape.)

---

## 7. Remediation playbooks (what the rulebook calls)

```yaml
# playbooks/restart_service.yml
- name: Remediate a failed service
  hosts: "{{ target_host }}"
  become: true
  gather_facts: false
  vars:
    service_name: nginx
    max_attempts: 3
  tasks:
    - name: Validate input
      ansible.builtin.assert:
        that:
          - target_host is defined
          - service_name is match('^[a-zA-Z0-9_.@-]+$')       # prevent injection from event data

    - name: Attempt restart with verification
      block:
        - name: Restart service
          ansible.builtin.service:
            name: "{{ service_name }}"
            state: restarted

        - name: Verify health endpoint
          ansible.builtin.uri:
            url: "http://localhost/health"
            status_code: 200
          register: health
          retries: 5
          delay: 5
          until: health.status == 200

        - name: Record result for downstream steps
          ansible.builtin.set_stats:
            data:
              remediation_status: resolved
              remediated_host: "{{ inventory_hostname }}"

      rescue:
        - name: Record failure for escalation
          ansible.builtin.set_stats:
            data:
              remediation_status: failed
              remediated_host: "{{ inventory_hostname }}"
        - name: Fail the play so callers see it
          ansible.builtin.fail:
            msg: "Remediation failed on {{ inventory_hostname }}"
```

Design rules for remediation:
- **Idempotent and safe to repeat** (events may be duplicated or re-delivered).
- **Validate event-derived inputs**: event data is untrusted. Never put it into `shell` commands unquoted; whitelist hosts/services, use `assert` and `quote` filter.
- **Verify the fix** (health check) before closing tickets.
- **Limit blast radius:** `limit` to the affected host, `throttle`, maximum attempts, and escalate to humans after N failures.
- Log what was done to the ticket for audit.

---

## 8. Self-heal chain with ServiceNow (capstone)

### Flow

```
Monitoring alert / curl
   ↓ (webhook)
EDA rulebook: condition status == "down"  (+ throttle per host)
   ↓ action: run_job_template / run_workflow_template  (or run_playbook locally)
Workflow:
   1. Open P1 incident in ServiceNow (servicenow.itsm.incident)         ← audit start
   2. Run remediation playbook (restart / re-provision)
   3. Verify health
   4. success → update incident work notes + resolve/close
      failure → keep incident open, set priority/assign to on-call, add notes (rescue)
```

### Rulebook

```yaml
# rulebooks/selfheal.yml
---
- name: Self-healing for web tier
  hosts: all
  sources:
    - ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000
        token: "{{ webhook_token }}"
  rules:
    - name: Web host down => incident + remediation
      condition: >
        event.payload.status == "down" and
        event.payload.host is match("^web[0-9]+$")
      throttle:
        group_by_attributes:
          - event.payload.host
        once_within: 15 minutes
      action:
        run_playbook:
          name: playbooks/selfheal_chain.yml
          extra_vars:
            target_host: "{{ event.payload.host }}"
            alert_source: "{{ event.meta.endpoint | default('webhook') }}"
            alert_detail: "{{ event.payload | to_json }}"
          verbosity: 1

    - name: Recovered alert => log only
      condition: event.payload.status == "up"
      action:
        debug:
          msg: "{{ event.payload.host }} recovered"
```

### Orchestration playbook (local run)

```yaml
# playbooks/selfheal_chain.yml
- name: Self-heal chain
  hosts: localhost
  gather_facts: false
  vars:
    sn_instance:
      host: "{{ lookup('ansible.builtin.env', 'SN_HOST') }}"
      username: "{{ lookup('ansible.builtin.env', 'SN_USERNAME') }}"
      password: "{{ lookup('ansible.builtin.env', 'SN_PASSWORD') }}"
  tasks:
    - name: Open P1 incident
      servicenow.itsm.incident:
        instance: "{{ sn_instance }}"
        state: new
        short_description: "[AUTO] {{ target_host }} is down"
        description: |
          Detected by Event-Driven Ansible.
          Source: {{ alert_source }}
          Payload: {{ alert_detail }}
        impact: high
        urgency: high
        other:
          cmdb_ci: "{{ target_host }}"          # field depends on your instance; validate CI mapping
      register: inc
      no_log: true

    - name: Remediate and verify
      block:
        - name: Run remediation
          ansible.builtin.include_tasks: remediate.yml
          vars:
            service_name: nginx

        - name: Resolve incident with evidence
          servicenow.itsm.incident:
            instance: "{{ sn_instance }}"
            number: "{{ inc.record.number }}"
            state: resolved
            close_code: "Solution provided"
            close_notes: "Auto-remediated by EDA. Service restarted and health check passed."
            other:
              work_notes: "Playbook run completed at {{ now(utc=true).isoformat() }}"
          no_log: true

      rescue:
        - name: Escalate: keep incident open, add failure details
          servicenow.itsm.incident:
            instance: "{{ sn_instance }}"
            number: "{{ inc.record.number }}"
            state: in_progress
            other:
              work_notes: "Auto-remediation FAILED: {{ ansible_failed_result.msg | default('see AAP job') }}. Human action required."
          no_log: true
```

(`remediate.yml` would delegate to the target host: e.g. `delegate_to: "{{ target_host }}"` tasks or `ansible.builtin.import_playbook` pattern from section 7. Field names for resolve states, `close_code`, and `other:` keys vary by your instance's configuration; validate on your PDI.)

### Same chain in AAP (preferred in enterprises)

- **Workflow template** "Self-Heal Web": nodes = `Open Incident` → `Remediate` (success) → `Close Incident`; failure paths → `Escalate Incident`.
- Rulebook action: `run_workflow_template` with `extra_vars` from the event.
- Credentials (ServiceNow, SSH) live in AAP, no secrets in the rulebook.
- Job history in AAP + the incident work notes = audit trail (GxP evidence).

```yaml
action:
  run_workflow_template:
    name: "Self-Heal Web"
    organization: "Default"
    job_args:
      extra_vars:
        target_host: "{{ event.payload.host }}"
        alert_detail: "{{ event.payload | to_json }}"
```

### Duplicate and loop prevention (interview-worthy)

- **Throttle** per host (above).
- **Deduplicate incidents:** before creating, query for an open incident for that CI (`servicenow.itsm.incident_info` with `query`), update it instead of creating a duplicate.
- **Loop guard:** if remediation triggers a new alert (e.g. restarting causes a "down" event), the throttle window absorbs it. Also add a circuit breaker: after N failures within a window, stop auto-remediation and page a human.
- **Maintenance windows:** have the rule check a fact/variable or call a ServiceNow change-window check before acting.

```yaml
- name: Check for existing open incident
  servicenow.itsm.incident_info:
    instance: "{{ sn_instance }}"
    query:
      - active: = true
        short_description: LIKE {{ target_host }}
  register: existing
  no_log: true
```

---

## 9. Decision Environments (DE)

A **decision environment** is an OCI container image that provides the runtime for `ansible-rulebook`: the rules engine, Java (JDK), Python, `ansible.eda` collection plus any source plugin collections and Python libraries. It is to EDA what an **execution environment** is to playbooks. (An EE runs the *playbooks* the actions launch; the DE runs the *rule engine*.)

### Build one

```yaml
# decision-environment/decision-environment.yml
version: 3
images:
  base_image:
    name: quay.io/ansible/ansible-rulebook:main       # or Red Hat's registry image for AAP:
    # registry.redhat.io/ansible-automation-platform-25/de-minimal-rhel9:latest
dependencies:
  galaxy: requirements.yml
  python:
    - aiokafka
    - requests
  system: []
  ansible_core:
    package_pip: ansible-core
  ansible_runner:
    package_pip: ansible-runner
```

```yaml
# decision-environment/requirements.yml
collections:
  - name: ansible.eda
  - name: servicenow.itsm
```

```bash
ansible-builder build -f decision-environment/decision-environment.yml -t registry.example.com/eda/de-itsm:1.0.0
podman push registry.example.com/eda/de-itsm:1.0.0
```

Use a minimal base image from the vendor for AAP support; make sure the base contains the JDK and `ansible-rulebook` (the `ansible-builder` base for DE differs from the EE base).

### Run a rulebook inside a DE locally

```bash
podman run --rm -it -p 5000:5000 \
  -v "$PWD:/work:Z" -w /work \
  -e SN_HOST -e SN_USERNAME -e SN_PASSWORD \
  registry.example.com/eda/de-itsm:1.0.0 \
  ansible-rulebook --rulebook rulebooks/selfheal.yml -i inventories/hosts.yml --verbose
```

### In AAP (EDA controller)

1. **Decision Environments** → Add: name, image URL (e.g. your registry), credential for the registry.
2. **Projects** → Add: Git repo containing `rulebooks/` (and playbooks if `run_playbook`).
3. **Credentials**: Red Hat Ansible Automation Platform credential (so activations can call `run_job_template`), source credentials (token for webhooks, Kafka SASL), vault.
4. **Rulebook Activations** → Create: choose project, rulebook, **decision environment**, credentials, **restart policy** (`always` / `on-failure` / `never`), log level, optional variables.
5. Activation runs as a container (on the EDA controller's podman or on OpenShift). Observe **History/Rule Audit**: which events matched which rules and which actions ran. The audit view is valuable compliance evidence.
6. Platform 2.5: use **Event Streams** (platform-managed webhook endpoints with tokens/HMAC) so external systems POST to a stable URL and events are routed to activations; avoid exposing activation ports directly.

Key concepts to explain:
- **Activation = long-running process**; if it crashes the restart policy governs recovery.
- Multiple activations can run the same rulebook with different variables (per environment).
- DE and EE are versioned and scanned separately; pin tags.
- Scale: one activation per event stream/ruleset; keep rules cheap; use Kafka for high volume buffering.

---

## 10. Reliability, security and governance

**Reliability**
- Events are **at-most-once in memory**: if the activation restarts you may lose in-flight events; use a durable broker (Kafka/SQS) for critical sources.
- Make remediation playbooks **idempotent** and tolerant of duplicates.
- Use `throttle` and incident deduplication to avoid storms.
- Keep a **kill switch**: an activation can be disabled, or a fact/variable (`auto_remediation_enabled`) checked in the condition.
- Timeouts and retries on every job; escalate to a human after N failures.
- Monitor the monitor: heartbeat events (`tick` source or a monitoring heartbeat) with a "not seen in X minutes" rule that alerts if EDA or the source is silent.

**Security**
- Treat event payload as **untrusted input**: validate, whitelist targets, never interpolate into shell unquoted.
- Authenticate the webhook (token/HMAC/TLS), restrict network access (firewall/reverse proxy), do not expose a lab ngrok URL permanently.
- Least privilege: the credential used by auto-remediation can only run the approved templates and touch the intended inventory.
- Secrets via AAP credentials/vault, never in rulebooks or Git.
- Limit the actions an event can trigger to a **pre-approved catalogue** (a mapping of alert → template), not arbitrary playbooks derived from payload content.

**Governance (GxP/enterprise)**
- Every automated action leaves evidence: AAP job history, EDA rule audit, incident/change work notes with job link.
- Change-control for rulebooks: PRs, lint (`ansible-rulebook --check`? use `ansible-lint` on playbooks; validate rulebook YAML with `yamllint` and run test events in CI), peer review, versioned DE images.
- Distinguish **safe** auto-remediation (restart a service in non-prod/with approval) from **risky** (reboot, delete) that should open a ticket for approval via a workflow approval node.

---

## 11. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Rulebook starts but nothing matches | Wrong event path (`event.payload.x` vs `event.x`), run with `--print-events`; check JSON content-type; check source plugin payload shape |
| `Could not find a JDK` / JVM errors | Install JDK 17, set `JAVA_HOME`/`LD_LIBRARY_PATH`; or run via the DE container |
| Webhook returns 401/403 | Token/HMAC header mismatch |
| Webhook connection refused | Port not exposed (`-p 5000:5000`), `host: 0.0.0.0` vs `127.0.0.1`, firewall |
| `run_playbook` fails to find playbook | Path relative to the working dir/project; in activations, path relative to the project root |
| `run_job_template` fails | Missing `--controller-url/--controller-token`, wrong name/organization, no permission, template not set to accept `extra_vars` (needs *Prompt on launch*) or `limit` |
| Same event triggers repeated actions | Missing `throttle`; self-triggering loop; source re-sending |
| Rule never fires after the first time | Event consumed/TTL; use `match_multiple_events` or correct `default_events_ttl`; check fact retention |
| Variables not rendering in actions | Using `{{ }}` in a condition (not needed there) or wrong quoting; in actions, always quote values in YAML |
| Activation keeps restarting in AAP | Crash in source plugin (bad parameters), DE missing a Python dependency, credential error |
| Playbook runs but incident not created | ServiceNow auth/env vars missing in the DE or AAP credential; module requires correct `instance` dict; check `no_log` hid the error (temporarily enable output in a lab) |

Useful flags: `--verbose`, `--debug`, `--print-events`, `--env-vars`, `--vars`, `--id`, `--worker`, `--websocket-url` (AAP-managed runs).

---

## 12. Interview quick-fire

1. **What is EDA and when do you use it instead of a scheduled job or pipeline?** It reacts to events in near real time through rulebooks (source → condition → action); use it for remediation, enrichment, and triggers where polling/scheduling is too slow or wasteful.
2. **Explain rulebook anatomy.** Rulesets contain sources (event producers), rules (condition + action), and `hosts`; actions include `run_playbook`, `run_job_template`, `run_workflow_template`, `set_fact`, `post_event`, `debug`.
3. **Decision environment vs execution environment?** DE is the container runtime for the rule engine (`ansible-rulebook`, JDK, source plugins); EE is the runtime for the playbooks that actions launch. Both are built with `ansible-builder`.
4. **How do you prevent remediation storms?** `throttle` by host, incident deduplication, circuit breakers (stop after N failures), maintenance-window checks, idempotent playbooks.
5. **How would you secure the webhook?** Token or HMAC verification, TLS, network restrictions or AAP Event Streams, validate/whitelist payload fields, least-privilege credentials.
6. **Walk me through your self-heal demo.** Alert → webhook → rule matches `status == "down"` → workflow: open P1 incident → remediation playbook with health verification → resolve incident with evidence, or escalate on failure; throttle prevents duplicates.
7. **What if the rulebook activation crashes?** AAP restart policy brings it back; in-memory events may be lost, so critical sources should be durable (Kafka/SQS) and playbooks idempotent; monitor with a heartbeat rule.
8. **How do you test rulebooks?** `generic`/`range` sources with deterministic events, `--print-events` and `--debug`, curl against a webhook, CI jobs that run the rulebook against fixtures, and Molecule/lint for the playbooks they call.
9. **How do you keep humans in the loop?** Workflow approval nodes for risky actions, incidents/changes for traceability, escalation when remediation fails, kill switch and per-env activations.
10. **What are limits of EDA?** Not a monitoring system (needs sources), state/correlation windows are in-memory, event delivery guarantees depend on the source/broker, version differences across `ansible-rulebook`/AAP releases.

---

## 13. Revision checklist

- [ ] Install `ansible-rulebook`, JDK and `ansible.eda`; run a rulebook with `--print-events`
- [ ] First rulebook: webhook source + `event.payload.status == "down"` + `debug` action; test with curl (match and non-match)
- [ ] Write 5 conditions: `and/or`, `in`, regex `match`, `is defined`, numeric threshold
- [ ] Add `throttle` and prove it with a loop of 10 curl calls
- [ ] `run_playbook` with `extra_vars` from the event; remediation playbook with assert, health verification and `set_stats`
- [ ] Self-heal chain: open incident → remediate → resolve; failure path escalates (test both by breaking the service/health check)
- [ ] Incident dedupe using `incident_info` before creating a new incident
- [ ] Secure the webhook with a token; show a 401 without it
- [ ] Use `generic` source events to unit-test rule matching
- [ ] Build a decision environment with `ansible-builder`; run the rulebook inside it with podman
- [ ] (With AAP/lab) create DE, project, credentials and activation; inspect the rule audit
- [ ] Switch the action from `run_playbook` to `run_workflow_template`; keep secrets in AAP credentials
- [ ] Draw the architecture diagram (Mermaid in the README) and record the 3-minute demo: approve → provision → kill service → self-heal
