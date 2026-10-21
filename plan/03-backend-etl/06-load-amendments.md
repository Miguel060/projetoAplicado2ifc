# 03.06 — Load parliamentary amendments destined to Videira

## Goal

For each matched federal deputy (step 05), list the parliamentary amendments
whose money **reached beneficiaries in Videira**, with how much was received,
plus the amendment's own metadata. Source: CGU bulk file
(`02-data-sources/03-portal-transparencia.md`), source id `cgu_amendments`.

## Contract change — adopted 2026-10-21 (already in `01-foundation/03-data-model.md` and `04-json-contract.md`)

Verified 2026-10-21: filtering `EmendasParlamentares.csv` by
`Código Município IBGE = 4219309` (what the foundation docs assume) yields
**3 rows in 12 years**, because most amendments are booked at state level or
"multiple localities". The money that actually reached Videira is only visible
in `EmendasParlamentares_PorFavorecido.csv` (84 rows, R$ 13.2 M received,
2015–2026), which has **no IBGE column** and gives **received amounts per
beneficiary**, not committed/liquidated/paid.

The current contract (`amendment` table, `Amendment` type) only has
whole-amendment `committed / liquidated / paid`. Exporting those as "for
Videira" would overstate amounts (an amendment can fund several
municipalities). The **additive** change below (still `schemaVersion: 1`) was
adopted; the foundation docs are the authoritative version:

```sql
-- amendment: add
  received_in_municipality REAL NOT NULL,   -- SUM(Valor Recebido) for beneficiaries in the municipality
-- committed/liquidated/paid become explicitly "whole amendment, all localities"

CREATE TABLE amendment_beneficiary (
  code          TEXT NOT NULL REFERENCES amendment(code),
  name          TEXT NOT NULL,            -- legal entities only; natural persons aggregated (D10)
  received      REAL NOT NULL,
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (code, name)
);
```

```ts
export interface Amendments {
  totals: { committed: number; liquidated: number; paid: number; receivedInMunicipality: number };
  items: Amendment[];   // desc by year, then by receivedInMunicipality
  source: "cgu_amendments";
}
export interface Amendment {
  /* existing fields … */
  receivedInMunicipality: number;
  beneficiaries: { name: string; received: number }[]; // desc by received
}
```

In the totals, `receivedInMunicipality` is the headline figure; `committed`,
`liquidated` and `paid` describe whole amendments (all localities) and the UI
labels them that way.

## Files to create

```
backend/internal/amendments/sources.go
backend/internal/amendments/load.go        # Load
backend/internal/amendments/match.go       # author → deputy
backend/config/name-overrides.json         # (created in step 01)
```

## Types

```go
package amendments

type Amendment struct {
	Code, AuthorCode, AuthorName, Type, Number string
	Year                       int
	FunctionName, SubfunctionName *string
	Committed, Liquidated, Paid float64 // whole amendment
	ReceivedInMunicipality      float64
	Beneficiaries               map[string]float64 // legal entities; natural persons under one fixed label
	DeputyID                    int
	URL                         *string // detail page; nil while config.PortalAmendmentURLPattern is empty
}

type Override struct {
	AuthorCode string `json:"authorCode"`
	DeputyID   int    `json:"deputyId"`
	Note       string `json:"note"`
}

func Load(ctx context.Context, c *fetch.Client, s *fetch.Sources,
	deps camara.Deputies, overridesPath string) (map[int][]Amendment, error)
```

## Algorithm

1. **Open the zip** `config.CGUAmendmentsZip` (32.5 MB; range-read); on 404
   retry with `config.CGUAmendmentsAlt` (302 → same file). Encoding
   `csvx.Windows1252`.
2. **Per-beneficiary pass** — member `EmendasParlamentares_PorFavorecido.csv`
   (24 MB compressed): keep rows where
   `Get("UF Favorecido") == "SC"` and
   `textnorm.Norm(Get("Município Favorecido")) == textnorm.Norm(config.MunicipalityName)`.
   For each kept row, accumulate into `byCode[Código da Emenda]`:
   - `ReceivedInMunicipality += Money(Valor Recebido)`;
   - if `Tipo Favorecido == "Pessoa Jurídica"`: `Beneficiaries[Favorecido] += value`;
     else `Beneficiaries["Pessoas físicas (valores agregados)"] += value` —
     **never** the person's name or `Código do Favorecido` (D10);
   - keep `AuthorCode`, `AuthorName`, `Type`, `Number` from the row.
