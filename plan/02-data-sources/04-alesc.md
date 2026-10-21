# 04 — ALESC (Assembleia Legislativa de Santa Catarina)

## Status

| | |
|---|---|
| Public API | **None found** (2026-10-21). The site exposes HTML pages, a transparency portal (`transparencia.alesc.sc.gov.br`) and the e-Legis portal (`portalelegis.alesc.sc.gov.br`), no machine-readable deputy or bill data. |
| Decision (D3) | MVP shows **no ALESC activity data**. State-deputy profiles show TSE data and, when the candidate was elected state deputy in 2022 (`running_for_reelection`), a link to ALESC. |
| Source id | none (no data is ingested, so nothing is attributed to ALESC). |

## What the ETL does

Only builds a link:

```go
// internal/config
const AlescDeputiesURL = "https://www.alesc.sc.gov.br/deputados/"
```

`stateAssembly = { "profileUrl": AlescDeputiesURL }` when
`office == state_deputy && running_for_reelection`; otherwise `null`.

A per-deputy URL is **not** generated: ALESC deputy page slugs are not
derivable from TSE data, and a wrong deep link is worse than a list page.

## Why `running_for_reelection` does not come from `ST_REELEICAO`

The 2026 TSE complementary file has `ST_REELEICAO = #NE` for every SC deputy
candidate. The flag is derived from the TSE candidacy history instead
(elected to the same office in 2022) — see `01-tse.md` §3.

## Phase 2 notes (not in the MVP)

- Scrape `https://www.alesc.sc.gov.br/deputados/` for the current list and
  each deputy page URL; match by normalised parliamentary name.
- e-Legis for authored bills; respect `robots.txt`, rate-limit to ≤ 1 req/s,
  cache aggressively, and treat any layout change as "source unavailable"
  (keep the last good snapshot) rather than publishing partial data.
- Would add source id `alesc_bills` and a nullable `stateAssembly.bills`
  block — non-breaking under the contract rules.
