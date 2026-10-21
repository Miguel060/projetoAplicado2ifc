# STEPS — build it one step at a time

The executable version of the plan. Each step is **one pull request** small
enough for one working session, and leaves `main` working. To build the
project, say: **"do step 1"**, then **"do step 2"**, and so on.

`07-delivery/01-implementation-order.md` is the calendar/ownership view of the
same work; this file is the execution order.

## How every step runs

1. **Check the dependencies** listed in the step are merged into `main`.
   If not, stop and say which one is missing.
2. **Read** the plan files listed under *Read* — they hold the detail (code
   sketches, exact commands, pitfalls). This file only sequences them.
3. **Branch:** `git switch main && git pull && git switch -c step-NN-<slug>`.
4. **Do** the items in order. Production code only — no test suites.
5. **Verify** every item under *Done when*. All must pass.
6. **Commit** with the given message (Conventional Commits), push, open a PR
   into `main`. CI must be green; merge. Merging to `main` deploys the site
   automatically (`deploy.yml`); changes under `backend/` also refresh the
   data (`refresh-data.yml`).
7. **Stop.** Do not start the next step's work. Report what was done and
   anything that deviated from the plan (and update the plan file if a
   decision changed — `00-overview/02-decisions.md` first).

Markers: 👤 = needs a specific person (AWS account owner or repository
admin) to run or approve an action that has outside effects. Prepare
everything, show the exact command, and wait for that person's go-ahead.
Every AWS command uses `--profile personal` and the account preflight
(`make preflight`) first.

## Overview

| # | Step | Area | Depends on | Deadline (Modelo 4) |
|---|---|---|---|---|
| 1 | Repository skeleton | repo | — | 09/11 |
| 2 | Backend skeleton | etl | 1 | 09/11 |
| 3 | Frontend skeleton | web | 1 | 09/11 |
| 4 | Infrastructure code | infra | 1 | 09/11 |
| 5 | 👤 Deploy the stack + repository variables | infra | 4 | 09/11 |
| 6 | CI workflow | ci | 2, 3, 4 | 09/11 |
| 7 | Deploy workflow + 👤 branch protection | ci | 5, 6 | 09/11 |
| 8 | Frontend data access layer | web | 3 | 09/11 |
| 9 | Landing page — first real deploy | web | 7, 8 | 09/11 |
| 10 | Fetch and parse plumbing | etl | 2 | 12/11 |
| 11 | TSE: candidates and votes | etl | 10 | 12/11 |
| 12 | Selection rule | etl | 11 | 12/11 |
| 13 | TSE: assets, finance, social, photos | etl | 12 | 12/11 |
| 14 | SQLite store + JSON export (TSE-only snapshot) | etl | 13 | 12/11 |
| 15 | Validation + publish script | etl | 14 | 12/11 |
| 16 | Refresh workflow — 👤 first snapshot live | ci | 7, 15 | 12/11 |
| 17 | Câmara loaders | etl | 14 | 12/11 |
| 18 | Parliamentary amendments | etl | 17 | 12/11 |
| 19 | Search page | web | 8, 14 | 17/11 |
| 20 | Profile page — part 1 (TSE blocks) | web | 19 | 17/11 |
| 21 | Profile page — part 2 (Câmara, amendments, ALESC) | web | 18, 20 | 17/11 |
| 22 | About page | web | 8, 14 | 17/11 |
| 23 | Accessibility, responsiveness and neutrality pass | web | 9, 19–22 | 17/11 |
| 24 | Launch checklist and fixes | all | 16, 21, 23 | 19/11 |
| 25 | 👤 Release sign-off | all | 24 | 19/11 |

**Parallel tracks.** After step 7: the backend track (10 → 18) and the
frontend track (8 → 9, then 19 → 23 once step 14 exists locally) run in
parallel. Steps 19–22 use a local snapshot from `make ingest`; they do not
wait for step 16.

---

## Phase A — foundation and the first deploy

