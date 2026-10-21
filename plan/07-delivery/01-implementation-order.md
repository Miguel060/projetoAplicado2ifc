# 07.01 — Implementation order (master checklist)

The whole project as one tree of checkboxes, in the order it should be built,
mapped to the dates already recorded in **Modelo 4 – Registro de Atividades**.
Each leaf points at the plan file that explains it. Tick boxes in a copy of
this file or in GitHub issues — whichever the group prefers.

The dates are **deadlines, not start dates**. Anything that can start earlier
should: the gap between 2026-10-28 and 2026-11-09 is the best place to absorb
surprises.

> To execute, use **`plan/STEPS.md`**: the same work as 25 numbered,
> one-PR steps ("do step 1", "do step 2"…). This file is the calendar and
> ownership view.

## Roles (assign one person to each)

People are referred to by role. With three members, a natural mapping is one
role per person; the group should adjust it to everyone's strengths and
availability.

| Role | Owns | Why this grouping |
|---|---|---|
| **Repository owner/admin** — also **backend lead** | GitHub settings (Actions, variables, ruleset); Go project setup; fetch/parse layer; TSE loaders; selection rule; JSON/SQLite export | Only an admin can create repository variables and rulesets. The TSE side is the backbone every other step consumes. |
| **AWS account owner** — also **infra/CI lead** | CDK stack and manual `cdk deploy`; `ci.yml`, `deploy.yml`, `refresh-data.yml`; Câmara loaders; CGU amendments; validation and publish | The AWS account is theirs; publishing and the deploy role belong together. Câmara/CGU are independent of the TSE code once the selection types exist. |
| **Frontend lead** | Vite/React/shadcn setup; data access layer; search, profile and About pages; accessibility/responsiveness implementation | The frontend only depends on the JSON contract, so it can progress in parallel from day one. |
| **Everyone** | Contract reviews; neutrality review; launch checklist; Modelo 4 records | Decisions about impartiality must not rest on one person. |

## Critical path

```
JSON contract frozen (28/10)
   ├──► ETL TSE-only export (target 11/11) ──► first real snapshot published (12/11) ──┐
   │                                                                                    ├──► profile page with real data (17/11) ──► QA (19/11)
   └──► frontend skeleton + components against the contract types (09/11–12/11) ───────┘
AWS stack + repository variables + CI (09/11) ──► landing page live via deploy.yml (09/11) ──► refresh-data.yml green (12/11)
```

The frontend never waits for the full ETL: as soon as the TSE part exports
(`chamber` and `amendments` are `null` until the Câmara/CGU loaders land —
the contract allows that), the frontend lead runs `make ingest` locally and
works on real data.

## The tree

```
✅ = done   ☐ = to do        [role] · ⏱ rough effort · ⛓ depends on
```

### Milestone 0 — 2026-10-23 (Fri): requirements, MVP scope, task split

- [ ] **Read and accept the overview** — [everyone] ⏱ 1 h
  - [ ] `00-overview/01-project.md` — goals, selection rule, out of scope
  - [ ] `00-overview/02-decisions.md` — D1–D13; if anything changes, edit the entry and date it
  - [ ] `00-overview/03-architecture.md` — the picture and the security model
  - [ ] `00-overview/04-tech-stack.md` — languages and pinned versions
  - [ ] `00-overview/05-aws-cost.md` — expected ≈ US$0.10/month
- [ ] **Assign the roles** (table above) — [everyone]
- [ ] **Choose the group contact e-mail** for the ETL `User-Agent` (it becomes the `INGEST_CONTACT` repository variable later; never committed) — [everyone]
- [ ] **Record the activity in Modelo 4** — [everyone]

### Milestone 1 — 2026-10-28 (Wed): data sources and data model

- [ ] **Map the sources** — ⛓ milestone 0
  - [ ] `02-data-sources/01-tse.md` — confirm SC members, columns, encodings, sentinels — [backend lead] ⏱ 2 h
  - [ ] `02-data-sources/02-camara.md` — API endpoints and bulk files, CPF in `/deputados/{id}` — [AWS account owner] ⏱ 2 h
  - [ ] `02-data-sources/03-portal-transparencia.md` — bulk amendments zip, per-beneficiary file (Videira beneficiaries) — [AWS account owner] ⏱ 1 h
  - [ ] `02-data-sources/04-alesc.md` — confirm "link only" for the MVP — [frontend lead] ⏱ 30 min
