# GitHub and CI/CD Revision Guide (GitOps for Automation Engineers)

Scope: Git workflow, branch protection and rulesets, PR workflow, CODEOWNERS, GitHub Actions (syntax, matrix, caching, artifacts, environments, secrets), reusable workflows, OIDC to AWS (zero static credentials), lint and scan gates (ansible-lint, yamllint, tflint, checkov, trivy, secret scanning), Molecule in CI, drift detection, release and promotion patterns.

How to use: create a scratch repo and build the pipeline step by step. Every section ends in something you can run.

> Verify action versions (`@v4` etc.) and UI labels against current GitHub docs; they change. The concepts are stable.

---

## 0. Mental model: GitOps for automation

- **Git is the source of truth.** Playbooks, roles, Terraform, workflows, even AAP config live in a repo. Nothing is changed by hand in prod.
- **Change flow:** branch → commit → pull request → automated checks → human review → merge to `main` → deploy/release.
- **`main` is protected:** no direct pushes, required checks, required reviews.
- **Pipelines have no static secrets:** short-lived credentials via OIDC.
- **Evidence for audit (GxP):** PR approvals, check results, signed commits, workflow run logs, release tags are all retained and linkable to a change request.

---

## 1. Git essentials you must have in muscle memory

```bash
git clone git@github.com:org/repo.git
git switch -c feature/add-nginx-role          # new branch (old: git checkout -b)
git status; git diff; git diff --staged
git add -p                                    # stage hunks interactively
git commit -m "feat(nginx): add reload handler"
git fetch origin && git rebase origin/main    # keep branch up to date, linear history
git push -u origin feature/add-nginx-role
git log --oneline --graph --decorate -20
git stash; git stash pop
git restore path/file                         # discard working-tree changes
git revert <sha>                              # safe undo on shared branches (new commit)
git reset --hard <sha>                        # LOCAL ONLY; never on shared branches
git cherry-pick <sha>
git tag -a v1.2.0 -m "Release 1.2.0" && git push origin v1.2.0
git bisect start; git bisect bad; git bisect good <sha>   # find the commit that broke it
git reflog                                    # recover "lost" commits
```

Concepts:
- **Merge vs rebase vs squash:** merge keeps history with a merge commit; rebase rewrites your branch onto the latest base (linear); squash collapses the PR into one commit on `main`. Many enterprises pick **squash merge** for a clean history, one PR = one commit = one change.
- **Conventional commits** (`feat:`, `fix:`, `chore:`, `docs:`) enable changelogs and semantic versioning automation.
- **`.gitignore`:** `*.retry`, `.vault_pass`, `.terraform/`, `*.tfstate*`, `collections/ansible_collections/`, `.env`, `__pycache__/`, `.venv/`.
- **Never commit secrets.** If it happens: rotate the secret first, then clean history (`git filter-repo`), not the other way round.

---

## 2. Repository layout (automation repo)

```
automation-repo/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                 # PR checks
│   │   ├── deploy.yml             # on merge/tag, OIDC to AWS
│   │   ├── drift.yml              # scheduled terraform plan
│   │   └── reusable-lint.yml      # callable workflow
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   └── dependabot.yml
├── ansible/                       # playbooks, roles, inventories, requirements.yml
├── terraform/                     # modules, envs
├── scripts/                       # python
├── .ansible-lint
├── .yamllint
├── .pre-commit-config.yaml
└── README.md
```

### Pull request template

```markdown
<!-- .github/pull_request_template.md -->
## What and why
## Change request / ticket
CHG0000000 / JIRA-123
## How was it tested
- [ ] ansible-lint / molecule
- [ ] terraform plan reviewed
- [ ] Rollback plan described
## Risk
- [ ] Low  - [ ] Medium  - [ ] High
```

### CODEOWNERS (required reviewers by path)

```
# .github/CODEOWNERS
*                       @org/automation-team
/terraform/             @org/cloud-platform @org/security
/ansible/roles/nginx/   @org/web-team
/.github/workflows/     @org/devops-leads
```

### Dependabot (keep actions and pip packages patched)

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
  - package-ecosystem: pip
    directory: /
    schedule: { interval: weekly }
  - package-ecosystem: terraform
    directory: /terraform
    schedule: { interval: weekly }
