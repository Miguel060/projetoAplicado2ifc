# 02 — Decision log

Every decision below was taken with the group on 2026-10-21. Each entry says
what was decided, what was rejected, and why. If a decision changes, edit the
entry and note the date — do not delete the history.

---

## D1 — Who is "linked to Videira"

**Decided:** top **20** candidates by valid nominal votes in Videira, per
office (federal deputy, state deputy), **plus** every candidate for those
offices born in Videira. Ties at position 20 are all included. `topN` is a
constant in `backend/internal/config`.

**Rejected:**

| Option | Why not |
|---|---|
| Only born in Videira | The 2026 SC file has **4** people born in Videira across all offices — too few to be useful. |
| All SC candidates (~645 for deputy) | Videira's representation gets lost in noise; profile volume makes manual QA impossible before 2026-11-19. |
| Hand-picked list | Opens the panel to accusations of bias. An objective, published rule does not. |

---

## D2 — The "Proposals" block for deputies

**Decided:** for 2023–2027 federal deputies, show **authored bills** (PL, PLP,
PEC, PDL) from the Câmara dos Deputados API. For everyone else, a neutral
"no proposal registered in an official source for this office" message that
explains the TSE only publishes proposals for executive offices.

**Rejected:** manual curation from campaign material (needs an admin panel,
lots of work, and every curated line is a bias risk); hiding the block (users
would not understand why it is missing).

---

## D3 — ALESC (state deputies)

**Decided:** ALESC has **no public API** (verified 2026-10-21). State deputy
profiles show the TSE data and, when the candidate is running for re-election
(elected state deputy in 2022, per the TSE candidacy history), a link to ALESC. Scraping ALESC is phase 2.

**Rejected:** scraping in the MVP (fragile, breaks when the site changes);
manual entry (same objections as D2).

---

## D4 — Profile data blocks

**Decided:** all four groups enter the MVP:

1. Votes + result (TSE)
2. Declared assets + campaign finance (TSE)
3. Activity in the Câmara: authored bills, roll-call votes, parliamentary quota (Câmara dos Deputados)
4. Parliamentary amendments whose money reached beneficiaries in Videira (Portal da Transparência / CGU — per-beneficiary bulk file; filtering by IBGE code alone finds only 3 amendments in 12 years)

**Deliberately not shown even though the TSE publishes them:** CPF, voter ID,
e-mail, criminal record certificates, marital status, race/colour. See D10.

---

## D5 — Languages and stack

**Decided:** **Go** for the backend (data pipeline), **TypeScript + React**
for the frontend, **TypeScript (AWS CDK)** for infrastructure. Full detail in
`04-tech-stack.md`.

Why Go for the backend: the group chose it; it streams the large TSE files
(the national votes zip is **454 MB**) with low memory, compiles to a single
binary, and needs almost no dependencies.

Why CDK in TypeScript and not Go: the same pattern (S3 + CloudFront + GitHub
OIDC) already runs in this AWS account for another project, in TypeScript,
and the frontend is TypeScript anyway. Copying a working stack is less risk
than translating it.

---

## D6 — Do we need a backend? (and why the runtime is static)

The question asked was: *can we build it with no backend at all?* Three
readings of "no backend" were evaluated.

### Option A — the browser calls the official sources directly → **not viable**

Verified on 2026-10-21:

| Problem | Evidence |
|---|---|
| **Secrets in the browser** | If any source needs a key (the Portal da Transparência REST API does — it returned `401` without one), the key ships inside the JavaScript bundle. Anyone copies it and abuses it under the group's name, and the key is tied to a person's gov.br account. |
| **CORS** | The Portal da Transparência API answered the CORS preflight with `403`. The Câmara API returned no `Access-Control-Allow-Origin` for a foreign origin. Browsers block both. |
| **Size** | The votes-by-municipality file is a **454 MB** zip. No visitor's phone can download and parse that to compute "top 20 in Videira". |
| **Joins** | Linking TSE candidate ↔ Câmara deputy ↔ amendment author would run on every visitor's device, on every visit. |
| **Availability** | If a government site is down or slow, our site is down or slow. |
| **Provenance** | "Fetched at" would mean "now", for every visitor, with no stable snapshot to point at. |

### Option B — a backend that runs as a **job**, and a static site → **chosen**

The backend exists and does the real work, but it is a **Go command-line
program run by GitHub Actions** (weekly + on demand), not a server:

1. downloads the official files and calls the Câmara API;
2. joins, filters (D1) and aggregates them;
3. writes a **SQLite database** — the project's data model;
4. exports small JSON files from it and publishes them to S3, versioned;
5. the React site, served by CloudFront from the same bucket, reads those JSON files.

What the backend is *for*, concretely:

- processing files far too big for a browser;
- joining three official sources by stable keys (CPF, deputy id, IBGE code);
- applying the selection rule once, identically for all;
- recording the provenance (source, generation date, fetch date) of every block;
- validating the result and refusing to publish broken data;
- keeping any credential (today: only the AWS deploy role) out of the browser.