- [ ] **Freeze the contracts** (review together, then changes need a PR touching both sides) — [everyone] ⏱ 1–2 h
  - [ ] `01-foundation/03-data-model.md` — SQLite schema, source ids, result codes, slug rule
  - [ ] `01-foundation/04-json-contract.md` — files, URLs, cache headers, TypeScript types
- [ ] **Optional, no code:** low-fidelity wireframes of home, profile and About — [frontend lead]
- [ ] **Record the activity in Modelo 4** — [everyone]

### Milestone 2 — 2026-11-09 (Mon): development environment

- [ ] **Machines ready** — `01-foundation/01-prerequisites.md` — [everyone] ⏱ 30 min each
- [ ] **Repository skeleton** — `01-foundation/02-repository-layout.md` — [repository owner] ⏱ 1 h
  - [ ] root files: `.gitignore`, `.editorconfig`, `.nvmrc`, `biome.json`, `Makefile`, `README.md`
- [ ] **GitHub settings** — `05-infrastructure/02-github-oidc-and-variables.md` — [repository owner] ⏱ 30 min
  - [ ] Actions enabled; workflow permissions = read
  - [ ] `INGEST_CONTACT` variable created (the other four come after the stack deploy)
- [ ] **Backend skeleton** — `03-backend-etl/01-go-project-setup.md` — [backend lead] ⏱ 2 h
  - [ ] `go.mod`, `cmd/ingest`, `internal/config`, flags, `go run ./cmd/ingest -h` works
- [ ] **Frontend skeleton** — `04-frontend/01-project-setup.md` — [frontend lead] ⏱ 3 h
  - [ ] Vite + React + TS strict, Tailwind 4, shadcn/ui init, tokens, layout shell, routes `/`, `/candidatos`, `/candidatos/:slug`, `/sobre`, dev data server
- [ ] **Infrastructure** — `05-infrastructure/01-cdk-stack.md` — [AWS account owner] ⏱ 2 h ⛓ prerequisites
  - [ ] `infra/` package, tsconfig, cdk.json, `bin/app.ts`, `lib/site-stack.ts`, `lib/spa-rewrite.js`
  - [ ] `cdk synth` clean; trust policy shows exactly the two subjects
  - [ ] `cdk diff` / `cdk deploy --profile personal` (preflight first)
  - [ ] outputs recorded and handed to the repository owner
- [ ] **Repository variables + ruleset** — `05-infrastructure/02-github-oidc-and-variables.md` — [repository owner] ⏱ 20 min ⛓ stack outputs
  - [ ] `AWS_DEPLOY_ROLE_ARN`, `SITE_BUCKET`, `SITE_DISTRIBUTION_ID`, `SITE_URL`
  - [ ] ruleset on `main` (after `ci.yml` has run once so the checks can be selected)
- [ ] **CI** — `06-ci-cd/01-ci.md` — [AWS account owner] ⏱ 1 h ⛓ the three skeletons
- [ ] **Deploy** — `06-ci-cd/02-deploy-site.md` (`deploy.yml`) — [AWS account owner] ⏱ 1 h ⛓ CI, variables
  - [ ] first merge to `main` publishes the skeleton; smoke checks green (data pointer warning expected)
- [ ] **Landing page — the first real deploy** — `04-frontend/03-landing-page.md` — [frontend lead] ⏱ 3 h ⛓ frontend skeleton, deploy
  - [ ] `/` explains the panel neutrally and links to `/candidatos` and `/sobre`
  - [ ] with no snapshot yet it shows the "Dados em preparação" fallback (this is what is live until milestone 3)
  - [ ] merged → `deploy.yml` green; `SITE_URL` shows the landing page
- [ ] **Record the activity in Modelo 4** (link the repository and the first green deploy) — [everyone]

### Milestone 3 — 2026-11-12 (Thu): data collection and processing

- [ ] **Shared plumbing** — `03-backend-etl/02-fetch-and-parse.md` — [backend lead] ⏱ 4 h
  - [ ] HTTP client with retries and `User-Agent`, download cache, remote-zip reader (range requests), Latin-1/BOM CSV, BR numbers/dates, sentinels, text normalisation
- [ ] **TSE** — `03-backend-etl/03-load-tse.md` — [backend lead] ⏱ 4 h ⛓ plumbing
  - [ ] candidates + complementary, votes (municipality code resolved at run time), assets, finance aggregates, social links, photos
