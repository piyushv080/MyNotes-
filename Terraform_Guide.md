# Terraform Revision Guide (for Automation Engineers)

Scope: core workflow, HCL language, the EC2 stack (VPC, security group, instance), variables/outputs/locals, state and S3 remote backend with locking, modules, environments, import/moved/lifecycle, drift detection, CI, and how Terraform complements Ansible.

How to use: build the stack in section 3 in a scratch AWS account, then refactor it into a module (section 7) and add the remote backend (section 5). Always run `terraform destroy` when finished to avoid charges.

> Terraform vs OpenTofu: OpenTofu is a compatible fork with the same workflow (`tofu` instead of `terraform`). Concepts here apply to both. Check provider and attribute names against current registry docs (`registry.terraform.io`), because AWS provider arguments change between major versions.

---

## 0. Mental model

- **Declarative infrastructure:** you describe the desired end state; Terraform computes a **plan** (diff between config, state and real world) and **applies** it.
- **State** (`terraform.tfstate`) maps your resource blocks to real object IDs. It is the source of truth for what Terraform manages. Lose or corrupt it and Terraform loses track.
- **Providers** (`hashicorp/aws`) are plugins that talk to APIs. **Resources** create things; **data sources** read existing things.
- **Dependency graph:** references (`aws_subnet.main.id`) create implicit ordering; Terraform parallelises everything else.
- **Where it fits in your world:** Terraform provisions *infrastructure* (VPC, EC2, IAM, S3); Ansible configures *what runs on it* (packages, files, services) and orchestrates *processes* (patching, deployments, remediation).

---

## 1. Core workflow and commands

```bash
terraform version
terraform init                         # download providers/modules, configure backend
terraform fmt -recursive               # canonical formatting (CI: fmt -check)
terraform validate                     # syntax and internal consistency (no API calls)
terraform plan -out=tfplan             # compute and save the plan
terraform show tfplan                  # human-readable plan
terraform apply tfplan                 # apply EXACTLY the reviewed plan
terraform apply -auto-approve          # CI only, after an approved plan
terraform destroy                      # tear down everything in this state
terraform output                       # show outputs
terraform output -raw instance_id
terraform state list                   # resources in state
terraform state show aws_instance.web
terraform providers
terraform graph | dot -Tpng > graph.png
terraform console                      # evaluate expressions interactively
terraform plan -target=aws_instance.web   # emergency use only; skips the full graph
terraform plan -refresh-only           # detect drift without proposing config changes
terraform plan -detailed-exitcode      # 0 no change, 1 error, 2 changes
terraform plan -var-file=prod.tfvars -var 'instance_type=t3.small'
terraform apply -replace=aws_instance.web      # force recreate one resource (replaces "taint")
```

Plan symbols: `+` create, `-` destroy, `~` update in place, `-/+` replace (destroy then create; look at "forces replacement" in the output), `<=` data read.

**Habit:** read the plan summary line (`Plan: 2 to add, 1 to change, 1 to destroy`). Any unexpected destroy is a stop-and-ask.

---

## 2. HCL language essentials

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"          # pin major; commit .terraform.lock.hcl
    }
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = {
      Project   = "enterprise-app"
      ManagedBy = "terraform"
      Env       = var.env
    }
  }
}
```

### Variables, locals, outputs

```hcl
variable "env" {
  type        = string
  description = "Deployment environment"
  validation {
    condition     = contains(["dev", "test", "prod"], var.env)
    error_message = "env must be dev, test or prod."
  }
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "allowed_cidrs" {
  type    = list(string)
  default = []
}

variable "tags" {
  type    = map(string)
  default = {}
}

variable "db_password" {
  type      = string
  sensitive = true                   # hidden in plan/outputs (but still in state!)
}

locals {
  name_prefix = "app-${var.env}"
  common_tags = merge(var.tags, { Name = local.name_prefix })
}

output "instance_ids" {
  value       = aws_instance.web[*].id
  description = "IDs of the web instances"
}
```

Set values via, in increasing precedence: defaults → `terraform.tfvars` / `*.auto.tfvars` → `-var-file` → `TF_VAR_name` env vars → `-var`.

```hcl
# prod.tfvars
env            = "prod"
instance_count = 3
allowed_cidrs  = ["10.0.0.0/8"]
```

### Expressions you need

```hcl
# count and for_each
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.micro"
  tags          = { Name = "${local.name_prefix}-${count.index}" }
}

