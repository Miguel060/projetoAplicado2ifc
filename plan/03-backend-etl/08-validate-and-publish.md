# 03.08 — Validate and publish a data snapshot

## Goal

1. **Refuse to publish broken data.** `validate.Check` runs inside `ingest`
   before `latest.json` is written; any failure exits non-zero and nothing is
   uploaded — the snapshot already live keeps being served.
2. **Publish atomically.** Upload the immutable `data/<version>/` first, then
   switch `data/latest.json`, then invalidate that single path, then prune old
   versions (keep the last 5).

The same publish commands are used by the `refresh-data.yml` workflow (CI)
and by `make publish-data-local` (manual fallback).

## Files to create

```
backend/internal/validate/validate.go
backend/scripts/publish-data.sh        # the one implementation of the publish order
Makefile                               # add the publish-data-local target
```

## Part A — `validate.Check(ctx, dir, previous string, c *fetch.Client) error`

Collect **all** problems, then return them joined (`errors.Join`) so one run
shows every issue. Checks:

| # | Check | Why |
|---|---|---|
| 1 | `index.json` decodes into the contract types; `schemaVersion == 1`; `version` matches the directory name | contract integrity |
| 2 | For each office: `counts[o] >= min(TopN, candidates with > 0 votes in the municipality)` and `counts[o] <= TopN + 15` | catches filter bugs and runaway ties |
| 3 | `counts` equals the number of summaries per office; slugs unique; `candidates/<slug>.json` exists for each and its `summary` equals the index entry | index ↔ profile consistency |
| 4 | Every summary: `office` and `result.code` are valid enum values; `selectionReasons` non-empty; `votes.municipality > 0` unless `bornInMunicipality` | rule D1 really applied |
| 5 | `photo` non-null ⇒ file exists and starts with `FF D8 FF` | no broken images |
| 6 | All 11 source ids present in `sources`; each has `fetchedAt`, `https://` `datasetUrl` and `fileUrl` | provenance on every block (goal 3) |
| 7 | Every profile: every block's `source` id exists in `index.sources`; array fields are arrays (decode once into `map[string]any` and walk the known array paths — `null` fails) | the frontend never gets `null` where it maps |
| 8 | `chamber == null` ⇔ `amendments == null`; `stateAssembly != null` only for `state_deputy` | contract rules |
| 9 | No JSON key named `cpf` anywhere; SQLite has no column matching `%cpf%` (re-checked here) | D10 |
| 10 | `sha256(dataset.sqlite) == index.dataset.sha256` and `bytes` match | download integrity |
| 11 | Sizes: `index.json` < 1 MB; each profile < 512 KB; total snapshot < 50 MB | catches accidental detail dumps |
| 12 | **Against the live snapshot** (`-previous`): read the live `latest.json` (URL or local path), then the live `<version>/index.json` next to it. **Fail** if the new snapshot has fewer than 50 % of the live candidates in total, or **loses an office entirely** (live count > 0, new count = 0), or a source id present live is missing now. Empty flag / HTTP 404 / file not found ⇒ no live snapshot yet → **info log, skip** (not an error). Other fetch or decode failures are errors. | a silently broken upstream file must not replace good data |

```go
func Check(ctx context.Context, dir, previous string, c *fetch.Client) error {
	var errs []error
	idx, err := readIndex(filepath.Join(dir, "index.json"))
	if err != nil {
		return fmt.Errorf("validate: index: %w", err)
	}
	errs = append(errs, checkCounts(idx)...)
	errs = append(errs, checkProfiles(dir, idx)...)
	errs = append(errs, checkSources(idx)...)
	errs = append(errs, checkPrivacy(dir)...)
	errs = append(errs, checkDataset(dir, idx)...)
	errs = append(errs, checkSizes(dir)...)
	if previous != "" {
		errs = append(errs, checkAgainstLive(ctx, c, previous, idx)...)
	}
	return errors.Join(errs...) // nil when the slice holds only nils
}

// checkAgainstLive implements check 12. previous is an http(s) URL or a local path.
func checkAgainstLive(ctx context.Context, c *fetch.Client, previous string, idx *export.DataIndex) []error {
	var ptr export.LatestPointer
	base, found, err := readPointer(ctx, c, previous, &ptr) // found=false on 404 / os.ErrNotExist
	if err != nil {
		return []error{fmt.Errorf("live pointer %s: %w", previous, err)}
	}
	if !found {
		slog.Info("no live snapshot yet; skipping comparison", "previous", previous)
		return nil
	}
	// base = previous without the trailing "latest.json" (URL path or directory)
	var live export.DataIndex
	if _, found, err := readJSON(ctx, c, join(base, ptr.Version, "index.json"), &live); err != nil || !found {
		return []error{fmt.Errorf("live index %s: found=%v err=%v", ptr.Version, found, err)}
	}
	var errs []error
	if 2*len(idx.Candidates) < len(live.Candidates) {
		errs = append(errs, fmt.Errorf("candidates dropped from %d to %d (> 50%%)", len(live.Candidates), len(idx.Candidates)))
	}
	for office, n := range live.Counts {
		if n > 0 && idx.Counts[office] == 0 {
			errs = append(errs, fmt.Errorf("office %s disappeared (live had %d)", office, n))
		}
	}
	for id := range live.Sources {
		if _, ok := idx.Sources[id]; !ok {
			errs = append(errs, fmt.Errorf("source %s missing (present in live)", id))
		}
	}
	return errs
}
```

