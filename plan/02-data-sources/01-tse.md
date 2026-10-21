# 01 — TSE (Tribunal Superior Eleitoral)

Publisher: **TSE – Portal de Dados Abertos** (CKAN). No authentication, no
key, CORS open. Files are served by `cdn.tse.jus.br`, which **supports HTTP
range requests** (verified 2026-10-21) — this is what lets the ETL read only
the Santa Catarina member out of national zips.

| | |
|---|---|
| Dataset pages | <https://dadosabertos.tse.jus.br/dataset/candidatos-2026>, <https://dadosabertos.tse.jus.br/dataset/resultados-2026>, <https://dadosabertos.tse.jus.br/dataset/prestacao-de-contas-eleitorais-2026> |
| CKAN API (to list resources) | `https://dadosabertos.tse.jus.br/api/3/action/package_show?id=candidatos-2026` |
| Election | 2026 general election, **1st round on 2026-10-04**. Deputies have a single round (`NR_TURNO = 1`). |
| Election code | `CD_ELEICAO = 6259` ("Eleições Gerais Estaduais 2026") for SC deputies. |
| Offices used | `CD_CARGO = 6` DEPUTADO FEDERAL (231 candidates in SC), `CD_CARGO = 7` DEPUTADO ESTADUAL (414). Other codes in the file: 3 governor, 4 vice-governor, 5 senator, 9/10 senator substitutes — **ignored in the MVP**. |
| Cadence | Candidate files are regenerated often during and after the election (the copy fetched on 2026-10-21 says `DT_GERACAO = 21/10/2026 16:31:26`). Votes are final after totalisation. Campaign finance gets its **final** accounts after the legal deadline (30 days after the election), then corrections. A weekly refresh is enough. |

## Common file format (all TSE CSVs)

| Property | Value |
|---|---|
| Container | `.zip` with one CSV per UF (`…_SC.csv`), plus `…_BRASIL.csv` / `…_BR.csv` and a `leiame.pdf` |
| Encoding | **ISO-8859-1 (Latin-1)**, no BOM |
| Line endings | CRLF |
| Separator | `;` |
| Quoting | every field in `"…"` (numbers sometimes unquoted) |
| Decimal | comma, no thousands separator: `150000,00` |
| Dates | `DD/MM/YYYY`; times `HH:MM:SS` (Brasília time, UTC−03:00) |
| Generation timestamp | `DT_GERACAO` + `HH_GERACAO` on every row → becomes `source.generated_at` (convert to ISO with `-03:00`) |
| Null sentinels | `#NULO`, `#NULO#`, `#NE`, `#NE#`, `-1`, `-3`, `-4` (in code / numeric columns). Treat all as **null**. The 2026 files use `#NULO` and `#NE` (no trailing `#`). |

> ⚠️ `#NE` ("não exigido/não divulgado") is used **for whole columns** in 2026.
> Verified on the SC file: `DS_SITUACAO_CANDIDATURA` and `ST_REELEICAO` are
> `#NE` for **all 645** deputy candidates. Do not build logic on them — see
> the replacements below.

## Files used

Base URL: `https://cdn.tse.jus.br/estatistica/sead/odsele/`

| Source id | File (zip) | Zip size | Member we read | Member size |
|---|---|---|---|---|
| `tse_candidates` | `consulta_cand/consulta_cand_2026.zip` | 3.2 MB | `consulta_cand_2026_SC.csv` | 0.4 MB |
| `tse_candidates` | `consulta_cand_complementar/consulta_cand_complementar_2026.zip` | small | `consulta_cand_complementar_2026_SC.csv` | small |
| `tse_candidates` | `historico_candidatura/historico_candidatura_2026.zip` | 4.4 MB | `historico_candidatura_2026_SC.csv` | 0.6 MB |
| `tse_votes` | `votacao_candidato_munzona/votacao_candidato_munzona_2026.zip` | **453.7 MB** | `votacao_candidato_munzona_2026_SC.csv` | 96.3 MB (**5.2 MB compressed**) |
| `tse_assets` | `bem_candidato/bem_candidato_2026.zip` | 3.8 MB | `bem_candidato_2026_SC.csv` | 0.9 MB |
| `tse_finance` | `prestacao_contas/prestacao_de_contas_eleitorais_candidatos_2026.zip` | **205.9 MB** | `receitas_candidatos_2026_SC.csv` (4.5 MB), `despesas_contratadas_candidatos_2026_SC.csv` (22.8 MB) | — |
| `tse_social` | `consulta_cand/rede_social_candidato_2026.zip` | 3.0 MB | `rede_social_candidato_2026_SC.csv` | 0.3 MB |
| `tse_photos` | `https://cdn.tse.jus.br/estatistica/sead/eleicoes/eleicoes2026/fotos/foto_cand2026_SC_div.zip` | 4.25 MB | `FSC<SQ_CANDIDATO>_div.jpg` (one per candidate) | ~10–40 KB each |

