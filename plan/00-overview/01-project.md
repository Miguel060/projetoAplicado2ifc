# 01 — Project

## One sentence

A **non-partisan web panel** that gathers, in one place and with the official
source of every value, the public information about the **2026 candidates for
federal and state deputy linked to Videira/SC**: who they are, how Videira
voted for them, what they declared, how they financed their campaign and, for
the ones who already hold a seat in the Câmara dos Deputados, what they did
there and which parliamentary amendments they sent to Videira.

## Context

| | |
|---|---|
| Course | Projeto Aplicado II — IFC (Instituto Federal Catarinense) |
| Group | Three IFC students (names in the course deliverables) |
| Partner (APL) | Câmara Municipal de Vereadores de Videira/SC (Legislative branch, public body) |
| Area | Website / App |
| Problem (from the Diagnóstico Participativo) | Information about candidates is spread across several official sources (TSE, Câmara dos Deputados, Portal da Transparência), with no single channel that gathers it in a neutral way. |
| Budget | **Zero.** The partner has no budget for the project. |
| Deadline | MVP published in a test environment by **2026-11-19** (Modelo 4). |

## Goals of the MVP

Routes: `/` landing · `/candidatos` search · `/candidatos/:slug` profile ·
`/sobre` methodology.

0. A **landing page** (`/`) that explains what the panel is for, how it works,
   who appears in it, its impartiality commitment and its official sources,
   with a call to action to the search. It shows live figures (number of
   candidates, sources, last update) when data exists and "Dados em
   preparação" when it does not, so it can be **deployed before the first
   data snapshot** — it is the first thing published.
1. **Search and filter** (`/candidatos`) the selected candidates by name,
   ballot number, office, result and party.
2. A **profile page** per candidate with, in this fixed order:
   1. Identification (photo, ballot name, number, party/federation, office, result)
   2. Votes in Videira and in SC
   3. Personal data published by the TSE (occupation, education, age, birthplace)
   4. Declared assets
   5. Campaign finance (revenue by origin, expenses by type)
   6. "Proposals": bills authored in the Câmara dos Deputados (incumbents only)
   7. Activity in the Câmara dos Deputados: roll-call votes and parliamentary quota (CEAP)
   8. Parliamentary amendments destined to Videira
   9. Official social media links registered with the TSE
   10. Link to ALESC for state deputies running for re-election
3. **Every block shows its source**: name of the official dataset, the date the
   source generated it, the date we fetched it, and a link to it.
4. **Automatic refresh**: the data is rebuilt weekly and on demand, with no
   manual editing anywhere.
5. **Responsive, accessible (WCAG 2.1 AA) and impartial**: the same template
   for every candidate, no scores, no opinions, no party colours.

## Who is selected ("linked to Videira")

Decided rule, applied identically to everyone:

> For each office (federal deputy, state deputy), the **20 candidates with the
> most valid nominal votes in Videira** in the 2026 election, **plus** every
> candidate for those offices **born in Videira** with an eligible candidacy.
> Ties at position 20 are all included.

`topN = 20` is a configuration constant, not code. Expected result: ~40–45
profiles. The rule is printed on the About page.

## What "proposals" means here

The TSE only publishes *propostas de governo* for **executive** offices
(president, governor). Deputies do not file proposals. Decision:

- For candidates who were **federal deputies in the 2023–2027 legislature**,
  the "Proposals" block shows their **authored bills** (PL, PLP, PEC, PDL) from
  the Câmara dos Deputados API.
- For everyone else it shows a neutral message: *there is no proposal
  registered in an official source for this office* — and explains why.
- TSE *propostas de governo* come in when the scope grows to governor/senator.

## Out of scope for the MVP

| Out | Why / when |
|---|---|
| Senator and governor | Phase 2 (Modelo 4: "expansão posterior"). The data model already supports other offices. |
| City councillors (vereadores) | Not in the Modelo 4 scope. |
| ALESC activity (state deputies) | ALESC has no public API. MVP links to ALESC; scraping is phase 2. |
| Manual curation of any data | Every value comes from an official source automatically. This is what keeps the panel defensibly impartial. |
| Accounts, login, comments, ratings | Not needed, and ratings/comments would break impartiality. |
| Admin panel | Nothing is edited by hand, so there is nothing to administer. |
| Analytics / trackers | Privacy, and a neutral public-interest site should not profile visitors. |
| Automated test suites | Explicit instruction: **production code only**. Quality is ensured by compilers, type checks, linters, runtime validation in the ETL and a manual QA checklist (see `07-delivery/02-launch-checklist.md`). |
| Custom domain | Optional. CloudFront serves HTTPS on its own `*.cloudfront.net` name. |

## Schedule (from Modelo 4 — Registro de Atividades)

| Date | Activity | Plan section |
|---|---|---|
| 2026-10-23 | Requirements, MVP scope, task split | `00-overview/`, `07-delivery/01-implementation-order.md` |
| 2026-10-28 | Technical mapping of data sources, data model | `02-data-sources/`, `01-foundation/03-data-model.md`, `01-foundation/04-json-contract.md` |
| 2026-11-09 | Dev environment: repository, app skeleton, database | `01-foundation/`, `03-backend-etl/01-go-project-setup.md`, `04-frontend/01-project-setup.md`, `05-infrastructure/` |
| 2026-11-12 | Data collection and processing (TSE, Câmara), with source and update date | `03-backend-etl/` |
| 2026-11-17 | Search, listing and profile screens with links to official sources | `04-frontend/` |
| 2026-11-19 | Functional, responsive and accessibility checks; fixes; publish MVP | `06-ci-cd/`, `07-delivery/02-launch-checklist.md` |