`readPointer`/`readJSON` use `c.GetJSON` for `http(s)://` (no cache, a 404
returns `found=false`) and `os.ReadFile` otherwise (`errors.Is(err,
fs.ErrNotExist)` ⇒ `found=false`). The live files are fetched through
CloudFront, so the deploy role is not needed for validation.

An intentional large change (e.g. lowering `TopN`) is published by running
the workflow manually with `skip_previous_check = true`, which omits the
flag.

What happens on failure:

```
ingest → validate.Check returns error → main logs "ingest failed" → exit 1
→ latest.json is NOT written → the workflow's publish step never runs
→ CloudFront keeps serving data/latest.json → previous <version>
```

GitHub e-mails the failed run to whoever triggered/owns the schedule.

## Part B — the publish order (`backend/scripts/publish-data.sh`)

**Single source of truth for the order and headers is the `publish` job of
`06-ci-cd/03-refresh-data.md`.** This script is the same sequence, command for
command, for publishing by hand; if one changes, change the other in the same
pull request.

Inputs (environment):

| Variable | Value |
|---|---|
| `BUCKET` | bucket name (CDK output `BucketName`) |
| `DISTRIBUTION_ID` | CloudFront id (CDK output `DistributionId`) |
| `SITE_URL` | `https://<distribution>.cloudfront.net` (CDK output `SiteUrl`) |
| `AWS_ARGS` | `--profile personal` — the project's AWS CLI profile name |
| `DATA_DIR` | default `backend/dist/data` |