3. **Attribution filter**: keep only codes whose `Type` starts with
   `Emenda Individual` (covers "Transferências com Finalidade Definida" and
   "Transferências Especiais"). Relator, committee and caucus amendments are
   not attributable to a candidate; count them for the log only.
4. **Match author → deputy** (`match.go`):
   1. overrides by `AuthorCode` (from `name-overrides.json`) win;
   2. else build `nameIndex` from matched deputies: `Norm(ParliamentaryName)`,
      `Norm(ElectoralName)`, `Norm(CivilName)` → deputy id; the author matches
      when `Norm(AuthorName)` is a key; a key that maps to two different
      deputies is removed (ambiguous) and logged;
   3. unmatched individual authors are **not errors** (most are senators or
      deputies who were not selected); log them once, aggregated:
      `amendments unmatched_individual_authors="CODE NAME (n rows); …"` so the
      group can add an override if one of them is in fact a selected deputy.
5. **Enrichment pass** — member `EmendasParlamentares.csv` (3.9 MB
   compressed): for codes kept in step 4, set `Year = Int(Ano da Emenda)`,
   `FunctionName`, `SubfunctionName`, `Committed`, `Liquidated`, `Paid`.
   If a code is missing there, derive `Year` from the first 4 digits of the
   code and leave the other values at 0 / nil (log a warning).
6. `URL`: if `config.PortalAmendmentURLPattern != ""`, `fmt.Sprintf(pattern, Code)`; else `nil`
   (pattern unverified as of 2026-10-21 — see `02-data-sources/03-portal-transparencia.md`).
7. Round every sum to 2 decimals; return `map[deputyID][]Amendment` sorted by
   `Year` desc, then `ReceivedInMunicipality` desc, then `Code`.
8. Record `cgu_amendments`: `FileURL = CGUAmendmentsZip`,
   `DatasetURL = https://portaldatransparencia.gov.br/download-de-dados/emendas-parlamentares`,
   `GeneratedAt` = modified time of the `PorFavorecido` member.

Amendments from earlier legislatures by the same parliamentarian are kept
(grouped by year in the UI).

## Name normalisation details

`textnorm.Norm` (lowercase, no accents, collapsed spaces) is enough for most
cases (`FULANO DE TAL` ↔ `Fulano de Tal`). Common misses and their fix:

| Miss | Fix |
|---|---|
| Author uses a nickname not present in any Câmara name field | override by author code |
| Author code `S/I`, name `Sem informação` | not attributable — skip |
| Two deputies normalise to the same name | ambiguous → removed from the index, override required |

## Pitfalls

- Windows-1252, comma decimals (`399688,80`).
- `UF` in `EmendasParlamentares.csv` is the full state name (`SANTA CATARINA`),
  while `UF Favorecido` in the per-beneficiary file is `SC` — do not reuse
  one filter for both.
- The same code repeats for each payment month (`Ano/Mês`) and beneficiary —
  aggregate, do not insert duplicates (`code` is the primary key).
- Do not attach relator/caucus amendments to the caucus coordinator or any
  candidate — that would be an editorial attribution.

## Done when

```bash
DS=backend/dist/data/$(jq -r .version backend/dist/data/latest.json)/dataset.sqlite
sqlite3 "$DS" "SELECT d.parliamentary_name, COUNT(*), ROUND(SUM(a.received_in_municipality),2)
               FROM amendment a JOIN chamber_deputy d USING (deputy_id)
               GROUP BY d.deputy_id ORDER BY 3 DESC;"
sqlite3 "$DS" "SELECT COUNT(*) FROM amendment_beneficiary WHERE name LIKE '%CPF%';"   # 0
(cd backend && go run ./cmd/ingest -out dist/data -cache .cache -offline 2>&1 | grep unmatched_individual_authors)   # review; add overrides if needed
```