```

### pre-commit (catch issues before the PR)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: detect-private-key
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks: [{ id: yamllint }]
  - repo: https://github.com/ansible/ansible-lint
    rev: v24.9.2
    hooks: [{ id: ansible-lint }]
```

```bash
pip install pre-commit && pre-commit install
```

---

## 3. Branch protection and rulesets

Settings → Branches → **Branch protection rules** (or the newer **Rulesets**, which can apply org-wide and support more targets). For `main`:

| Setting | Why |
|---|---|
| **Require a pull request before merging** | No direct pushes |
| **Required approvals: 1-2** | Four-eyes principle (a GxP expectation) |
| **Dismiss stale approvals on new commits** | Approval applies to what was reviewed |
| **Require review from Code Owners** | Right people approve the right paths |
| **Require status checks to pass** (select `lint`, `test`, `scan`, `plan`) | Automated quality gate |
| **Require branches to be up to date before merging** | Checks ran against the latest `main` |
| **Require conversation resolution** | No unresolved review comments |
| **Require signed commits** | Authenticity (GPG/SSH signing) |
| **Require linear history** | Squash/rebase only |
| **Do not allow bypassing / include administrators** | Admins follow the same rules |
| **Restrict force pushes and deletions** | Protects history |
| **Merge queue** (optional) | Serialises merges, tests each combined result |

Gotchas:
- Required checks are matched **by job name**; a renamed job means a stuck PR. Skipped jobs (path filters) can also block required checks; use a final "gate" job that always runs (see section 5).
- Enable **environment protection rules** for deploy approvals (section 8).
- Repository rules apply per branch pattern: protect `main` and `release/*`.

Configure with the CLI/API (config as code), e.g.:

```bash
gh api -X PUT repos/ORG/REPO/branches/main/protection --input - <<'JSON'
{
  "required_status_checks": { "strict": true, "contexts": ["lint", "test", "scan"] },
  "enforce_admins": true,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": true
  },
  "restrictions": null,
  "required_linear_history": true,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "required_conversation_resolution": true
}
JSON
```

---

## 4. The PR workflow (what a reviewer expects)

1. Create an issue/ticket, link it to a change request.
2. Branch from latest `main`: `feature/...`, `fix/...`.
3. Small commits, clear messages, small PRs (under ~400 lines is reviewable).
4. Open the PR (draft first if early). Fill the template.
5. CI runs: lint, test, scan, `terraform plan` posted as a comment.
6. Reviewers check logic, security, idempotency, rollback, variable naming, secrets.
7. Address comments with new commits (no force-push during review unless agreed).
8. **Squash and merge** after approvals and green checks.
9. Pipeline deploys (dev automatically; prod behind an approval).
10. Tag and release notes; update the change record.

```bash
gh pr create --fill --draft
gh pr checks --watch
gh pr view --web
gh pr merge --squash --delete-branch
```

---

## 5. GitHub Actions: syntax and patterns

### Anatomy

```yaml
name: ci
on:
  pull_request:
    branches: [main]
    paths: ["ansible/**", "terraform/**", ".github/workflows/**"]
  push:
    branches: [main]
  workflow_dispatch:                         # manual run button
    inputs:
      env: { description: Target env, type: choice, options: [dev, test, prod], default: dev }

permissions:                                 # least privilege for GITHUB_TOKEN
  contents: read

concurrency:                                 # cancel superseded runs on the same PR
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  ANSIBLE_FORCE_COLOR: "1"

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install ansible-core ansible-lint yamllint
      - run: yamllint .
      - run: ansible-lint
```

Key vocabulary:
- **Workflow** (YAML file) → **jobs** (run in parallel by default, `needs:` creates order) → **steps** (`uses:` an action or `run:` a command).
- **Triggers:** `push`, `pull_request`, `workflow_dispatch`, `schedule` (cron, UTC), `workflow_call` (reusable), `release`, `repository_dispatch` (external webhook, e.g. from ServiceNow), `pull_request_target` (dangerous: runs with secrets from the base branch; avoid on untrusted PRs).
- **Contexts:** `github.*`, `env.*`, `secrets.*`, `vars.*`, `matrix.*`, `needs.*`, `steps.<id>.outputs.*`.
- **Expressions:** `if: github.ref == 'refs/heads/main' && github.event_name == 'push'`, `${{ fromJSON(...) }}`, `success()`, `failure()`, `always()`.
- **Runners:** GitHub-hosted (`ubuntu-latest`) or **self-hosted** (inside your network to reach private AAP/ServiceNow; harden them and never use for public repos).