### Option C — a permanent Go API + PostgreSQL (RDS) → rejected

Works, but:

| | Option B (chosen) | Option C |
|---|---|---|
| Monthly cost | ~US$0.01–0.10 | ~US$15 (RDS) + ~US$32 (NAT Gateway, see below) |
| Attack surface | none at runtime: no server, no public database, no SQL | API endpoint, database, network rules |
| Things to secure | an S3 bucket behind CloudFront | VPC, security groups, Secrets Manager, rate limiting, CORS, SQL injection, DB patching |
| Fails when a source is down | no — last good snapshot stays live | depends on design |
| Benefit for this MVP | — | none: there is no login, no user-written data, no real-time data |

The NAT trap: a private RDS lives in a VPC; the job that downloads data from
the internet must then run inside the VPC too, and reaching the internet from
a private subnet needs a NAT Gateway (~US$32/month). Making the RDS public to
avoid that is a security regression.

**When to revisit:** if a future version needs user accounts, user-submitted
content, full-text search over thousands of documents, or data fresher than
weekly. None of that is planned.

---

## D7 — Where it runs

**Decided:** AWS, in an account already used by another of the group's
projects (account `436812234154`, region `us-east-1`, CLI profile name
`personal`). CI/CD on **GitHub Actions** with **OIDC** (no stored AWS keys).
Details in `01-foundation/01-prerequisites.md` and `05-infrastructure/`.

> ⚠️ Machines often hold more than one AWS profile. Every AWS command in this
> project passes `--profile personal` explicitly, the CDK stack pins the
> account id, and every workflow checks the account before writing — so a
> wrong profile fails instead of deploying somewhere else.

---

## D8 — Design system

**Decided:** **shadcn/ui** (Radix primitives + Tailwind CSS 4) is the design
system, and the only component source. Every component comes from the
shadcn CLI and lives in `frontend/src/components/ui/`.

**Rejected:** Charcoal UI (charcoalui.com), evaluated 2026-10-21 and not
approved:

| | Finding |
|---|---|
| What it is | 165 React components (102 free, MIT), built with Motion + Tailwind CSS 4, installed through the shadcn CLI. |
| Why not | A catalogue of **animated effects** (text reveal, shimmer, typewriter, animated backgrounds, cursor effects), not a design system: no documented colour, typography or spacing tokens. |
| Fit with this product | A public-interest transparency panel tied to a legislative body must look sober and neutral. Flashy motion reads as marketing, can trigger motion sensitivity, and the `motion` runtime adds weight on low-end phones. |

Also rejected: gov.br Design System (DSGov) — the federal government's
visual identity would make a student project look like an official federal
service.

---

## D9 — Secrets

**Decided:** the project has **no data-source secrets**. The Portal da
Transparência amendments come from its **bulk download**
(`EmendasParlamentares.zip`, 32 MB, no key required), not from the REST API
that requires a key. The only credential is the AWS deploy role, obtained by
GitHub OIDC at run time. GitHub repository **variables** (not secrets) hold
the role ARN, bucket and distribution id — none of them is sensitive.

---

## D10 — Personal data (LGPD) and minimisation

The TSE files contain CPF, voter ID, e-mail. Rules:

- **CPF** is read only in memory to join TSE ↔ Câmara. It is **never** written
  to SQLite, JSON or logs.
- Voter ID and e-mail are never read into the model.
- Campaign donors and suppliers are often private individuals: the profile
  shows **only aggregates** (by origin / by expense type), never donor or
  supplier names or documents. The full detail stays one click away on the
  TSE's own site.
- Criminal record certificates (PDFs) are not shown: they are documents, not
  data, and rendering them out of context is the opposite of neutral.

---

## D11 — Scope of the docs

**Decided:** these documents cover the production code **and its tests**
(D15). They are written in English, organised as a tree, and
**versioned publicly in the repository** under `plan/`.

Because they are public, they never contain personal data, e-mail
addresses, credentials, or anyone's local machine setup. People are referred
to by role (repository owner, AWS account owner, backend lead, frontend lead).
Identifiers that must appear in the code anyway — the AWS account id, the
GitHub owner/repository ids — are allowed.

(Changed 2026-10-21: the first version kept `plan/` git-ignored and excluded tests.)

---

## D12 — Repository and GitHub permissions

The repository is `Miguel060/projetoAplicado2ifc` (public; owner id
`77062840`, repository id `1366189075`). Not every member is an admin.
Therefore:

- The **repository owner/admin** creates the repository variables, the
  branch protection rule and enables Actions (or grants admin to another
  member).
- The **AWS account owner** deploys the AWS stack and hands the stack
  outputs (bucket, distribution id, role ARN, URL) to the admin.

---

## D13 — Deploy design

**Decided:** the same design as the group's other project already running in
this AWS account, which has proven reliable:

