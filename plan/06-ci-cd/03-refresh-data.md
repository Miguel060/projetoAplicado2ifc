# 06.03 — Refresh the data (`.github/workflows/refresh-data.yml`)

**Owner:** the infra/CI lead (the AWS account owner), with the backend lead.
**When:** milestone 3 (by 2026-11-12), as soon as `go run ./cmd/ingest`
produces a valid snapshot locally (`03-backend-etl/08-validate-and-publish.md`).
**Depends on:** `05-infrastructure/02-github-oidc-and-variables.md` (all five
repository variables, including `INGEST_CONTACT`), `06-ci-cd/02-deploy-site.md`
(the site that reads the data).

This is the "backend" of the project in production: a scheduled job, not a
server. It runs the backend test suite, then the Go ETL on a GitHub runner,
then publishes an immutable, versioned snapshot to
`s3://<bucket>/data/<version>/` and switches `data/latest.json` to it.

## Why GitHub Actions and not AWS Lambda

Decision **D16** (`00-overview/02-decisions.md`). In short — cost is not the
reason, both are nearly free:

- the ETL runs **after `go test`, from the same checkout**, and a push that
  touches `backend/` re-runs it with the new code: one release path, not two;
- every group member sees the logs and history, receives failure e-mails and
  has a "Run workflow" button **without AWS access**;
- nothing new to build and secure in AWS (no function, schedule, log group or
  alerting) — only the existing deploy role with short-lived OIDC credentials;
- generous limits (6 h per job, ~14 GB disk) versus Lambda's 15 min, and
  `make ingest` locally is the same command.

Accepted costs: the 60-day pause of scheduled workflows (below), approximate
cron start times, and a skipped refresh if GitHub is down (the site keeps
serving the last snapshot). Revisit if the data must refresh hourly or
independently of GitHub.

## Triggers

| Trigger | When | Why |
|---|---|---|
| `schedule` | Mondays 09:00 UTC (06:00 in Brasília) | Weekly refresh: Câmara votes/bills/CEAP and CGU amendments keep changing; TSE files get corrections. |
| `workflow_dispatch` | Actions → Refresh data → Run workflow (branch **main**) | On demand, e.g. right after the TSE republishes a file. |
| `push` to `main` touching `backend/**` or this file | After merging ETL changes | The live data must reflect the current ETL logic, not last week's. |

`deploy.yml` also runs on every push to `main` (it re-deploys the site). The
two workflows touch disjoint prefixes — `deploy.yml` excludes `data/*` from
its sync — so they can safely run at the same time.

### ⚠️ Scheduled workflows stop after 60 days of inactivity

In **public** repositories GitHub automatically **disables** scheduled
workflows when the repository has had no activity for 60 days. After the
course ends, the weekly refresh will silently stop. That is safe (the last
snapshot stays live — there is no lifecycle rule that could delete it), but
if the panel must stay fresh, someone has to re-enable the workflow in the
Actions tab or push a commit at least every 60 days.

Failure e-mails for scheduled runs go to the person who **last edited the
`cron` line** in this file — make sure that is someone who will read them.

## Publish order (why it is safe)

```
1. ETL writes backend/dist/data/<version>/…  and, only if validation passed, latest.json
2. upload data/<version>/photos/*        immutable      ┐ nobody can see these yet:
3. upload data/<version>/**/*.json       immutable      │ latest.json still points
4. upload data/<version>/dataset.sqlite  immutable      ┘ at the previous version
5. upload data/latest.json               max-age=60     ← the atomic "go live"
6. invalidate /data/latest.json and wait
7. prune: keep the newest 5 versions; never delete the live one or the new one
8. smoke check through CloudFront
```

A failure at any step before 5 leaves the site exactly as it was. A failure
after 5 leaves the new snapshot live and complete.

## Cache headers (must match `01-foundation/04-json-contract.md`)

| Object | Content-Type | Cache-Control |
|---|---|---|
| `data/<version>/photos/*.jpg` | `image/jpeg` (inferred) | `public, max-age=31536000, immutable` |
| `data/<version>/index.json`, `candidates/*.json` | `application/json; charset=utf-8` | `public, max-age=31536000, immutable` |
| `data/<version>/dataset.sqlite` | `application/vnd.sqlite3` + `Content-Disposition: attachment` | `public, max-age=31536000, immutable` |
| `data/latest.json` | `application/json; charset=utf-8` | `public, max-age=60` |

## Tests before data (decision D15)