```bash
#!/usr/bin/env bash
# Manual mirror of the `publish` job in .github/workflows/refresh-data.yml.
# Order: account check → photos → JSON → dataset → latest.json → invalidate+wait → prune → smoke.
set -euo pipefail

: "${BUCKET:?}" "${DISTRIBUTION_ID:?}" "${SITE_URL:?}"
DATA_DIR="${DATA_DIR:-backend/dist/data}"
AWS_ARGS="${AWS_ARGS:-}"
aws() { command aws $AWS_ARGS "$@"; }

# 0. The pointer exists only if validation passed.
test -f "$DATA_DIR/latest.json" || { echo "No latest.json: validation did not pass." >&2; exit 1; }
VERSION="$(jq -r '.version' "$DATA_DIR/latest.json")"
[[ "$VERSION" =~ ^[0-9]{8}T[0-9]{6}Z$ ]] || { echo "Unexpected version '$VERSION'" >&2; exit 1; }
SRC="$DATA_DIR/$VERSION"

# 1. Refuse to publish into the wrong account.
account="$(aws sts get-caller-identity --query Account --output text)"
echo "Publishing as account $account"
[ "$account" = "436812234154" ] || { echo "Expected account 436812234154, got $account. Stopping." >&2; exit 1; }

# 2. Upload photos (immutable; content type inferred from .jpg)
aws s3 sync "$SRC" "s3://$BUCKET/data/$VERSION" \
  --exclude "*" --include "photos/*" \
  --cache-control "public, max-age=31536000, immutable" \
  --only-show-errors

# 3. Upload JSON (immutable)
aws s3 sync "$SRC" "s3://$BUCKET/data/$VERSION" \
  --exclude "*" --include "*.json" \
  --content-type "application/json; charset=utf-8" \
  --cache-control "public, max-age=31536000, immutable" \
  --only-show-errors

# 4. Upload the dataset (immutable, offered as a download)
aws s3 cp "$SRC/dataset.sqlite" "s3://$BUCKET/data/$VERSION/dataset.sqlite" \
  --content-type "application/vnd.sqlite3" \
  --content-disposition "attachment; filename=\"dataset-$VERSION.sqlite\"" \
  --cache-control "public, max-age=31536000, immutable" \
  --only-show-errors

# 5. The atomic switch: latest.json → new version
aws s3 cp "$DATA_DIR/latest.json" "s3://$BUCKET/data/latest.json" \
  --content-type "application/json; charset=utf-8" \
  --cache-control "public, max-age=60" \
  --only-show-errors

# 6. Invalidate the pointer in CloudFront and wait
id="$(aws cloudfront create-invalidation \
  --distribution-id "$DISTRIBUTION_ID" \
  --paths "/data/latest.json" \
  --query 'Invalidation.Id' --output text)"
echo "Invalidation $id"
aws cloudfront wait invalidation-completed --distribution-id "$DISTRIBUTION_ID" --id "$id"

# 7. Prune old snapshots (keep the newest 5, never the live one or the new one)
live="$(aws s3 cp "s3://$BUCKET/data/latest.json" - | jq -r '.version')"
versions=()
while IFS= read -r line; do versions+=("$line"); done < <(
  aws s3api list-objects-v2 --bucket "$BUCKET" --prefix data/ --delimiter / \
    --query 'CommonPrefixes[].Prefix' --output text \
  | tr '\t' '\n' | grep -E '^data/[0-9]{8}T[0-9]{6}Z/$' | sort
)
keep_from=$(( ${#versions[@]} - 5 ))
for i in "${!versions[@]}"; do
  prefix="${versions[$i]}"
  if [ "$i" -ge "$keep_from" ] || [ "$prefix" = "data/$live/" ] || [ "$prefix" = "data/$VERSION/" ]; then
    echo "keep   $prefix"; continue
  fi
  echo "delete $prefix"
  aws s3 rm "s3://$BUCKET/$prefix" --recursive --only-show-errors
done

# 8. Smoke check the live data through CloudFront
served=""
for attempt in $(seq 1 10); do
  served="$(curl -s "$SITE_URL/data/latest.json" | jq -r '.version // empty' 2>/dev/null || true)"
  [ "$served" = "$VERSION" ] && break
  sleep 6
done
[ "$served" = "$VERSION" ] || { echo "CloudFront serves '$served', expected $VERSION" >&2; exit 1; }
type="$(curl -s -o /dev/null -w '%{content_type}' "$SITE_URL/data/$VERSION/index.json")"
[[ "$type" == application/json* ]] || { echo "index.json served as '$type'" >&2; exit 1; }
index="$(curl -sf "$SITE_URL/data/$VERSION/index.json")"
jq -e '.schemaVersion == 1 and (.candidates | length) > 0' <<<"$index" > /dev/null \
  || { echo "index.json of $VERSION does not load" >&2; exit 1; }
slug="$(jq -r '.candidates[0].slug' <<<"$index")"
curl -sf "$SITE_URL/data/$VERSION/candidates/$slug.json" \
  | jq -e --arg s "$slug" '.summary.slug == $s' > /dev/null \
  || { echo "profile $slug does not load" >&2; exit 1; }
echo "Live: $VERSION"
```

The only differences from the workflow are mechanical: `AWS_ARGS` (CI uses
OIDC credentials, no profile), `::error::` annotations replaced by `stderr`,
and a `while read` loop instead of `mapfile` (macOS ships bash 3.2).

Why this order:

- Steps 2–4 before 5: a visitor can only ever be pointed at a version whose
  files are all already there.
- Prune **after** go-live and never by S3 lifecycle rule: if the weekly cron
  ever stops, an age-based rule would eventually delete the version that is
  still live (`00-overview/03-architecture.md`).
- No `--delete` on the data sync: each version is a new prefix.
- Permissions used: `s3:ListBucket`, `s3:GetObject` (reading the live
  pointer), `s3:PutObject`, `s3:DeleteObject` on the bucket and
  `cloudfront:CreateInvalidation` / `cloudfront:GetInvalidation` (the
  `wait`) on the distribution — what the deploy role grants
  (`05-infrastructure/`).

