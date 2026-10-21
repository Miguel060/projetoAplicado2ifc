# 03.01 — Go project setup and the `ingest` command

## Goal

A compilable Go module with the `ingest` command skeleton, configuration
constants, flags and the orchestration order every later step plugs into.

## Files to create

```
backend/
├── go.mod
├── go.sum
├── cmd/ingest/main.go
├── config/name-overrides.json
└── internal/
    ├── config/config.go
    ├── fetch/        (step 02)
    ├── csvx/         (step 02)
    ├── textnorm/     (step 02)
    ├── tse/          (step 03)
    ├── selection/    (step 04)
    ├── camara/       (step 05)
    ├── amendments/   (step 06)
    ├── store/        (step 07)
    ├── export/       (step 07)
    └── validate/     (step 08)
```

## 1. Initialise the module

```bash
cd backend
go mod init github.com/Miguel060/projetoAplicado2ifc/backend
go get modernc.org/sqlite@latest golang.org/x/text@latest
go mod tidy
```

Resulting `go.mod` (versions = whatever `go get` resolved; commit `go.sum`):

```go
module github.com/Miguel060/projetoAplicado2ifc/backend

go 1.25

require (
	golang.org/x/text vX.Y.Z
	modernc.org/sqlite vX.Y.Z
)
```

No other direct dependencies (see `00-overview/04-tech-stack.md`).

## 2. Configuration constants — `internal/config/config.go`

Everything that is a *decision* lives here, not scattered in code.

```go
package config

const (
	ElectionYear  = 2026
	ElectionRound = 1
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

	// User-Agent template; %s = contact from env INGEST_CONTACT (see UserAgent()).
	userAgentFormat    = "projetoAplicado2ifc/1.0 (+https://github.com/Miguel060/projetoAplicado2ifc; %s)"
	contactEnvVar      = "INGEST_CONTACT"
	contactPlaceholder = "<GROUP_CONTACT_EMAIL>"

	HTTPTimeoutSeconds = 120 // per request; big zips are read in ranges
	APIRequestsPerSec  = 4   // Câmara API politeness
	MaxAttempts        = 3   // retries for 429/5xx/network errors
)

// Years with roll-call votes and CEAP in legislature 57 (bulk files per year).
var ChamberYears = []int{2023, 2024, 2025, 2026}

// Base URLs (sources are defined in step 02; listed here for one place to edit).
const (
	TSEBase          = "https://cdn.tse.jus.br/estatistica/sead/odsele/"
	TSEPhotosSC      = "https://cdn.tse.jus.br/estatistica/sead/eleicoes/eleicoes2026/fotos/foto_cand2026_SC_div.zip"
	CamaraAPI        = "https://dadosabertos.camara.leg.br/api/v2"
	CamaraFiles      = "https://dadosabertos.camara.leg.br/arquivos"
	CamaraCotas      = "https://www.camara.leg.br/cotas"
	CGUAmendmentsZip = "https://dadosabertos-download.cgu.gov.br/PortalDaTransparencia/saida/emendas-parlamentares/EmendasParlamentares.zip"
	CGUAmendmentsAlt = "https://portaldatransparencia.gov.br/download-de-dados/emendas-parlamentares/UNICO"
)

// Verified at implementation time, or left empty → tse_page_url = NULL (see 02-data-sources/01-tse.md).
const DivulgaCandURLPattern = ""

// Portal da Transparência amendment detail page, e.g. "https://portaldatransparencia.gov.br/emendas/detalhe?codigoEmenda=%s".
// UNVERIFIED (the site answers scripted requests with a "Human Verification" page, HTTP 405).
// Leave empty until someone confirms it in a browser → amendment.url / Amendment.url = null.
const PortalAmendmentURLPattern = ""

// UserAgent builds the User-Agent sent to every government server.
// The group contact e-mail is NOT committed: it comes from the INGEST_CONTACT
// environment variable (a GitHub repository variable in CI). Locally, when it is
// unset, the placeholder is used and a warning is logged once.
func UserAgent(log *slog.Logger) string {
	contact := strings.TrimSpace(os.Getenv(contactEnvVar))
	if contact == "" {
		log.Warn("INGEST_CONTACT is not set; using placeholder in User-Agent", "placeholder", contactPlaceholder)
		contact = contactPlaceholder
	}
	return fmt.Sprintf(userAgentFormat, contact)
}
```

### The contact e-mail (`INGEST_CONTACT`)

| Where | How |
|---|---|
| CI (`refresh-data.yml`) | **Required.** Repository variable `INGEST_CONTACT`, passed as `env: INGEST_CONTACT: ${{ vars.INGEST_CONTACT }}`. `main()` fails fast when running in CI (`GITHUB_ACTIONS=true`) and the variable is empty. |
| Local runs | Optional: `export INGEST_CONTACT=<the group contact e-mail>` before `make ingest`; if unset, the placeholder `<GROUP_CONTACT_EMAIL>` is sent and a warning is logged. |
| Repository | Never committed — not in code, config files or docs. |

Resulting header: `projetoAplicado2ifc/1.0 (+https://github.com/Miguel060/projetoAplicado2ifc; <INGEST_CONTACT>)`.

```go
// in run(), before creating the HTTP client:
if os.Getenv("GITHUB_ACTIONS") == "true" && strings.TrimSpace(os.Getenv("INGEST_CONTACT")) == "" {
	return errors.New("INGEST_CONTACT must be set in CI (repository variable)")
}
ua := config.UserAgent(log)
client := fetch.New(fetch.Options{UserAgent: ua, /* … */})
```


