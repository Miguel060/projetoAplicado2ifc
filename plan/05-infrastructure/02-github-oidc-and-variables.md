# 05.02 — GitHub OIDC, repository variables and branch protection

**Owners:** the AWS account owner (AWS side) and the **repository
owner/admin** (GitHub side — only an admin can create variables and rulesets;
push access is not enough). **When:** right after
`05-infrastructure/01-cdk-stack.md` Step 9, by 2026-11-09.
**Unblocks:** `06-ci-cd/02-deploy-site.md` (`deploy.yml`), `06-ci-cd/03-refresh-data.md` (`refresh-data.yml`).

## How CI gets into AWS without any stored key

```
GitHub Actions job (permissions: id-token: write)
  │ 1. asks GitHub for a short-lived OIDC token
  │    claims: aud=sts.amazonaws.com, sub=repo:Miguel060/projetoAplicado2ifc:ref:refs/heads/main
  ▼
AWS STS  AssumeRoleWithWebIdentity(role = ProjetoAplicado2Deploy, token)
  │ 2. checks the token signature against the account's GitHub OIDC provider
  │ 3. checks the role's trust policy: aud and sub must match EXACTLY
  ▼
temporary credentials (≤ 1 hour), scoped to: this bucket's objects + this distribution's invalidations
```

There is **no AWS access key** in the repository, in GitHub secrets, or
anywhere else. A leaked workflow log reveals nothing reusable.

## Who can and cannot assume the role

| Run | `sub` claim | Allowed? |
|---|---|---|
| Push to `main` | `repo:Miguel060/projetoAplicado2ifc:ref:refs/heads/main` | ✅ |
| Scheduled run (cron runs on the default branch) | same | ✅ |
| Manual "Run workflow" **from `main`** | same | ✅ |
| Manual run from any other branch | `…:ref:refs/heads/<branch>` | ❌ |
| Pull request (any, including from forks) | `…:pull_request` | ❌ — and PR workflows never request `id-token: write` |
| Any other repository, fork or rename | different `repo:` | ❌ |

