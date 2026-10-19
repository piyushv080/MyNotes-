# Ansible Revision Guide (Enterprise Automation Engineer)

Scope: roles, collections, inventory design, vault/credentials, error handling, idempotency, serial/forks, ansible-lint, Molecule, execution environments, AAP concepts. Custom modules/plugins at the end (optional).

How to use: read one section per sitting, type the examples out (don't paste), run them against a local container or a test EC2 box.

---

## 0. Mental model

- **Control node** runs `ansible-core`, connects to **managed nodes** (SSH/WinRM/API), runs **modules**, and removes them.
- **Declarative, idempotent**: describe the desired state, run it 100 times, get the same result. A second run should show `changed=0`.
- **Everything is data**: inventory (what), variables (config), playbooks (orchestration), roles (reusable units), collections (distribution packaging).
- Enterprise lens: auditability, least privilege, no plaintext secrets, repeatable runs, safe rollout, clear failure handling.

---

## 1. Project layout and `ansible.cfg`

```
repo/
├── ansible.cfg
├── requirements.yml            # collections (and roles) pinned
├── site.yml                    # entry playbook
├── inventories/
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all/
│   │       │   ├── vars.yml
│   │       │   └── vault.yml   # encrypted
│   │       └── webservers.yml
│   └── prod/
│       └── ...
├── playbooks/
├── roles/
│   └── nginx/
├── collections/                # installed locally (gitignored or vendored)
├── molecule/                   # tests (inside role or at root)
└── .ansible-lint
```

```ini
# ansible.cfg
[defaults]
inventory          = inventories/dev
roles_path         = roles
collections_path   = collections
forks              = 20
interpreter_python = auto_silent
callbacks_enabled  = profile_tasks, timer     # shows slow tasks
retry_files_enabled = False
# host_key_checking: leave True in enterprise; manage known_hosts properly

[ssh_connection]
pipelining = True                              # fewer SSH round trips (needs no requiretty)
ssh_args   = -o ControlMaster=auto -o ControlPersist=60s

[privilege_escalation]
become = False                                 # opt in per play/task, least privilege
```

Commands you should know cold:

```bash
ansible-playbook site.yml -i inventories/dev --check --diff        # dry run + show changes
ansible-playbook site.yml --limit web01 --tags config               # narrow scope
ansible-playbook site.yml --list-tasks --list-hosts
ansible-playbook site.yml --start-at-task "Deploy config" --step
ansible-inventory -i inventories/dev --graph                        # see group tree
ansible-inventory -i inventories/dev --host web01                   # see merged vars
ansible-config dump --only-changed                                  # effective config
ansible-doc ansible.builtin.template                                # module docs offline
```

---

## 2. Roles

### Structure

```
roles/nginx/
├── defaults/main.yml     # lowest precedence, meant to be overridden (public API of the role)
├── vars/main.yml         # high precedence, internal constants (hard to override)
├── tasks/main.yml
├── handlers/main.yml
├── templates/nginx.conf.j2
├── files/                # static files for copy:
├── meta/main.yml         # dependencies, galaxy metadata
└── molecule/default/     # tests
```

Create a skeleton: `ansible-galaxy role init roles/nginx`

### Example role

```yaml
# roles/nginx/defaults/main.yml
nginx_port: 80
nginx_worker_processes: auto
nginx_server_name: "{{ inventory_hostname }}"
nginx_package: nginx
```

```yaml
# roles/nginx/tasks/main.yml
- name: Install nginx
  ansible.builtin.package:
    name: "{{ nginx_package }}"
    state: present

- name: Render nginx config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
    validate: nginx -t -c %s        # fail BEFORE replacing a good config
  notify: Reload nginx

- name: Ensure nginx is enabled and running
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

```yaml
# roles/nginx/handlers/main.yml
- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

```jinja
{# roles/nginx/templates/nginx.conf.j2 #}
# Managed by Ansible - do not edit by hand
worker_processes {{ nginx_worker_processes }};
events { worker_connections 1024; }
http {
  server {
    listen {{ nginx_port }};
    server_name {{ nginx_server_name }};
{% for loc in nginx_locations | default([]) %}
    location {{ loc.path }} {
      proxy_pass {{ loc.upstream }};
    }
{% endfor %}
  }
}
```

```yaml
# site.yml
- name: Configure web tier
  hosts: webservers
  become: true
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
```

### Key points to remember

- **defaults vs vars**: put anything a caller may change in `defaults/`. Use `vars/` only for internal constants. Prefix every variable with the role name (`nginx_`) to avoid collisions.
- **Handlers** run once, at the end of the play (or on `meta: flush_handlers`), only if notified by a task that reported `changed`.
- **`import_role` vs `include_role`**: import is static (parsed at start, tags/`when` applied to every task); include is dynamic (evaluated at runtime, needed for loops over roles or computed role names).
- **Role dependencies** go in `meta/main.yml` (`dependencies: [{role: common}]`). Keep them minimal.
- **Tags**: `tags: [nginx, config]` on tasks; run with `--tags config`.
- Roles should be idempotent, OS-aware (use `ansible_facts['os_family']` + `vars/Debian.yml` pattern), and documented (README with variables table).

OS-specific vars pattern:

```yaml
- name: Load OS-specific variables
  ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', params) }}"
  vars:
    params:
      files:
        - "{{ ansible_facts['distribution'] }}.yml"
        - "{{ ansible_facts['os_family'] }}.yml"
        - default.yml
      paths: [vars]
```

---

## 3. Collections

A collection packages modules, plugins, roles and playbooks (`namespace.name`, e.g. `servicenow.itsm`, `amazon.aws`, `community.general`).

```yaml
# requirements.yml  (PIN versions for reproducibility)
collections:
  - name: amazon.aws
    version: ">=8.0.0,<9.0.0"
  - name: servicenow.itsm
    version: ">=2.5.0"
  - name: community.hashi_vault
  - name: ansible.eda
roles: []
```

```bash
ansible-galaxy collection install -r requirements.yml -p collections/
ansible-galaxy collection list
```

- **Always use FQCN**: `ansible.builtin.copy`, `amazon.aws.ec2_instance`. ansible-lint enforces it, and it avoids name clashes.
- Python dependencies are separate (`boto3`, `botocore` for amazon.aws; `requests` for others). In enterprise this is solved by **execution environments** (section 11).
- Private collections: host on Automation Hub / Artifactory, configure in `ansible.cfg` under `[galaxy]` with `server_list`.
- Pin ansible-core and collection versions together; upgrade deliberately, test with Molecule.

---

## 4. Inventory design

### Static YAML inventory

```yaml
# inventories/dev/hosts.yml
all:
  children:
    webservers:
      hosts:
        web01.dev.example.com:
        web02.dev.example.com:
    dbservers:
      hosts:
        db01.dev.example.com:
          ansible_port: 2222
    dev:
      children:
        webservers:
        dbservers:
```

### group_vars / host_vars

```yaml
# inventories/dev/group_vars/all/vars.yml
env: dev
ntp_servers: [ntp1.example.com, ntp2.example.com]
```

```yaml
# inventories/dev/group_vars/webservers.yml
nginx_port: 8080
```

```yaml
# inventories/dev/host_vars/web01.dev.example.com.yml
nginx_worker_processes: 4
```

Design principles:
- **One inventory per environment** (dev/test/prod) with the same variable names, different values. The same playbook runs everywhere.
- Group by **function** (webservers), **environment** (dev), **location** (eu_west) and combine with children groups.
- Keep secrets in `vault.yml` next to `vars.yml`, never inline.
- Avoid putting logic in inventory. Keep it as data.

### Variable precedence (high level, low to high)

1. role `defaults/`
2. inventory `group_vars/all` → `group_vars/<group>` → `host_vars/<host>`
3. play `vars:` / `vars_files`
4. role `vars/`, `include_vars`, `set_fact`, task `vars:`
5. `-e` extra vars (**always wins**)

### Dynamic inventory: AWS

```yaml
# inventories/aws_ec2.yml   (filename must end in aws_ec2.yml)
plugin: amazon.aws.aws_ec2
regions: [eu-west-1]
filters:
  tag:Managed: ansible
  instance-state-name: running
hostnames:
  - tag:Name
  - private-ip-address
compose:
  ansible_host: private_ip_address
keyed_groups:
  - key: tags.Env
    prefix: env              # -> env_dev, env_prod
  - key: tags.Role
    prefix: role             # -> role_web
```

```bash
ansible-inventory -i inventories/aws_ec2.yml --graph
ansible role_web -i inventories/aws_ec2.yml -m ansible.builtin.ping
```

Use **tags** as the source of truth. That is why tagging standards matter. Enable `cache: true` for large estates.

### Inventory in AAP

Inventory sources (AWS, ServiceNow CMDB via `servicenow.itsm.now` plugin, VMware, Azure) sync on a schedule; dynamic groups are built the same way with `keyed_groups`.

```yaml
# inventories/snow.now.yml (CMDB as inventory)
plugin: servicenow.itsm.now
table: cmdb_ci_server
columns: [name, ip_address, sys_class_name, os]
keyed_groups:
  - key: os | lower | replace(' ', '_')
    prefix: os
```

---

## 5. Vault and credential handling

### Ansible Vault basics

```bash
ansible-vault create   inventories/dev/group_vars/all/vault.yml
ansible-vault edit     inventories/dev/group_vars/all/vault.yml
ansible-vault encrypt  secrets.yml
ansible-vault rekey    inventories/dev/group_vars/all/vault.yml
ansible-vault view     inventories/dev/group_vars/all/vault.yml
ansible-vault encrypt_string 's3cr3t' --name 'vault_db_password'
```

Run:

```bash
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass     # chmod 600, never commit
ansible-playbook site.yml --vault-id dev@prompt --vault-id prod@~/.vault_prod
```

### Naming convention (the "vault_ prefix" pattern)

```yaml
# group_vars/all/vault.yml (encrypted)
vault_db_password: "S3cr3t!"
vault_snow_password: "..."

# group_vars/all/vars.yml (plaintext, greppable)
db_password: "{{ vault_db_password }}"
snow_password: "{{ vault_snow_password }}"
```

This lets teams grep which variables are secrets without decrypting.

### Rules for zero plaintext secrets

- **`no_log: true`** on any task handling secrets:

```yaml
- name: Create DB user
  community.mysql.mysql_user:
    name: app
    password: "{{ db_password }}"
    state: present
  no_log: true
```

- Never pass secrets on the command line (`-e password=...` is visible in `ps` and shell history). Use `-e @secrets.yml` (vaulted) instead.
- Prefer **short-lived credentials**: IAM roles/instance profiles, OIDC from CI, not static access keys.
- Read from environment on the controller, set by the CI/AAP credential:

```yaml
vars:
  aws_access_key: "{{ lookup('ansible.builtin.env', 'AWS_ACCESS_KEY_ID') }}"
```

### External secret managers (enterprise)

```yaml
- name: Read secret from HashiCorp Vault
  ansible.builtin.set_fact:
    db_password: "{{ lookup('community.hashi_vault.hashi_vault',
                     'secret=kv/data/app/db:password',
                     url=vault_url, auth_method='approle',
                     role_id=vault_role_id, secret_id=vault_secret_id) }}"
  no_log: true
```

In **AAP**: define a *Credential* (machine, cloud, ServiceNow, Vault) and optionally a *credential plugin* so the secret is fetched from HashiCorp Vault/CyberArk at job run time. The job template just references the credential; the playbook sees env vars/extra vars. No secret is stored in the repo or in Controller.

---

## 6. Error handling

### block / rescue / always

```yaml
- name: Deploy application safely
  hosts: appservers
  tasks:
    - name: Deployment with ServiceNow incident on failure
      block:
        - name: Run deployment
          ansible.builtin.command: /opt/deploy.sh
          register: deploy_result
          changed_when: "'deployed' in deploy_result.stdout"

      rescue:
        - name: Show what failed
          ansible.builtin.debug:
            msg: "Task '{{ ansible_failed_task.name }}' failed: {{ ansible_failed_result.msg | default('n/a') }}"

        - name: Open ServiceNow incident
          servicenow.itsm.incident:
            instance:
              host: "{{ lookup('env', 'SN_HOST') }}"
              username: "{{ lookup('env', 'SN_USERNAME') }}"
              password: "{{ lookup('env', 'SN_PASSWORD') }}"
            state: new
            short_description: "Deployment failed on {{ inventory_hostname }}"
            description: "{{ ansible_failed_result | to_nice_json }}"
            impact: medium
            urgency: medium
          delegate_to: localhost
          no_log: true

        - name: Roll back
          ansible.builtin.command: /opt/rollback.sh

      always:
        - name: Collect log
          ansible.builtin.fetch:
            src: /var/log/deploy.log
            dest: logs/{{ inventory_hostname }}.log
            flat: true
          ignore_errors: true
```

- `rescue` runs only if the block failed; **the play continues** after a successful rescue (host is not marked failed unless rescue also fails).
- `always` runs regardless.
- Special vars in rescue: `ansible_failed_task`, `ansible_failed_result`.

### Fine-grained control

```yaml
- name: Check service health
  ansible.builtin.uri:
    url: "http://{{ inventory_hostname }}:8080/health"
    status_code: [200, 503]
  register: health
  failed_when: health.status != 200 or health.json.status != 'UP'   # custom failure
  changed_when: false                                               # a read never "changes"
  retries: 5                                                        # + until: = retry loop
  delay: 10
  until: health.status == 200

- name: Optional cleanup
  ansible.builtin.command: /opt/cleanup.sh
  ignore_errors: true          # last resort; prefer failed_when
  # ignore_unreachable: true   # also tolerate down hosts
```

### Play-level failure policies

```yaml
- hosts: webservers
  any_errors_fatal: true          # one failure aborts the whole play on all hosts
  max_fail_percentage: 20         # abort rolling batches if >20% fail (works with serial)
  force_handlers: true            # run handlers even if the play fails
```

Guidance:
- Fail **early and loudly** for unsafe states (use `ansible.builtin.assert` for preconditions).
- Rescue should either **recover** or **raise a ticket**; do not swallow errors silently.
- Use `assert` to validate input:

```yaml
- name: Validate required variables
  ansible.builtin.assert:
    that:
      - server_name is defined
      - env in ['dev', 'test', 'prod']
    fail_msg: "server_name and valid env are required"
```

---

## 7. Idempotency

**Rule:** prefer purpose-built modules over `command`/`shell`.

| Bad (not idempotent) | Good |
|---|---|
| `shell: apt install nginx` | `ansible.builtin.package: name=nginx state=present` |
| `shell: echo "x" >> file` | `ansible.builtin.lineinfile` / `blockinfile` / `template` |
| `command: useradd bob` | `ansible.builtin.user: name=bob` |
| `shell: mkdir /opt/app` | `ansible.builtin.file: path=/opt/app state=directory` |

When you must use `command`/`shell`, make it idempotent yourself:

```yaml
- name: Initialise database (only once)
  ansible.builtin.command: /opt/app/init_db.sh
  args:
    creates: /opt/app/.db_initialised          # skip if file exists

- name: Check current version
  ansible.builtin.command: /opt/app/bin/version
  register: ver
  changed_when: false                          # read-only task

- name: Upgrade if needed
  ansible.builtin.command: /opt/app/upgrade.sh
  when: ver.stdout is version('2.0', '<')
  register: up
  changed_when: "'upgraded' in up.stdout"
```

Other points:
- `shell` with pipes needs `set -o pipefail` (ansible-lint rule `risky-shell-pipe`).
- Use `--check --diff` to preview. Modules that don't support check mode are skipped; mark read-only tasks with `check_mode: false` so facts are still gathered in check mode.
- **Proof of idempotency**: run twice; second run must show `changed=0`. Molecule automates this (section 10).
- Avoid timestamps/random values in templates, they force "changed" every run.
- Use `state: present/absent` semantics, not "do this now".

---

## 8. Scale and performance: forks, serial, async

### forks and strategy

```ini
forks = 50          # parallel hosts per task (default 5). Limited by controller CPU/RAM.
```

```yaml
- hosts: all
  strategy: free        # default 'linear' waits for every host per task; 'free' lets hosts run independently
```

Other speedups: `pipelining = True`, `gather_facts: false` (or `gather_subset: [min]`), fact caching, avoid `with_items` over large lists with module calls per item (pass lists to `package`), `ControlPersist`.

### serial: rolling deployments

```yaml
- name: Rolling update
  hosts: webservers
  serial: ["10%", "50%", "100%"]     # canary, then half, then the rest
  max_fail_percentage: 10
  tasks:
    - name: Take out of load balancer
      ansible.builtin.uri:
        url: "https://lb.example.com/api/disable/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost

    - name: Update app
      ansible.builtin.package:
        name: myapp
        state: latest

    - name: Wait for app
      ansible.builtin.wait_for:
        port: 8080
        delay: 5
        timeout: 120

    - name: Put back in load balancer
      ansible.builtin.uri:
        url: "https://lb.example.com/api/enable/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost
```

### async / poll (long tasks, avoid SSH timeouts)

```yaml
- name: Start long job (fire and forget)
  ansible.builtin.command: /opt/backup.sh
  async: 3600          # max runtime seconds
  poll: 0              # do not wait
  register: backup_job

- name: Do other work ...
  ansible.builtin.debug: msg="working"

- name: Wait for job to finish
  ansible.builtin.async_status:
    jid: "{{ backup_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 120
  delay: 30
```

### Other scale tools

```yaml
- name: Only once per play (e.g. DB migration)
  ansible.builtin.command: /opt/migrate.sh
  run_once: true

- name: Limit concurrency on a fragile API
  ansible.builtin.uri:
    url: https://api.example.com/register
  throttle: 3

- name: Run on controller, not target
  delegate_to: localhost
```

In AAP: use **instance groups / execution nodes / automation mesh** to scale jobs, **job slicing** to split a big inventory across parallel jobs, and `forks` per job template.

---

## 9. ansible-lint

```bash
pip install ansible-lint
ansible-lint                     # lints repo according to .ansible-lint
ansible-lint --fix               # auto-fix some rules
ansible-lint -p roles/nginx      # parseable output
```

```yaml
# .ansible-lint
profile: production        # min < basic < moderate < safety < shared < production
exclude_paths:
  - .github/
  - collections/
skip_list:
  - yaml[line-length]
warn_list:
  - experimental
enable_list:
  - no-same-owner
```

Common findings and fixes:

| Rule | Meaning | Fix |
|---|---|---|
| `fqcn[action-core]` | short module name | `ansible.builtin.copy` |
| `name[missing]` / `name[casing]` | unnamed / lowercase task names | add `name: Install nginx` |
| `no-changed-when` | `command` with no `changed_when` | add `changed_when:` |
| `risky-shell-pipe` | pipe without pipefail | `set -o pipefail;` + `args: executable: /bin/bash` |
| `yaml[truthy]` | `yes/no` | use `true/false` |
| `no-handler` | task `when: x.changed` | use `notify` |
| `var-naming[no-role-prefix]` | role vars not prefixed | `nginx_port` |
| `risky-file-permissions` | missing `mode` | add `mode: "0644"` |

CI gate (GitHub Actions):

```yaml
name: lint
on: [pull_request]
jobs:
  ansible-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ansible/ansible-lint@main
```

Pair with `yamllint` and `ansible-playbook --syntax-check`.

---

## 10. Molecule (role testing)

Flow: `create → converge → idempotence → verify → destroy`.

```bash
pip install molecule "molecule-plugins[podman]" ansible-core
cd roles/nginx
molecule init scenario --driver-name podman default
molecule test              # full cycle
molecule converge          # apply role, keep container (dev loop)
molecule verify            # run checks only
molecule login             # shell into the instance
molecule destroy
```

```yaml
# molecule/default/molecule.yml
driver:
  name: podman
platforms:
  - name: ubuntu2204
    image: docker.io/geerlingguy/docker-ubuntu2204-ansible:latest
    pre_build_image: true
    privileged: true
    command: /lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
provisioner:
  name: ansible
  env:
    ANSIBLE_ROLES_PATH: "${MOLECULE_PROJECT_DIRECTORY}/../"
verifier:
  name: ansible
```

```yaml
# molecule/default/converge.yml
- name: Converge
  hosts: all
  become: true
  roles:
    - role: nginx
      vars:
        nginx_port: 8081
```

```yaml
# molecule/default/verify.yml
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
    - name: Check nginx package
      ansible.builtin.package_facts:
    - name: Assert nginx installed
      ansible.builtin.assert:
        that: "'nginx' in ansible_facts.packages"

    - name: Check port responds
      ansible.builtin.uri:
        url: http://localhost:8081
        status_code: 200
```

- The built-in **idempotence** step fails if the second converge reports any change. That is your proof for section 7.
- Test matrix in CI (multiple OS images) with GitHub Actions `strategy.matrix`.

```yaml
jobs:
  molecule:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        distro: [ubuntu2204, rockylinux9]
    steps:
      - uses: actions/checkout@v4
      - run: pip install molecule "molecule-plugins[podman]" ansible-core
      - run: molecule test
        working-directory: roles/nginx
        env:
          MOLECULE_DISTRO: ${{ matrix.distro }}
```

---

## 11. Execution Environments (EE)

An EE is an **OCI container image** containing `ansible-core` + `ansible-runner` + collections + Python/system dependencies. It replaces "works on my laptop" with an identical runtime on laptop, CI and AAP.

```yaml
# execution-environment.yml  (ansible-builder v3 schema)
version: 3
images:
  base_image:
    name: quay.io/ansible/community-ee-minimal:latest   # or registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel9
dependencies:
  ansible_core:
    package_pip: ansible-core>=2.16,<2.17
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements.yml          # collections
  python: requirements.txt          # boto3, requests, ...
  system: bindep.txt                # OS packages
additional_build_steps:
  append_final:
    - RUN echo "Built for Sanofi automation" > /etc/ee-info
```

```text
# requirements.txt
boto3
botocore
requests
```

```text
# bindep.txt
git [platform:rpm]
```

```bash
pip install ansible-builder ansible-navigator
ansible-builder build -t registry.example.com/ee/infra:1.0.0 -f execution-environment.yml
ansible-navigator run site.yml --eei registry.example.com/ee/infra:1.0.0 -m stdout
podman push registry.example.com/ee/infra:1.0.0
```

In AAP: register the image under *Execution Environments*, assign it to a project/job template. Version tags, never `latest` in prod. Scan images with `trivy`. A **Decision Environment (DE)** is the same idea for Event-Driven Ansible (`ansible-rulebook` + EDA collections).

---

## 12. AAP concepts

| Component | Role |
|---|---|
| **Automation Controller** (formerly Tower) | Web UI + REST API + RBAC + scheduling + job runs. The part most people mean by "AAP". |
| **Automation Hub** | Private registry for certified/validated/custom **collections** and EE images |
| **Event-Driven Ansible controller** | Runs rulebooks (events → actions) using decision environments |
| **Automation Mesh** | Overlay network of control, hybrid, execution and **hop** nodes to run jobs near targets (regions, DMZs) |
| **Platform gateway** (2.5+) | Unified UI/auth entry point |

### Controller objects (know the chain)

```
Organization
├─ Credentials  (machine SSH, cloud, ServiceNow, vault password, Source Control)
├─ Projects     (Git repo + branch → playbooks)
├─ Inventories  (static / dynamic sources, groups, hosts)
├─ Execution Environments
├─ Job Templates  = Project + Playbook + Inventory + Credentials + EE + (Survey, Limit, Forks, Tags)
├─ Workflow Templates = chained job templates with success/failure/always paths and approval nodes
├─ Schedules, Notifications (Slack/email/webhook), Teams, Users, RBAC
└─ Instance Groups (which execution nodes a job may use; capacity/forks)
```

Concepts to be able to explain:

- **Job template vs workflow**: single run vs orchestrated pipeline (e.g. snapshot → patch → verify → on failure rollback + open incident).
- **Survey**: user-facing form that produces `extra_vars` (used from ServiceNow catalog → AAP).
- **RBAC**: Execute vs Admin roles on templates; Use on credentials, so users can run jobs without seeing secrets.
- **Inventory sync, smart inventories, job slicing**: scale and flexibility.
- **Instance groups and capacity**: capacity measured in forks; containers groups for OpenShift-based installs.
- **Clustering/HA**: multiple control nodes behind a load balancer, external PostgreSQL (HA), execution nodes scaled separately.
- **Auditing**: every job stores who/when/what/inventory/output/artifacts; stream logs to Splunk/ELK; use for GxP evidence.
- **AAP vs core Ansible**: core is the engine (CLI, free). AAP adds RBAC, UI/API, scheduling, credential vaulting, workflow, audit, EEs, mesh, Hub, EDA and Red Hat support. This is the classic opener question.

### Launch a job from outside (ServiceNow/GitHub/curl)

```bash
curl -s -X POST https://aap.example.com/api/controller/v2/job_templates/42/launch/ \
  -H "Authorization: Bearer $AAP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"extra_vars": {"server_name": "app01", "env": "dev"}}'
# then poll: GET /api/controller/v2/jobs/<id>/
```

Config-as-code for AAP: manage objects with the `ansible.controller` collection (or `infra.aap_configuration`) so templates, projects and credentials live in Git.

```yaml
- name: Create job template
  ansible.controller.job_template:
    name: Provision EC2
    organization: Default
    project: Infra Repo
    playbook: playbooks/provision_ec2.yml
    inventory: Localhost
    credentials: [AWS Dev]
    execution_environment: infra-ee
    ask_variables_on_launch: true
    state: present
```

---

## 13. Putting it together: a production-style playbook

```yaml
# playbooks/provision_and_register.yml
- name: Provision EC2 and register in CMDB
  hosts: localhost
  gather_facts: false
  vars:
    server_name: "{{ server_name }}"      # passed from ServiceNow (extra vars)
    env: "{{ env | default('dev') }}"
  pre_tasks:
    - name: Validate inputs
      ansible.builtin.assert:
        that:
          - server_name is match('^[a-z0-9-]{3,30}$')
          - env in ['dev', 'test', 'prod']

    - name: Open change request
      servicenow.itsm.change_request:
        state: new
        type: standard
        short_description: "Provision {{ server_name }} ({{ env }})"
      register: cr
      no_log: true

  tasks:
    - name: Provision with rollback on failure
      block:
        - name: Create EC2 instance
          amazon.aws.ec2_instance:
            name: "{{ server_name }}"
            instance_type: t3.micro
            image_id: "{{ ami_id }}"
            vpc_subnet_id: "{{ subnet_id }}"
            security_groups: [web-sg]
            tags:
              Env: "{{ env }}"
              Managed: ansible
            wait: true
            state: running
          register: ec2

        - name: Register CI in CMDB
          servicenow.itsm.configuration_item:
            sys_class_name: cmdb_ci_server
            name: "{{ server_name }}"
            ip_address: "{{ ec2.instances[0].private_ip_address }}"
          no_log: true

      rescue:
        - name: Terminate half-built instance
          amazon.aws.ec2_instance:
            filters:
              "tag:Name": "{{ server_name }}"
            state: absent

        - name: Raise incident
          servicenow.itsm.incident:
            state: new
            short_description: "Provisioning failed for {{ server_name }}"
            description: "{{ ansible_failed_result | to_nice_json }}"
          no_log: true

      always:
        - name: Write execution log to change work notes
          servicenow.itsm.api:
            resource: change_request
            action: patch
            sys_id: "{{ cr.record.sys_id }}"
            data:
              work_notes: "Run complete for {{ server_name }}. Result: {{ 'FAILED' if ansible_failed_task is defined else 'OK' }}"
          no_log: true
```

(Credentials for both ServiceNow and AWS come from environment variables supplied by AAP credentials. Nothing is stored in the repo.)

---

## 14. Optional: custom module and filter plugin

### Custom module

```python
#!/usr/bin/python
# library/hello_file.py  (or plugins/modules/ in a collection)
from ansible.module_utils.basic import AnsibleModule
import os

DOCUMENTATION = r"""
module: hello_file
short_description: Ensure a file contains a greeting
options:
  path: {description: File path, required: true, type: path}
  name: {description: Name to greet, required: true, type: str}
"""


def run():
    module = AnsibleModule(
        argument_spec=dict(path=dict(type="path", required=True),
                           name=dict(type="str", required=True)),
        supports_check_mode=True,
    )
    path, name = module.params["path"], module.params["name"]
    desired = f"Hello {name}\n"
    current = open(path).read() if os.path.exists(path) else None
    changed = current != desired                      # idempotency: compare first

    if changed and not module.check_mode:
        with open(path, "w") as f:
            f.write(desired)
    module.exit_json(changed=changed, path=path)


if __name__ == "__main__":
    run()
```

```yaml
- name: Use custom module
  hello_file:
    path: /tmp/hello.txt
    name: Sanofi
```

Key rules: return `changed` accurately, support check mode, fail with `module.fail_json(msg=...)`, document args, unit-test with `pytest`.

### Filter plugin

```python
# filter_plugins/custom.py
class FilterModule:
    def filters(self):
        return {"to_tag_dict": self.to_tag_dict}

    @staticmethod
    def to_tag_dict(tags):
        """[{'Key':'a','Value':'b'}] -> {'a':'b'}"""
        return {t["Key"]: t["Value"] for t in tags}
```

```yaml
- ansible.builtin.debug:
    msg: "{{ ec2.instances[0].tags | to_tag_dict }}"
```

(Often unnecessary: `items2dict`, `community.general.json_query` and `selectattr` cover many cases.)

---

## 15. Interview quick-fire (answer in 2-3 sentences each)

1. **AAP vs core Ansible?** Core is the CLI engine; AAP adds RBAC, API/UI, scheduling, credential management, workflows, EEs, mesh, Hub, EDA, audit and support.
2. **How do you keep secrets out of Git?** Vault for static, external secret manager through AAP credential plugins for runtime, `no_log`, short-lived cloud creds via roles/OIDC.
3. **How do you prove a playbook is idempotent?** Run twice and expect `changed=0`; Molecule's idempotence step automates it.
4. **Rolling deployment to 500 hosts?** `serial` with canary batches, `max_fail_percentage`, health checks, LB drain via `delegate_to`, rescue with rollback.
5. **defaults vs vars in a role?** defaults are overridable public inputs (lowest precedence); vars are internal constants.
6. **A job hangs/times out?** Check connectivity/become, use `async/poll`, increase `timeout`, check `forks`, mesh/hop node routing, `-vvv`.
7. **How do you make 20 teams automate consistently?** Shared roles/collections in Hub, EE versions, lint + Molecule gates in CI, protected branches, naming/tagging standards.
8. **How do you provide audit evidence (GxP)?** Controller job history, CR linked to the job, logs pushed to work notes, Git history and PR approvals, log streaming to SIEM.
9. **What is an EE and why?** A container with ansible-core, collections and dependencies; consistent runtime across dev/CI/prod and removes controller dependency drift.
10. **Static vs dynamic inventory?** Static for stable infra; dynamic (aws_ec2, ServiceNow CMDB) built from source of truth with `keyed_groups` and tags.

---

## 16. Revision checklist

- [ ] Build a role with defaults, handlers, template + `validate`, and run it twice (`changed=0`)
- [ ] Write a Molecule scenario for it with a verify step
- [ ] `requirements.yml` pinned; install into `collections/`
- [ ] Two inventories (dev/prod) with identical var names, plus one `aws_ec2` dynamic inventory
- [ ] Vault file with the `vault_` prefix pattern; `no_log` on secret tasks
- [ ] A `block/rescue/always` creating a ServiceNow incident (test against your PDI)
- [ ] Rolling update with `serial: ["10%","50%","100%"]`
- [ ] An `async` + `async_status` example
- [ ] `.ansible-lint` with production profile passing; GitHub Action gate
- [ ] Build an EE with `ansible-builder`, run with `ansible-navigator`
- [ ] Explain every Controller object and the AAP-vs-core answer out loud
- [ ] (Optional) one custom module with check-mode support, one filter plugin