The `ingest` job runs `go vet` and the **same `go test` command as CI** —
race detector, unit tests, the fake-upstream pipeline integration test, the
golden snapshot comparison and the **85 % coverage gate** — **before**
`go run ./cmd/ingest`. A scheduled run therefore never publishes data
produced by code whose tests fail, even if something reached `main` by an
admin bypass. The tests touch no network, so a failure here is always a code
problem, never a government site being down.

## Two jobs, on purpose

`ingest` tests and runs the ETL, parses third-party files, and runs
**without** `id-token: write`, so nothing in the ETL can mint AWS credentials. `publish` has the token
permission but runs **no repository code** — only the AWS CLI on the
artifact the first job produced.

## The file

```yaml
name: Refresh data

on:
  schedule:
    # Mondays 09:00 UTC = 06:00 in Brasília. GitHub may start scheduled runs
    # some minutes late; nothing depends on the exact time.
    - cron: "0 9 * * 1"
  workflow_dispatch:
    inputs:
      skip_previous_check:
        description: "Skip the comparison with the live snapshot (only for an intentional large change)"
        type: boolean
        default: false
  push:
    branches: [main]
    paths:
      - "backend/**"
      - ".github/workflows/refresh-data.yml"

# One refresh at a time, and never cancel one midway: a half-published
# snapshot is harmless (latest.json is written last) but wastes the run.
concurrency:
  group: refresh-data
  cancel-in-progress: false

permissions:
  contents: read

defaults:
  run:
    shell: bash

jobs:
  ingest:
    name: Test, then build the data snapshot
    runs-on: ubuntu-latest
    timeout-minutes: 40
    outputs:
      version: ${{ steps.version.outputs.version }}
    defaults:
      run:
        working-directory: backend
    env:
      # Group contact for the User-Agent sent to government servers. Required
      # in CI (the ETL fails fast without it). Never printed by the ETL.
      INGEST_CONTACT: ${{ vars.INGEST_CONTACT }}
      SITE_URL: ${{ vars.SITE_URL }}
      SKIP_PREVIOUS_CHECK: ${{ inputs.skip_previous_check || false }}
    steps:
      - uses: actions/checkout@v5

      - uses: actions/setup-go@v5
        with:
          go-version-file: backend/go.mod
          cache-dependency-path: backend/go.sum

      - name: go vet
        run: go vet ./...

      # Same command and gate as ci.yml (job "Backend (Go)"): no snapshot is
      # built by code whose tests fail. Offline: the tests use a fake upstream.
      - name: go test (race detector, coverage of ./internal/...)
        run: go test -race -covermode=atomic -coverpkg=./internal/... -coverprofile=coverage.out ./...

      - name: Coverage gate (>= 85%)
        run: |
          go tool cover -func=coverage.out | tail -n 1 | tee -a "$GITHUB_STEP_SUMMARY"
          go tool cover -func=coverage.out | awk '/^total:/ { sub("%","",$3); if ($3+0 < 85) { print "coverage " $3 "% < 85%"; exit 1 } else print "coverage " $3 "%" }'

      # Downloads every source fresh (the runner starts with an empty .cache),
      # validates, and exits non-zero instead of producing a broken snapshot.
      # latest.json is written only when validation passed. GITHUB_SHA is set
      # by Actions and recorded in ingest_run.git_sha.
      - name: Run the ETL
        run: |
          args=(-out dist/data -cache .cache -log-format json)
          if [ "$SKIP_PREVIOUS_CHECK" != "true" ] && [ -n "$SITE_URL" ]; then
            args+=(-previous "$SITE_URL/data/latest.json")
          fi
          go run ./cmd/ingest "${args[@]}"

      - name: Read and check the new version
        id: version
        run: |
          version="$(jq -r '.version' dist/data/latest.json)"
          if ! [[ "$version" =~ ^[0-9]{8}T[0-9]{6}Z$ ]]; then
            echo "::error::Unexpected version '$version' in latest.json"
            exit 1
          fi
          dir="dist/data/$version"
          for f in index.json dataset.sqlite; do
            [ -s "$dir/$f" ] || { echo "::error::$dir/$f is missing or empty"; exit 1; }
          done
          jq -e --arg v "$version" '.schemaVersion == 1 and .version == $v and (.candidates | length) > 0' \
            "$dir/index.json" > /dev/null \
            || { echo "::error::index.json does not match latest.json or has no candidates"; exit 1; }
          # Only the files the contract defines may be published.
          unexpected="$(cd "$dir" && find . -type f ! -name '*.json' ! -path './photos/*.jpg' ! -name 'dataset.sqlite')"
          [ -z "$unexpected" ] || { echo "::error::Unexpected files in the snapshot: $unexpected"; exit 1; }
          echo "version=$version" >> "$GITHUB_OUTPUT"
          echo "New snapshot $version: $(jq '.candidates | length' "$dir/index.json") candidates"

      # Public data only (no CPF, decision D10). Kept 14 days for debugging.
      - uses: actions/upload-artifact@v4
        with:
          name: data-snapshot
          path: backend/dist/data
          retention-days: 14
          if-no-files-found: error

  publish:
    name: Publish the snapshot
    needs: ingest
    runs-on: ubuntu-latest
    timeout-minutes: 20

    permissions:
      id-token: write
      contents: read

    env:
      AWS_REGION: us-east-1
      DEPLOY_ROLE_ARN: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
      BUCKET: ${{ vars.SITE_BUCKET }}
      DISTRIBUTION_ID: ${{ vars.SITE_DISTRIBUTION_ID }}
      SITE_URL: ${{ vars.SITE_URL }}
      VERSION: ${{ needs.ingest.outputs.version }}

    steps:
      - uses: actions/download-artifact@v4
        with:
          name: data-snapshot
          path: snapshot

      - name: Check the repository variables are set
        run: |
          missing=0
          for name in DEPLOY_ROLE_ARN BUCKET DISTRIBUTION_ID SITE_URL VERSION; do
            if [ -z "${!name}" ]; then echo "::error::$name is empty (see plan 05-infrastructure/02)."; missing=1; fi
          done
          exit "$missing"

      - name: Assume the deploy role
        uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ env.DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Refuse to publish into the wrong account
        run: |
          account="$(aws sts get-caller-identity --query Account --output text)"
          echo "Publishing as account $account"
          if [ "$account" != "436812234154" ]; then
            echo "::error::Expected account 436812234154, got $account. Stopping."
            exit 1
          fi

      - name: Upload photos (immutable)
        run: |
          aws s3 sync "snapshot/$VERSION" "s3://$BUCKET/data/$VERSION" \
            --exclude "*" --include "photos/*" \
            --cache-control "public, max-age=31536000, immutable" \
            --only-show-errors

      - name: Upload JSON (immutable)
        run: |
          aws s3 sync "snapshot/$VERSION" "s3://$BUCKET/data/$VERSION" \
            --exclude "*" --include "*.json" \
            --content-type "application/json; charset=utf-8" \
            --cache-control "public, max-age=31536000, immutable" \
            --only-show-errors

      - name: Upload the dataset (immutable, offered as a download)
        run: |
          aws s3 cp "snapshot/$VERSION/dataset.sqlite" "s3://$BUCKET/data/$VERSION/dataset.sqlite" \
            --content-type "application/vnd.sqlite3" \
            --content-disposition "attachment; filename=\"dataset-$VERSION.sqlite\"" \
            --cache-control "public, max-age=31536000, immutable" \
            --only-show-errors

      # The atomic switch. Until this lands, every visitor reads the previous
      # snapshot in full.
      - name: Switch latest.json to the new version
        run: |
          aws s3 cp "snapshot/latest.json" "s3://$BUCKET/data/latest.json" \
            --content-type "application/json; charset=utf-8" \
            --cache-control "public, max-age=60" \
            --only-show-errors

      - name: Invalidate the pointer in CloudFront
        run: |
          id="$(aws cloudfront create-invalidation \
            --distribution-id "$DISTRIBUTION_ID" \
            --paths "/data/latest.json" \
            --query 'Invalidation.Id' --output text)"
          echo "Invalidation $id"
          aws cloudfront wait invalidation-completed --distribution-id "$DISTRIBUTION_ID" --id "$id"

      # Pruning lives here, not in an S3 lifecycle rule: if the cron ever
      # stops, a lifecycle rule would eventually delete the version that is
      # still live. The regex guarantees nothing outside data/<version>/ is touched.
      - name: Prune old snapshots (keep the newest 5, never the live one)
        run: |
          live="$(aws s3 cp "s3://$BUCKET/data/latest.json" - | jq -r '.version')"
          mapfile -t versions < <(
            aws s3api list-objects-v2 --bucket "$BUCKET" --prefix data/ --delimiter / \
              --query 'CommonPrefixes[].Prefix' --output text \
            | tr '\t' '\n' | grep -E '^data/[0-9]{8}T[0-9]{6}Z/$' | sort
          )
          keep_from=$(( ${#versions[@]} - 5 ))
          for i in "${!versions[@]}"; do
            prefix="${versions[$i]}"
            if [ "$i" -ge "$keep_from" ] || [ "$prefix" = "data/$live/" ] || [ "$prefix" = "data/$VERSION/" ]; then
              echo "keep   $prefix"
              continue
            fi
            echo "delete $prefix"
            aws s3 rm "s3://$BUCKET/$prefix" --recursive --only-show-errors
          done

      - name: Smoke check the live data through CloudFront
        run: |
          served=""
          for attempt in $(seq 1 10); do
            served="$(curl -s "$SITE_URL/data/latest.json" | jq -r '.version // empty' 2>/dev/null || true)"
            [ "$served" = "$VERSION" ] && break
            sleep 6
          done
          [ "$served" = "$VERSION" ] || { echo "::error::CloudFront serves '$served', expected $VERSION"; exit 1; }

          type="$(curl -s -o /dev/null -w '%{content_type}' "$SITE_URL/data/$VERSION/index.json")"
          [[ "$type" == application/json* ]] || { echo "::error::index.json served as '$type'"; exit 1; }

          index="$(curl -sf "$SITE_URL/data/$VERSION/index.json")"
          jq -e '.schemaVersion == 1 and (.candidates | length) > 0' <<<"$index" > /dev/null \
            || { echo "::error::index.json of $VERSION does not load"; exit 1; }

          slug="$(jq -r '.candidates[0].slug' <<<"$index")"
          curl -sf "$SITE_URL/data/$VERSION/candidates/$slug.json" \
            | jq -e --arg s "$slug" '.summary.slug == $s' > /dev/null \
            || { echo "::error::profile $slug does not load"; exit 1; }

          echo "Live: $VERSION ($(jq '.candidates | length' <<<"$index") candidates; first profile $slug OK)"
```