### Matrix

```yaml
  molecule:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        role: [nginx, common, hardening]
        distro: [ubuntu2204, rockylinux9]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install molecule "molecule-plugins[docker]" ansible-core
      - run: molecule test
        working-directory: ansible/roles/${{ matrix.role }}
        env:
          MOLECULE_DISTRO: ${{ matrix.distro }}
```

### Passing data between steps and jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.ver.outputs.version }}
    steps:
      - id: ver
        run: echo "version=1.2.${{ github.run_number }}" >> "$GITHUB_OUTPUT"
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.version }}"
```

### Artifacts and caching

```yaml
      - uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: terraform/tfplan
          retention-days: 7
      - uses: actions/download-artifact@v4
        with: { name: tfplan }
      - uses: actions/cache@v4
        with:
          path: ~/.ansible/collections
          key: collections-${{ hashFiles('ansible/requirements.yml') }}
```

### Gate job for required checks (avoids skipped-check deadlocks)

```yaml
  ci-gate:
    if: always()
    needs: [lint, molecule, scan]
    runs-on: ubuntu-latest
    steps:
      - run: |
          if [[ "${{ contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') }}" == "true" ]]; then
            echo "A required job failed"; exit 1; fi
```

Mark only `ci-gate` as the required status check.

### Workflow security rules

- Set `permissions:` explicitly (default to `contents: read`); add `id-token: write` only on jobs needing OIDC.
- **Pin third-party actions to a commit SHA** (`uses: org/action@<40-char-sha>`) rather than a mutable tag. Dependabot keeps them updated.
- Never interpolate untrusted input directly into `run:` (script injection). Pass through `env:`:

```yaml
      - env:
          TITLE: ${{ github.event.pull_request.title }}      # safe
        run: echo "$TITLE"                                     # NOT: echo "${{ github.event.pull_request.title }}"
```

- Secrets are masked in logs but can still leak through `echo $SECRET | base64` or artifacts. Do not print them.
- Fork PRs do not get secrets (good). Do not "fix" that with `pull_request_target`.

---

## 6. Reusable workflows and composite actions

### Reusable workflow (callee)

```yaml
# .github/workflows/reusable-lint.yml  (can live in a central repo: org/shared-workflows)
name: reusable-lint
on:
  workflow_call:
    inputs:
      python-version: { type: string, default: "3.12" }
      path:           { type: string, default: "." }
    secrets: {}                                 # none needed here
    outputs:
      result:
        value: ${{ jobs.lint.outputs.result }}

jobs:
  lint:
    runs-on: ubuntu-latest
    outputs:
      result: ${{ steps.done.outputs.result }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "${{ inputs.python-version }}", cache: pip }
      - run: pip install ansible-core ansible-lint yamllint
      - run: yamllint ${{ inputs.path }}
      - run: ansible-lint ${{ inputs.path }}
      - id: done
        run: echo "result=ok" >> "$GITHUB_OUTPUT"
```

### Caller

```yaml
# .github/workflows/ci.yml
name: ci
on: [pull_request]
permissions: { contents: read }
jobs:
  lint:
    uses: org/shared-workflows/.github/workflows/reusable-lint.yml@v1      # pin a tag/SHA
    with:
      path: ansible
  deploy-dev:
    needs: lint
    uses: ./.github/workflows/reusable-deploy.yml
    with: { env: dev }
    secrets: inherit                          # or pass explicitly
```

Facts:
- Reusable workflow = **a whole job set**, called with `uses:` at the **job level**. Composite action = **a bundle of steps**, used with `uses:` at the **step level** (`action.yml` with `runs: using: composite`).
- Limits: nesting depth is capped (currently 4 levels); `env` context from the caller is not inherited, pass inputs.
- Why: **standardise** pipelines across dozens of repos (one change to the shared workflow updates all, with version pinning for controlled rollout). This is the answer to "how do you make 20 teams automate consistently".

### Composite action

```yaml
# .github/actions/setup-ansible/action.yml
name: setup-ansible
description: Install ansible-core and collections
inputs:
  requirements: { default: ansible/requirements.yml }
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v5
      with: { python-version: "3.12", cache: pip }
    - shell: bash
      run: |
        pip install ansible-core ansible-lint
        ansible-galaxy collection install -r "${{ inputs.requirements }}"
