# 03.01 — Go project setup, `internal/pipeline` and the `ingest` command

## Goal

A compilable Go module where:

- the whole ETL is **one function**, `pipeline.Run(ctx, Options) (Result, error)`,
  that the CLI and the integration test both call;
- `cmd/ingest/main.go` only maps flags and environment variables to
  `Options` (it is excluded from coverage, so it must stay trivial);
- every source URL is an `Options` field defaulting to the real one, and the
  clock is injectable — so tests run the real pipeline against an in-process
  fake upstream and get byte-identical output (`09-integration-tests.md`).

## Files to create

```
.gitattributes                         # repository root: backend/testdata/** -text
testdata/textnorm-vectors.json         # repository root: shared Go/TS normalisation vectors (step 02)
backend/
├── go.mod
├── go.sum
├── cmd/ingest/main.go                 # flags + env → pipeline.Options; exit code
├── config/name-overrides.json
├── internal/
│   ├── config/config.go               # decisions (constants) + URLs + DefaultURLs()
│   ├── config/config_test.go
│   ├── pipeline/options.go            # Options, Result, withDefaults
│   ├── pipeline/pipeline.go           # Run: the orchestration order
│   ├── pipeline/options_test.go
│   ├── pipeline/pipeline_test.go      # integration test + golden snapshot (step 09)
│   ├── testsrv/                       # fake upstream, used only by *_test.go (step 09)
│   ├── fetch/        (step 02)
│   ├── csvx/         (step 02)
│   ├── textnorm/     (step 02)
│   ├── tse/          (step 03)
│   ├── selection/    (step 04)
│   ├── camara/       (step 05)
│   ├── amendments/   (step 06)
│   ├── store/        (step 07)
│   ├── export/       (step 07)
│   └── validate/     (step 08)
└── testdata/
    ├── sources/                       # synthetic upstream files, real layouts and encodings (step 09)
    ├── config/name-overrides.json     # overrides used by the integration test
    └── golden/data/                   # expected pipeline output = frontend e2e fixture (step 09)
```

`.gitattributes` (repository root):

```gitattributes
# Fixtures are byte-exact: never convert encodings or line endings (Latin-1, CRLF, BOM matter).
backend/testdata/** -text
```

## 1. Initialise the module

```bash
cd backend
go mod init github.com/Miguel060/projetoAplicado2ifc/backend
go get modernc.org/sqlite@latest golang.org/x/text@latest
go get github.com/google/go-cmp@latest     # test-only (imported from *_test.go files only)
go mod tidy
```

Resulting `go.mod` (versions = whatever `go get` resolved; commit `go.sum`):

```go
module github.com/Miguel060/projetoAplicado2ifc/backend

go 1.25

require (
	github.com/google/go-cmp vX.Y.Z // test-only: readable diffs in tests
	golang.org/x/text vX.Y.Z
	modernc.org/sqlite vX.Y.Z
)
```

No other direct dependencies (see `00-overview/04-tech-stack.md`). Go modules
cannot mark a dependency as test-only; the rule is enforced by review:
`go-cmp` is never imported by a non-test file (a `grep` in step 09's CI
check).

## 2. Configuration — `internal/config/config.go`

Everything that is a *decision* lives here. URLs are **defaults**, not
hard-wired: loaders read them from `fetch.Env.URLs` (step 02), which the
pipeline fills from `Options.URLs`.