### Step 1 — Repository skeleton
- **Role:** repository owner · **Depends on:** — · **Branch:** `step-01-repo-skeleton`
- **Read:** `01-foundation/02-repository-layout.md` (whole), `00-overview/04-tech-stack.md` (version policy)
- **Do:**
  1. Create `.gitignore` with the exact content in 02-repository-layout.md (keep `plan/` tracked).
  2. Create `.editorconfig` (UTF-8, LF, 2 spaces; tabs for `*.go` and `Makefile`), `.nvmrc` (`24`).
  3. Create root `biome.json` covering `frontend/` and `infra/` (ignore `dist`, `cdk.out`, `components/ui` formatting rules as in the frontend doc).
  4. Create `Makefile` with `ingest`, `dev`, `build-site`, `check`, `preflight`, `infra-diff`, `infra-deploy` (targets may point at folders created by later steps).
  5. Create a short public `README.md`: what the project is, the three folders, `make` commands, link to `plan/README.md`.
- **Produces:** `.gitignore`, `.editorconfig`, `.nvmrc`, `biome.json`, `Makefile`, `README.md`
- **Done when:** `git status` shows only those files; `make -n check` prints the commands; no personal data in any file.
- **Commit:** `chore(repo): add gitignore, editorconfig, nvmrc, biome and makefile`
- **Don't:** create `backend/`, `frontend/` or `infra/` yet.

### Step 2 — Backend skeleton
- **Role:** backend lead · **Depends on:** 1 · **Branch:** `step-02-backend-skeleton`
- **Read:** `03-backend-etl/01-go-project-setup.md` (whole)
- **Do:**
  1. `go mod init github.com/Miguel060/projetoAplicado2ifc/backend` (`go 1.25`).
  2. `internal/config/config.go` with every constant (year, UF, municipality, IBGE code, topN, offices, URLs, User-Agent built from `INGEST_CONTACT`).
  3. `config/name-overrides.json` (empty object, with its documented shape).
  4. `cmd/ingest/main.go`: all flags (`-out`, `-cache`, `-previous`, `-log-format`, …) and the orchestration order in `main()` as stubs that log "not implemented yet" per stage.
- **Produces:** `backend/go.mod`, `backend/cmd/ingest/main.go`, `backend/internal/config/config.go`, `backend/config/name-overrides.json`
- **Done when:** `cd backend && go vet ./... && go build ./...` clean; `go run ./cmd/ingest -h` lists the flags; `gofmt -l .` prints nothing.
- **Commit:** `feat(etl): scaffold go module and ingest command`
- **Don't:** implement any loader.

### Step 3 — Frontend skeleton
- **Role:** frontend lead · **Depends on:** 1 · **Branch:** `step-03-frontend-skeleton`
- **Read:** `04-frontend/01-project-setup.md` (whole), `04-frontend/07-accessibility-responsiveness-neutrality.md` (layout/landmarks rules only)
- **Do:**
  1. Scaffold Vite + React 19 + TypeScript strict with the pinned versions and scripts (`dev`, `build`, `preview`, `typecheck`, `lint`, `format`).
  2. Tailwind CSS 4, `shadcn init`, add the listed shadcn components; design tokens in `src/index.css`; self-hosted Inter.
  3. `index.html` (`lang="pt-BR"`, meta), app shell (header, skip link, main, footer).
  4. Router with the four routes (`/`, `/candidatos`, `/candidatos/:slug`, `/sobre`) as placeholder pages, plus NotFound and error pages.
  5. The dev-only middleware that serves `../backend/dist/data` at `/data`.
- **Produces:** `frontend/**` (no data access yet)
- **Done when:** `npm run typecheck && npm run lint && npm run build` clean; `npm run dev` shows the shell and all four routes; keyboard Tab reaches the skip link first.
- **Commit:** `feat(web): scaffold vite react app with tailwind and shadcn`
- **Don't:** fetch data; write page content.