resource "aws_subnet" "private" {
  for_each          = toset(var.azs)            # keys are stable; prefer for_each over count for sets/maps
  vpc_id            = aws_vpc.main.id
  availability_zone = each.value
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, index(var.azs, each.value))
}

# conditionals and dynamic blocks
instance_type = var.env == "prod" ? "m5.large" : "t3.micro"

dynamic "ingress" {
  for_each = var.allowed_cidrs
  content {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [ingress.value]
  }
}

# for expressions and splat
names = [for i in aws_instance.web : i.tags["Name"]]
ids_by_name = { for i in aws_instance.web : i.tags["Name"] => i.id }
all_ips = aws_instance.web[*].private_ip

# functions: merge, lookup, concat, flatten, cidrsubnet, file, templatefile, jsonencode, try, coalesce
user_data = templatefile("${path.module}/user_data.sh.tpl", { env = var.env })
```

`count` vs `for_each`: with `count`, removing item 0 shifts indexes and can destroy/recreate everything after it. `for_each` keys by name and is safer for lists that change.

### Data sources (read existing things)

```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

data "aws_caller_identity" "current" {}
data "aws_availability_zones" "available" { state = "available" }
```

---

## 3. The EC2 stack (VPC + security group + instance)

Layout:

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── backend.tf
├── user_data.sh.tpl
└── envs/
    ├── dev.tfvars
    └── prod.tfvars
```

```hcl
# versions.tf
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" {
  region = var.region
  default_tags { tags = { Project = "ent-app", Env = var.env, ManagedBy = "terraform" } }
}
```

```hcl
# main.tf
data "aws_availability_zones" "available" { state = "available" }

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023*-x86_64"]
  }
}

# --- Network ---
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags                 = { Name = "${var.env}-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.env}-igw" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, 1)
  availability_zone       = data.aws_availability_zones.available.names[0]
  map_public_ip_on_launch = true
  tags                    = { Name = "${var.env}-public-1" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "${var.env}-public-rt" }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# --- Security group ---
resource "aws_security_group" "web" {
  name_prefix = "${var.env}-web-"
  description = "Web tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS from allowed CIDRs"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidrs
  }
  ingress {
    description = "SSH from admin CIDR only"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.admin_cidrs
  }
  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  lifecycle { create_before_destroy = true }
}

# --- IAM role for SSM (preferred over SSH) ---
resource "aws_iam_role" "ec2" {
  name = "${var.env}-ec2-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Action    = "sts:AssumeRole"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.ec2.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "ec2" {
  name = "${var.env}-ec2-profile"
  role = aws_iam_role.ec2.name
}

# --- Instances ---
resource "aws_instance" "web" {
  count                  = var.instance_count
  ami                    = data.aws_ami.al2023.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2.name
  user_data              = templatefile("${path.module}/user_data.sh.tpl", { env = var.env })

  metadata_options {
    http_tokens   = "required"        # IMDSv2 only (checkov CKV_AWS_79)
    http_endpoint = "enabled"
  }

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true                # checkov CKV_AWS_8 / EBS encryption
  }

  monitoring = true

  tags = {
    Name    = "${var.env}-web-${count.index + 1}"
    Role    = "web"
    Managed = "ansible"               # picked up by Ansible aws_ec2 dynamic inventory
  }

  lifecycle {
    ignore_changes = [ami]            # do not replace instances when a newer AMI is published
  }
}
```

```hcl
# variables.tf
variable "region" {
  type    = string
  default = "eu-west-1"
}

variable "env" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.20.0.0/16"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "allowed_cidrs" {
  type = list(string)
}

variable "admin_cidrs" {
  type = list(string)
}
```

```hcl
# outputs.tf
output "vpc_id"       { value = aws_vpc.main.id }
output "instance_ids" { value = aws_instance.web[*].id }
output "private_ips"  { value = aws_instance.web[*].private_ip }
output "public_ips"   { value = aws_instance.web[*].public_ip }
```