Reading the votes member through range requests transfers ~5 MB instead of
454 MB; the finance members likewise. Always read by **member name**, never
by position.

### 1. `consulta_cand_2026_SC.csv` — candidates (50 columns)

Join key: **`SQ_CANDIDATO`** (string; keep as text, it is 12 digits).

| Column | Use | → model |
|---|---|---|
| `CD_CARGO`, `DS_CARGO` | filter 6/7; office | `candidate.office_code`, `office` |
| `SQ_CANDIDATO` | key | `candidate.tse_id` |
| `NR_CANDIDATO` | ballot number | `number` |
| `NM_CANDIDATO` | civil name | `full_name` |
| `NM_URNA_CANDIDATO` | ballot name | `ballot_name` |
| `NM_SOCIAL_CANDIDATO` | social name (`#NULO` for 642 of 645) | `social_name` |
| `NR_CPF_CANDIDATO` | **in memory only**, to match Câmara deputies (D10) | — never stored |
| `NR_PARTIDO`, `SG_PARTIDO`, `NM_PARTIDO` | party | `party_*` |
| `SG_FEDERACAO`, `NM_FEDERACAO` | federation (nullable) | `federation_*` |
| `SG_UF_NASCIMENTO` | birth UF | `birth_uf` |
| `DT_NASCIMENTO` | birth date | `birth_date` (ISO) |
| `DS_GRAU_INSTRUCAO` | education | `education` |
| `DS_OCUPACAO` | occupation | `occupation` |
| `CD_SIT_TOT_TURNO`, `DS_SIT_TOT_TURNO` | result (`ELEITO POR QP`, `ELEITO POR MÉDIA`, `SUPLENTE`, `NÃO ELEITO`, `#NULO`) | `result_tse_label`, `result_code` (mapping in `01-foundation/03-data-model.md`) |
| `DT_ELEICAO`, `CD_ELEICAO`, `NR_TURNO` | election metadata | `index.json.election` |
| `DT_GERACAO`, `HH_GERACAO` | provenance | `source.generated_at` |

**Ignored on purpose:** `DS_EMAIL`, `NR_TITULO_ELEITORAL_CANDIDATO` (personal
data, D10), `DS_ESTADO_CIVIL`, `DS_COR_RACA`, `DS_GENERO` (D4),
`DS_SITUACAO_CANDIDATURA` (`#NE` for everyone in 2026), coalition columns.

Result distribution for SC deputies (2026-10-21): SUPLENTE 388, NÃO ELEITO 189,
ELEITO POR QP 45, #NULO 12, ELEITO POR MÉDIA 11.

### 2. `consulta_cand_complementar_2026_SC.csv` — complementary (49 columns)

Join on `SQ_CANDIDATO` (all 645 deputy rows match).

| Column | Use | → model |
|---|---|---|
| `NM_MUNICIPIO_NASCIMENTO` | birthplace (selection rule) — `-3` when not disclosed | `birth_city` |
| `NR_IDADE_DATA_POSSE` | age at inauguration (always numeric for deputies) | `age_at_inauguration` |
| `ST_DECLARAR_BENS` | `S`/`N` — declared assets (498 S, 147 N) | `declared_assets` |
| `DS_SITUACAO_CANDIDATO_TOT` | **candidacy status** — `DEFERIDO` 613, `INDEFERIDO` 10, `INDEFERIDO EM PRAZO RECURSAL OU COM RECURSO` 8, `RENÚNCIA` 2, `#NULO` 12 | `candidacy_status` |
| `NM_TIPO_DESTINACAO_VOTOS` | `Válido`, `Nulo técnico`, `Anulado sub judice`, `Anulado` — explains zero valid votes | logged only |