### Step 4 — Infrastructure code
- **Role:** AWS account owner · **Depends on:** 1 · **Branch:** `step-04-infra-code`
- **Read:** `05-infrastructure/01-cdk-stack.md` Steps 1–7, `05-infrastructure/02-github-oidc-and-variables.md` (trust + imported provider)
- **Do:**
  1. `infra/package.json` (pinned CDK versions), `tsconfig.json`, `cdk.json`.
  2. `bin/app.ts` with the pinned account `436812234154` / `us-east-1`.
  3. `lib/spa-rewrite.js` (CloudFront Function) and `lib/site-stack.ts`: private bucket, distribution with OAC, function association, response-headers policy (CSP), **imported** OIDC provider, deploy role with the two exact subjects and minimal permissions, outputs.
- **Produces:** `infra/**`
- **Done when:** `cd infra && npm ci && npm run typecheck && npx cdk synth -q` clean (no credentials needed); the synthesized trust policy contains exactly the two subjects and no wildcard.
- **Commit:** `feat(infra): add cdk stack for bucket, distribution and deploy role`
- **Don't:** run `cdk deploy` (that is step 5).

### Step 5 — 👤 Deploy the stack and create the repository variables
- **Role:** AWS account owner (deploy) + repository admin (variables) · **Depends on:** 4 merged
- **Read:** `05-infrastructure/01-cdk-stack.md` Steps 8–10, `05-infrastructure/02-github-oidc-and-variables.md` (variables, Actions settings)
- **Do:**
  1. 👤 `make preflight` → must print nothing and succeed (account `436812234154`).
  2. 👤 `make infra-diff`, review it, then `make infra-deploy`.
  3. Record the outputs (`BucketName`, `DistributionId`, `SiteUrl`, `DeployRoleArn`) — they are not secret.
  4. 👤 Repository admin: enable Actions (workflow permissions = read) and create the variables `AWS_DEPLOY_ROLE_ARN`, `SITE_BUCKET`, `SITE_DISTRIBUTION_ID`, `SITE_URL`, `INGEST_CONTACT`.
- **Produces:** live AWS resources; repository variables. No code change (no PR needed).
- **Done when:** the read-only checks in 01-cdk-stack.md Step 10 pass (bucket not public, distribution deployed, `curl -I $SITE_URL` answers); all five variables visible in Settings.
- **Don't:** store any AWS key anywhere; create a second OIDC provider.

### Step 6 — CI workflow
- **Role:** AWS account owner · **Depends on:** 2, 3, 4 · **Branch:** `step-06-ci`
- **Read:** `06-ci-cd/01-ci.md` (whole)
- **Do:** add `.github/workflows/ci.yml` exactly as documented (backend, frontend, infra jobs; `workflow_call` enabled).
- **Done when:** the PR shows the three CI jobs green; `make check` passes locally.
- **Commit:** `ci: add ci workflow`

### Step 7 — Deploy workflow + 👤 branch protection
- **Role:** AWS account owner + repository admin · **Depends on:** 5, 6 · **Branch:** `step-07-deploy`
- **Read:** `06-ci-cd/02-deploy-site.md` (whole), `05-infrastructure/02-github-oidc-and-variables.md` (branch protection)
- **Do:**
  1. Add `.github/workflows/deploy.yml` (every push to `main` + manual; `verify` reuses `ci.yml`; OIDC; account check; site sync excluding `data/*`; `index.html` no-cache; invalidation; smoke checks — a missing `latest.json` is a warning only).
  2. Merge → the skeleton site goes live.
  3. 👤 Repository admin: protect `main` (PR required, CI checks required).
- **Done when:** the `Deploy` run on `main` is green; `$SITE_URL/`, `/candidatos`, `/sobre` and `/candidatos/nao-existe` return 200; `curl -I $SITE_URL` shows the security headers.
- **Commit:** `ci: deploy the site on every push to main`