```bash
terraform init
terraform plan  -var-file=envs/dev.tfvars -out=tfplan
terraform apply tfplan
terraform output -json > outputs.json            # hand to Ansible (see section 11)
terraform destroy -var-file=envs/dev.tfvars      # clean up!
```

Free-tier note: `t3.micro`/`t2.micro` eligibility depends on region and account age. Set an AWS Budget alert before you start.

---

## 4. State fundamentals

State holds resource IDs, attributes and **sensitive values in plain text** (passwords, keys, generated secrets). Therefore:

- **Never commit state** (`*.tfstate*`, `.terraform/` in `.gitignore`).
- Store remotely, encrypted, access-controlled, versioned.
- Use **locking** so two applies cannot run at once.
- Treat state access as privileged: anyone who can read state can read secrets in it.

State commands:

```bash
terraform state list
terraform state show aws_instance.web[0]
terraform state mv aws_instance.web aws_instance.app       # rename in state without recreating
terraform state rm aws_instance.legacy                     # stop managing (does NOT destroy)
terraform import aws_instance.web i-0abc123def4567890      # adopt an existing resource
terraform force-unlock <LOCK_ID>                           # only after confirming no run is active
terraform state pull > backup.tfstate                      # always back up before surgery
```

Declarative alternatives (preferred over CLI surgery, reviewable in PRs):

```hcl
moved {                                   # rename/refactor without destroy
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}

import {                                  # config-driven import (Terraform >= 1.5)
  to = aws_instance.legacy
  id = "i-0abc123def4567890"
}
# terraform plan -generate-config-out=generated.tf   (drafts the resource block)

removed {                                 # stop managing without destroying (>= 1.7)
  from = aws_instance.old
  lifecycle { destroy = false }
}
```

---

## 5. S3 remote state with locking

### Bootstrap the backend (once, separately, with local state)

```hcl
# bootstrap/main.tf
resource "aws_s3_bucket" "tfstate" {
  bucket = "ent-tfstate-111122223333-eu-west-1"      # globally unique
  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }     # recover earlier state versions
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.tfstate.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "tfstate" {
  bucket                  = aws_s3_bucket.tfstate.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_kms_key" "tfstate" {
  description         = "Terraform state encryption"
  enable_key_rotation = true
}

# Legacy locking table (still widely used; see note on S3-native locking below)
resource "aws_dynamodb_table" "tflock" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Backend configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "ent-tfstate-111122223333-eu-west-1"
    key            = "enterprise-app/dev/terraform.tfstate"     # one key per stack/environment
    region         = "eu-west-1"
    encrypt        = true
    kms_key_id     = "alias/tfstate"
    dynamodb_table = "terraform-locks"                          # classic locking
    # use_lockfile = true                                       # newer: S3-native locking (no DynamoDB)
  }
}
```

Notes to remember:
- **Locking:** the classic approach uses a **DynamoDB table** with a `LockID` hash key; Terraform writes a lock item before any state-modifying operation. Newer Terraform versions (1.10+) add **S3-native locking** via `use_lockfile = true` (conditional writes), and DynamoDB locking is being deprecated. Know both; check which your Terraform version supports.
- The backend block **cannot use variables**. Pass settings at init time for multiple environments:

```bash
terraform init -backend-config="key=enterprise-app/prod/terraform.tfstate" -reconfigure
terraform init -migrate-state           # move existing local state to the new backend
```

- **Least-privilege IAM** for the pipeline role: `s3:GetObject/PutObject/ListBucket` on the state prefix, `kms:Decrypt/Encrypt/GenerateDataKey`, DynamoDB `GetItem/PutItem/DeleteItem` on the lock table.
- **Versioning + `state pull`** gives you recovery from mistakes. Consider MFA delete or object lock for prod.
- Separate states per environment and per layer (network, compute, data) to limit blast radius and run times.
- Reading another stack's outputs: `data "terraform_remote_state"` (couples stacks) or better, SSM Parameter Store / explicit data sources.

### Locking failure behaviour

