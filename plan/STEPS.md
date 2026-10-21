# STEPS — build it one step at a time

The executable version of the plan. Each step is **one pull request** small
enough for one working session, ships **its own tests**, and leaves `main`
working. To build the project, say: **"do step 1"**, then **"do step 2"**,
and so on.

`07-delivery/01-implementation-order.md` is the calendar/ownership view of the
same work; this file is the execution order. The testing rules every step
follows are in `01-foundation/05-testing-strategy.md`.

## How every step runs

1. **Check the dependencies** listed in the step are merged into `main`.
   If not, stop and say which one is missing.
2. **Read** the plan files listed under *Read* — they hold the detail (code
   sketches, exact commands, pitfalls, and each doc's **"Tests for this
   step"** section). This file only sequences them.
3. **Branch:** `git switch main && git pull && git switch -c step-NN-<slug>`.
4. **Do** the items in order, **writing the tests together with the code**
   (unit tests next to the code; fixtures synthetic only — never a row from a
   real file).
5. **Test:** run the step's test targets locally. From step 6 on, CI enforces
   them: backend and frontend coverage **≥ 85 %**, e2e green in **Chromium
   and WebKit**, infra assertions green.
6. **Browser check (frontend steps):** before opening the PR, exercise the
   change in **Claude in Chrome** against the local production build
   (`npm run build && npm run preview`), following the doc's **"Browser check
   (Claude in Chrome)"** section and `04-frontend/08-testing.md`: every new
   control, keyboard-only pass, 360 px and desktop widths, zero console
   errors, no unexpected 4xx/5xx, a GIF named `step-NN-<slug>.gif`. If Claude
   in Chrome is unavailable or the check needs scripting, use Playwright and
   say so in the PR. Any regression worth keeping becomes a Playwright spec.
7. **Verify** every item under *Done when*. All must pass.
8. **Commit** with the given message (Conventional Commits), push, open a PR
   into `main`. CI must be green; merge. Merging to `main` deploys the site
   automatically (`deploy.yml`, which runs the full test suite first);
   changes under `backend/` also refresh the data (`refresh-data.yml`, which
   runs the Go tests first).
9. **Stop.** Do not start the next step's work. Report what was done, the
   coverage numbers, and anything that deviated from the plan (update the
   plan file if a decision changed — `00-overview/02-decisions.md` first).

Markers: 👤 = needs a specific person (AWS account owner or repository
admin) to run or approve an action that has outside effects. Prepare
everything, show the exact command, and wait for that person's go-ahead.
Every AWS command uses `--profile personal` and the account preflight
(`make preflight`) first.

## Overview

| # | Step | Area | Depends on | Deadline (Modelo 4) |
|---|---|---|---|---|
| 1 | Repository skeleton + shared test vectors | repo | — | 09/11 |
| 2 | Backend skeleton (+ tests) | etl | 1 | 09/11 |
| 3 | Frontend skeleton + test tooling (Vitest, Playwright, axe) | web | 1 | 09/11 |
| 4 | Infrastructure code + assertion tests | infra | 1 | 09/11 |
| 5 | 👤 Deploy the stack + repository variables | infra | 4 | 09/11 |
| 6 | CI workflow with all test gates | ci | 2, 3, 4 | 09/11 |
| 7 | Deploy workflow + 👤 branch protection | ci | 5, 6 | 09/11 |
| 8 | Frontend data access layer (+ tests) | web | 3 | 09/11 |
| 9 | Landing page — first real deploy (+ tests, browser check) | web | 7, 8 | 09/11 |
| 10 | Fetch and parse plumbing + fake upstream (`testsrv`) | etl | 2 | 12/11 |
| 11 | TSE: candidates and votes (+ fixtures, tests) | etl | 10 | 12/11 |
| 12 | Selection rule (+ tests) | etl | 11 | 12/11 |
| 13 | TSE: assets, finance, social, photos (+ fixtures, tests) | etl | 12 | 12/11 |
| 14 | SQLite + JSON export + **pipeline integration test and golden snapshot** | etl | 13 | 12/11 |
| 15 | Validation + publish script (+ tests) | etl | 14 | 12/11 |
| 16 | Refresh workflow — 👤 first snapshot live | ci | 7, 15 | 12/11 |
| 17 | Câmara loaders (+ fixtures, tests, golden update) | etl | 14 | 12/11 |
| 18 | Parliamentary amendments (+ fixtures, tests, golden update) | etl | 17 | 12/11 |
| 19 | Search page (+ tests, browser check) | web | 8, 14 | 17/11 |
| 20 | Profile page — part 1, TSE blocks (+ tests, browser check) | web | 19 | 17/11 |
| 21 | Profile page — part 2, Câmara/amendments/ALESC (+ tests, browser check) | web | 18, 20 | 17/11 |
| 22 | About page (+ tests, browser check) | web | 8, 14 | 17/11 |
| 23 | Accessibility, responsiveness and neutrality pass (+ e2e, browser check) | web | 9, 19–22 | 17/11 |
| 24 | Launch checklist and fixes | all | 16, 21, 23 | 19/11 |
| 25 | 👤 Release sign-off | all | 24 | 19/11 |

**Parallel tracks.** After step 7: the backend track (10 → 18) and the
frontend track (8 → 9, then 19 → 23) run in parallel. Steps 19–22 develop
against a local snapshot (`make ingest`) and are tested end-to-end against
the **golden snapshot** from step 14 — they do not wait for step 16.

---

## Phase A — foundation and the first deploy

### Step 1 — Repository skeleton + shared test vectors
- **Role:** repository owner · **Depends on:** — · **Branch:** `step-01-repo-skeleton`
- **Read:** `01-foundation/02-repository-layout.md` (whole), `01-foundation/05-testing-strategy.md` (whole), `00-overview/04-tech-stack.md` (version policy)
- **Do:**
  1. `.gitignore` (with the test-output entries) and `.gitattributes` (`backend/testdata/** -text`), exactly as in 02-repository-layout.md.
  2. `.editorconfig` (UTF-8, LF, 2 spaces; tabs for `*.go` and `Makefile`), `.nvmrc` (`24`).
  3. Root `biome.json` covering `frontend/` and `infra/`.
  4. `Makefile` with `ingest`, `dev`, `build-site`, `test*`, `check`, `preflight`, `infra-diff`, `infra-deploy`.
  5. `testdata/textnorm-vectors.json`: the shared input → expected pairs (accents, cedilla, case, extra spaces, hyphens, empty string) used by both test suites.
  6. A short public `README.md`: what the project is, the folders, `make` commands, how to run tests, link to `plan/README.md`.
- **Tests:** none yet (no code); the vectors file is validated as JSON (`jq . testdata/textnorm-vectors.json`).
- **Done when:** `make -n check` prints the commands; `jq` parses the vectors; no personal data in any file.
- **Commit:** `chore(repo): add repository skeleton and shared test vectors`

### Step 2 — Backend skeleton
- **Role:** backend lead · **Depends on:** 1 · **Branch:** `step-02-backend-skeleton`
- **Read:** `03-backend-etl/01-go-project-setup.md` (whole, incl. "Tests for this step")
- **Do:**
  1. `go mod init github.com/Miguel060/projetoAplicado2ifc/backend` (`go 1.25`).
  2. `internal/config` (constants, User-Agent from `INGEST_CONTACT`), `config/name-overrides.json`.
  3. `internal/pipeline`: `Options` (every source URL overridable, injectable clock) and `Run(ctx, Options)` with each stage stubbed.
  4. `cmd/ingest/main.go`: flags → `Options` → `pipeline.Run`.
- **Tests:** config (User-Agent with and without `INGEST_CONTACT`), flag → `Options` mapping, `Options` defaults point at the real URLs, `Run` calls the stages in order (stubbed).
- **Done when:** `go vet ./... && go build ./...` clean; `go run ./cmd/ingest -h` lists the flags; `make test-backend` green with coverage ≥ 85 %.
- **Commit:** `feat(etl): scaffold go module, pipeline and ingest command`

### Step 3 — Frontend skeleton + test tooling
- **Role:** frontend lead · **Depends on:** 1 · **Branch:** `step-03-frontend-skeleton`
- **Read:** `04-frontend/01-project-setup.md` (whole), `04-frontend/08-testing.md` (setup parts), `04-frontend/07-accessibility-responsiveness-neutrality.md` (layout/landmarks)
- **Do:**
  1. Vite + React 19 + TypeScript strict, pinned versions and scripts (incl. `test`, `test:coverage`, `test:e2e`).
  2. Tailwind CSS 4, `shadcn init` + listed components, tokens, self-hosted Inter; `index.html` (`lang="pt-BR"`).
  3. App shell (skip link, header, main, footer); router with `/`, `/candidatos`, `/candidatos/:slug`, `/sobre` placeholders, NotFound and error pages.
  4. Dev middleware serving `../backend/dist/data` at `/data`; preview middleware serving the e2e data dir.
  5. `vitest.config.ts` (jsdom, coverage thresholds 85 %), `playwright.config.ts` (Chromium + WebKit iPhone 13), `e2e/fixtures.ts` (axe + console-error guard); `npx playwright install --with-deps chromium webkit`.
- **Tests:** component tests for the shell (skip link first in tab order, landmarks) and the router (each route renders, unknown path → NotFound); e2e smoke: every route loads in both browsers, zero axe violations, zero console errors.
- **Browser check:** shell and the four placeholder routes (see 01-project-setup.md).
- **Done when:** typecheck/lint/build clean; `make test-frontend` (≥ 85 %) and `make test-e2e` green; browser check done.
- **Commit:** `feat(web): scaffold vite react app with shadcn and test tooling`

### Step 4 — Infrastructure code + assertion tests
- **Role:** AWS account owner · **Depends on:** 1 · **Branch:** `step-04-infra-code`
- **Read:** `05-infrastructure/01-cdk-stack.md` (all steps up to the local check, plus the tests step), `05-infrastructure/02-github-oidc-and-variables.md`
- **Do:** `infra/` package (pinned versions, `test` script), `bin/app.ts` (account pinned), `lib/spa-rewrite.js`, `lib/site-stack.ts` (bucket, OAC distribution, function, headers policy with CSP, **imported** OIDC provider, deploy role with the two exact subjects and minimal permissions, outputs).
- **Tests:** `infra/test/site-stack.test.ts` (every security invariant in the testing strategy) and `infra/test/spa-rewrite.test.ts` (rewrite table).
- **Done when:** `npm ci && npm run typecheck && npm test && npx cdk synth -q` clean, no credentials needed.
- **Commit:** `feat(infra): add cdk stack with security assertion tests`
- **Don't:** run `cdk deploy` (step 5).

### Step 5 — 👤 Deploy the stack and create the repository variables
- **Role:** AWS account owner (deploy) + repository admin (variables) · **Depends on:** 4 merged
- **Read:** `05-infrastructure/01-cdk-stack.md` (deploy, outputs, verify), `05-infrastructure/02-github-oidc-and-variables.md` (variables, Actions settings)
- **Do:**
  1. 👤 `make preflight`, `make infra-diff` (review), `make infra-deploy`.
  2. Record the outputs (`BucketName`, `DistributionId`, `SiteUrl`, `DeployRoleArn`) — not secret.
  3. 👤 Repository admin: enable Actions (workflow permissions = read) and create `AWS_DEPLOY_ROLE_ARN`, `SITE_BUCKET`, `SITE_DISTRIBUTION_ID`, `SITE_URL`, `INGEST_CONTACT`.
- **Done when:** the read-only verification in 01-cdk-stack.md passes (bucket not public, distribution deployed, `curl -I $SITE_URL` answers); five variables visible. No PR (no code change).
- **Don't:** store any AWS key anywhere; create a second OIDC provider.

### Step 6 — CI workflow with all test gates
- **Role:** AWS account owner · **Depends on:** 2, 3, 4 · **Branch:** `step-06-ci`
- **Read:** `06-ci-cd/01-ci.md` (whole)
- **Do:** `.github/workflows/ci.yml` with jobs `backend` (fmt, vet, tests with race + 85 % gate, build), `frontend` (typecheck, lint, Vitest with 85 % thresholds, build), `e2e` (Playwright Chromium + WebKit, report artifact on failure), `infra` (typecheck, tests, synth); `workflow_call` enabled.
- **Done when:** the PR shows the four jobs green; `make check` passes locally; a deliberately failing test on a scratch branch turns CI red (then delete the branch).
- **Commit:** `ci: add ci workflow with test and coverage gates`

### Step 7 — Deploy workflow + 👤 branch protection
- **Role:** AWS account owner + repository admin · **Depends on:** 5, 6 · **Branch:** `step-07-deploy`
- **Read:** `06-ci-cd/02-deploy-site.md` (whole), `05-infrastructure/02-github-oidc-and-variables.md` (branch protection)
- **Do:**
  1. `.github/workflows/deploy.yml`: every push to `main` + manual; `verify` = `ci.yml` (all tests); OIDC; account check; site sync excluding `data/*`; `index.html` no-cache; invalidation; smoke checks (missing `latest.json` = warning only).
  2. Merge → the skeleton goes live.
  3. 👤 Repository admin: protect `main` — PR required, checks `backend`, `frontend`, `e2e`, `infra` required.
- **Done when:** the `Deploy` run is green (verify included); `/`, `/candidatos`, `/sobre`, `/candidatos/nao-existe` return 200; `curl -I $SITE_URL` shows the security headers.
- **Commit:** `ci: deploy the site on every push to main after the full test suite`

### Step 8 — Frontend data access layer
- **Role:** frontend lead · **Depends on:** 3 · **Branch:** `step-08-data-access`
- **Read:** `01-foundation/04-json-contract.md`, `04-frontend/02-data-access.md` (whole, incl. tests)
- **Do:** `src/data/types.ts`, fetchers (`latest.json` short-lived; versioned files immutable), memoised index, route loaders, error mapping (schema mismatch, network, 404), pt-BR formatters, text normalisation.
- **Tests:** fetchers and loaders with mocked `fetch` (success, 404, network error, schema mismatch, memoisation), formatters (BRL, numbers, percentages, dates in America/Sao_Paulo), text normalisation against `testdata/textnorm-vectors.json`.
- **Browser check:** with and without a local snapshot, every route still renders (degraded states visible).
- **Done when:** `make test-frontend` ≥ 85 %; `make test-e2e` green; browser check done.
- **Commit:** `feat(web): add data access layer and formatters`

### Step 9 — Landing page (first real deploy)
- **Role:** frontend lead · **Depends on:** 7, 8 · **Branch:** `step-09-landing`
- **Read:** `04-frontend/03-landing-page.md` (whole), D14
- **Do:** implement `/` with every section, the live-figures strip and the "Dados em preparação" fallback; no mention of any public body as partner.
- **Tests:** component tests (sections present in order, figures from mocked index, fallback when the pointer is 404/invalid); e2e: landing with no data (404 intercepted) and with the golden/local snapshot, CTA navigates to `/candidatos`, axe clean, keyboard reachable.
- **Browser check:** the landing at 360 px and desktop, CTA and anchors, fallback state.
- **Done when:** tests green (≥ 85 %), browser check done, deploy run green and `$SITE_URL/` shows the landing with the fallback.
- **Commit:** `feat(web): add landing page`

---

## Phase B — the data pipeline

### Step 10 — Fetch and parse plumbing + fake upstream
- **Role:** backend lead · **Depends on:** 2 · **Branch:** `step-10-fetch-parse`
- **Read:** `03-backend-etl/02-fetch-and-parse.md` (whole), `03-backend-etl/09-integration-tests.md` (fake upstream, fixtures rules)
- **Do:** `fetch.Client` (timeouts, retries/backoff, rate limit, User-Agent), URL-keyed cache, remote zip reader over HTTP ranges with full-download fallback, provenance registry, `csvx`, value parsers, `textnorm`; `internal/testsrv` (fixture serving, in-memory zips, `Range`, failure injection).
- **Tests:** retry on 500/504/429 with backoff, no retry on 404, cache hit on second call, remote zip reads one member via ranges and falls back when ranges are ignored, Latin-1/Windows-1252/BOM decoding, BR numbers in both formats, sentinels → null, `textnorm` against the shared vectors.
- **Done when:** `make test-backend` ≥ 85 %; no test touches the network.
- **Commit:** `feat(etl): add http client, cache, remote zip, csv parsing and fake upstream`

### Step 11 — TSE: candidates and votes
- **Role:** backend lead · **Depends on:** 10 · **Branch:** `step-11-tse-candidates-votes`
- **Read:** `02-data-sources/01-tse.md`, `03-backend-etl/03-load-tse.md` (Types, Step A, Step B, tests), `03-backend-etl/09-integration-tests.md` (TSE fixtures)
- **Do:** synthetic TSE fixtures (candidates, complementary, history, votes); `LoadCandidates` (CPF in memory only) and `LoadVotes` (municipality code resolved at run time).
- **Tests:** loaders against the fixtures through `testsrv`: column mapping, status from `DS_SITUACAO_CANDIDATO_TOT`, re-election from history, municipality code resolution (and failure when absent), state/municipality totals.
- **Done when:** `make test-backend` ≥ 85 %; optional manual run against the real sources matches the reference values in the doc (645 SC deputy candidates; Videira TSE code `83798`).
- **Commit:** `feat(etl): load tse candidates and votes`

### Step 12 — Selection rule
- **Role:** backend lead · **Depends on:** 11 · **Branch:** `step-12-selection`
- **Read:** `03-backend-etl/04-select-candidates.md` (whole), D1
- **Do:** top 20 per office by valid votes in Videira (ties at 20 included, competition rank) ∪ born in Videira with `DEFERIDO`; reasons recorded.
- **Tests:** ties exactly at 20, ties crossing 20, fewer than 20 candidates, born-in-Videira with 0 votes, `INDEFERIDO` excluded, both reasons on one candidate, determinism.
- **Done when:** `make test-backend` ≥ 85 %; the doc's expected result matches on a manual real run.
- **Commit:** `feat(etl): select candidates linked to the municipality`

### Step 13 — TSE: assets, finance, social, photos
- **Role:** backend lead · **Depends on:** 12 · **Branch:** `step-13-tse-enrichment`
- **Read:** `03-backend-etl/03-load-tse.md` (Step C, tests)
- **Do:** fixtures + `LoadAssets`, `LoadFinance` (aggregates only), `LoadSocial` (http/https only), `LoadPhotos` (selected only).
- **Tests:** aggregation sums and ordering, no donor/supplier identity retained, non-http social URLs dropped, missing photo → null.
- **Done when:** `make test-backend` ≥ 85 %.
- **Commit:** `feat(etl): load assets, campaign finance, social links and photos`

### Step 14 — SQLite + JSON export + pipeline integration test and golden snapshot
- **Role:** backend lead · **Depends on:** 13 · **Branch:** `step-14-export`
- **Read:** `01-foundation/03-data-model.md`, `01-foundation/04-json-contract.md`, `03-backend-etl/07-export-json.md`, `03-backend-etl/09-integration-tests.md` (pipeline test, golden)
- **Do:** embedded `schema.sql` + `store.Write`; export structs; slugs (`^[a-z0-9-]+$`, unique) and search text; `index.json`, profiles, photos, `dataset.sqlite` + sha256, `latest.json` (`chamber`/`amendments` null for now). Then `internal/pipeline/pipeline_test.go` running the whole pipeline against `testsrv` and writing/comparing `backend/testdata/golden/data/` (`-update` to regenerate).
- **Tests:** store round trip, no CPF anywhere in the SQLite, slug regex and collision abort, export shapes match the contract, the integration test against the golden snapshot.
- **Done when:** `make test-backend` ≥ 85 %; `make ingest` produces a real local snapshot; `make test-e2e` passes against the new golden snapshot; hand the local snapshot to the frontend lead.
- **Commit:** `feat(etl): export sqlite and json snapshot with golden integration test`

### Step 15 — Validation + publish script
- **Role:** AWS account owner · **Depends on:** 14 · **Branch:** `step-15-validate-publish`
- **Read:** `03-backend-etl/08-validate-and-publish.md` (whole)
- **Do:** `validate.Check` (all gates incl. `-previous`), `backend/scripts/publish-data.sh`, `make publish-data-local`, rollback command.
- **Tests:** every gate passes on the golden snapshot and fails on a mutated copy (missing profile, empty office, missing source, bad slug); `-previous` 404 = skip, 50 % drop = fail; the script passes `bash -n` (and `shellcheck` if available).
- **Done when:** `make test-backend` ≥ 85 %.
- **Commit:** `feat(etl): validate snapshots and add the publish script`
- **Don't:** publish to S3 yet.

### Step 16 — Refresh workflow (👤 first snapshot live)
- **Role:** AWS account owner · **Depends on:** 7, 15 · **Branch:** `step-16-refresh-data`
- **Read:** `06-ci-cd/03-refresh-data.md` (whole), D16
- **Do:** `.github/workflows/refresh-data.yml`: weekly cron, manual, push to `main` touching `backend/**`; build job runs `go vet` + the Go tests (85 % gate) **before** the ETL; artifact → publish job in the documented order; prune keep 5; smoke checks. Merge; 👤 run manually if the push trigger did not fire.
- **Done when:** the run is green; `curl $SITE_URL/data/latest.json` shows the new version; the landing shows real figures.
- **Commit:** `ci: test, refresh and publish the data snapshot`

### Step 17 — Câmara loaders
- **Role:** AWS account owner · **Depends on:** 14 · **Branch:** `step-17-camara`
- **Read:** `02-data-sources/02-camara.md`, `03-backend-etl/05-load-camara.md` (whole), `03-backend-etl/09-integration-tests.md` (Câmara fixtures)
- **Do:** fixtures (API pages with `rel=next`, bulk votes, CEAP zip); `MatchDeputies` (CPF join, then forget), `LoadBills`, `LoadRollCallVotes`, `LoadExpenses`; fill `chamber`; regenerate the golden snapshot.
- **Tests:** CPF match and no CPF retained, pagination, 504 retry, bill-type filter, vote totals and recent ordering, CEAP join on `ideCadastro`, non-incumbent → `chamber: null`.
- **Done when:** `make test-backend` ≥ 85 %; golden updated and `make test-e2e` green.
- **Commit:** `feat(etl): load chamber of deputies activity`

### Step 18 — Parliamentary amendments
- **Role:** AWS account owner · **Depends on:** 17 · **Branch:** `step-18-amendments`
- **Read:** `02-data-sources/03-portal-transparencia.md`, `03-backend-etl/06-load-amendments.md` (whole)
- **Do:** fixtures (CGU zip with both members, Windows-1252); per-beneficiary rows in Videira + metadata; author matching with overrides; natural persons aggregated; fill `amendments`; regenerate the golden snapshot.
- **Tests:** municipality filter, `receivedInMunicipality` sums, natural persons never named, override file applied, unmatched authors logged, ordering.
- **Done when:** `make test-backend` ≥ 85 %; golden updated and `make test-e2e` green.
- **Commit:** `feat(etl): load parliamentary amendments received in the municipality`

---

## Phase C — the screens

### Step 19 — Search page
- **Role:** frontend lead · **Depends on:** 8, 14 · **Branch:** `step-19-search`
- **Read:** `04-frontend/04-search-page.md` (whole, incl. tests and browser check)
- **Do:** `/candidatos`: search, filters and sort in the URL, identical cards, visible sort-rule sentence, result count via `aria-live`, empty state.
- **Tests:** filter/sort/search units (accent-insensitive, combined filters, ties), URL round trip, component test of the page with fixture data; e2e: search, each filter, sort, back/forward restores state, empty state, axe clean, both browsers.
- **Browser check:** every control, keyboard-only, 360 px, GIF `step-19-search.gif`.
- **Done when:** `make test-frontend` ≥ 85 %, `make test-e2e` green, browser check done.
- **Commit:** `feat(web): add candidate search page`

### Step 20 — Profile page, part 1 (TSE blocks)
- **Role:** frontend lead · **Depends on:** 19 · **Branch:** `step-20-profile-tse`
- **Read:** `04-frontend/05-profile-page.md` (layout, `SourceNote`, TSE blocks, "Proposals" empty state, tests, browser check)
- **Do:** `/candidatos/:slug` with the fixed section order; `SourceNote` on every block.
- **Tests:** component tests per block (data, empty state, source note), unknown slug → NotFound; e2e: open a profile from search, back keeps filters, axe clean.
- **Browser check:** three different candidates, links open in new tab with the hint, GIF `step-20-profile-tse.gif`.
- **Done when:** tests green (≥ 85 %), browser check done.
- **Commit:** `feat(web): add candidate profile with tse data`

### Step 21 — Profile page, part 2 (Câmara, amendments, ALESC)
- **Role:** frontend lead · **Depends on:** 18, 20 · **Branch:** `step-21-profile-activity`
- **Read:** `04-frontend/05-profile-page.md` (bills, Câmara activity, amendments, ALESC, tests, browser check)
- **Do:** bills as "Proposals" for incumbents; roll-call votes and CEAP tables with accessible bars; amendments with "Recebido por beneficiários em Videira" as the headline and whole-amendment values labelled; ALESC link for state deputies running for re-election.
- **Tests:** component tests for incumbent vs non-incumbent, amendments headline and labels, aggregated natural persons; e2e on the golden incumbent and state-deputy profiles, axe clean.
- **Browser check:** incumbent profile end to end, tables at 360 px (scroll region focusable), GIF `step-21-profile-activity.gif`.
- **Done when:** tests green (≥ 85 %), browser check done.
- **Commit:** `feat(web): add chamber activity and amendments to the profile`

### Step 22 — About page
- **Role:** backend lead (content) + frontend lead (review) · **Depends on:** 8, 14 · **Branch:** `step-22-about`
- **Read:** `04-frontend/06-about-page.md` (whole), D1, D10, D14
- **Do:** `/sobre`: selection rule, sources with dates, impartiality and privacy statements, dataset download (size + sha256), credits with the "not an official site" disclaimer.
- **Tests:** component test (every source listed with both dates, download link, disclaimer present, no partner mention); e2e: download link responds, axe clean.
- **Browser check:** links, download, 360 px, GIF `step-22-about.gif`.
- **Done when:** tests green (≥ 85 %), browser check done.
- **Commit:** `feat(web): add about and methodology page`

### Step 23 — Accessibility, responsiveness and neutrality pass
- **Role:** frontend lead · **Depends on:** 9, 19–22 · **Branch:** `step-23-a11y`
- **Read:** `04-frontend/07-accessibility-responsiveness-neutrality.md` (whole), `04-frontend/08-testing.md`
- **Do:** apply every rule; fix what the manual procedure finds (VoiceOver/NVDA, 200 % zoom, 320 px, real phone, Lighthouse).
- **Tests:** e2e keyboard journeys across all routes, reflow at 320 px (no horizontal scroll), reduced-motion respected, axe clean everywhere in both browsers.
- **Browser check:** full site tour, GIF `step-23-a11y.gif`.
- **Done when:** the doc's checklist ticked; Lighthouse accessibility ≥ 95 on all routes; tests green (≥ 85 %).
- **Commit:** `fix(web): accessibility, responsiveness and neutrality pass`

---

## Phase D — launch

### Step 24 — Launch checklist and fixes
- **Role:** everyone · **Depends on:** 16, 21, 23
- **Read:** `07-delivery/02-launch-checklist.md` (whole, incl. "Tests")
- **Do:** run every section; one issue per finding; fix blockers in small PRs, each with a regression test (`fix(<scope>): …`).
- **Done when:** no blocker open; all CI jobs green on `main`; coverage ≥ 85 % on backend and frontend; data spot-checks of 3 candidates match the official sites.

### Step 25 — 👤 Release sign-off
- **Role:** AWS account owner + repository admin · **Depends on:** 24
- **Read:** `07-delivery/02-launch-checklist.md` (Security, Operations and cost, Sign-off), `01-foundation/01-prerequisites.md` (credential hygiene)
- **Do:** 👤 credential hygiene (least-privilege IAM user for CDK, no root keys, MFA); 👤 final manual refresh-data run; sign-off table filled; record the URL in Modelo 4.
- **Done when:** sign-off complete; site live with the latest snapshot; budgets in place.