## 3. `config/name-overrides.json`

Empty at first; filled only when the ETL logs an unmatched amendment author
(step 06). Keyed by the CGU **author code**, value = Câmara deputy id.

```json
{
  "comment": "CGU 'Código do Autor da Emenda' -> Câmara deputy id. Only for authors the automatic name match misses. Add the author name in 'note' for reviewers.",
  "overrides": [
    { "authorCode": "0000", "deputyId": 0, "note": "EXAMPLE – delete" }
  ]
}
```

## 4. Flags — `cmd/ingest/main.go`

| Flag | Default | Meaning |
|---|---|---|
| `-out` | `dist/data` | Output root. The run writes `<out>/<version>/…` and, only after validation, `<out>/latest.json`. |
| `-cache` | `.cache` | Download cache directory (keyed by URL hash). |
| `-offline` | `false` | Use only cached files; fail if one is missing. For fast local iteration on parsing/export. |
| `-refresh` | `false` | Ignore the cache and re-download everything (CI starts with an empty cache anyway). |
| `-version` | UTC now, `20060102T150405Z` | Override the snapshot version (useful to reproduce a run). |
| `-previous` | `""` | The **live** `latest.json`: an `http(s)://` URL **or** a local file path. Validation follows it to the live `<version>/index.json` (same base URL / sibling directory) and compares counts (step 08, check 12). Empty flag, HTTP 404 or missing file ⇒ "no live snapshot yet" → comparison skipped with an **info** log, not an error (the very first run has no live data). Any other failure (5xx after retries, invalid JSON) **is** an error. CI passes `"$SITE_URL/data/latest.json"` unless the manual `skip_previous_check` input is set. |
| `-overrides` | `config/name-overrides.json` | Amendment author overrides. |
| `-log-format` | `text` | `text` locally, `json` in CI. |

Environment: `INGEST_CONTACT` (see §2) — required in CI, optional locally.

Exit codes: `0` success; `1` any error (fetch, parse, validation). On a
non-zero exit `latest.json` is **not** written, so nothing can be published.

## 5. `main()` orchestration order

```go
func main() {
	f := parseFlags()
	log := newLogger(f.LogFormat)
	started := time.Now().UTC()
	version := f.Version
	if version == "" {
		version = started.Format("20060102T150405Z")
	}
	if err := run(context.Background(), f, log, version, started); err != nil {
		log.Error("ingest failed", "err", err)
		os.Exit(1)
	}
}

func run(ctx context.Context, f flags, log *slog.Logger, version string, started time.Time) error {
	if os.Getenv("GITHUB_ACTIONS") == "true" && strings.TrimSpace(os.Getenv("INGEST_CONTACT")) == "" {
		return errors.New("INGEST_CONTACT must be set in CI")
	}
	client := fetch.New(fetch.Options{UserAgent: config.UserAgent(log), CacheDir: f.Cache, Offline: f.Offline, Refresh: f.Refresh, Log: log})
	sources := fetch.NewSources() // provenance registry, filled by every loader

	// 1. TSE candidates (all SC deputy candidates, CPF in memory only)
	cands, err := tse.LoadCandidates(ctx, client, sources)            // step 03
	// 2. TSE votes: resolves municipality code, per-candidate totals
	votes, err := tse.LoadVotes(ctx, client, sources)                 // step 03
	// 3. Selection rule (D1)
	sel, err := selection.Select(cands, votes, config.TopN)           // step 04
	// 4. TSE enrichment for the selected only
	assets, err := tse.LoadAssets(ctx, client, sources, sel.IDs())     // step 03
	finance, err := tse.LoadFinance(ctx, client, sources, sel.IDs())
	social, err := tse.LoadSocial(ctx, client, sources, sel.IDs())
	photos, err := tse.LoadPhotos(ctx, client, sources, sel.IDs())     // bytes in memory, ~45 × 30 KB
	// 5. Câmara: match by CPF, then forget CPFs
	deps, err := camara.MatchDeputies(ctx, client, sources, sel)       // step 05
	sel.ForgetCPFs()                                                   // D10: zero every CPF string
	bills, err := camara.LoadBills(ctx, client, sources, deps)
	rcv, err := camara.LoadRollCallVotes(ctx, client, sources, deps)
	ceap, err := camara.LoadExpenses(ctx, client, sources, deps)
	// 6. CGU amendments to the municipality, attributed to matched deputies
	amds, err := amendments.Load(ctx, client, sources, deps, f.Overrides) // step 06
	// 7. Persist + export
	dir := filepath.Join(f.Out, version)
	ds := store.Dataset{ /* everything above + run metadata */ }
	if err := store.Write(filepath.Join(dir, "dataset.sqlite"), ds); err != nil { … }   // step 07
	if err := export.Write(dir, ds, photos); err != nil { … }
	// 8. Validate (aborts on failure; latest.json not written)
	if err := validate.Check(ctx, dir, f.Previous, client); err != nil { … }           // step 08
	// 9. Go-live marker for the publisher
	return export.WriteLatest(f.Out, version, time.Now().UTC())
}
```

(Each `err` is checked and wrapped — `fmt.Errorf("tse candidates: %w", err)` —
omitted above for readability.)

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
- Keep `config.go` the only place with magic values; reviewers check decisions there.
- Logs must never print CPFs, raw TSE rows or HTTP bodies of personal-data files.

## Done when

```bash
cd backend
gofmt -l .            # prints nothing
go vet ./...          # no findings
go build ./...        # compiles
go run ./cmd/ingest -h   # shows the flags above
```
