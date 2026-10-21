# 03.07 — Store (SQLite) and export (JSON)

## Goal

1. Write everything to a **fresh** `dataset.sqlite` using the schema in
   `01-foundation/03-data-model.md`.
2. Export the JSON snapshot defined in `01-foundation/04-json-contract.md`
   **by querying that SQLite file**, so the published JSON can never disagree
   with the published dataset.
3. Copy the selected photos.

Output of one run:

```
backend/dist/data/
├── <version>/
│   ├── dataset.sqlite
│   ├── index.json
│   ├── candidates/<slug>.json     (one per selected candidate)
│   └── photos/<tseId>.jpg
└── latest.json                    (written only by step 08, after validation)
```

## Files to create

```
backend/internal/store/schema.sql   # exactly the DDL of 01-foundation/03-data-model.md
backend/internal/store/store.go     # Dataset, Write
backend/internal/export/types.go    # Go mirror of the TS contract
backend/internal/export/export.go   # Write, WriteLatest
backend/internal/export/slug.go     # Slug, uniqueness check
```

## Part A — `store.Write`

```go
//go:embed schema.sql
var schemaSQL string

type Dataset struct {
	Run        RunMeta              // version, started/finished, git sha, election, municipality codes, topN
	Sources    []fetch.Source
	Selected   *selection.Result
	Assets     map[string][]tse.Asset
	Finance    map[string]tse.Finance
	Social     map[string][]tse.SocialLink
	PhotoPaths map[string]string    // tseID → "photos/<tseID>.jpg" (only when a photo exists)
	Deputies   camara.Deputies
	Bills      map[int][]camara.Bill
	Votes      map[int][]camara.RollCallVote
	Expenses   map[int]camara.Expenses
	Amendments map[int][]amendments.Amendment
}

func Write(path string, ds Dataset) error
```

Steps:

1. `os.MkdirAll(dir)`; `os.Remove(path)` if it exists — the file is always rebuilt.
2. `db, _ := sql.Open("sqlite", path)` (driver name of `modernc.org/sqlite`),
   then `PRAGMA journal_mode=DELETE; PRAGMA foreign_keys=ON;` (single file,
   no `-wal`/`-shm` side files to forget when uploading).
3. `db.Exec(schemaSQL)`.
4. One transaction; prepared statements; insert in FK order:
   `source` → `ingest_run` → `candidate` → `vote_total` (municipality with
   rank, state with `rank = NULL`) → `selection` → `asset` →
   `campaign_revenue_by_origin` → `campaign_expense_by_type` → `social_link`
   → `chamber_deputy` → `chamber_bill` → `chamber_roll_call_vote` →
   `chamber_expense_by_year` → `chamber_expense_by_category` → `amendment`
   (→ `amendment_beneficiary` if the contract change of step 06 is accepted).
5. `candidate.slug` comes from `export.Slug` (Part C) so SQLite and JSON agree.
6. Commit, `VACUUM`, close.
7. Assert: `SELECT COUNT(*) FROM pragma_table_info(t) WHERE name LIKE '%cpf%'`
   is 0 for every table (defence in depth for D10).

`git_sha`: `os.Getenv("GITHUB_SHA")` in CI; locally
`debug.ReadBuildInfo()` → `vcs.revision`, else NULL.

## Part B — export types (mirror of the TS contract)

Rules that make Go produce exactly the contract:

- **No `omitempty` anywhere** — absent values are `null`, never missing.
- Nullable scalars are pointers (`*string`, `*int`); nullable blocks are
  pointers to structs (`*ChamberActivity`).
- Slices are initialised to `[]T{}` before marshalling, so they encode as
  `[]`, never `null`.
- Money rounded to 2 decimals, shares to 4, before marshalling.