```
Error: Error acquiring the state lock
Lock Info: ID, Path, Operation, Who, Created
```

Another run is active (or crashed). Check who/when; if a CI job died, `terraform force-unlock <ID>`. Never force-unlock while someone is mid-apply.

---

## 6. Lifecycle, dependencies, provisioners

```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true                 # replace safely (new first)
    prevent_destroy       = true                 # block accidental destroy (stateful resources)
    ignore_changes        = [ami, tags["LastPatched"]]   # tolerate external/Ansible changes
    replace_triggered_by  = [aws_security_group.web]
    precondition {
      condition     = var.instance_type != "t2.nano"
      error_message = "Instance type too small."
    }
  }
  depends_on = [aws_iam_role_policy_attachment.ssm]      # only when the dependency is not visible by reference
}
```

- **Provisioners** (`remote-exec`, `local-exec`) are a last resort. They are not idempotent, not part of the plan, and fail awkwardly. Use `user_data`/cloud-init for first boot and **Ansible for configuration**.
- `ignore_changes` is how you stop Terraform from fighting Ansible over attributes that Ansible (or AWS autoscaling) legitimately changes.
- `check` blocks and `precondition/postcondition` let you assert assumptions (e.g. the AMI is encrypted).

---

## 7. Modules

A module is a directory of `.tf` files with **inputs (variables)** and **outputs**. The root config calls modules.

```
terraform/
├── modules/
│   └── ec2-stack/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── versions.tf
│       └── README.md
└── envs/
    ├── dev/
    │   ├── main.tf
    │   ├── backend.tf
    │   └── terraform.tfvars
    └── prod/
        ├── main.tf
        ├── backend.tf
        └── terraform.tfvars
```

```hcl
# modules/ec2-stack/variables.tf
variable "env" {
  type = string
}

variable "vpc_cidr" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "allowed_cidrs" {
  type = list(string)
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/ec2-stack/outputs.tf
output "instance_ids" { value = aws_instance.web[*].id }
output "private_ips"  { value = aws_instance.web[*].private_ip }
output "sg_id"        { value = aws_security_group.web.id }
```

```hcl
# envs/prod/main.tf  (the caller)
module "web" {
  source = "../../modules/ec2-stack"          # local path
  # source  = "git::https://github.com/org/tf-modules.git//ec2-stack?ref=v1.3.0"   # versioned, from Git
  # source  = "app.terraform.io/org/ec2-stack/aws"; version = "~> 1.3"             # private registry

  env            = "prod"
  vpc_cidr       = "10.30.0.0/16"
  instance_type  = "m5.large"
  instance_count = 3
  allowed_cidrs  = ["10.0.0.0/8"]
  tags           = { CostCenter = "1234" }
}

output "web_ips" { value = module.web.private_ips }
```

Module design rules:
- **Small, single-purpose, opinionated defaults**, with required inputs for things that must be decided by the caller.
- **Version everything**: Git tag or registry version; never `ref=main` in prod.
- Expose only needed **outputs**; do not leak internals.
- Do not put provider blocks inside reusable modules (pass providers in); keep `required_providers` constraints only.
- Document inputs/outputs (`terraform-docs`), test with `terraform validate`, `tflint`, `checkov`, and `terraform test` (native test framework) or Terratest.
- Prefer composition (network module + compute module) over one giant module.
- Use `moved` blocks when refactoring existing resources into a module so Terraform does not destroy/recreate.

### Native tests (Terraform >= 1.6)

```hcl
# modules/ec2-stack/tests/basic.tftest.hcl
variables {
  env           = "dev"
  vpc_cidr      = "10.99.0.0/16"
  allowed_cidrs = ["10.0.0.0/8"]
}

run "plan_creates_one_instance" {
  command = plan
  assert {
    condition     = length(aws_instance.web) == 1
    error_message = "Expected exactly one instance"
  }
}
```

Run: `terraform test`.

---

## 8. Environments and workspaces

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Directory per env** (recommended) | `envs/dev`, `envs/prod` each with own backend key and tfvars, calling shared modules | Clear, explicit, different versions per env, easy RBAC | Some duplication |
| **Workspaces** | `terraform workspace new prod`, state key `env:/prod/...` | Little duplication | Easy to apply to the wrong workspace; same code/backend for all envs; weak isolation |
| **Terragrunt** | Wrapper to DRY backend/vars | Less boilerplate | Extra tool |