## Step by step

1. Prerequisites: `deploy.yml` is green, the five repository variables exist,
   and `make ingest` produces a valid snapshot locally.
2. On a branch, add `.github/workflows/refresh-data.yml` with the content above.
3. Open a PR → review → merge. Because the file is under the `paths` filter,
   the merge itself triggers the first refresh.
4. Watch **Actions → Refresh data**: `Test, then build the data snapshot`
   (tests ≈1–2 min, then the ETL ≈5–10 min, mostly downloads) →
   `Publish the snapshot`.
5. Open `SITE_URL/candidatos`: the search page lists the candidates and the footer shows
   the new "Dados atualizados em …" date.
6. Check the next Monday that the scheduled run happened (the Actions tab
   shows `schedule` as the event).

## Manual fallback (from a local machine)

If GitHub Actions is unavailable, the AWS account owner can run the same
sequence locally with `make publish-data-local`
(`03-backend-etl/08-validate-and-publish.md`). It performs the same uploads
with the same headers, in the same order, with `--profile personal` on every
command and the account preflight first. Use it only in an emergency: a
local run is not recorded in the Actions history.

## Rollback

Point `latest.json` back at the previous snapshot — see
`07-delivery/02-launch-checklist.md` → "Rollback". The previous 4 versions
are still in the bucket.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `go test` or the coverage gate fails before the ETL | Code on `main` with failing tests (should be impossible with branch protection) | Fix on a branch → PR → merge; the push re-runs this workflow. The live snapshot is untouched. |
| `INGEST_CONTACT must be set in CI` | Variable missing | Repository admin creates `INGEST_CONTACT` (05.02). |
| ETL fails in validation comparing with the live snapshot | A source changed shape or returned partial data — or the change is intentional | Read the log first. If intentional, run manually with **skip_previous_check**. |
| ETL fails downloading | Government site down or slow | Re-run later; the live snapshot is untouched. |
| `Unexpected files in the snapshot` | ETL writes something the contract does not define | Update `01-foundation/04-json-contract.md` and this workflow together, or fix the ETL. |
| Smoke check: CloudFront serves the old version | Invalidation not finished, or `latest.json` uploaded with a long `Cache-Control` | Re-run the smoke step; check the object's `Cache-Control` with `aws s3api head-object --profile personal`. |
| Scheduled run did not happen | 60-day inactivity, or the workflow was disabled | Actions tab → Refresh data → Enable workflow. |

## Done when

- [ ] A merge touching `backend/` publishes a new snapshot end to end.
- [ ] The run log shows `go test` and `coverage ≥ 85%` **before** the `Run the ETL` step.
- [ ] `curl -s "$SITE_URL/data/latest.json"` shows the new `version`.
- [ ] `aws s3 ls "s3://<bucket>/data/" --profile personal` lists at most 5 version prefixes plus `latest.json`.
- [ ] The first scheduled Monday run is green.