```go
package config

const (
	ElectionYear  = 2026
	ElectionRound = 1
	ElectionDate  = "2026-10-04"
	UF            = "SC"

	// Municipality as written in TSE and CGU files (upper case, accents as published).
	MunicipalityName    = "VIDEIRA"
	MunicipalityDisplay = "Videira"
	IBGECode            = "4219309"
	ExpectedTSECode     = "83798" // resolved at run time; a different value only logs a warning

	TopN = 20 // D1: per office, ties at position TopN all included

	OfficeFederal = 6 // CD_CARGO
	OfficeState   = 7

	Legislature      = 57
	LegislatureStart = "2023-02-01"
	BillTypes        = "PL,PLP,PEC,PDL"

	MaxBillsExported       = 50
	MaxRecentVotesExported = 20

	AlescDeputiesURL = "https://www.alesc.sc.gov.br/deputados/"

	// User-Agent template; %s = group contact (env INGEST_CONTACT, mapped by cmd/ingest).
	UserAgentFormat    = "projetoAplicado2ifc/1.0 (+https://github.com/Miguel060/projetoAplicado2ifc; %s)"
	ContactPlaceholder = "<GROUP_CONTACT_EMAIL>"

	HTTPTimeoutSeconds = 120 // per request; big zips are read in ranges
	APIRequestsPerSec  = 4   // Câmara API politeness
	MaxAttempts        = 3   // retries for 429/5xx/network errors

	VersionLayout = "20060102T150405Z"
)

// Years with roll-call votes and CEAP in legislature 57 (bulk files per year).
var ChamberYears = []int{2023, 2024, 2025, 2026}

// Verified at implementation time, or left empty → tse_page_url = NULL (see 02-data-sources/01-tse.md).
const DivulgaCandURLPattern = ""

// Portal da Transparência amendment detail page, e.g. "https://portaldatransparencia.gov.br/emendas/detalhe?codigoEmenda=%s".
// UNVERIFIED (the site answers scripted requests with a "Human Verification" page, HTTP 405).
// Leave empty until someone confirms it in a browser → amendment.url / Amendment.url = null.
const PortalAmendmentURLPattern = ""

// URLs are every upstream base URL. Tests point them at the fake upstream.
type URLs struct {
	TSEBase          string // ".../odsele/" (trailing slash)
	TSEPhotosSC      string // full URL of the SC photos zip
	CamaraAPI        string // ".../api/v2" (no trailing slash)
	CamaraFiles      string // ".../arquivos"
	CamaraCotas      string // ".../cotas"
	CGUAmendmentsZip string
	CGUAmendmentsAlt string // used when the direct URL answers 404
}

func DefaultURLs() URLs {
	return URLs{
		TSEBase:          "https://cdn.tse.jus.br/estatistica/sead/odsele/",
		TSEPhotosSC:      "https://cdn.tse.jus.br/estatistica/sead/eleicoes/eleicoes2026/fotos/foto_cand2026_SC_div.zip",
		CamaraAPI:        "https://dadosabertos.camara.leg.br/api/v2",
		CamaraFiles:      "https://dadosabertos.camara.leg.br/arquivos",
		CamaraCotas:      "https://www.camara.leg.br/cotas",
		CGUAmendmentsZip: "https://dadosabertos-download.cgu.gov.br/PortalDaTransparencia/saida/emendas-parlamentares/EmendasParlamentares.zip",
		CGUAmendmentsAlt: "https://portaldatransparencia.gov.br/download-de-dados/emendas-parlamentares/UNICO",
	}
}

// UserAgent formats the header; an empty contact becomes the placeholder.
func UserAgent(contact string) string {
	if strings.TrimSpace(contact) == "" {
		contact = ContactPlaceholder
	}
	return fmt.Sprintf(UserAgentFormat, strings.TrimSpace(contact))
}
```

### The contact e-mail (`INGEST_CONTACT`)

| Where | How |
|---|---|
| CI (`refresh-data.yml`) | **Required.** Repository variable `INGEST_CONTACT`, passed as `env: INGEST_CONTACT: ${{ vars.INGEST_CONTACT }}`. `cmd/ingest` sets `Options.RequireContact = (GITHUB_ACTIONS == "true")`; `Run` fails before any request when the contact is empty. |
| Local runs | Optional: `export INGEST_CONTACT=<the group contact e-mail>` before `make ingest`; if unset, the placeholder `<GROUP_CONTACT_EMAIL>` is sent and a warning is logged. |
| Repository | Never committed — not in code, config files, fixtures or docs. |

Resulting header: `projetoAplicado2ifc/1.0 (+https://github.com/Miguel060/projetoAplicado2ifc; <INGEST_CONTACT>)`.

## 3. `config/name-overrides.json`

Empty at first; filled only when the ETL logs an unmatched amendment author
(step 06). Keyed by the CGU **author code**, value = Câmara deputy id.

```json
{
  "comment": "CGU 'Código do Autor da Emenda' -> Câmara deputy id. Only for authors the automatic name match misses. Add the author name in 'note' for reviewers.",
  "overrides": []
}
```

The integration test uses its own file, `backend/testdata/config/name-overrides.json`,
with one synthetic override (step 09).

## 4. `internal/pipeline` — Options, Result, Run

```go
package pipeline

type Options struct {
	OutDir        string // required: writes <OutDir>/<Version>/… and, after validation, <OutDir>/latest.json
	CacheDir      string // "" → no disk cache (tests use t.TempDir())
	Offline       bool
	Refresh       bool
	Version       string // "" → Now().UTC().Format(config.VersionLayout)
	Previous      string // live latest.json (URL or path); "" → skip comparison
	OverridesPath string // "" → no overrides

	Contact        string // group contact for the User-Agent
	RequireContact bool   // true in CI: empty Contact is an error
	GitSHA         string // recorded in ingest_run.git_sha; "" → NULL

	URLs         config.URLs // zero value → config.DefaultURLs(); unit tests may point these at the fake upstream,
	                         // the integration test keeps the real URLs and injects HTTP.Transport instead (step 09)
	TopN         int         // 0 → config.TopN
	MinCandidates int        // 0 → 600: sanity floor of SC deputy candidates in the TSE file; the integration test sets 1
	ChamberYears []int       // nil → config.ChamberYears
	HTTP         fetch.Tuning // zero value → production retries/backoff/rate (step 02)

	Now func() time.Time // nil → time.Now; every timestamp in the output comes from here
	Log *slog.Logger     // nil → slog.New(slog.DiscardHandler)
}

type Result struct {
	Version     string
	SnapshotDir string         // <OutDir>/<Version>
	Counts      map[string]int // per office
	Warnings    []string       // e.g. unmatched amendment authors (also logged)
}

// withDefaults fills zero values; it never touches the environment.
func (o Options) withDefaults() (Options, error)

func Run(ctx context.Context, o Options) (Result, error)
```

