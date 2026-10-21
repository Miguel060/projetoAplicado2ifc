# 06.02 — Deploy the site (`.github/workflows/deploy.yml`)

**Owner:** the infra/CI lead (the AWS account owner). **When:** milestone 2
(by 2026-11-09), as soon as the frontend skeleton builds.
**Depends on:** `06-ci-cd/01-ci.md`, `05-infrastructure/01-cdk-stack.md`,
`05-infrastructure/02-github-oidc-and-variables.md` (the repository variables; this workflow uses four of them).

The same model as the PacketLaboratory project in this AWS account:

- **one** deploy workflow, `deploy.yml`;
- it runs on **every push to `main`** (no `paths` filter) and on the manual
  "Run workflow" button;
- its first job, `verify`, **is** `ci.yml` (reused via `workflow_call`), so a
  red build can never reach CloudFront;
- then one `deploy` job builds the React app and publishes it with
  short-lived OIDC credentials;
- infrastructure is **not** deployed here — `cdk deploy` stays manual
  (`05-infrastructure/01-cdk-stack.md` Step 8).

Data snapshots are published by a different workflow, `refresh-data.yml`
(`06-ci-cd/03-refresh-data.md`). The two touch **disjoint prefixes** of the
bucket — this one everything except `data/`, the other only `data/` — so they
may run at the same time after a push that changes both `frontend/` and
`backend/`.

## What gets uploaded, and with which cache headers

| Files | Cache-Control | Why |
|---|---|---|
| `assets/*` (Vite output, content-hashed names) | `public, max-age=31536000, immutable` | A hashed name never changes content; cache forever. Uploaded **first** and old ones are **not** deleted, so a tab opened before the deploy can still load what its `index.html` referenced. |
| Other root files (`favicon.svg`, `robots.txt`, …) | `public, max-age=3600` | Not hashed, so not immutable. Removed from S3 if removed from the build (`--delete`). |
| `index.html` | `no-cache, must-revalidate` | The one stable name that points at the new hashed assets. If it were cached, a deploy would never reach anyone. Uploaded **last**. |
| `data/**` | — (never touched) | `--exclude "data/*"`. Without it, `--delete` would wipe every published snapshot. |

Differences from PacketLaboratory's `deploy.yml`, all deliberate: the
`data/*` exclusion (that project has no data), the separate cache class for
non-hashed root files, keeping old hashed assets, and the extra smoke checks
for the SPA rewrite and the data pointer.

## The file

