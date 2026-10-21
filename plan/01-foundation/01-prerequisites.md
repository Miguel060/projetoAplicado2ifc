# 01 — Prerequisites

Do this once per machine, before any code.

## 1. Tools

| Tool | Version | Check | Install (macOS) |
|---|---|---|---|
| Go | 1.25.x | `go version` | `brew install go` |
| Node.js | 24 LTS | `node -v` → `v24.*` | `brew install node@24` or `nvm install 24` |
| npm | 11.x (ships with Node 24) | `npm -v` | — |
| AWS CLI (AWS account owner only) | v2 | `aws --version` | `brew install awscli` |
| Git | any recent | `git --version` | — |
| GitHub CLI (optional) | any | `gh --version` | `brew install gh` |
| SQLite CLI (optional, to inspect the dataset) | 3.x | `sqlite3 --version` | preinstalled on macOS |

Windows/Linux: same versions; use the official installers or the distro
package manager. Nothing in the project is macOS-specific.

## 2. GitHub access

- Repository: **`Miguel060/projetoAplicado2ifc`** (public).
- Each member clones it with **their own personal GitHub account** — not a
  work account. If your machine has more than one GitHub identity, check
  before the first commit:

  ```bash
  git remote -v              # must point at Miguel060/projetoAplicado2ifc with your personal credentials
  git config user.email      # must be your personal (or GitHub noreply) address
  ```

  Set the identity for this clone only, if needed:
  `git config --local user.name "…"` and `git config --local user.email "…"`.

- **Admin tasks (repository owner):**
  1. Settings → Actions → General → allow GitHub Actions.
  2. Settings → Secrets and variables → Actions → **Variables** tab: create the
     variables listed in `05-infrastructure/02-github-oidc-and-variables.md`
     once the AWS stack exists.
  3. Settings → Branches → protect `main`: require a pull request and the
     `CI` check to pass.
  4. Optionally grant admin to the AWS account owner so they can manage the
     variables directly.

## 3. AWS access (AWS account owner only)

Only the AWS account owner needs AWS credentials. Everyone else never touches
AWS: CI deploys through GitHub OIDC.

| | |
|---|---|
| Account | `436812234154` |
| Region | `us-east-1` |
| CLI profile name used in every command | `personal` |

### The rule

> Every `aws` and `cdk` command for this project passes `--profile personal`.
> Never rely on the default profile, and do not export `AWS_PROFILE`
> globally — machines often hold profiles for other accounts.

### Preflight before any AWS command

```bash
aws sts get-caller-identity --profile personal --query Account --output text
# must print exactly: 436812234154
```

Anything else: stop.

### Already in place in the account (verified 2026-10-21)

- CDK bootstrap stack `CDKToolkit` in `us-east-1` → **do not bootstrap again**.
- GitHub OIDC provider
  `arn:aws:iam::436812234154:oidc-provider/token.actions.githubusercontent.com`,
  owned by another project's stack in the same account. An account can hold
  only **one** provider per URL, so this project **imports** it. Consequence:
  that other stack must not be destroyed without first moving the provider
  into a stack of its own.
- Cost budgets (see `00-overview/05-aws-cost.md`).

### Credential hygiene (before the launch checklist is signed off)

- Deploy infrastructure with a **least-privilege IAM user or Identity Center
  user** (CloudFormation, S3, CloudFront, IAM role management, and
  `sts:AssumeRole` on the CDK bootstrap roles `cdk-hnb659fds-*`).
- **Never use root access keys**; enable MFA on the root user.
- No AWS keys anywhere in the repository or in GitHub secrets — CI uses OIDC.

## 4. Editor (recommended)

- VS Code with: Go (golang.go), Biome (biomejs.biome), Tailwind CSS IntelliSense.
- Format on save: Go → `gofmt`; TS/TSX/JSON/CSS → Biome.

## 5. Group contact for the User-Agent

The ETL sends a descriptive `User-Agent` to government servers (good
citizenship, and lets them contact the group instead of blocking it):

```
projetoAplicado2ifc/1.0 (+https://github.com/Miguel060/projetoAplicado2ifc; <GROUP_CONTACT_EMAIL>)
```

Choose a group contact address and set it through the `INGEST_CONTACT`
environment variable / repository variable rather than committing it (see
`03-backend-etl/01-go-project-setup.md`).
