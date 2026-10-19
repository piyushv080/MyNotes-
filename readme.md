# Complete Tracker — One Project, 30 Min/Day, 14 Weeks

**Project:** `closed-loop-automation-platform` — ServiceNow = brain, AWS = muscle, Ansible = nervous system, Python = glue.
**Daily:** 5 min read the linked doc → 20 min build → 5 min journal.
**How to use:** work top to bottom. Click the link, do the task, change `[ ]` to `[x]`. Never open a tab that isn't in this file.

---

## BLOCK 1 — ServiceNow: The Governance Brain (Weeks 1–3)

Do not open AWS or Ansible during this block.

| # | Status | Topic | Task | Link |
|---|---|---|---|---|
| 1.1 | [ ] | Get a sandbox | Claim free Personal Developer Instance (PDI) | [developer.servicenow.com](https://developer.servicenow.com/dev.do#!/guides/xanadu/now-platform/pdi-guide/personal-developer-instance-guide-introduction) |
| 1.2 | [ ] | Service Catalog | Build catalog item "Request Enterprise App Server" with vars `server_name`, `env`, `requested_by` | [Catalog item docs](https://www.servicenow.com/docs/bundle/zurich-servicenow-platform/page/product/service-catalog-management/task/t_DefineACatalogItem.html) |
| 1.3 | [ ] | Flow Designer | Add an approval action, then a REST step on approval | [Flow Designer](https://www.servicenow.com/docs/bundle/zurich-build-workflows/page/administer/flow-designer/concept/flow-designer.html) |
| 1.4 | [ ] | Table API | In REST API Explorer: `POST /api/now/table/incident`, then `PATCH` to update state | [Table API ref](https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html) |
| 1.5 | [ ] | REST API Explorer | Learn to generate code snippets from the explorer UI | [REST API Explorer](https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_RESTAPIExplorer.html) |
| 1.6 | [ ] | OAuth2 | Create OAuth app registry entry; fetch a bearer token with curl | [OAuth setup](https://www.servicenow.com/docs/bundle/zurich-platform-security/page/administer/security/task/t_SettingUpOAuth.html) |
| 1.7 | [ ] | CMDB + IRE | Understand how IRE prevents duplicate CIs (critical interview topic) | [IRE docs](https://www.servicenow.com/docs/bundle/zurich-servicenow-platform/page/product/configuration-management/concept/identification-reconciliation.html) |
| 1.8 | [ ] | Scripting basics | `GlideRecord` vs `GlideRecordSecure` vs `GlideAggregate`; write one Script Include | [Glide API](https://developer.servicenow.com/dev.do#!/reference/api/zurich/server/) |
| 1.9 | [ ] | MID Server (read only) | How a cloud instance reaches private on-prem networks | [MID Server](https://www.servicenow.com/docs/bundle/zurich-servicenow-platform/page/product/mid-server/concept/mid-server-landing.html) |
| 1.10 | [ ] | IntegrationHub | What a spoke is; the REST step; when to use it vs raw REST message | [IntegrationHub](https://www.servicenow.com/docs/bundle/zurich-build-workflows/page/administer/integrationhub/concept/integrationhub.html) |

---

## BLOCK 2 — AWS + Ansible: The Muscle (Weeks 4–6)

Only terminal, VS Code, AWS free tier.

| # | Status | Topic | Task | Link |
|---|---|---|---|---|
| 2.1 | [ ] | AWS account | Sign up free tier; create IAM user with programmatic access | [AWS Free Tier](https://aws.amazon.com/free/) |
| 2.2 | [ ] | Ansible install | Install `ansible-core` + `amazon.aws` collection + `boto3` | [Install guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) |
| 2.3 | [ ] | EC2 provisioning | Playbook creating VPC, security group, EC2 instance | [amazon.aws.ec2_instance](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_instance_module.html) |
| 2.4 | [ ] | Role structure | Refactor into a role: `tasks/ vars/ defaults/ templates/ handlers/` | [Roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html) |
| 2.5 | [ ] | Jinja2 templates | Template an Nginx config with variables | [Templating](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html) |
| 2.6 | [ ] | Ansible Vault | Encrypt AWS keys — zero plaintext secrets | [Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) |
| 2.7 | [ ] | Error handling | `block` / `rescue` / `always`, `failed_when`, `ignore_errors` | [Error handling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html) |
| 2.8 | [ ] | Dynamic inventory | Replace static hosts file with AWS tag-based inventory | [aws_ec2 plugin](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html) |
| 2.9 | [ ] | Scale & performance | `forks`, `serial: ["10%","50%","100%"]`, `async`/`poll` | [Strategies](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html) |
| 2.10 | [ ] | Idempotency | Make the playbook safe to run 100 times | [Check mode](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html) |
| 2.11 | [ ] | Terraform | Same EC2 stack in Terraform + S3 remote state with locking | [S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3) |
| 2.12 | [ ] | Terraform modules | Split into a reusable module with inputs/outputs | [Modules](https://developer.hashicorp.com/terraform/language/modules) |

---

## BLOCK 3 — Integration: Tying the Knot (Weeks 7–9)

| # | Status | Topic | Task | Link |
|---|---|---|---|---|
| 3.1 | [ ] | Tunnel | Install ngrok so ServiceNow cloud can reach your laptop | [ngrok](https://ngrok.com/docs/getting-started/) |
| 3.2 | [ ] | The handshake | ServiceNow approve → webhook → ngrok → Ansible run → EC2 created | [Webhook via REST step](https://www.servicenow.com/docs/bundle/zurich-build-workflows/page/administer/integrationhub/task/rest-step.html) |
| 3.3 | [ ] | servicenow.itsm collection | Install it, read the module index | [Collection docs](https://docs.ansible.com/ansible/latest/collections/servicenow/itsm/index.html) |
| 3.4 | [ ] | CMDB write-back | End of playbook registers the new instance as a CI | [configuration_item module](https://docs.ansible.com/ansible/latest/collections/servicenow/itsm/configuration_item_module.html) |
| 3.5 | [ ] | Incident module | Create / update / close incidents from a playbook | [incident module](https://docs.ansible.com/ansible/latest/collections/servicenow/itsm/incident_module.html) |
| 3.6 | [ ] | Change requests | Open a CR before the change, close it after | [change_request module](https://docs.ansible.com/ansible/latest/collections/servicenow/itsm/change_request_module.html) |
| 3.7 | [ ] | Audit trail (GxP) | Push execution logs into CR work notes — your compliance answer | [itsm.api module](https://docs.ansible.com/ansible/latest/collections/servicenow/itsm/api_module.html) |
| 3.8 | [ ] | GitOps | GitHub repo, protect `main`, PR-only workflow | [Protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) |
| 3.9 | [ ] | CI pipeline | GitHub Actions workflow with jobs + matrix | [Actions syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions) |
| 3.10 | [ ] | Lint gate | `ansible-lint` on every PR | [ansible-lint](https://ansible.readthedocs.io/projects/lint/) |
| 3.11 | [ ] | IaC scanning | `tflint` + `checkov` in the pipeline | [Checkov](https://www.checkov.io/1.Welcome/What%20is%20Checkov.html) |
| 3.12 | [ ] | Container scanning | `trivy` on your image / filesystem | [Trivy](https://trivy.dev/latest/docs/) |
| 3.13 | [ ] | Zero static creds | OIDC — GitHub Actions assumes an AWS role, no stored keys | [OIDC to AWS](https://docs.github.com/en/actions/concepts/security/openid-connect) |
| 3.14 | [ ] | Drift detection | Nightly scheduled `terraform plan -detailed-exitcode` | [Scheduled workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule) |

---

## BLOCK 4 — EDA + Showcase (Weeks 10–12)

| # | Status | Topic | Task | Link |
|---|---|---|---|---|
| 4.1 | [ ] | EDA concepts | What a rulebook / source / condition / action is | [EDA docs](https://docs.ansible.com/ansible/latest/collections/ansible/eda/index.html) |
| 4.2 | [ ] | Install | `ansible-rulebook` + `ansible.eda` collection | [ansible-rulebook](https://ansible.readthedocs.io/projects/rulebook/en/stable/installation.html) |
| 4.3 | [ ] | First rulebook | Webhook source, `condition: event.payload.status == "down"` | [Rulebook reference](https://ansible.readthedocs.io/projects/rulebook/en/stable/rulebooks.html) |
| 4.4 | [ ] | Simulate the alert | `curl -X POST localhost:5000/endpoint -d '{"status":"down","host":"web01"}'` | [Webhook source](https://docs.ansible.com/ansible/latest/collections/ansible/eda/webhook_source.html) |
| 4.5 | [ ] | Self-heal chain | Rule → open P1 incident → remediation playbook → close ticket | [EDA lab guide (p.10–15)](https://people.redhat.com/mskinner/rhug/q1.2024/Ansible-EDA.pdf) |
| 4.6 | [ ] | Architecture README | Diagram + data flow + design decisions | [Mermaid in GitHub](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/creating-diagrams) |
| 4.7 | [ ] | 3-min video | Min 1 approve → min 2 provision → min 3 kill & self-heal | [Loom](https://www.loom.com/) |
| 4.8 | [ ] | LinkedIn post | Video + repo link + business-value caption | — |

---

## BLOCK 5 — Read-Only Gap Fillers (Weeks 13–14)

You cannot build these on a laptop. Reading + being able to speak to them is enough, and interviewers ask all of them.

| # | Status | Topic | Why it's asked | Link |
|---|---|---|---|---|
| 5.1 | [ ] | AAP vs core Ansible | The single most common opener at platform level | [AAP planning guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/planning_your_installation/index) |
| 5.2 | [ ] | Execution Environments | How playbooks + deps get packaged as OCI containers | [EE guide](https://docs.ansible.com/ansible/latest/getting_started_ee/index.html) |
| 5.3 | [ ] | ansible-builder | The command that builds an EE | [ansible-builder](https://ansible.readthedocs.io/projects/builder/en/latest/) |
| 5.4 | [ ] | Automation Mesh | Hop/execution nodes across regions & DMZs | [Mesh docs](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/automation_mesh_for_managed_cloud_or_operator-based_installations/index) |
| 5.5 | [ ] | Decision Environments | The EDA equivalent of an EE | [Decision environments](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_decisions/eda-decision-environments) |
| 5.6 | [ ] | Controller clustering | HA, instance groups, capacity | [Controller admin](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/configuring_automation_execution/index) |
| 5.7 | [ ] | Enterprise secrets | Credential plugins for HashiCorp Vault / CyberArk | [Credential plugins](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/configuring_automation_execution/assembly-controller-secret-management) |
| 5.8 | [ ] | Async at scale | Message brokers for long jobs (SQS / Kafka) to avoid timeouts | [SQS patterns](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/queue-based-load-leveling.html) |
| 5.9 | [ ] | Automated remediation patterns | How big enterprises design closed loops | [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/) |
| 5.10 | [ ] | Kubernetes basics | Only if a JD mentions it — pods, services, scaling | [K8s concepts](https://kubernetes.io/docs/concepts/) |
| 5.11 | [ ] | Behavioral + negotiation | The round nobody preps and everyone loses money on | [STAR method](https://www.themuse.com/advice/star-interview-method) |

---

## PYTHON TRACK — Run in Parallel, 15 Min Every Alternate Day

They will ask programming. Not LeetCode-hard — DevOps-flavoured: API calls, JSON, dicts, error handling. Do these in the context of your project so it's not extra work.

| # | Status | Topic | Task | Link |
|---|---|---|---|---|
| P1 | [ ] | Core data structures | dict/list/set comprehensions, slicing, unpacking | [Data structures](https://docs.python.org/3/tutorial/datastructures.html) |
| P2 | [ ] | JSON handling | Parse nested API responses; `json.loads` / `dumps`, `.get()` with defaults | [json module](https://docs.python.org/3/library/json.html) |
| P3 | [ ] | HTTP calls | `requests` — GET/POST/PATCH, headers, bearer token auth | [requests quickstart](https://requests.readthedocs.io/en/latest/user/quickstart/) |
| P4 | [ ] | **Project task** | Script that creates a ServiceNow incident via Table API with an OAuth bearer token | [Table API](https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html) |
| P5 | [ ] | Error handling | `try/except/else/finally`, custom exceptions, retries with backoff | [Errors](https://docs.python.org/3/tutorial/errors.html) |
| P6 | [ ] | boto3 | List EC2 instances, filter by tag, start/stop | [boto3 EC2](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html) |
| P7 | [ ] | **Project task** | Script that reconciles AWS instances against ServiceNow CIs and reports drift | — |
| P8 | [ ] | Functions & typing | args/kwargs, type hints, dataclasses | [typing](https://docs.python.org/3/library/typing.html) |
| P9 | [ ] | File & env handling | `pathlib`, `os.environ`, reading YAML with `PyYAML` | [PyYAML](https://pyyaml.org/wiki/PyYAMLDocumentation) |
| P10 | [ ] | Custom Ansible module | Write a small module in Python — big differentiator | [Developing modules](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_general.html) |
| P11 | [ ] | Custom filter plugin | A Jinja2 filter in Python for your playbooks | [Filter plugins](https://docs.ansible.com/ansible/latest/dev_guide/developing_plugins.html#filter-plugins) |
| P12 | [ ] | Testing | `pytest` basics + one test for your reconcile script | [pytest](https://docs.pytest.org/en/stable/getting-started.html) |
| P13 | [ ] | Interview classics | String/list manipulation, log parsing, counting with `collections.Counter` | [collections](https://docs.python.org/3/library/collections.html) |
| P14 | [ ] | Bash minimum | Enough shell to not fumble: pipes, `grep`/`awk`/`sed`, exit codes, `set -euo pipefail` | [Bash guide](https://mywiki.wooledge.org/BashGuide) |

---

## MEMORIZE-BY-HAND DRILLS

15 min, alternate days, blank file, no Google. Rotate through these four.
1. A role directory tree from scratch.
2. `block` / `rescue` / `always` with a ServiceNow incident in the rescue.
3. An EDA rulebook with one source, one condition, one action.
4. A Python function that POSTs to a REST API with bearer auth and handles a non-200.
```yaml
block:
  - name: Attempt critical infrastructure deployment
    ansible.builtin.command: /opt/deploy.sh
rescue:
  - name: Open a ServiceNow incident automatically
    servicenow.itsm.incident:
      short_description: "Deployment failed on {{ inventory_hostname }}"
      state: new
always:
  - name: Push execution log to change work notes
    servicenow.itsm.api:
      resource: change_request
      action: patch
```

---

## WEEKLY RHYTHM

| Day | 30 min |
|---|---|
| Mon | Next table row |
| Tue | Next table row + Python (15/15) |
| Wed | Next table row |
| Thu | Next table row + Python (15/15) |
| Fri | **Talk, don't type** — explain this week's slice out loud for 2 min, no notes |
| Sat | Memorize-by-hand drill |
| Sun | Off. Genuinely off. |

Missed a day? Do not catch up. Move to the next row.