```yaml
name: Deploy

on:
  # Every push to main deploys the site. No paths filter and no workflow_run
  # indirection: one rule, easy to reason about. A backend-only push simply
  # republishes the same site, which is harmless and cheap.
  push:
    branches: [main]
  # So a deploy can be repeated without an empty commit (run it from main;
  # the deploy role trusts nothing else).
  workflow_dispatch:

# Never two deploys at once, and never cancel one midway: a half-finished sync
# would leave the bucket serving a mix of two builds.
concurrency:
  group: deploy-production
  cancel-in-progress: false

permissions:
  contents: read

jobs:
  # The same checks CI runs, reused rather than restated. `vite build` does not
  # type-check, so without this a type error would deploy perfectly happily.
  verify:
    uses: ./.github/workflows/ci.yml

  deploy:
    name: Publish to CloudFront
    needs: verify
    runs-on: ubuntu-latest
    timeout-minutes: 15

    permissions:
      # Only this job may mint the OIDC token that is exchanged for the AWS
      # role. No access key exists anywhere in the repository.
      id-token: write
      contents: read

    defaults:
      run:
        shell: bash

    env:
      AWS_REGION: us-east-1
      DEPLOY_ROLE_ARN: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
      BUCKET: ${{ vars.SITE_BUCKET }}
      DISTRIBUTION_ID: ${{ vars.SITE_DISTRIBUTION_ID }}
      SITE_URL: ${{ vars.SITE_URL }}

    steps:
      - uses: actions/checkout@v5

      - uses: actions/setup-node@v5
        with:
          node-version-file: .nvmrc
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install
        working-directory: frontend
        run: npm ci

      - name: Build
        working-directory: frontend
        run: npm run build

      - name: Check the repository variables are set
        run: |
          missing=0
          for name in DEPLOY_ROLE_ARN BUCKET DISTRIBUTION_ID SITE_URL; do
            if [ -z "${!name}" ]; then
              echo "::error::Repository variable for $name is empty (see plan 05-infrastructure/02)."
              missing=1
            fi
          done
          exit "$missing"

      - name: Assume the deploy role
        uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ env.DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Refuse to deploy into the wrong account
        run: |
          account="$(aws sts get-caller-identity --query Account --output text)"
          echo "Deploying as account $account"
          if [ "$account" != "436812234154" ]; then
            echo "::error::Expected account 436812234154, got $account. Stopping."
            exit 1
          fi

      # Hashed filenames never change contents, so they are cached forever.
      # No --delete: an open tab may still need the previous build's files.
      - name: Upload hashed assets
        run: |
          aws s3 sync frontend/dist/assets "s3://$BUCKET/assets" \
            --cache-control "public, max-age=31536000, immutable" \
            --only-show-errors

      # Everything else except index.html and the data snapshots. Excluded
      # paths are neither uploaded nor deleted by --delete.
      - name: Upload the other static files
        run: |
          aws s3 sync frontend/dist "s3://$BUCKET" \
            --delete \
            --exclude "data/*" \
            --exclude "assets/*" \
            --exclude "index.html" \
            --cache-control "public, max-age=3600" \
            --only-show-errors

      # Last, and never cached: the moment this lands, visitors get the new build.
      - name: Upload index.html, uncached
        run: |
          aws s3 cp frontend/dist/index.html "s3://$BUCKET/index.html" \
            --cache-control "no-cache, must-revalidate" \
            --content-type "text/html; charset=utf-8" \
            --only-show-errors

      # Every SPA route is rewritten to /index.html before the cache lookup, so
      # these two paths cover all routes.
      - name: Invalidate CloudFront
        run: |
          id="$(aws cloudfront create-invalidation \
            --distribution-id "$DISTRIBUTION_ID" \
            --paths "/index.html" "/" \
            --query 'Invalidation.Id' --output text)"
          echo "Invalidation $id"
          aws cloudfront wait invalidation-completed --distribution-id "$DISTRIBUTION_ID" --id "$id"

      # The URL comes from a repository variable instead of being looked up:
      # asking CloudFront would need cloudfront:GetDistribution, and the role is
      # scoped to exactly what publishing requires.
      - name: Smoke check the live site
        run: |
          status() { curl -s -o /dev/null -w '%{http_code}' "$SITE_URL$1"; }
          ctype()  { curl -s -o /dev/null -w '%{content_type}' "$SITE_URL$1"; }

          # 1. The site answers.
          code=""
          for attempt in $(seq 1 10); do
            code="$(status /)"
            [ "$code" = "200" ] && break
            sleep 6
          done
          [ "$code" = "200" ] || { echo "::error::$SITE_URL/ returned $code"; exit 1; }
          echo "/ -> 200"

          # 2. Every SPA route is served as HTML by the CloudFront Function rewrite:
          #    landing, search, a profile slug that does not exist (the app shows
          #    its not-found state; the HTTP status is still 200), and About.
          for path in / /candidatos /candidatos/nao-existe /sobre; do
            code="$(status "$path")"; type="$(ctype "$path")"
            if [ "$code" != "200" ] || [[ "$type" != text/html* ]]; then
              echo "::error::$path returned $code ($type); expected 200 text/html"
              exit 1
            fi
            echo "$path -> $code $type"
          done

          # 3. The security headers policy is attached.
          headers="$(curl -sI "$SITE_URL/")"
          for h in content-security-policy strict-transport-security x-frame-options x-content-type-options referrer-policy; do
            grep -qi "^$h:" <<<"$headers" || { echo "::error::Missing response header $h"; exit 1; }
          done
          echo "security headers present"

          # 4. A missing data file is a real 404, never the HTML fallback.
          code="$(status /data/smoke-check-missing.json)"
          case "$code" in
            404) echo "missing data file -> 404" ;;
            403) echo "::warning::missing data file -> 403 (expected 404; check AccessLevel.LIST on the origin)" ;;
            *)   echo "::error::missing data file returned $code; the SPA rewrite is catching files"; exit 1 ;;
          esac

          # 5. The data pointer, if a snapshot has been published already. The
          #    landing page is deployed before the first ETL run, so a missing
          #    pointer is a warning here, never a failure.
          code="$(status /data/latest.json)"
          if [ "$code" = "200" ]; then
            curl -s "$SITE_URL/data/latest.json" \
              | jq -e '.schemaVersion == 1 and (.version | test("^[0-9]{8}T[0-9]{6}Z$"))' > /dev/null \
              || { echo "::error::/data/latest.json is not a valid pointer"; exit 1; }
            echo "data pointer OK: $(curl -s "$SITE_URL/data/latest.json" | jq -r .version)"
          else
            echo "::warning::/data/latest.json returned $code; no data snapshot published yet. Run the 'Refresh data' workflow."
          fi
```