```

Use: `- uses: ./.github/actions/setup-ansible`

---

## 7. Lint and scan gates

### Gate summary

| Gate | Tool | Catches |
|---|---|---|
| YAML syntax/style | `yamllint` | Indentation, truthy values, long lines |
| Ansible quality | `ansible-lint` (profile `production`) | FQCN, missing `name`, risky shell, permissions, idempotency smells |
| Ansible tests | `molecule` | Role behaviour, idempotence |
| Terraform format/validate | `terraform fmt -check`, `terraform validate` | Syntax errors |
| Terraform lint | `tflint` | Provider-specific mistakes (bad instance types, deprecated args) |
| IaC security | `checkov` (also `tfsec`/`trivy config`) | Open security groups, unencrypted volumes, public S3 |
| Container/filesystem | `trivy fs` / `trivy image` | CVEs in dependencies and images, misconfigurations, secrets |
| Secrets | GitHub **secret scanning + push protection**, `gitleaks` | Committed credentials |
| Dependencies | **Dependabot**, **dependency review action** | Vulnerable/outdated packages |
| Code analysis | **CodeQL** (Python/JS) | Injection, unsafe patterns |
| Python | `ruff`, `pytest`, `bandit` | Style, tests, insecure calls |

### A full CI workflow with gates

```yaml
name: ci
on:
  pull_request:
    branches: [main]
permissions:
  contents: read
  security-events: write       # to upload SARIF to the Security tab
  pull-requests: write         # to comment the plan
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install ansible-core ansible-lint yamllint ruff
      - run: ansible-galaxy collection install -r ansible/requirements.yml
      - run: yamllint .
      - run: ansible-lint ansible/
      - run: ruff check scripts/

  python-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install -r scripts/requirements.txt pytest
      - run: pytest -q scripts/

  terraform-checks:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: terraform } }
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform fmt -check -recursive
      - run: terraform init -backend=false
      - run: terraform validate
      - uses: terraform-linters/setup-tflint@v4
      - run: tflint --init && tflint --recursive

  checkov:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bridgecrewio/checkov-action@v12
        with:
          directory: terraform
          framework: terraform
          soft_fail: false                      # fail the PR on findings
          output_format: cli,sarif
          output_file_path: console,checkov.sarif
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with: { sarif_file: checkov.sarif }

  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: fs
          scan-ref: .
          scanners: vuln,secret,misconfig
          severity: HIGH,CRITICAL
          exit-code: "1"                        # fail on findings
          ignore-unfixed: true
          format: sarif
          output: trivy.sarif
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with: { sarif_file: trivy.sarif }

  molecule:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        role: [nginx, common]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install molecule "molecule-plugins[docker]" ansible-core
      - run: molecule test
        working-directory: ansible/roles/${{ matrix.role }}

  ci-gate:
    if: always()
    needs: [lint, python-tests, terraform-checks, checkov, trivy, molecule]
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo '${{ toJSON(needs) }}'
          [[ "${{ contains(needs.*.result, 'failure') }}" == "false" ]]
```

### Tuning findings (do not just disable gates)

```yaml
# .ansible-lint
profile: production
skip_list: []                # keep empty; justify any skip in a comment
warn_list: [experimental]
```

```hcl
# Terraform: inline checkov suppression with a reason (reviewable in PR)
resource "aws_security_group_rule" "egress_all" {
  # checkov:skip=CKV_AWS_23: description is added by the module caller
}
```

```text
# .trivyignore  (CVE with accepted risk, add expiry/comment)
CVE-2024-12345   # no fix available, mitigated by network policy, review 2026-01-01
```

Common checkov findings to know: `CKV_AWS_24` SSH open to 0.0.0.0/0, `CKV_AWS_8` unencrypted launch config volumes, `CKV_AWS_79` IMDSv1 enabled (set `metadata_options { http_tokens = "required" }`), `CKV_AWS_135` EBS optimisation, `CKV2_AWS_*` graph checks.

### Terraform plan on PR as a comment

```yaml
  plan:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read, pull-requests: write }
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/gha-terraform-plan   # read-only role
          aws-region: eu-west-1
      - uses: hashicorp/setup-terraform@v3
      - working-directory: terraform/envs/dev
        run: |
          terraform init -input=false
          terraform plan -input=false -no-color -out=tfplan | tee plan.txt
      - uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('terraform/envs/dev/plan.txt', 'utf8').slice(-60000);
            github.rest.issues.createComment({
              owner: context.repo.owner, repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '#### Terraform plan\n```\n' + plan + '\n```'
            });