### Step 8 — Frontend data access layer
- **Role:** frontend lead · **Depends on:** 3 · **Branch:** `step-08-data-access`
- **Read:** `01-foundation/04-json-contract.md` (whole), `04-frontend/02-data-access.md` (whole)
- **Do:** `src/data/types.ts` (copy of the contract), fetchers (`latest.json` short-lived; versioned files immutable), memoised index, route loaders, error mapping (schema mismatch, network, 404), pt-BR formatters, text normalisation identical to Go's.
- **Done when:** typecheck/lint/build clean; with no `backend/dist/data`, the app still renders every route (loaders degrade as documented).
- **Commit:** `feat(web): add data access layer and formatters`

### Step 9 — Landing page (first real deploy)
- **Role:** frontend lead · **Depends on:** 7, 8 · **Branch:** `step-09-landing`
- **Read:** `04-frontend/03-landing-page.md` (whole), `00-overview/02-decisions.md` D14
- **Do:** implement `/` with every section of the doc, the live-figures strip, and the "Dados em preparação" fallback. No mention of any public body as partner.
- **Done when:** locally and on `$SITE_URL` the landing renders fully with no data snapshot (fallback visible), CTA goes to `/candidatos`; keyboard-only and 320 px-wide checks pass; deploy run green.
- **Commit:** `feat(web): add landing page`

---

## Phase B — the data pipeline

### Step 10 — Fetch and parse plumbing
- **Role:** backend lead · **Depends on:** 2 · **Branch:** `step-10-fetch-parse`
- **Read:** `03-backend-etl/02-fetch-and-parse.md` (whole)
- **Do:** `fetch.Client` (timeouts, retries/backoff, polite rate limit, User-Agent), URL-keyed cache in `backend/.cache`, remote zip reader over HTTP ranges with full-download fallback, provenance registry, `csvx` (Latin-1/Windows-1252/BOM, `;`), value parsers (BR numbers both formats, dates, sentinels → null), `textnorm`.
- **Done when:** the manual checks in the doc pass (e.g. read only the `_SC.csv` member of the 454 MB votes zip, second run served from cache).
- **Commit:** `feat(etl): add http client, cache, remote zip and csv parsing`

### Step 11 — TSE: candidates and votes
- **Role:** backend lead · **Depends on:** 10 · **Branch:** `step-11-tse-candidates-votes`
- **Read:** `02-data-sources/01-tse.md`, `03-backend-etl/03-load-tse.md` (Types, Step A, Step B)
- **Do:** `LoadCandidates` (consulta_cand + complementar + historico; CPF kept in memory only) and `LoadVotes` (Videira code resolved at run time; municipality and state totals).
- **Done when:** a debug log prints 645 SC deputy candidates, Videira TSE code `83798`, valid votes federal 26,315 / state 26,788 (values of 2026-10-21; re-check if the TSE republishes).
- **Commit:** `feat(etl): load tse candidates and votes`

### Step 12 — Selection rule
- **Role:** backend lead · **Depends on:** 11 · **Branch:** `step-12-selection`
- **Read:** `03-backend-etl/04-select-candidates.md` (whole), D1
- **Do:** top 20 per office by valid votes in Videira (ties at 20 included, competition rank) ∪ born in Videira with `DEFERIDO` candidacy; reasons recorded.
- **Done when:** the "Expected result" section of the doc matches (counts per office, cutoff votes, born-in-Videira entries).
- **Commit:** `feat(etl): select candidates linked to the municipality`

### Step 13 — TSE: assets, finance, social, photos
- **Role:** backend lead · **Depends on:** 12 · **Branch:** `step-13-tse-enrichment`
- **Read:** `03-backend-etl/03-load-tse.md` (Step C)
- **Do:** `LoadAssets`, `LoadFinance` (aggregates only — no donor/supplier identities), `LoadSocial` (http/https only), `LoadPhotos` (selected only).
- **Done when:** the doc's checks pass; no CPF or donor/supplier name anywhere in memory dumps/logs.
- **Commit:** `feat(etl): load assets, campaign finance, social links and photos`