## Step by step

1. Confirm the prerequisites: the stack is deployed and the repository
   variables exist (`gh variable list` or Settings → Variables).
2. On a branch `ci/add-deploy`, create `.github/workflows/deploy.yml` with the
   content above. CI runs on the branch push; the deploy itself does not
   (it only triggers on `main`).
3. Open a PR, get it reviewed, merge. The merge is a push to `main`, so
   `Deploy` starts: `verify` (three CI jobs) → `Publish to CloudFront`.
4. Watch the run in the **Actions** tab. Every step must be green; the
   "Smoke check" step prints each path and its status.
5. Open `SITE_URL` in a browser. The first real deploy is the **landing
   page**; before the first data refresh it shows its "Dados em preparação"
   fallback, and step 5 of the smoke check prints a warning — both expected
   until `06-ci-cd/03-refresh-data.md` has run once.

## Verify by hand (any time)

```bash
SITE_URL=https://dxxxxxxxxxxxx.cloudfront.net

curl -sI "$SITE_URL/" | grep -iE '^(HTTP|cache-control|content-type|content-security-policy)'
#   HTTP/2 200 · cache-control: no-cache, must-revalidate · content-type: text/html; charset=utf-8 · CSP present

curl -sI "$SITE_URL/candidatos/qualquer-coisa-1234" | sed -n 1p   # HTTP/2 200 (SPA route)
curl -sI "$SITE_URL/data/nao-existe.json"            | sed -n 1p    # HTTP/2 404 (real file semantics)

ASSET=$(curl -s "$SITE_URL/" | grep -oE '/assets/[^"]+\.js' | head -1)
curl -sI "$SITE_URL$ASSET" | grep -i cache-control                  # public, max-age=31536000, immutable
```

## Rollback

The deploy role trusts only `main`, so a rollback is a commit on `main`:

```bash
git revert <bad-commit-sha>      # on a branch, then PR → merge
```

The merge redeploys the previous build. Old hashed assets are still in the
bucket, so the rollback is effective as soon as `index.html` is replaced.
Data is unaffected (see `07-delivery/02-launch-checklist.md` for data rollback).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Run not from `main`, or the OIDC `sub` shape differs | `05-infrastructure/02-github-oidc-and-variables.md` → Troubleshooting |
| "Repository variable … is empty" | Variables not created yet | Repository admin creates them (05.02) |
| "Expected account 436812234154, got …" | The role ARN variable points at another account | Fix `AWS_DEPLOY_ROLE_ARN` |
| `AccessDenied` on `s3 sync` | `SITE_BUCKET` is not the stack's bucket | Compare with the stack outputs |
| Smoke check: SPA route returns 404 | CloudFront Function not associated | `cdk diff` / redeploy the stack |
| Smoke check: missing data file returns 200 | Someone added `errorResponses` to the distribution | Remove them (05.01, "Design notes") |
| New build not visible | Browser cache of an old `index.html` from before the headers were set | Hard reload; check `cache-control` on `/` |

## Done when

- [ ] A merge to `main` runs `verify` then `Publish to CloudFront`, all green.
- [ ] `/`, `/candidatos`, `/candidatos/nao-existe` and `/sobre` all return `200 text/html`.
- [ ] `SITE_URL/data/nao-existe.json` returns `404`.
- [ ] The security headers appear on `curl -sI "$SITE_URL/"`.