**Replacement for `DS_SITUACAO_CANDIDATURA`:** use `DS_SITUACAO_CANDIDATO_TOT`.
"Eligible candidacy" in the selection rule means
`DS_SITUACAO_CANDIDATO_TOT = 'DEFERIDO'`.

**Not used:** `ST_REELEICAO` (`#NE` for all), quilombola/indigenous/FEFC
columns, process/protocol numbers.

### 3. `historico_candidatura_2026_SC.csv` — candidacy history

One row per **past** candidacy of each current candidate.

| Column | Use |
|---|---|
| `SQ_CANDIDATO_ATUAL` | join to the 2026 candidate |
| `ANO_ELEICAO`, `CD_CARGO` | previous election and office |
| `DS_SIT_TOT_TURNO` | previous result — **mixed case** here (`Eleito por QP`, `Eleito por média`, `Suplente`, `Não eleito`) |

**Replacement for `ST_REELEICAO`:** `running_for_reelection = 1` when the
candidate has a row with `ANO_ELEICAO = 2022`, the **same** `CD_CARGO` as in
2026, and `upper(DS_SIT_TOT_TURNO)` starting with `ELEITO`. The SC history file has
37 rows "elected state deputy in 2022" and 15 "elected federal deputy in 2022"
(some of those people run for a *different* office in 2026, so the flag count
is lower). The flag means "elected in 2022 to the same office" — not "held the
seat for the whole term"; say so on the About page.

### 4. `votacao_candidato_munzona_2026_SC.csv` — votes by municipality and zone

One row per candidate × municipality × zone (× in-transit flag).

| Column | Use |
|---|---|
| `NM_MUNICIPIO`, `CD_MUNICIPIO` | Videira = `NM_MUNICIPIO = 'VIDEIRA'`, **`CD_MUNICIPIO = 83798`** (TSE code, *not* IBGE) |
| `NR_ZONA` | Videira has a **single zone: 36** (643 rows) |
| `NR_TURNO` | always `1` for deputies |
| `ST_VOTO_EM_TRANSITO` | all `N` for Videira; still sum across values |
| `CD_CARGO`, `SQ_CANDIDATO` | keys |
| `QT_VOTOS_NOMINAIS_VALIDOS` | **the vote count we use** (ranking and display) |
| `QT_VOTOS_NOMINAIS` | logged only (differs when votes were annulled) |
| `DS_SIT_TOT_TURNO` | cross-check of the result |

Verified figures for Videira (2026-10-21):

| Office | Valid nominal votes (all candidates) | Candidates with rows | With > 0 votes | 20th place | 21st place |
|---|---|---|---|---|---|
| Federal deputy (6) | 26,315 | 224 | 175 | 288 | 234 |
| State deputy (7) | 26,788 | 398 | 225 | 112 | 107 |

State-level totals (`vote_total.scope = 'state'`) come from the **same file**:
sum every SC row per candidate (all municipalities).

**Rule:** resolve the TSE municipality code at run time
(`NM_MUNICIPIO = 'VIDEIRA'` in this file) and assert exactly one code is
found; log it; store it in `ingest_run.tse_code` and `index.json`. Keep
`83798` in config only as an expected value to warn on drift.

### 5. `bem_candidato_2026_SC.csv` — declared assets

| Column | → `asset` |
|---|---|
| `SQ_CANDIDATO` | `tse_id` |
| `NR_ORDEM_BEM_CANDIDATO` | `ord` |
| `DS_TIPO_BEM_CANDIDATO` | `type` |
| `DS_BEM_CANDIDATO` | `description` (verbatim; may contain addresses — published by TSE, shown as is) |
| `VR_BEM_CANDIDATO` | `value` (`150000,00` → 150000.00) |
| `DT_ULT_ATUAL_BEM_CANDIDATO` | logged only |