### Step 14 — SQLite store + JSON export (TSE-only snapshot)
- **Role:** backend lead · **Depends on:** 13 · **Branch:** `step-14-export`
- **Read:** `01-foundation/03-data-model.md`, `01-foundation/04-json-contract.md`, `03-backend-etl/07-export-json.md` (whole)
- **Do:** embedded `schema.sql` + `store.Write`; export structs mirroring the TS contract; slugs (`^[a-z0-9-]+$`, unique) and search text; `index.json`, `candidates/<slug>.json`, photos, `dataset.sqlite` + sha256, `latest.json`. `chamber`/`amendments` are `null` for now.
- **Done when:** `make ingest` produces `backend/dist/data/latest.json` + `<version>/…`; the `sqlite3`/`jq` checks in the doc pass; running twice gives identical JSON except timestamps; **hand the snapshot to the frontend lead** (they run `make ingest` themselves).
- **Commit:** `feat(etl): write sqlite dataset and export json snapshot`

### Step 15 — Validation + publish script
- **Role:** AWS account owner · **Depends on:** 14 · **Branch:** `step-15-validate-publish`
- **Read:** `03-backend-etl/08-validate-and-publish.md` (whole)
- **Do:** `validate.Check` (all gates, incl. `-previous` comparison; 404 = no live snapshot), `backend/scripts/publish-data.sh`, `make publish-data-local`, rollback command.
- **Done when:** a deliberately broken snapshot (e.g. delete a profile file) makes the ETL exit non-zero and writes no `latest.json`; the script passes `bash -n` and `shellcheck` if available.
- **Commit:** `feat(etl): validate snapshots and add the publish script`
- **Don't:** publish to S3 yet (step 16 does it through CI).

### Step 16 — Refresh workflow (👤 first snapshot live)
- **Role:** AWS account owner · **Depends on:** 7, 15 · **Branch:** `step-16-refresh-data`
- **Read:** `06-ci-cd/03-refresh-data.md` (whole)
- **Do:** add `.github/workflows/refresh-data.yml` (weekly cron, manual, push to `main` touching `backend/**`; build job → artifact → publish job in the documented order; prune keep 5; smoke checks). Merge; 👤 if the push trigger did not fire, run it manually.
- **Done when:** `curl $SITE_URL/data/latest.json` returns the new version; the landing page shows real figures instead of "Dados em preparação".
- **Commit:** `ci: refresh and publish the data snapshot`

### Step 17 — Câmara loaders
- **Role:** AWS account owner · **Depends on:** 14 · **Branch:** `step-17-camara`
- **Read:** `02-data-sources/02-camara.md`, `03-backend-etl/05-load-camara.md` (whole)
- **Do:** `MatchDeputies` (CPF join, then forget the CPF), `LoadBills` (API, PL/PLP/PEC/PDL, paginated, retry on 504), `LoadRollCallVotes` (bulk per year), `LoadExpenses` (CEAP bulk, join on `ideCadastro`); fill `chamber` in the export.
- **Done when:** the doc's checks pass; profiles of incumbent federal deputies have a non-null `chamber`; others stay `null`.
- **Commit:** `feat(etl): load chamber of deputies activity`

### Step 18 — Parliamentary amendments
- **Role:** AWS account owner · **Depends on:** 17 · **Branch:** `step-18-amendments`
- **Read:** `02-data-sources/03-portal-transparencia.md`, `03-backend-etl/06-load-amendments.md` (whole)
- **Do:** per-beneficiary rows in Videira + amendment metadata; author → deputy matching with `config/name-overrides.json`; natural persons aggregated; fill `amendments` (headline `receivedInMunicipality`).
- **Done when:** the doc's checks pass (incl. "no natural person named"); unmatched authors are logged, and resolved via overrides if they are selected deputies.
- **Commit:** `feat(etl): load parliamentary amendments received in the municipality`

---

## Phase C — the screens