```

(Use a **read-only** role for the PR plan job, and a separate role for apply.)

---

## 8. OIDC to AWS (zero static credentials)

### Why

Static `AWS_ACCESS_KEY_ID/SECRET` stored in GitHub secrets can leak, never rotate, and give long-lived access. With **OIDC**, GitHub issues a short-lived signed JWT per job; AWS verifies it and returns temporary credentials for a role. No stored keys; trust is limited to a repo/branch/environment.

### Flow

```
GitHub job ──(id-token: write)──► GitHub OIDC provider  ──► JWT (iss, sub, aud, ref, repo, environment)
GitHub job ──► AWS STS AssumeRoleWithWebIdentity (JWT) ──► IAM checks trust policy conditions
STS ──► temporary credentials (1 hour default) ──► job uses AWS API
```

### Step 1: create the OIDC identity provider in AWS (once per account)

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  # thumbprint_list is no longer required by AWS for GitHub's provider on recent provider versions;
  # keep one if your Terraform provider version still demands it.
}
```

### Step 2: IAM role with a tightly scoped trust policy

```hcl
data "aws_iam_policy_document" "trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      # ONLY this repo and ONLY the prod environment (or ref:refs/heads/main)
      values   = ["repo:my-org/automation-repo:environment:prod"]
    }
  }
}

resource "aws_iam_role" "gha_deploy" {
  name               = "gha-terraform-apply"
  assume_role_policy = data.aws_iam_policy_document.trust.json
  max_session_duration = 3600
}
# attach a least-privilege policy (not AdministratorAccess)
```

**The `sub` condition is the security control.** A wildcard like `repo:my-org/*` lets any repo in the org assume the role. Typical `sub` values:

| Scenario | `sub` claim |
|---|---|
| Push to a branch | `repo:ORG/REPO:ref:refs/heads/main` |
| Pull request | `repo:ORG/REPO:pull_request` |
| Job using an environment | `repo:ORG/REPO:environment:prod` |
| Tag | `repo:ORG/REPO:ref:refs/tags/v1.0.0` |

Use separate roles: **plan/read-only** (trusted for PRs) and **apply** (trusted only for `environment:prod` with required reviewers).

### Step 3: use it in the workflow

```yaml
name: deploy
on:
  push:
    branches: [main]

permissions:
  id-token: write          # REQUIRED to request the OIDC token
  contents: read

jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment: prod       # triggers required reviewers + environment-scoped trust
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/gha-terraform-apply
          role-session-name: gha-${{ github.run_id }}
          aws-region: eu-west-1
      - run: aws sts get-caller-identity           # prove who you are
      - uses: hashicorp/setup-terraform@v3
      - working-directory: terraform/envs/prod
        run: |
          terraform init -input=false
          terraform apply -input=false -auto-approve
```

### Environments and approvals

Settings → Environments → `prod`: **required reviewers**, **wait timer**, **deployment branches** (only `main`), **environment secrets/variables**. The job pauses until approved; the approval is audited. Pair it with a ServiceNow change request reference in the run name/inputs for GxP traceability.

### Troubleshooting OIDC

| Error | Cause |
|---|---|
| `Credentials could not be loaded` / `Not authorized to perform sts:AssumeRoleWithWebIdentity` | `sub` mismatch (wrong branch/environment), wrong provider ARN, wrong `aud` |
| `Could not get OIDC token` | Missing `permissions: id-token: write` |
| Works on `main`, fails on PR | Trust policy only allows branch/environment, not `pull_request` |
| Fork PRs fail | Forks do not receive OIDC tokens (by design) |

Debug by decoding the token claims or printing `${{ github.sub }}`-equivalent values (never the token). Check CloudTrail `AssumeRoleWithWebIdentity` events.

---

## 9. Drift detection (scheduled Terraform plan)