## Part C — how CI runs it (contract with `06-ci-cd/03-refresh-data.md`)

CI does **not** call the script; it runs the same commands as separate
workflow steps (clearer logs per step). The ETL step:

```bash
cd backend
args=(-out dist/data -cache .cache -log-format json)
if [ "$SKIP_PREVIOUS_CHECK" != "true" ] && [ -n "$SITE_URL" ]; then
  args+=(-previous "$SITE_URL/data/latest.json")
fi
go run ./cmd/ingest "${args[@]}"     # env: INGEST_CONTACT from the repository variable
```

The snapshot is passed to the `publish` job as an artifact, and the `publish`
job runs steps 1–8 above.

## Part D — `make publish-data-local` (manual fallback)

Add to the root `Makefile` (uses the `preflight` target defined in
`01-foundation/02-repository-layout.md`):

```makefile
publish-data-local: preflight   ## Publish backend/dist/data by hand (fallback when CI is unavailable)
	@test -n "$(BUCKET)" -a -n "$(DISTRIBUTION_ID)" -a -n "$(SITE_URL)" \
	  || (echo "Usage: make publish-data-local BUCKET=<bucket> DISTRIBUTION_ID=<id> SITE_URL=https://<distribution>.cloudfront.net" && exit 1)
	BUCKET="$(BUCKET)" DISTRIBUTION_ID="$(DISTRIBUTION_ID)" SITE_URL="$(SITE_URL)" \
	  AWS_ARGS="--profile personal" bash backend/scripts/publish-data.sh
```

Typical manual sequence:

```bash
make ingest                                   # writes backend/dist/data/<version> + latest.json
# (optionally compare with the live snapshot first:
#  cd backend && go run ./cmd/ingest -out dist/data -cache .cache -previous https://<distribution>.cloudfront.net/data/latest.json)
make publish-data-local BUCKET=<bucket> DISTRIBUTION_ID=<id> SITE_URL=https://<distribution>.cloudfront.net
```

The bucket, distribution id and site URL come from the CDK stack outputs
(`05-infrastructure/`). `preflight` (account `436812234154` via
`--profile personal`) runs before anything is uploaded, and the script checks
the account again.

## Part E — rollback (one command, no rebuild)

Point the pointer back at the previous version (still in S3 thanks to keep-5):

```bash
PREV=20261102T060000Z    # pick from: aws s3 ls s3://<bucket>/data/ --profile personal
printf '{"schemaVersion":1,"version":"%s","generatedAt":"%s"}\n' "$PREV" "$(date -u +%Y-%m-%dT%H:%M:%SZ)" > /tmp/latest.json
aws s3 cp /tmp/latest.json s3://<bucket>/data/latest.json --profile personal \
  --cache-control "public, max-age=60" --content-type "application/json; charset=utf-8"
aws cloudfront create-invalidation --profile personal --distribution-id <id> --paths "/data/latest.json"
```

Then fix the cause; the next successful refresh moves the pointer forward again.

## Pitfalls

- macOS ships bash 3.2 (no `mapfile`) — the script uses a `while read`
  loop; the prune arithmetic is otherwise identical to the workflow.
- `aws s3 sync` guesses `.sqlite` as `application/octet-stream` and JSON
  without charset — hence explicit content types.
- Do not invalidate `/data/*`: versioned paths never change, and wildcard
  invalidations waste the free quota.
- Publishing must never run if `ingest` failed: in CI `publish` is a
  separate job with `needs: ingest`, which GitHub skips after a failure;
  locally there is no `latest.json` to publish (step 0 of the script).

## Done when

```bash
# Local dry run of validation (no AWS):
cd backend && go run ./cmd/ingest -out dist/data -cache .cache -offline && cat dist/data/latest.json
# Break something on purpose (e.g. set TopN=0 in a scratch branch) → exit code 1, no latest.json.

# After a real publish:
curl -s https://<distribution>.cloudfront.net/data/latest.json | jq .
curl -sI https://<distribution>.cloudfront.net/data/<version>/index.json | grep -i cache-control   # immutable
curl -sI https://<distribution>.cloudfront.net/data/latest.json | grep -i cache-control            # max-age=60
aws s3 ls s3://<bucket>/data/ --profile personal        # at most 5 version prefixes + latest.json
```