### Step 19 — Search page
- **Role:** frontend lead · **Depends on:** 8, 14 (local snapshot) · **Branch:** `step-19-search`
- **Read:** `04-frontend/04-search-page.md` (whole)
- **Do:** `/candidatos`: search box, filters and sort in the URL, identical cards, visible sort rule sentence, result count via `aria-live`, empty state.
- **Done when:** the doc's checks pass on the local snapshot; back/forward restores filters; works at 320 px.
- **Commit:** `feat(web): add candidate search page`

### Step 20 — Profile page, part 1 (TSE blocks)
- **Role:** frontend lead · **Depends on:** 19 · **Branch:** `step-20-profile-tse`
- **Read:** `04-frontend/05-profile-page.md` (layout, `SourceNote`, identification, votes, personal data, assets, campaign finance, social links, official links)
- **Do:** `/candidatos/:slug` with the fixed section order; `SourceNote` on every block; the "Proposals" block with its neutral message when `chamber` is null.
- **Done when:** every TSE block renders with its source note on 3 different candidates; unknown slug → NotFound; "back to search" keeps filters.
- **Commit:** `feat(web): add candidate profile with tse data`

### Step 21 — Profile page, part 2 (Câmara, amendments, ALESC)
- **Role:** frontend lead · **Depends on:** 18, 20 · **Branch:** `step-21-profile-activity`
- **Read:** `04-frontend/05-profile-page.md` (proposals/bills, Câmara activity, amendments, ALESC)
- **Do:** bills as "Proposals" for incumbents; roll-call votes and CEAP tables with accessible bars; amendments with "Recebido por beneficiários em Videira" as the headline and whole-amendment values labelled as such; ALESC link for state deputies running for re-election.
- **Done when:** an incumbent federal deputy shows all blocks; a non-incumbent shows the neutral empty states; no beneficiary that is a natural person is named.
- **Commit:** `feat(web): add chamber activity and amendments to the profile`

### Step 22 — About page
- **Role:** backend lead (content) + frontend lead (review) · **Depends on:** 8, 14 · **Branch:** `step-22-about`
- **Read:** `04-frontend/06-about-page.md` (whole), D1, D10, D14
- **Do:** `/sobre`: selection rule, sources with dates from `index.sources`, impartiality and privacy statements, dataset download (size + sha256), project credits with the "not an official site" disclaimer.
- **Done when:** every source in `index.json` is listed with both dates and a working link; the download works.
- **Commit:** `feat(web): add about and methodology page`

### Step 23 — Accessibility, responsiveness and neutrality pass
- **Role:** frontend lead · **Depends on:** 9, 19–22 · **Branch:** `step-23-a11y`
- **Read:** `04-frontend/07-accessibility-responsiveness-neutrality.md` (whole)
- **Do:** apply every implementation rule; run the manual verification procedure (keyboard only, VoiceOver/NVDA, Lighthouse + axe extension, 200 % zoom, 320 px, real phone); fix what fails.
- **Done when:** the doc's checklist is fully ticked; Lighthouse accessibility ≥ 95 on all four routes.
- **Commit:** `fix(web): accessibility, responsiveness and neutrality pass`

---

## Phase D — launch

### Step 24 — Launch checklist and fixes
- **Role:** everyone · **Depends on:** 16, 21, 23
- **Read:** `07-delivery/02-launch-checklist.md` (whole)
- **Do:** run sections 1–7; open one issue per finding; fix blockers in small PRs (`fix(<scope>): …`); re-run the affected checks.
- **Done when:** no blocker open; data spot-checks of 3 candidates match the official sites.

### Step 25 — 👤 Release sign-off
- **Role:** AWS account owner + repository admin · **Depends on:** 24
- **Read:** `07-delivery/02-launch-checklist.md` (Security, Operations and cost, Sign-off), `01-foundation/01-prerequisites.md` (credential hygiene)
- **Do:** 👤 credential hygiene done (least-privilege IAM user for CDK, no root keys, MFA); 👤 final manual refresh-data run; sign-off table filled; record the URL in Modelo 4.
- **Done when:** sign-off table complete; site live with the latest snapshot; budgets in place.