Drift = real infrastructure differs from code/state (someone changed a security group in the console). Detect it nightly and alert.

```yaml
# .github/workflows/drift.yml
name: drift-detection
on:
  schedule:
    - cron: "0 2 * * *"            # 02:00 UTC daily (schedules run from the default branch)
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
  issues: write

jobs:
  drift:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        env: [dev, prod]
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/gha-terraform-plan   # read-only
          aws-region: eu-west-1
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_wrapper: false }          # needed to read the real exit code
      - name: Plan
        id: plan
        working-directory: terraform/envs/${{ matrix.env }}
        run: |
          terraform init -input=false
          set +e
          terraform plan -input=false -detailed-exitcode -lock=false -no-color | tee plan.txt
          echo "exitcode=${PIPESTATUS[0]}" >> "$GITHUB_OUTPUT"
      - name: Open or update drift issue
        if: steps.plan.outputs.exitcode == '2'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const body = fs.readFileSync('terraform/envs/${{ matrix.env }}/plan.txt','utf8').slice(-60000);
            const title = 'Drift detected: ${{ matrix.env }}';
            const { data: open } = await github.rest.issues.listForRepo({
              ...context.repo, state: 'open', labels: 'drift' });
            const existing = open.find(i => i.title === title);
            const text = 'Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\n```\n' + body + '\n```';
            if (existing) {
              await github.rest.issues.createComment({ ...context.repo, issue_number: existing.number, body: text });
            } else {
              await github.rest.issues.create({ ...context.repo, title, labels: ['drift'], body: text });
            }
      - name: Fail on error (exit code 1)
        if: steps.plan.outputs.exitcode == '1'
        run: exit 1
```

Terraform `plan -detailed-exitcode`: **0** = no changes, **1** = error, **2** = changes present (drift or pending code changes). Notes:
- `-lock=false` for a read-only drift plan avoids blocking real applies (accept the small race).
- Alert targets: GitHub issue, Slack webhook, or **open a ServiceNow incident** via REST/`servicenow.itsm`; for auto-remediation use a protected workflow that runs `terraform apply` after approval.
- Ansible drift: run `ansible-playbook site.yml --check --diff` on a schedule; any `changed>0` means drift from the desired state.
- Scheduled workflows are disabled after 60 days of repo inactivity (public repos), run on the default branch only, and may be delayed under load.

```yaml
      - name: Ansible drift check
        run: ansible-playbook ansible/site.yml -i ansible/inventories/prod --check --diff
```

---

## 10. Deployment and promotion patterns

### Build once, promote through environments

```yaml
on:
  push:
    tags: ["v*"]
jobs:
  dev:
    uses: ./.github/workflows/reusable-deploy.yml
    with: { env: dev }
  test:
    needs: dev
    uses: ./.github/workflows/reusable-deploy.yml
    with: { env: test }
  prod:
    needs: test
    uses: ./.github/workflows/reusable-deploy.yml
    with: { env: prod }        # environment 'prod' has required reviewers
```

### Triggering AAP or ServiceNow from Actions

```yaml
      - name: Launch AAP job template
        env:
          AAP_URL: ${{ vars.AAP_URL }}
          AAP_TOKEN: ${{ secrets.AAP_TOKEN }}
        run: |
          curl -fsS -X POST "$AAP_URL/api/controller/v2/job_templates/42/launch/" \
            -H "Authorization: Bearer $AAP_TOKEN" -H "Content-Type: application/json" \
            -d '{"extra_vars": {"git_sha": "${{ github.sha }}"}}'
```

(Reaching a private AAP needs a **self-hosted runner** inside the network.)

### Inbound trigger from ServiceNow (repository_dispatch)

```bash
curl -X POST https://api.github.com/repos/ORG/REPO/dispatches \
  -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github+json" \
  -d '{"event_type":"provision-request","client_payload":{"server_name":"app01","env":"dev","chg":"CHG0030001"}}'
```

```yaml
on:
  repository_dispatch:
    types: [provision-request]
jobs:
  provision:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Provision ${{ github.event.client_payload.server_name }} for ${{ github.event.client_payload.chg }}"
```

Use a fine-scoped token (fine-grained PAT or GitHub App) stored in ServiceNow's credential store.

### Releases and versioning