| Workflow | Trigger | Does |
|---|---|---|
| `ci.yml` | every pull request and push; also callable (`workflow_call`) | vet/build Go, typecheck/lint/build web, synth infra |
| `deploy.yml` | **every push to `main`** (no path filter) + manual button | runs `ci.yml` as its first job; if green, publishes the site to S3/CloudFront and smoke-checks it |
| `refresh-data.yml` | weekly cron + manual button + push to `main` touching `backend/**` | runs the Go ETL and publishes a new data snapshot |
| — infra | manual, by the AWS account owner | `cdk deploy --profile personal`; changes rarely, and keeps CI's role minimal |

Why: merging to `main` *is* the release — no release branches, tags or
`workflow_run` chains (those fire only after another workflow and fail
silently in edge cases). Deploys never run in parallel and are never
cancelled midway (a half-finished sync would serve a mix of two builds).

---

## D14 — Landing page and how the site presents itself

**Decided (2026-10-21):**

| | |
|---|---|
| Routes | `/` landing · `/candidatos` search · `/candidatos/:slug` profile · `/sobre` methodology |
| Landing content | what the panel is for, what you find in it, how it works, who appears (selection rule), impartiality commitment, official sources, call to action "Ver candidatos" |
| Live figures | number of candidates, number of official sources, last update — read from `latest.json` / `index.json`; if no snapshot exists yet (or it fails to load), the page still renders fully and shows "Dados em preparação" |
| Release order | the landing is the **first page deployed** (right after infra + CI exist), before the ETL has ever run — it proves the whole delivery pipeline early |
| Partner | the site does **not** mention the Câmara Municipal de Videira anywhere (no name, no logo). The partnership appears only in the course deliverables. The site states it is an academic project of IFC students and not an official site of any public body. |

**Rejected:** a hero section on top of the search at `/` (the explanation
competes with the tool, and that page cannot ship before the data does).

---

## D15 — Tests gate every deploy

**Decided (2026-10-21), replacing "production code only":** backend,
frontend and infrastructure all carry unit and integration tests, written in
the same PR as the code they cover, and run before any deploy. Full contract:
`01-foundation/05-testing-strategy.md`.

| | Decision |
|---|---|
| Backend | Go `testing`; unit tests per package; one **integration test of the whole pipeline** against an in-process fake upstream (no network); its output is a committed **golden snapshot** |
| Frontend | Vitest + Testing Library (unit/component); **Playwright** end-to-end on the production build against the golden snapshot, with **axe** accessibility checks |
| Browsers (e2e) | **Chromium** (desktop) + **WebKit** (iPhone 13) |
| Infrastructure | CDK assertion tests pinning the security invariants |
| Coverage gate | **85 %** on backend (`./internal/...`) and frontend (`src/`, generated UI excluded); infra coverage reported only |
| Per-step browser check | every frontend step is exercised interactively with **Claude in Chrome** before its PR (Playwright when scripting is needed) |
| Where they run | `ci.yml` (which `deploy.yml` reuses as its `verify` job) and `refresh-data.yml` (Go tests before the ETL) |
| Test data | synthetic only — real column layouts and encodings, invented values, fake CPFs (public repo, D10) |

Why the golden snapshot matters: the frontend's end-to-end tests run against
the backend's actual output, so a JSON contract break fails CI in the PR that
causes it.

Cost: more time per step. The 85 % gate was chosen deliberately over a
lower one; if the 2026-11-19 deadline is at risk, lower it in this entry —
never by skipping tests silently.

---

## D16 — The ingestion runs on GitHub Actions, not AWS Lambda

**Decided (2026-10-21).** Cost is *not* the reason — a Lambda would also be
nearly free, and a Lambda outside a VPC reaches the internet without a NAT
Gateway. The reasons:

| | GitHub Actions | AWS Lambda |
|---|---|---|
| Same pipeline as the code | the ETL runs after `go test`, from the same checkout; a push touching `backend/` re-runs it with the new code | build and deploy the function, then invoke it — a second release path to keep in sync |
| Who can see and run it | all members see logs and history, get failure e-mails, and have a "Run workflow" button — no AWS access needed | only people with AWS access see CloudWatch logs or trigger runs |
| Infrastructure to build and secure | none beyond the existing deploy role (short-lived OIDC credentials scoped to the bucket) | function, role, EventBridge schedule, log group, failure alerting (SNS), Go packaging for `provided.al2023` |
| Limits | 6 h per job; 4 vCPU / 16 GB RAM / ~14 GB disk; free for a public repository | 15 min per run; 512 MB `/tmp` by default — workable, but Câmara API retries on 504s can push the run time up |
| Local parity | `make ingest` is the same command | the deployed function differs from a local run |

Costs accepted: scheduled workflows pause after 60 days without repository
activity (documented in `06-ci-cd/03-refresh-data.md`), cron start times
are approximate, and a GitHub outage skips that refresh (the site keeps
serving the last snapshot).

**Revisit if:** the data must refresh hourly, or independently of GitHub.