`Run` never reads environment variables, flags or the real clock directly:
everything arrives through `Options`. That is what makes the integration test
deterministic.

### Orchestration order (`pipeline.go`)

```go
func Run(ctx context.Context, opts Options) (Result, error) {
	o, err := opts.withDefaults()
	if err != nil {
		return Result{}, err
	}
	if o.RequireContact && strings.TrimSpace(o.Contact) == "" {
		return Result{}, errors.New("INGEST_CONTACT must be set in CI")
	}
	if strings.TrimSpace(o.Contact) == "" {
		o.Log.Warn("INGEST_CONTACT is not set; using placeholder in User-Agent")
	}
	started := o.Now().UTC()
	env := &fetch.Env{
		Client: fetch.New(fetch.Options{
			UserAgent: config.UserAgent(o.Contact), CacheDir: o.CacheDir,
			Offline: o.Offline, Refresh: o.Refresh, Tuning: o.HTTP, Now: o.Now, Log: o.Log,
		}),
		Sources:      fetch.NewSources(),
		URLs:         o.URLs,
		ChamberYears: o.ChamberYears,
		Now:          o.Now,
		Log:          o.Log,
	}

	cands, err := tse.LoadCandidates(ctx, env, o.MinCandidates)         // step 03 (CPF in memory only)
	votes, err := tse.LoadVotes(ctx, env)                               // step 03
	sel, err := selection.Select(cands, votes, o.TopN)                  // step 04
	assets, err := tse.LoadAssets(ctx, env, sel.IDs())                  // step 03
	finance, err := tse.LoadFinance(ctx, env, sel.IDs())
	social, err := tse.LoadSocial(ctx, env, sel.IDs())
	photos, err := tse.LoadPhotos(ctx, env, sel.IDs())
	deps, err := camara.MatchDeputies(ctx, env, sel)                    // step 05
	sel.ForgetCPFs()                                                    // D10
	cands = nil                                                         // drop the 645 CPFs too
	bills, err := camara.LoadBills(ctx, env, deps)
	rcv, err := camara.LoadRollCallVotes(ctx, env, deps)
	ceap, err := camara.LoadExpenses(ctx, env, deps)
	amds, warns, err := amendments.Load(ctx, env, deps, o.OverridesPath) // step 06

	dir := filepath.Join(o.OutDir, o.Version)
	ds := store.Dataset{Run: store.RunMeta{Version: o.Version, StartedAt: started,
		FinishedAt: o.Now().UTC(), GitSHA: o.GitSHA, TopN: o.TopN, /* … */} /* , … */}
	err = store.Write(filepath.Join(dir, "dataset.sqlite"), ds)         // step 07
	err = export.Write(dir, filepath.Join(dir, "dataset.sqlite"), photos)
	err = validate.Check(ctx, dir, o.Previous, env.Client)              // step 08 (aborts → no latest.json)
	err = export.WriteLatest(o.OutDir, o.Version, ds.Run.FinishedAt)    // the go-live marker
	return Result{Version: o.Version, SnapshotDir: dir, Counts: sel.Counts(), Warnings: warns}, nil
}
```

(Each `err` is checked and wrapped — `fmt.Errorf("tse candidates: %w", err)` —
omitted above for readability.)

## 5. `cmd/ingest/main.go` — flags and environment → Options

| Flag | Default | `Options` field |
|---|---|---|
| `-out` | `dist/data` | `OutDir` |
| `-cache` | `.cache` | `CacheDir` |
| `-offline` | `false` | `Offline` — use only cached files; fail if one is missing |
| `-refresh` | `false` | `Refresh` — ignore the cache |
| `-version` | `""` (= now, `20060102T150405Z`) | `Version` |
| `-previous` | `""` | `Previous` — the **live** `latest.json`: an `http(s)://` URL **or** a local file path. Validation follows it to the live `<version>/index.json` and compares counts (step 08, check 12). Empty flag, HTTP 404 or missing file ⇒ "no live snapshot yet" → comparison skipped with an **info** log, not an error. Any other failure (5xx after retries, invalid JSON) **is** an error. CI passes `"$SITE_URL/data/latest.json"` unless the manual `skip_previous_check` input is set. |
| `-overrides` | `config/name-overrides.json` | `OverridesPath` |
| `-log-format` | `text` | `Log` = text or JSON `slog` handler on stderr |