- Tag with semantic versions (`v1.4.2`); create a GitHub Release with auto-generated notes (`gh release create v1.4.2 --generate-notes`).
- Version roles/collections and execution environment images; pin versions in `requirements.yml` and AAP.
- Publish an EE image from a workflow (`ansible-builder build`, `docker/login-action`, `docker/build-push-action`) and scan it with `trivy image` before push.

---

## 11. Debugging workflows

- Re-run with debug logging: set repository secret/variable `ACTIONS_STEP_DEBUG=true`.
- `gh run list`, `gh run view <id> --log-failed`, `gh run rerun <id> --failed`.
- Run locally with **act** (`act pull_request`) for quick iteration (not 100% faithful).
- Validate syntax with `actionlint` (add it to pre-commit): catches bad expressions, wrong `needs`, deprecated syntax.
- Common failures: indentation, wrong `working-directory`, missing `permissions`, required check name mismatch, path filters causing skipped required checks, cache key never changing, rate limits on `gh api`.
- Add `timeout-minutes` and `concurrency` to every workflow to avoid runaway or overlapping jobs.

---

## 12. Interview quick-fire

1. **How do you protect `main`?** PR required, 1-2 approvals incl. code owners, required status checks (lint, test, scan, plan) with up-to-date branches, signed commits, linear history, admins included, no force pushes.
2. **OIDC vs stored AWS keys?** OIDC exchanges a short-lived GitHub-signed JWT for temporary AWS credentials; trust is restricted by the `sub` claim (repo, branch, environment). Nothing to leak or rotate. Needs `id-token: write`.
3. **Reusable workflow vs composite action?** Reusable = whole jobs, called at job level via `workflow_call`; composite = a bundle of steps used inside a job. Use reusable workflows to standardise pipelines across repos.
4. **How do you secure Actions?** Least-privilege `permissions`, pin actions by SHA, avoid `pull_request_target`, no untrusted input in `run:`, environment protection, secret scanning/push protection, Dependabot for actions.
5. **What do your PR gates run?** yamllint, ansible-lint (production profile), Molecule, terraform fmt/validate/tflint, checkov, trivy fs, secret scan, Python tests, and a plan comment; one final gate job is the required check.
6. **How do you detect drift?** Nightly `terraform plan -detailed-exitcode` (exit 2 = drift) with a read-only OIDC role, open an issue/incident automatically; `ansible-playbook --check --diff` for configuration drift.
7. **How is this audit-ready for GxP?** Approved PRs tied to a change record, immutable run logs, environment approvals, signed commits, tagged releases, and execution evidence written into the change work notes.
8. **Merge vs squash vs rebase?** Merge preserves all commits, squash makes one commit per PR (clean history, easy revert), rebase keeps a linear history but rewrites commits.
9. **Where do secrets live?** Not in the repo. Environment-scoped GitHub secrets for unavoidable tokens, OIDC for cloud, AAP credentials/secret managers for runtime, and `no_log` in Ansible.
10. **Runner choice?** GitHub-hosted for public/cloud targets; self-hosted, ephemeral and hardened, when jobs must reach private networks (AAP, ServiceNow MID, on-prem).

---

## 13. Revision checklist

- [ ] Create a repo, protect `main` (PR, approvals, required checks, linear history), prove you cannot push directly
- [ ] Add CODEOWNERS, a PR template, Dependabot, and pre-commit with yamllint and ansible-lint
- [ ] CI workflow: lint + matrix Molecule + Python tests, with `concurrency` and `permissions`
- [ ] Add checkov and trivy with SARIF upload; fix or justify at least one finding
- [ ] Add a `ci-gate` job and make only that the required check
- [ ] Build a reusable workflow in a central repo and call it from two repos
- [ ] Create the AWS OIDC provider and a role with a `sub`-scoped trust policy; run `aws sts get-caller-identity` from Actions
- [ ] Use separate read-only (plan) and apply roles; add an environment with required reviewers
- [ ] Post `terraform plan` as a PR comment
- [ ] Nightly drift workflow: introduce drift by hand in the console, confirm exit code 2 creates an issue
- [ ] Trigger a workflow from ServiceNow or curl with `repository_dispatch`; launch an AAP job from Actions
- [ ] Tag a release, generate notes, promote dev → test → prod
- [ ] Break the pipeline on purpose five ways and debug each with `gh run view --log-failed`