Candidates with `ST_DECLARAR_BENS = 'N'` have no rows → `assets.declared = false`.

### 6. Campaign finance (`prestacao_de_contas_eleitorais_candidatos_2026.zip`)

Members for SC: `receitas_candidatos_2026_SC.csv`,
`receitas_candidatos_doador_originario_2026_SC.csv`,
`despesas_contratadas_candidatos_2026_SC.csv`,
`despesas_pagas_candidatos_2026_SC.csv`. **We read only the first and third.**

| File | Columns used | Aggregation |
|---|---|---|
| `receitas_candidatos_2026_SC.csv` | `SQ_CANDIDATO`, `DS_ORIGEM_RECEITA`, `VR_RECEITA`, `TP_PRESTACAO_CONTAS` | `SUM(VR_RECEITA)` by `DS_ORIGEM_RECEITA` → `campaign_revenue_by_origin` |
| `despesas_contratadas_candidatos_2026_SC.csv` | `SQ_CANDIDATO`, `DS_ORIGEM_DESPESA`, `VR_DESPESA_CONTRATADA`, `TP_PRESTACAO_CONTAS` | `SUM(VR_DESPESA_CONTRATADA)` by `DS_ORIGEM_DESPESA` → `campaign_expense_by_type` |

`TP_PRESTACAO_CONTAS` distinguishes partial and final reports. Rule: per
candidate, if any row is final (value starting with `Final`, case-insensitive)
use only final rows; otherwise use the partial rows. Log which one was used.

**Ignored on purpose (D10):** every donor and supplier identity column —
`NR_CPF_CNPJ_DOADOR`, `NM_DOADOR`, `NM_DOADOR_RFB`, `NR_CPF_CNPJ_FORNECEDOR`,
`NM_FORNECEDOR`, `NM_FORNECEDOR_RFB`, receipt/document numbers, and the whole
`doador_originario` and `despesas_pagas` files. `despesas_pagas` has no
`SQ_CANDIDATO` (it keys on `SQ_PRESTADOR_CONTAS`) — another reason to use
`despesas_contratadas`.

### 7. `rede_social_candidato_2026_SC.csv` — social media

`SQ_CANDIDATO`, `NR_ORDEM_REDE_SOCIAL`, `DS_URL`. Keep only `http://` /
`https://` URLs that parse with `net/url` and have a host; normalise `http` →
`https` is **not** done (keep the official value). The UI label is the host
without `www.`.

### 8. Photos — `foto_cand2026_SC_div.zip`

Members `FSC<SQ_CANDIDATO>_div.jpg`. Match case-insensitively with
`^F?SC(\d+)_div\.jpe?g$`. Copy only the selected candidates'; missing photo →
`photo_path = NULL` (UI shows the same neutral placeholder for everyone).

## Links back to the TSE per candidate

The citizen-facing site is DivulgaCandContas
(`https://divulgacandcontas.tse.jus.br/divulga/`). It is an Angular app behind
a WAF: scripted requests get **HTTP 403** (2026-10-21), so the deep-link
pattern **could not be verified**. Earlier elections used
`#/candidato/<ano>/<id-eleicao-divulga>/<UF>/<SQ_CANDIDATO>`, but the
DivulgaCand election id is **not** the open-data `CD_ELEICAO` (6259).

→ `candidate.tse_page_url` / `officialLinks.tseCandidatePage` stays **NULL**
unless someone opens a candidate in a browser during implementation, confirms
the pattern, and records it in `internal/config`. Every block still links to
its dataset page.

## Pitfalls

- Latin-1: reading as UTF-8 silently corrupts names (`JOÃO` → `JO�O`).
- `SQ_CANDIDATO` must stay a string (precision) — never parse to float.
- CPF may lose leading zeros if parsed as a number: keep as string, left-pad to 11.
- Candidates with annulled votes have `QT_VOTOS_NOMINAIS_VALIDOS = 0` but non-zero `QT_VOTOS_NOMINAIS`.
- The TSE re-publishes files with corrections; never cache them across CI runs.
- The `_BRASIL.csv` member contains every UF — never read it by accident.
