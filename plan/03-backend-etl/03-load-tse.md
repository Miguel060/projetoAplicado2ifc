# 03.03 — Load TSE data

## Goal

Turn the TSE files described in `02-data-sources/01-tse.md` into typed Go
values: all SC deputy candidates (with CPF **in memory only**), their votes
in Videira and in SC, and — for the selected candidates only — assets,
campaign finance, social links and photos.

## Files to create

```
backend/internal/tse/sources.go     # URLs, member names, source ids/names/dataset URLs
backend/internal/tse/candidates.go  # LoadCandidates (consulta_cand + complementar + historico)
backend/internal/tse/votes.go       # LoadVotes
backend/internal/tse/assets.go      # LoadAssets
backend/internal/tse/finance.go     # LoadFinance
backend/internal/tse/social.go      # LoadSocial
backend/internal/tse/photos.go      # LoadPhotos
backend/internal/tse/result.go      # ResultCode mapping
```

## Types

```go
package tse

type Office string
const (
	FederalDeputy Office = "federal_deputy" // CD_CARGO 6
	StateDeputy   Office = "state_deputy"   // CD_CARGO 7
)

type Candidate struct {
	TSEID         string // SQ_CANDIDATO
	OfficeCode    int
	Office        Office
	Number        string
	BallotName    string
	FullName      string
	SocialName    *string
	CPF           string // in memory only — never stored/exported/logged (D10)
	BirthDate     *time.Time
	AgeAtInauguration *int
	BirthCity     *string
	BirthUF       *string
	Occupation    *string
	Education     *string
	PartyNumber   int
	PartyAcronym  string
	PartyName     string
	FederationAcronym *string
	FederationName    *string
	CandidacyStatus string // DS_SITUACAO_CANDIDATO_TOT (complementary file), e.g. "DEFERIDO"
	ResultCode     string  // see result.go
	ResultTSELabel string  // DS_SIT_TOT_TURNO verbatim
	RunningForReelection bool // derived from historico (elected to same office in 2022)
	DeclaredAssets bool       // ST_DECLARAR_BENS == "S"
}

type Candidates map[string]*Candidate // by TSEID

type Votes struct {
	MunicipalityCode string                  // resolved CD_MUNICIPIO (expected 83798)
	ElectionDate     time.Time               // DT_ELEICAO
	ByCandidate      map[string]CandidateVotes
	OfficeTotals     map[Office]OfficeTotals
}
type CandidateVotes struct{ Municipality, State int }
type OfficeTotals struct{ Municipality, State, CandidatesInOffice int }

type Asset struct { Order int; Type, Description string; Value float64 }
type Finance struct {
	RevenueByOrigin  map[string]float64
	ExpenseByType    map[string]float64
	ReportKind       string // "final" | "partial" (logged)
}
type SocialLink struct { Order int; URL string }
```

## Step A — `LoadCandidates`

1. `OpenZipMember(TSEBase+"consulta_cand/consulta_cand_2026.zip", "consulta_cand_2026_SC.csv")`,
   `csvx.Open(…, csvx.Latin1)`, `Require` every column listed in the data-source doc.
2. For each row with `CD_CARGO ∈ {6, 7}`:
   - build `Candidate`; strings through `csvx.Str` (sentinels → nil);
   - `CPF = leftPad11(digitsOnly(NR_CPF_CANDIDATO))`; empty if sentinel;
   - `ResultCode = mapResult(DS_SIT_TOT_TURNO)` (table in `01-foundation/03-data-model.md`);
   - first row only: record source `tse_candidates` with `GeneratedAt =
     TimestampBR(DT_GERACAO, HH_GERACAO)`.
3. Complementary member `consulta_cand_complementar_2026_SC.csv` (same zip
   family, `consulta_cand_complementar/…`): for each row whose
   `SQ_CANDIDATO` is in the map, set `BirthCity` (`NM_MUNICIPIO_NASCIMENTO`,
   `-3` → nil), `AgeAtInauguration`, `DeclaredAssets`, `CandidacyStatus =
   DS_SITUACAO_CANDIDATO_TOT` (`#NULO` → `"SEM INFORMAÇÃO"` so the column stays NOT NULL).
4. History member `historico_candidatura_2026_SC.csv`: for each row with
   `ANO_ELEICAO = 2022`, `CD_CARGO` equal to the current candidate's office
   code, and `strings.HasPrefix(strings.ToUpper(DS_SIT_TOT_TURNO), "ELEITO")`
   → `RunningForReelection = true` for `SQ_CANDIDATO_ATUAL`.
5. Assert: `len(cands) > 600` (SC had 645 on 2026-10-21) — a much smaller
   number means a filter or encoding bug; abort.

```go
func mapResult(label string) string {
	switch strings.TrimSpace(strings.ToUpper(label)) {
	case "ELEITO POR QP":    return "elected_by_quota"
	case "ELEITO POR MÉDIA": return "elected_by_average"
	case "ELEITO":           return "elected"
	case "SUPLENTE":         return "substitute"
	case "NÃO ELEITO":       return "not_elected"
	case "", "#NULO", "#NULO#": return "no_result"
	default:                 return "other"
	}
}
```