The trust policy accepts the two subject shapes GitHub can emit for `main`,
and nothing else:

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::436812234154:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:sub": [
        "repo:Miguel060/projetoAplicado2ifc:ref:refs/heads/main",
        "repo:Miguel060@77062840/projetoAplicado2ifc@1366189075:ref:refs/heads/main"
      ]
    }
  }
}
```

Permissions of the role (the complete list):

| Action | Resource | Used by |
|---|---|---|
| `s3:ListBucket` | the bucket | `aws s3 sync`, pruning old data versions |
| `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` | `bucket/*` | uploads, reading the live `latest.json`, pruning |
| `cloudfront:CreateInvalidation`, `cloudfront:GetInvalidation` | the distribution | invalidating `index.html` / `latest.json` and waiting for it |

## The imported OIDC provider (and the dependency it creates)

An AWS account can have only **one** OIDC provider per URL. The provider for
`https://token.actions.githubusercontent.com` already exists in account
`436812234154` — it was created by the **PacketLaboratory** stack
(`arn:aws:iam::436812234154:oidc-provider/token.actions.githubusercontent.com`,
verified 2026-10-21). Our stack **imports** it by ARN
(`iam.OpenIdConnectProvider.fromOpenIdConnectProviderArn`); creating a second
one would fail with `EntityAlreadyExists`.

**Consequence:** the provider's lifecycle belongs to PacketLaboratory. If that
stack is ever destroyed, CloudFormation deletes the provider and **every
deploy of this project starts failing** with `Not authorized to perform
sts:AssumeRoleWithWebIdentity`.

Before destroying PacketLaboratory, detach the provider first:

1. In PacketLaboratory's stack, set `RemovalPolicy.RETAIN` on the provider
   (`githubOidc.applyRemovalPolicy(RemovalPolicy.RETAIN)`) and deploy it.
2. Replace the provider construct there with the same
   `fromOpenIdConnectProviderArn(...)` import and deploy again. CloudFormation
   drops the resource from the stack but leaves it in the account.
3. Now both stacks import a provider that no stack owns. Destroying either is safe.

If it already happened: recreate the provider once by hand —

```bash
aws iam create-open-id-connect-provider --profile personal \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

— the ARN is the same, so neither stack needs to change.

## Repository variables (repository admin creates them)

All five are **variables, not secrets**: none is sensitive. A role ARN is
useless without a token that satisfies its trust policy, and the bucket and
distribution are not reachable except through CloudFront. Variables are also
printed unmasked in logs, which keeps debugging simple.

| Variable | Value (from `cdk deploy` outputs) | Example |
|---|---|---|
| `AWS_DEPLOY_ROLE_ARN` | `DeployRoleArn` | `arn:aws:iam::436812234154:role/ProjetoAplicado2Deploy` |
| `SITE_BUCKET` | `BucketName` | `projetoaplicado2-sitebucket1a2b3c4d-xyz` |
| `SITE_DISTRIBUTION_ID` | `DistributionId` | `E2ABCDEF123456` |
| `SITE_URL` | `SiteUrl`, **no trailing slash** | `https://d1234abcd.cloudfront.net` |
| `INGEST_CONTACT` | The group's contact e-mail, chosen by the group (not from CDK) | `<group contact e-mail>` |

`INGEST_CONTACT` is read only by `refresh-data.yml` and passed to the ETL,
which puts it in the `User-Agent` it sends to the government servers, so they
can contact the group instead of blocking it
(`01-foundation/01-prerequisites.md`). It is kept out of the repository on
purpose; the ETL must not print it in its logs.

How (repository admin only):

- Web UI: **Settings → Secrets and variables → Actions → Variables tab → New repository variable**, once per row.
- Or with the GitHub CLI, logged in as the admin:

  ```bash
  REPO=Miguel060/projetoAplicado2ifc
  gh variable set AWS_DEPLOY_ROLE_ARN  --repo $REPO --body "arn:aws:iam::436812234154:role/ProjetoAplicado2Deploy"
  gh variable set SITE_BUCKET          --repo $REPO --body "<BucketName>"
  gh variable set SITE_DISTRIBUTION_ID --repo $REPO --body "<DistributionId>"
  gh variable set SITE_URL             --repo $REPO --body "<SiteUrl>"
  gh variable set INGEST_CONTACT       --repo $REPO --body "<group contact e-mail>"
  gh variable list --repo $REPO
  ```

  `gh variable set` needs an account with **admin** permission on the
  repository; with only push access it fails with `403`.

**No repository secrets are needed at all** (decision D9). If someone proposes
adding one, it needs a written reason in `00-overview/02-decisions.md`.

## Actions settings (repository admin)

**Settings → Actions → General:**

| Setting | Value | Why |
|---|---|---|
| Actions permissions | Allow all actions and reusable workflows (or: allow GitHub-authored + `aws-actions/*`) | The workflows use `actions/*` and `aws-actions/configure-aws-credentials`. |
| Fork pull request workflows | Require approval for first-time contributors (default) | Public repo; avoids random fork runs. |
| Workflow permissions | **Read repository contents and packages permissions** | Every workflow declares what it needs; nothing writes to the repo. |
| Allow GitHub Actions to create and approve pull requests | **Off** | Not used. |

## Branch protection for `main` (repository admin)

**Settings → Rules → Rulesets → New branch ruleset** (or the classic
**Branches → Add rule**), target `main`:

- [x] **Require a pull request before merging**, 1 approval (another member reviews).
- [x] **Require status checks to pass** — all four CI jobs, which include
  every test suite and the 85 % coverage gates (decision D15):
  - `Backend (Go)`
  - `Frontend (React)`
  - `End-to-end (Playwright)`
  - `Infrastructure (CDK)`

  These names only appear in the picker after `ci.yml` has run once — open a
  first PR, let CI run, then add them.
- [x] **Block force pushes**.
- [x] **Restrict deletions**.
- Bypass list: the repository admin, for emergencies only.

Why it matters beyond hygiene: `ci.yml` does not run on pushes to `main`
(`deploy.yml` re-runs it as its `verify` job). Requiring PRs guarantees every commit on
`main` passed CI before it got there.

## Troubleshooting: `Not authorized to perform sts:AssumeRoleWithWebIdentity`

The error names no claim. Print the real `sub` from inside the failing job
(temporary step, remove afterwards):

```yaml
- name: Show the OIDC subject (debug only)
  run: |
    token=$(curl -sH "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq -r .value)
    echo "$token" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq '{sub, aud, ref, repository}'
```

| Printed `sub` | Cause | Fix |
|---|---|---|
| `…:ref:refs/heads/<other-branch>` | Workflow dispatched from a branch | Run it from `main`. |
| `…:pull_request` | Job ran on a PR | Deploy jobs must not run on PRs (they don't). |
| `repo:…@<ids>/…` with different ids | Repo transferred/recreated | Update `ownerId` / `repositoryId` in `infra/lib/config.ts` **and** the `SUBJECTS` expectation in `infra/test/site-stack.test.ts`, redeploy the stack. |
| Something with `job_workflow_ref`, `environment`, etc. | The owner customised the OIDC subject template | Reset it (`gh api -X PUT repos/Miguel060/projetoAplicado2ifc/actions/oidc/customization/sub -F use_default=true`) or add the exact new subject to `githubSubjects`. Never use a wildcard. |
| No token at all | Job lacks `permissions: id-token: write` | Add it to that job only. |

## Done when

- [ ] The five variables exist (`gh variable list` shows them).
- [ ] The ruleset protects `main` with the four checks.
- [ ] The first run of `deploy.yml` passes the "Refuse to deploy into the wrong account" step.