Enterprise default: **directory per environment**, separate state keys (even separate AWS accounts), promotion by changing the module version pin in each env.

---

## 9. Drift detection and management

**Drift** = the real infrastructure no longer matches state/config (manual console change, another tool, autoscaling, Ansible modifying tags).

### Detection

```bash
terraform plan -refresh-only                       # shows what changed outside Terraform
terraform plan -detailed-exitcode -lock=false      # exit code 2 => differences exist
```

Scheduled workflow (full version in the GitHub guide):

```yaml
on:
  schedule: [{ cron: "0 2 * * *" }]
jobs:
  drift:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/gha-terraform-plan
          aws-region: eu-west-1
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_wrapper: false }
      - working-directory: terraform/envs/prod
        run: |
          terraform init -input=false
          set +e
          terraform plan -input=false -detailed-exitcode -lock=false
          code=$?
          if [ $code -eq 2 ]; then echo "::warning::Drift detected"; fi
          exit $([ $code -eq 1 ] && echo 1 || echo 0)
```

### Responding to drift

| Situation | Action |
|---|---|
| Manual change was wrong | `terraform apply` to revert it to code |
| Manual change was right | Update the code to match, then plan to confirm zero diff |
| Attribute is legitimately managed elsewhere (Ansible, autoscaling) | `lifecycle { ignore_changes = [...] }` |
| Resource created outside Terraform | `import` block, then `plan` |
| Resource deleted outside Terraform | `plan` will recreate it (check data loss implications) |
| State out of sync only | `terraform apply -refresh-only` to accept real values into state |

Prevention: restrict console write access (SCPs/IAM), tag-based guardrails, AWS Config rules/CloudTrail alerts for changes to Terraform-managed resources, and pipeline-only applies.

Complement: Ansible drift = `ansible-playbook --check --diff` (configuration inside the OS), Terraform drift = `plan` (cloud resources). Both scheduled, both open a ticket (GitHub issue or ServiceNow incident) when changes are found.

---

## 10. Security and quality gates

```bash
terraform fmt -check -recursive
terraform validate
tflint --init && tflint --recursive
checkov -d . --framework terraform
trivy config .                               # misconfiguration scan (successor to tfsec)
```

Checklist:
- No secrets in `.tf` or `.tfvars` committed to Git. Pass via `TF_VAR_*` from the CI secret store, read from AWS Secrets Manager/SSM with data sources, or generate in AWS and never expose. Remember state still stores them.
- `sensitive = true` on variables/outputs hides CLI output only.
- Encrypt everything (EBS, S3, RDS), IMDSv2 required, no `0.0.0.0/0` on SSH/RDP, least-privilege IAM, mandatory tags (`default_tags`).
- Pin provider versions; commit `.terraform.lock.hcl`; use Dependabot for providers/modules.
- Apply from CI via **OIDC role** (separate plan and apply roles), apply only the saved plan from the merged commit, environment approval for prod.
- Policy as code: **Sentinel** (HCP Terraform), **OPA/Conftest**, or checkov custom policies for rules like "only approved instance types".

CI order: `fmt` → `validate` → `tflint` → `checkov/trivy` → `plan` (comment on PR) → review → merge → `apply` (approval) → post-apply Ansible.

---

## 11. Terraform and Ansible: how they complement each other

### Division of labour

| Concern | Terraform | Ansible |
|---|---|---|
| Model | Declarative, state-based | Procedural/declarative tasks, stateless (facts at runtime) |
| Best for | Cloud resources: VPC, subnets, SG, EC2, RDS, IAM, S3, DNS | OS and app config: packages, files, services, users, hardening, deploys, patching |
| Lifecycle | Create, update, destroy with a plan and a state file | Run to converge; ordering and rolling updates (`serial`) |
| Drift | `plan` shows differences from state | `--check --diff`; every run reconverges |
| Orchestration | Dependency graph | Cross-system workflows, ServiceNow/ITSM calls, remediation |
| Weakness | Poor at in-OS config and multi-step procedures | Weak at tracking and safely destroying cloud resources (no state) |