| Environment | `Options` field |
|---|---|
| `INGEST_CONTACT` | `Contact` |
| `GITHUB_ACTIONS == "true"` | `RequireContact` |
| `GITHUB_SHA` (else `debug.ReadBuildInfo()` `vcs.revision`) | `GitSHA` |

```go
func main() {
	f := parseFlags()
	o := pipeline.Options{
		OutDir: f.Out, CacheDir: f.Cache, Offline: f.Offline, Refresh: f.Refresh,
		Version: f.Version, Previous: f.Previous, OverridesPath: f.Overrides,
		Contact:        os.Getenv("INGEST_CONTACT"),
		RequireContact: os.Getenv("GITHUB_ACTIONS") == "true",
		GitSHA:         gitSHA(),
		Log:            newLogger(f.LogFormat),
	}
	res, err := pipeline.Run(context.Background(), o)
	if err != nil {
		o.Log.Error("ingest failed", "err", err)
		os.Exit(1)
	}
	o.Log.Info("ingest done", "version", res.Version, "counts", res.Counts)
}
```

Exit codes: `0` success; `1` any error (fetch, parse, validation). On a
non-zero exit `latest.json` is **not** written, so nothing can be published.

## 6. Run it

```bash
cd backend
go run ./cmd/ingest -out dist/data -cache .cache          # first run downloads (~70 MB with range reads)
go run ./cmd/ingest -out dist/data -cache .cache -offline # re-run from cache in seconds
```

Or from the repository root: `make ingest`.

## Pitfalls

- Never read `AWS_*` or any credential in the Go program: publishing is done
  by the AWS CLI (step 08). The binary has no cloud access at all.
- Never call `time.Now()`, `os.Getenv` or a hard-coded **upstream** URL inside
  `internal/…` — use `Options`/`fetch.Env`. Grep before each PR:
  `grep -rnE "time\.Now\(\)|os\.Getenv|cdn\.tse|dadosabertos|camara\.leg\.br/cotas|cgu\.gov\.br|portaldatransparencia" internal --include=*.go | grep -v _test.go`
  must only show `config.go` (default URLs/patterns), `testsrv/` (the transport
  mapping of real hosts to the fake upstream) and `options.go` (the
  `Now` default). Citizen-facing link templates written into the output
  (e.g. `https://www.camara.leg.br/propostas-legislativas/<id>`) are fine.
- Logs must never print CPFs, raw TSE rows or HTTP bodies of personal-data files.

## Tests for this step

| File | Test | Cases |
|---|---|---|
| `internal/config/config_test.go` | `TestUserAgent` | contact set → exact header string; `""` and `"   "` → placeholder; surrounding spaces trimmed |
| | `TestDefaultURLs` | every field non-empty, `https://`, parses with `url.Parse`; `TSEBase` ends with `/`; `CamaraAPI` does not |
| | `TestDecisionConstants` | guards against accidental edits: `TopN == 20`, `IBGECode == "4219309"`, offices 6/7, `BillTypes` has exactly PL, PLP, PEC, PDL |
| `internal/pipeline/options_test.go` | `TestWithDefaults` | zero `Options` (with `OutDir`) → `URLs == DefaultURLs()`, `TopN == 20`, `ChamberYears` default, `Log` non-nil, `Now` non-nil; explicit values are kept; `MinCandidates` defaults to 600; `Version` = injected `Now` formatted (`2099-01-01T00:00:00Z` → `20990101T000000Z`); missing `OutDir` → error; malformed `Version` (`"abc"`) → error |
| | `TestRunRequiresContactInCI` | `RequireContact: true`, `Contact: ""` → error **and** the fake upstream (`testsrv`) recorded **zero** requests |
| | `TestRunPlaceholderContact` | `Contact: ""`, `RequireContact: false` → every request's `User-Agent` contains `<GROUP_CONTACT_EMAIL>` (asserted from `testsrv` request log) |
| `internal/pipeline/pipeline_test.go` | integration + golden | see `09-integration-tests.md` |

Fixtures: none beyond the shared `testsrv` fixture set (step 09).

`cmd/ingest` has no tests and is excluded from coverage by construction
(`-coverpkg=./internal/...`); keep it to flag/env mapping only.

## Done when

```bash
cd backend
gofmt -l .               # prints nothing
go vet ./...             # no findings
go build ./...           # compiles
go run ./cmd/ingest -h   # shows the flags above
cd .. && make test-backend   # all tests pass, "coverage NN.N%" with NN.N >= 85
```