```go
type CandidateSummary struct {
	Slug                 string     `json:"slug"`
	TSEID                string     `json:"tseId"`
	BallotName           string     `json:"ballotName"`
	Number               string     `json:"number"`
	Office               string     `json:"office"`
	Party                Party      `json:"party"`
	Federation           *Federation `json:"federation"`
	Result               Result     `json:"result"`
	Votes                SummaryVotes `json:"votes"`
	BornInMunicipality   bool       `json:"bornInMunicipality"`
	SelectionReasons     []string   `json:"selectionReasons"`
	HasChamberRecord     bool       `json:"hasChamberRecord"`
	RunningForReelection bool       `json:"runningForReelection"`
	Photo                *string    `json:"photo"`
	SearchText           string     `json:"searchText"`
}
type Amendment struct {
	Code            string  `json:"code"`
	Year            int     `json:"year"`
	Type            string  `json:"type"`
	Number          string  `json:"number"`
	FunctionName    *string `json:"functionName"`
	SubfunctionName *string `json:"subfunctionName"`
	Committed       float64 `json:"committed"`
	Liquidated      float64 `json:"liquidated"`
	Paid            float64 `json:"paid"`
	URL             *string `json:"url"` // nullable: Portal detail page only when the pattern is verified
	// + receivedInMunicipality / beneficiaries if the step-06 contract change is accepted
}
// … DataIndex, CandidateProfile, ChamberActivity, Bill, RollCallVote,
//   Amendments, Amendment, Source, LatestPointer — one Go struct per TS interface,
//   field names = camelCase json tags, same order as the contract.
```

## Part C — slugs and search text

```go
func Slug(ballotName, number string) string {
	return textnorm.Slug(ballotName) + "-" + number // "fulano-de-tal-1234"
}
```

**Every slug must match `^[a-z0-9-]+$`** — lowercase ASCII letters, digits
and hyphens only, and in particular **no dots**: the CloudFront SPA-rewrite
function serves `index.html` only for extension-less paths, so a slug with a
`.` would make `/candidatos/<slug>` look like a file and return 404.
`textnorm.Slug` guarantees it (accents stripped, every other run of
characters → `-`), and the export **asserts** it anyway:

```go
var slugRe = regexp.MustCompile(`^[a-z0-9-]+$`)

func checkSlug(slug string) error {
	if !slugRe.MatchString(slug) || strings.HasPrefix(slug, "-") || strings.Contains(slug, "--") {
		return fmt.Errorf("invalid slug %q (must match ^[a-z0-9-]+$)", slug)
	}
	return nil
}
```

Build all slugs first; if any fails `checkSlug`, or two candidates produce
the same slug, **abort** (`duplicate slug %q`) — by construction it cannot happen (numbers are unique
per office; federal numbers have 4 digits and state numbers 5).

```go
searchText := textnorm.Norm(strings.Join(nonEmpty(
	c.BallotName, c.FullName, deref(c.SocialName), c.Number, c.PartyAcronym), " "))
```

## Part D — `export.Write(dir string, dbPath string, photos map[string][]byte)`

Open the SQLite file read-only (`file:<path>?mode=ro`) and:

1. **index.json**
   - `schemaVersion: 1`, `version`, `generatedAt` (= run finished time, UTC);
   - `election` from `ingest_run` + config (`municipality.name = config.MunicipalityDisplay`,
     `ibgeCode`, `tseCode`);
   - `selection: { topN, offices: ["federal_deputy","state_deputy"] }`;
   - `counts` per office;
   - `sources`: map id → `Source` (all 11 ids must be present; step 08 checks);
   - `dataset: { path: "dataset.sqlite", bytes, sha256 }` — computed by
     streaming the file through `sha256.New()` (hex, lower case);
   - `candidates`: summaries ordered by office (federal first), municipality
     rank asc, ballot name asc.