One line for interviews: *"Terraform builds the house, Ansible furnishes it and runs the operations."* Ansible can also provision cloud resources (`amazon.aws.ec2_instance`) and is fine for small/ad hoc cases or when the organisation standardised on it. Choose by team standards, but avoid managing the **same resource** with both tools.

### Integration patterns

**1. Terraform outputs → Ansible inventory (static handoff)**

```bash
terraform output -json private_ips | jq -r '.value[]' > hosts.txt
```

```hcl
# Generate an inventory file from Terraform
resource "local_file" "ansible_inventory" {
  filename = "${path.module}/inventory.ini"
  content  = templatefile("${path.module}/inventory.tpl", {
    web_ips = aws_instance.web[*].private_ip
  })
}
```

**2. Tag-based dynamic inventory (preferred, decoupled)**

Terraform tags instances (`Managed = "ansible"`, `Role = "web"`, `Env = "dev"`); Ansible's `amazon.aws.aws_ec2` plugin builds groups from tags. No files to pass around; new instances appear automatically.

**3. Terraform calls Ansible after apply (CI pipeline)**

```yaml
jobs:
  infra:
    # terraform apply ...
  configure:
    needs: infra
    steps:
      - run: ansible-playbook -i inventories/aws_ec2.yml site.yml --limit "env_${{ inputs.env }}"
```

Or from **AAP**: Terraform output triggers a job template (webhook/API), or AAP's inventory sync picks up the new hosts, then a workflow template configures them.

**4. Ansible runs Terraform** (`cloud.terraform.terraform` module in the `cloud.terraform` collection)

```yaml
- name: Apply Terraform from an Ansible workflow
  cloud.terraform.terraform:
    project_path: "{{ playbook_dir }}/../terraform/envs/{{ env }}"
    state: present
    force_init: true
    variables:
      env: "{{ env }}"
      instance_count: "{{ instance_count | int }}"
    backend_config:
      key: "enterprise-app/{{ env }}/terraform.tfstate"
  register: tf
- ansible.builtin.debug:
    msg: "{{ tf.outputs.private_ips.value }}"
```

Use this when ServiceNow requests drive everything through AAP (catalog → AAP workflow → Terraform → Ansible config → CMDB write-back) and the platform team wants one orchestration entry point.

**5. First-boot handoff:** `user_data` installs the SSM agent or `ansible-pull`; Ansible then configures via SSM (`community.aws.aws_ssm` connection) without SSH keys.

### End-to-end flow (ties to your portfolio project)

```
ServiceNow catalog (approve)
   → AAP workflow
      → Step 1: Terraform apply (VPC/SG/EC2, tags)        [infrastructure]
      → Step 2: Ansible roles (nginx, hardening, agents)   [configuration]
      → Step 3: servicenow.itsm: CI via IRE + CR work notes [record and audit]
   → on failure: rescue → incident, terraform destroy of partial stack
Nightly: terraform plan -detailed-exitcode + ansible --check → drift tickets
Alert: EDA rulebook → remediation playbook (restart / re-provision)
```

Design decisions to be ready to defend: why Terraform for infrastructure (state, plan, safe destroy), why Ansible for configuration (agentless, orchestration, ITSM integration), how state and secrets are protected, how drift is detected and who owns remediation, how audit evidence is produced.

---

## 12. Common errors and troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Error acquiring the state lock` | Another run active or crashed run; verify, then `force-unlock` |
| `Backend configuration changed` | Run `terraform init -reconfigure` or `-migrate-state` |
| `Provider produced inconsistent result` | Eventual consistency or provider bug; retry; upgrade provider |
| Resource shows `forces replacement` unexpectedly | Changed an immutable attribute (AMI, subnet); use `ignore_changes` or accept a planned replacement |
| `Cycle` error | Circular references; split resources or use separate SG rule resources (`aws_security_group_rule` / `aws_vpc_security_group_ingress_rule`) |
| `InvalidClientTokenId` / `ExpiredToken` | Credentials missing/expired; check OIDC role and `aws sts get-caller-identity` |
| `count`/`for_each` value depends on unknown values | Keys must be known at plan time; derive from inputs not resource attributes |
| Plan wants to destroy and recreate everything after removing one list item | `count` index shift; switch to `for_each` and use `moved` blocks |
| State drift after Ansible changed tags | Add `ignore_changes` for those tags |
| `Error: Unsupported argument` after upgrade | Provider major version change; read the upgrade guide, pin versions |
| Lock table / bucket access denied | Pipeline role lacks S3/KMS/DynamoDB permissions |

