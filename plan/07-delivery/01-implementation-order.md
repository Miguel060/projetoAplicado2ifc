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

## Every step ships with its tests (decision D15)

The testing contract is `01-foundation/05-testing-strategy.md`; the
step-by-step test guides are `03-backend-etl/09-integration-tests.md`
(backend, fake upstream, golden snapshot) and `04-frontend/08-testing.md`
(Vitest, Testing Library, Playwright, axe). Rules that apply to **every** leaf
below:

- A step is **done** only when its code **and its tests** are merged in the
  same PR, CI is green (four jobs) and coverage stays **≥ 85 %** on the backend
  and on the frontend. CI gates every deploy, so an untested step cannot ship.
- **Every frontend step** also gets its **interactive browser check with
  Claude in Chrome** before the PR is opened (every changed control, keyboard
  pass, 360 px and desktop widths, zero console errors, a GIF in the PR) —
  Playwright instead when the check needs scripting.
- Test effort is shown as `+ tests ⏱ …` next to each item. It is rough, and
  deliberately not small: budget for it, or lower the gate in D15 explicitly
  — never skip tests silently.
- Fixtures are **synthetic** (real layouts and encodings, invented values,
  fake CPFs): the repository is public.

## Critical path

```
JSON contract frozen (28/10)
   ├──► fixtures + fake upstream (testsrv) ──► ETL TSE-only export + golden snapshot (target 11/11) ──► first real snapshot published (12/11) ──┐
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
  - [ ] `00-overview/02-decisions.md` — all decisions (incl. D15 tests gate every deploy, D16 ingestion on GitHub Actions); if anything changes, edit the entry and date it
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
  - [ ] `01-foundation/05-testing-strategy.md` — gates, synthetic fixtures, golden snapshot, per-step browser check
- [ ] **Optional, no code:** low-fidelity wireframes of home, profile and About — [frontend lead]
- [ ] **Record the activity in Modelo 4** — [everyone]

### Milestone 2 — 2026-11-09 (Mon): development environment

- [ ] **Machines ready** — `01-foundation/01-prerequisites.md` — [everyone] ⏱ 30 min each
- [ ] **Repository skeleton** — `01-foundation/02-repository-layout.md` — [repository owner] ⏱ 1 h
  - [ ] root files: `.gitignore`, `.gitattributes` (byte-exact `backend/testdata/**`), `.editorconfig`, `.nvmrc`, `biome.json`, `Makefile` (incl. `test-*` targets), `README.md`
  - [ ] shared test vectors `testdata/textnorm-vectors.json` (read by both Go and TS tests)
- [ ] **GitHub settings** — `05-infrastructure/02-github-oidc-and-variables.md` — [repository owner] ⏱ 30 min
  - [ ] Actions enabled; workflow permissions = read
  - [ ] `INGEST_CONTACT` variable created (the other four come after the stack deploy)
- [ ] **Backend skeleton** — `03-backend-etl/01-go-project-setup.md` — [backend lead] ⏱ 2 h + tests ⏱ 1 h
  - [ ] `go.mod`, `cmd/ingest` (flags → `pipeline.Options`), `internal/config`, `internal/pipeline.Run`, `go run ./cmd/ingest -h` works
  - [ ] first tests (config, flag mapping) so the 85 % gate passes from day one
- [ ] **Frontend skeleton** — `04-frontend/01-project-setup.md` — [frontend lead] ⏱ 3 h + tests ⏱ 2 h
  - [ ] Vite + React + TS strict, Tailwind 4, shadcn/ui init, tokens, layout shell, routes `/`, `/candidatos`, `/candidatos/:slug`, `/sobre`, dev data server
  - [ ] Vitest + Testing Library + coverage thresholds, `playwright.config.ts` (Chromium + WebKit), first smoke spec — `04-frontend/08-testing.md`
  - [ ] Claude in Chrome check of the shell (routes, keyboard, 360 px)
- [ ] **Infrastructure** — `05-infrastructure/01-cdk-stack.md` — [AWS account owner] ⏱ 2 h + tests ⏱ 2 h ⛓ prerequisites
  - [ ] `infra/` package, tsconfig, cdk.json, `lib/config.ts`, `bin/app.ts`, `lib/site-stack.ts`, `lib/spa-rewrite.js`
  - [ ] assertion tests `infra/test/site-stack.test.ts` + `spa-rewrite.test.ts` green (Step 6b)
  - [ ] `cdk synth` clean; trust policy shows exactly the two subjects
  - [ ] `cdk diff` / `cdk deploy --profile personal` (preflight first)
  - [ ] outputs recorded and handed to the repository owner
- [ ] **Repository variables + ruleset** — `05-infrastructure/02-github-oidc-and-variables.md` — [repository owner] ⏱ 20 min ⛓ stack outputs
  - [ ] `AWS_DEPLOY_ROLE_ARN`, `SITE_BUCKET`, `SITE_DISTRIBUTION_ID`, `SITE_URL`
  - [ ] ruleset on `main` with the **four** checks (after `ci.yml` has run once so they can be selected)
- [ ] **CI** — `06-ci-cd/01-ci.md` — [AWS account owner] ⏱ 1.5 h ⛓ the three skeletons with their first tests
  - [ ] four jobs green: `Backend (Go)`, `Frontend (React)`, `End-to-end (Playwright)`, `Infrastructure (CDK)`
- [ ] **Deploy** — `06-ci-cd/02-deploy-site.md` (`deploy.yml`) — [AWS account owner] ⏱ 1 h ⛓ CI, variables
  - [ ] first merge to `main` runs `verify` (all tests) and publishes the skeleton; smoke checks green (data pointer warning expected)
- [ ] **Landing page — the first real deploy** — `04-frontend/03-landing-page.md` — [frontend lead] ⏱ 3 h + tests ⏱ 2 h ⛓ frontend skeleton, deploy
  - [ ] `/` explains the panel neutrally and links to `/candidatos` and `/sobre`
  - [ ] with no snapshot yet it shows the "Dados em preparação" fallback (this is what is live until milestone 3)
  - [ ] component tests + `e2e/landing.spec.ts` (with and without data, axe clean) + Claude in Chrome check
  - [ ] merged → `deploy.yml` green; `SITE_URL` shows the landing page
- [ ] **Record the activity in Modelo 4** (link the repository and the first green deploy) — [everyone]

### Milestone 3 — 2026-11-12 (Thu): data collection and processing

- [ ] **Shared plumbing** — `03-backend-etl/02-fetch-and-parse.md` — [backend lead] ⏱ 4 h + tests ⏱ 4 h
  - [ ] HTTP client with retries and `User-Agent`, download cache, remote-zip reader (range requests), Latin-1/BOM CSV, BR numbers/dates, sentinels, text normalisation
  - [ ] table-driven unit tests; fake upstream `internal/testsrv` (in-memory zips, `Range`, injected 500/504/429/slow) — `03-backend-etl/09-integration-tests.md`
- [ ] **TSE** — `03-backend-etl/03-load-tse.md` — [backend lead] ⏱ 4 h + tests ⏱ 3 h ⛓ plumbing
  - [ ] synthetic TSE fixtures (real layouts, Latin-1, invented values, fake CPFs) + loader tests
  - [ ] candidates + complementary, votes (municipality code resolved at run time), assets, finance aggregates, social links, photos
- [ ] **Selection** — `03-backend-etl/04-select-candidates.md` — [backend lead] ⏱ 2 h + tests ⏱ 1.5 h ⛓ TSE
  - [ ] top 20 per office in Videira, ties at 20 included, plus born in Videira
  - [ ] tests for ties at the cut-off, born-in-municipality only, empty office
- [ ] **Export (TSE-only first)** — `03-backend-etl/07-export-json.md` — [backend lead] ⏱ 3 h + tests ⏱ 3 h ⛓ selection
  - [ ] `dataset.sqlite`, `index.json`, `candidates/<slug>.json`, photos, `latest.json`
  - [ ] pipeline integration test + **first golden snapshot** committed (`backend/testdata/golden/data`) — `03-backend-etl/09-integration-tests.md`; from now on the e2e job runs against it
  - [ ] **hand the first local snapshot to the frontend lead** (target 11/11)
- [ ] **Câmara** — `03-backend-etl/05-load-camara.md` — [AWS account owner] ⏱ 4 h + tests ⏱ 3 h ⛓ plumbing, selection types
  - [ ] CPF match (CPF forgotten right after), bills, roll-call votes (bulk), CEAP (bulk)
  - [ ] fixtures sharing fake CPFs with the TSE ones; tests for match/no-match, 504 retries; golden snapshot updated
- [ ] **Amendments** — `03-backend-etl/06-load-amendments.md` — [AWS account owner] ⏱ 2 h + tests ⏱ 2 h ⛓ Câmara
  - [ ] bulk CGU file, per-beneficiary rows in Videira, author matching + `config/name-overrides.json`
  - [ ] tests for name matching, overrides, municipality filter; golden snapshot updated
- [ ] **Validate and publish** — `03-backend-etl/08-validate-and-publish.md` — [AWS account owner] ⏱ 2 h + tests ⏱ 2 h ⛓ export
  - [ ] one test per validation gate (each must be able to fail); coverage ≥ 85 % on `./internal/...`
- [ ] **Scheduled refresh** — `06-ci-cd/03-refresh-data.md` (`refresh-data.yml`) — [AWS account owner] ⏱ 1 h ⛓ validate, variables
  - [ ] the run shows `go test` + coverage gate **before** the ETL
  - [ ] first snapshot live: `curl $SITE_URL/data/latest.json`
- [ ] **Spot-check the numbers** of 3 candidates against the official sites (see `07-delivery/02-launch-checklist.md` §2) — [backend lead]
- [ ] **Record the activity in Modelo 4** — [everyone]

### Milestone 4 — 2026-11-17 (Tue): search, listing and profile screens

- [ ] **Data access layer** — `04-frontend/02-data-access.md` — [frontend lead] ⏱ 3 h + tests ⏱ 3 h ⛓ contract (can start 09/11; the landing page already uses `latest.json`)
  - [ ] unit tests for fetchers/loaders (mocked `fetch`), error mapping, schema mismatch; shared text-normalisation vectors
- [ ] **Search page** (`/candidatos`) — `04-frontend/04-search-page.md` — [frontend lead] ⏱ 6 h + tests ⏱ 4 h ⛓ data access
  - [ ] search box, filters in the URL, sorting, cards, result count announced
  - [ ] unit tests (search/filter/sort/URL params), component tests, `e2e/search.spec.ts` (keyboard, axe) + Claude in Chrome check
- [ ] **Profile page** (`/candidatos/:slug`) — `04-frontend/05-profile-page.md` — [frontend lead + backend lead] ⏱ 8 h + tests ⏱ 5 h ⛓ first golden snapshot
  - [ ] fixed section order, source note on every block, neutral empty states
  - [ ] component tests per section (with/without Câmara record), `e2e/profile.spec.ts` (axe, not-found slug) + Claude in Chrome check
- [ ] **About page** (`/sobre`) — `04-frontend/06-about-page.md` — [backend lead] ⏱ 3 h + tests ⏱ 2 h ⛓ index.json sources
  - [ ] selection rule, sources with dates, dataset download, impartiality statement
  - [ ] component tests + `e2e/about.spec.ts` (axe) + Claude in Chrome check
- [ ] **Accessibility, responsiveness, neutrality implementation pass** — `04-frontend/07-accessibility-responsiveness-neutrality.md` Parts 1–3 — [frontend lead] ⏱ 4 h + tests ⏱ 2 h
  - [ ] axe zero violations (`wcag2a/aa`, `wcag21a/aa`) on every route in Chromium and WebKit; keyboard specs; Claude in Chrome pass at 360 px and desktop
- [ ] Merge → `deploy.yml` runs `verify` (all tests) and publishes; the live site shows real data — [frontend lead]
- [ ] **Record the activity in Modelo 4** (screenshots of the three pages) — [everyone]

### Milestone 5 — 2026-11-19 (Thu): checks, fixes, MVP published

- [ ] **Launch checklist** — `07-delivery/02-launch-checklist.md` — [everyone] ⏱ 4–6 h
  - [ ] §0 tests: all CI jobs green on the release commit, coverage ≥ 85 % backend and frontend, e2e green in Chromium and WebKit, infra assertions green
  - [ ] §1 functional · §2 data spot-checks · §3 responsive on real phones · §4 accessibility manual pass · §5 neutrality review · §6 security · §7 operations and cost
- [ ] **Fix list** triaged: blockers fixed today (each fix with a regression test), the rest become issues — [everyone]
- [ ] **Final refresh + deploy**, sign-off table filled — [AWS account owner]
- [ ] **Hand the URL to the APL partner** as the test-environment MVP (the partnership is recorded in the course deliverables, never on the site) — [repository owner]
- [ ] **Record the activity in Modelo 4** — [everyone]

## After the MVP (not scheduled)

- [ ] Phase 2 scope from `00-overview/03-architecture.md` → "What would change for phase 2"
  (senator/governor, TSE government proposals, ALESC scraping, custom domain).
- [ ] Before the course ends: decide who keeps the weekly refresh alive (the
  60-day inactivity rule in `06-ci-cd/03-refresh-data.md`) or note in the
  About page that the data is frozen at the last update date.