- [ ] **Selection** — `03-backend-etl/04-select-candidates.md` — [backend lead] ⏱ 2 h ⛓ TSE
  - [ ] top 20 per office in Videira, ties at 20 included, plus born in Videira
- [ ] **Export (TSE-only first)** — `03-backend-etl/07-export-json.md` — [backend lead] ⏱ 3 h ⛓ selection
  - [ ] `dataset.sqlite`, `index.json`, `candidates/<slug>.json`, photos, `latest.json`
  - [ ] **hand the first local snapshot to the frontend lead** (target 11/11)
- [ ] **Câmara** — `03-backend-etl/05-load-camara.md` — [AWS account owner] ⏱ 4 h ⛓ plumbing, selection types
  - [ ] CPF match (CPF forgotten right after), bills, roll-call votes (bulk), CEAP (bulk)
- [ ] **Amendments** — `03-backend-etl/06-load-amendments.md` — [AWS account owner] ⏱ 2 h ⛓ Câmara
  - [ ] bulk CGU file, per-beneficiary rows in Videira, author matching + `config/name-overrides.json`
- [ ] **Validate and publish** — `03-backend-etl/08-validate-and-publish.md` — [AWS account owner] ⏱ 2 h ⛓ export
- [ ] **Scheduled refresh** — `06-ci-cd/03-refresh-data.md` (`refresh-data.yml`) — [AWS account owner] ⏱ 1 h ⛓ validate, variables
  - [ ] first snapshot live: `curl $SITE_URL/data/latest.json`
- [ ] **Spot-check the numbers** of 3 candidates against the official sites (see `07-delivery/02-launch-checklist.md` §2) — [backend lead]
- [ ] **Record the activity in Modelo 4** — [everyone]

### Milestone 4 — 2026-11-17 (Tue): search, listing and profile screens

- [ ] **Data access layer** — `04-frontend/02-data-access.md` — [frontend lead] ⏱ 3 h ⛓ contract (can start 09/11; the landing page already uses `latest.json`)
- [ ] **Search page** (`/candidatos`) — `04-frontend/04-search-page.md` — [frontend lead] ⏱ 6 h ⛓ data access
  - [ ] search box, filters in the URL, sorting, cards, result count announced
- [ ] **Profile page** (`/candidatos/:slug`) — `04-frontend/05-profile-page.md` — [frontend lead + backend lead] ⏱ 8 h ⛓ first real snapshot
  - [ ] fixed section order, source note on every block, neutral empty states
- [ ] **About page** (`/sobre`) — `04-frontend/06-about-page.md` — [backend lead] ⏱ 3 h ⛓ index.json sources
  - [ ] selection rule, sources with dates, dataset download, impartiality statement
- [ ] **Accessibility, responsiveness, neutrality implementation pass** — `04-frontend/07-accessibility-responsiveness-neutrality.md` Parts 1–3 — [frontend lead] ⏱ 4 h
- [ ] Merge → `deploy.yml` publishes; the live site shows real data — [frontend lead]
- [ ] **Record the activity in Modelo 4** (screenshots of the three pages) — [everyone]

### Milestone 5 — 2026-11-19 (Thu): checks, fixes, MVP published

- [ ] **Launch checklist** — `07-delivery/02-launch-checklist.md` — [everyone] ⏱ 4–6 h
  - [ ] §1 functional · §2 data spot-checks · §3 responsive on real phones · §4 accessibility manual pass · §5 neutrality review · §6 security · §7 operations and cost
- [ ] **Fix list** triaged: blockers fixed today, the rest become issues — [everyone]
- [ ] **Final refresh + deploy**, sign-off table filled — [AWS account owner]
- [ ] **Hand the URL to the APL partner** as the test-environment MVP (the partnership is recorded in the course deliverables, never on the site) — [repository owner]
- [ ] **Record the activity in Modelo 4** — [everyone]

## After the MVP (not scheduled)

- [ ] Phase 2 scope from `00-overview/03-architecture.md` → "What would change for phase 2"
  (senator/governor, TSE government proposals, ALESC scraping, custom domain).
- [ ] Before the course ends: decide who keeps the weekly refresh alive (the
  60-day inactivity rule in `06-ci-cd/03-refresh-data.md`) or note in the
  About page that the data is frozen at the last update date.