Debug: `TF_LOG=DEBUG terraform plan 2> debug.log` (can leak secrets, clean up after), `terraform console`, `terraform graph`.

---

## 13. Interview quick-fire

1. **Why remote state with locking?** Shared team source of truth, encryption and versioning, and locking prevents two concurrent applies from corrupting state. S3 holds state; DynamoDB (or S3-native lockfile on newer versions) provides the lock.
2. **Terraform vs Ansible?** Terraform is stateful and declarative for provisioning cloud resources with a plan/apply cycle; Ansible is agentless and procedural-or-declarative for configuration and orchestration. Use Terraform for infrastructure, Ansible for what runs inside it and for operational workflows.
3. **What is in state and why is it sensitive?** Resource IDs and all attributes, including secrets Terraform handled; so encrypt, restrict access, never commit it.
4. **`count` vs `for_each`?** `count` is index-based, so removing an item shifts others and can force replacements; `for_each` keys by value and is stable.
5. **How do you handle drift?** Scheduled `plan -detailed-exitcode` (2 = changes) or `-refresh-only`; decide revert-to-code vs update-code vs `ignore_changes`; restrict console access to prevent it.
6. **How do you refactor without recreating resources?** `moved` blocks (or `state mv`), `import` blocks to adopt existing resources, `removed` blocks to stop managing.
7. **How do you structure environments?** Directory per environment, separate state keys/accounts, shared versioned modules, promotion by bumping module versions.
8. **How do you pass infra details to Ansible?** Prefer tag-based dynamic inventory; otherwise Terraform outputs or generated inventory; in AAP use inventory sources and workflow templates.
9. **How do you keep credentials out of Terraform pipelines?** OIDC to an assumable role, separate plan/apply roles, no static keys; secrets from a secret manager, never in `.tfvars` in Git.
10. **What does `terraform plan -detailed-exitcode` return?** 0 no changes, 1 error, 2 changes present.
11. **What is a module and how do you version it?** A reusable directory with inputs/outputs; consumed by Git tag or registry version so changes roll out deliberately.
12. **What would you do if two engineers' applies collide?** The lock blocks the second; wait, or investigate a stale lock and `force-unlock` only after confirming nothing is running.

---

## 14. Revision checklist

- [ ] Write the EC2 stack (VPC, subnet, IGW, route table, SG, IAM/SSM, instance with IMDSv2 and encrypted root) and apply it in a scratch account
- [ ] Destroy it and confirm in the console nothing remains (cost control)
- [ ] Create the S3 bucket (versioning, KMS, public access block) and lock table in a bootstrap stack
- [ ] Migrate local state to S3: `terraform init -migrate-state`; run two applies at once to see the lock error
- [ ] Refactor the stack into `modules/ec2-stack` with variables/outputs; use `moved` blocks so the plan shows no destroys
- [ ] Create `envs/dev` and `envs/prod` with separate state keys and tfvars
- [ ] Import a manually created resource using an `import` block
- [ ] Change a tag in the console; detect with `plan -refresh-only` and `-detailed-exitcode`; fix it three different ways
- [ ] Add `tflint`, `checkov`, `trivy config` and `terraform test` to the CI workflow
- [ ] Tag instances (`Managed=ansible`, `Env`, `Role`) and configure them with Ansible via `aws_ec2` dynamic inventory
- [ ] Run Terraform from an Ansible playbook with `cloud.terraform.terraform`
- [ ] Explain "Terraform builds, Ansible configures" out loud with the end-to-end ServiceNow → AAP → Terraform → Ansible → CMDB flow