## Step B — `LoadVotes`

Streams the 96 MB SC member once (≈5 MB over the wire).

1. `OpenZipMember(TSEBase+"votacao_candidato_munzona/votacao_candidato_munzona_2026.zip",
   "votacao_candidato_munzona_2026_SC.csv")`, Latin-1.
2. For every row with `CD_CARGO ∈ {6,7}` and `NR_TURNO = 1`:
   - `v := Int(QT_VOTOS_NOMINAIS_VALIDOS)`;
   - `ByCandidate[sq].State += v`; `OfficeTotals[office].State += v`;
   - if `NM_MUNICIPIO == config.MunicipalityName`: collect `CD_MUNICIPIO`
     into a set; `ByCandidate[sq].Municipality += v`;
     `OfficeTotals[office].Municipality += v`; mark `sq` as "present in the
     municipality" (to count `CandidatesInOffice`).
3. After the loop:
   - the municipality code set must have **exactly one** element → `MunicipalityCode`;
     if it differs from `config.ExpectedTSECode`, log a warning (not an error);
   - `CandidatesInOffice[office]` = number of distinct candidates of that office
     with rows in the municipality (federal 224, state 398 on 2026-10-21);
   - `ElectionDate` from `DT_ELEICAO` (assert = 2026-10-04).
4. Record `tse_votes` (GeneratedAt from the first row).

Rows for the same candidate repeat per zone and in-transit flag — always
**sum**, never overwrite.

## Step C — enrichment for the selected (`ids map[string]bool`)

Only the ~45 selected candidates are kept; rows for others are skipped as
early as possible (check `SQ_CANDIDATO` before parsing anything else).

### `LoadAssets`
`bem_candidato/bem_candidato_2026.zip` → `bem_candidato_2026_SC.csv`.
Per row: `Asset{Order: Int(NR_ORDEM_BEM_CANDIDATO), Type, Description, Value: Money(VR_BEM_CANDIDATO)}`.
Return `map[tseID][]Asset` sorted by `Order`. Record `tse_assets`.

### `LoadFinance`
`prestacao_contas/prestacao_de_contas_eleitorais_candidatos_2026.zip` (206 MB,
range-read):

1. `receitas_candidatos_2026_SC.csv`: per row of a selected `SQ_CANDIDATO`,
   remember `(kind = final|partial from TP_PRESTACAO_CONTAS, origin =
   DS_ORIGEM_RECEITA, value = Money(VR_RECEITA))`.
2. `despesas_contratadas_candidatos_2026_SC.csv`: same with
   `DS_ORIGEM_DESPESA` / `VR_DESPESA_CONTRATADA`.
3. Per candidate: if any final row exists, keep only final rows; else partial.
   Sum by origin / by type. Round to 2 decimals after summing.
4. **Never** read donor/supplier columns (they are not even `Require`d).
5. Record `tse_finance`.

### `LoadSocial`
`consulta_cand/rede_social_candidato_2026.zip` → `rede_social_candidato_2026_SC.csv`.
Keep `DS_URL` only if `url.Parse` succeeds, scheme is `http`/`https` and
host is non-empty. Sort by `NR_ORDEM_REDE_SOCIAL`. Record `tse_social`.

### `LoadPhotos`
`config.TSEPhotosSC` (4.25 MB): iterate members, match
`^F?SC(\d+)_div\.jpe?g$` (case-insensitive), keep bytes for selected ids
(`map[tseID][]byte`). Validate JPEG magic `FF D8 FF`; otherwise skip and
warn. Record `tse_photos` (GeneratedAt = newest member mtime).

## Pitfalls

- `#NE` columns in 2026: `DS_SITUACAO_CANDIDATURA`, `ST_REELEICAO`,
  `DS_DETALHE_SITUACAO_CAND` — do not use them (see data-source doc).
- `historico` labels are mixed case; the 2026 files are upper case — compare
  upper-cased.
- `QT_VOTOS_NOMINAIS_VALIDOS` is 0 for annulled candidacies; they still exist
  in the file and still count in `CandidatesInOffice`.
- Do not keep the full `Candidates` map alive after selection longer than
  needed — it holds 645 CPFs. `selection.Select` copies what it needs and
  step 05 calls `ForgetCPFs()`.

## Done when

```bash
cd backend && go run ./cmd/ingest -out dist/data -cache .cache 2>&1 | grep -E "tse_(candidates|votes)"
# expected log lines (values as of 2026-10-21):
#   tse_candidates loaded=645 federal=231 state=414
#   tse_votes municipality_code=83798 federal_total=26315 state_total=26788
sqlite3 dist/data/$(jq -r .version dist/data/latest.json)/dataset.sqlite \
  "SELECT office, COUNT(*), SUM(declared_assets) FROM candidate GROUP BY office;"
```