2. **candidates/<slug>.json** for each candidate, following
   `CandidateProfile`:
   - `identity` ← `candidate`;
   - `votes.municipality` ← `vote_total(scope='municipality')` +
     `candidatesInOffice` from `ingest_run`/selection metadata;
     `votes.state` ← `vote_total(scope='state')`;
   - `assets` ← `asset` ordered by `ord`; `total` = sum; `declared` ←
     `candidate.declared_assets`;
   - `campaignFinance` ← the two aggregate tables, each `desc by total`;
     totals = sums;
   - `social.links` ← `social_link` by `ord`; `host` = URL host without `www.`;
   - `chamber` ← `null` when no `chamber_deputy` row; else:
     - `bills.total` = count; `byType` = counts desc by count, then type;
       `items` = 50 most recent (`presented_at` desc, `bill_id` desc);
     - `rollCallVotes.total` = count; `byVote` desc by count, then vote;
       `recent` = 20 most recent by `voted_at` desc; `date = voted_at[:10]`;
     - `expenses.total` = sum of `byYear`; `byYear` asc; `byCategory` desc;
   - `amendments` ← `null` when `chamber` is `null`; else totals + items
     ordered as in step 06 (empty `items: []` when none reached Videira);
     each item's `url` ← `amendment.url` — **`null`** while the Portal da
     Transparência detail-page pattern is unverified (`config.PortalAmendmentURLPattern`
     empty); the UI then links to the `cgu_amendments` dataset page instead;
   - `stateAssembly` ← `{ profileUrl: config.AlescDeputiesURL }` when
     `office = 'state_deputy' AND running_for_reelection = 1`, else `null`;
   - `officialLinks.tseCandidatePage` ← `candidate.tse_page_url` (NULL unless verified).
3. **photos/<tseId>.jpg** — write the bytes kept by `LoadPhotos`; set
   `photo` in the summary only when the file was written.

Write each JSON file to `<name>.tmp` then `os.Rename` (no half-written files
if the process dies). Use `json.NewEncoder(f)` with `SetEscapeHTML(false)`;
compact output (no indentation) — the files are small and served gzipped.

## Part E — `export.WriteLatest(out, version string, finished time.Time)`

Called by `main()` **only after `validate.Check` passes** (step 08):

```json
{ "schemaVersion": 1, "version": "<version>", "generatedAt": "<finished, RFC 3339 UTC>" }
```

Written to `<out>/latest.json` via tmp + rename.

## Determinism

Two runs over the same cached inputs must produce byte-identical JSON
(except `version`, `generatedAt`, `fetchedAt`). That is what makes snapshot
diffs reviewable:

- every list has an explicit sort with a final unique tie-breaker (id/code);
- maps are only marshalled where the contract uses `Record<…>` (Go sorts map
  keys when encoding);
- floats rounded before encoding.

## Pitfalls

- A `nil` slice marshals to `null` and breaks the frontend's `.map()` — always
  initialise.
- `modernc.org/sqlite` registers driver name `"sqlite"` (not `"sqlite3"`).
- Do not export from the in-memory structs "because it is easier" — the whole
  point is that the JSON is a view of the published dataset.
- Never include `full_name`-based or CPF-based identifiers in file names; only
  slug and `tseId`.

## Done when

```bash
cd backend
V=$(ls dist/data | grep -v latest.json | sort | tail -1)
ls dist/data/$V/candidates | wc -l                     # ≈ 40–45
jq '.counts, (.candidates | length), (.sources | keys)' dist/data/$V/index.json
jq '.candidates[0] | keys' dist/data/$V/index.json     # exactly the CandidateSummary fields
f=$(ls dist/data/$V/candidates | head -1)
jq '{chamber: (.chamber != null), amendments: (.amendments != null), assets: (.assets.items | length)}' dist/data/$V/candidates/$f
shasum -a 256 dist/data/$V/dataset.sqlite; jq -r .dataset.sha256 dist/data/$V/index.json   # identical
grep -rl '"cpf"' dist/data/$V || echo "no cpf anywhere"
```
