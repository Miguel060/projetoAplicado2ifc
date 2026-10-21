# Implementation plan — Projeto Aplicado II (IFC)

Non-partisan panel of the 2026 candidates for federal and state deputy linked
to **Videira/SC**, built from official open data (TSE, Câmara dos Deputados,
CGU – Portal da Transparência), with the source of every value.

> This folder is **versioned and public** in the repository. It must never
> contain personal data, e-mail addresses, credentials or anyone's local
> machine setup. Scope: production code **and its tests** — every step ships
> its own tests, and nothing deploys unless they pass (D15).

## At a glance

| Question | Answer | Where |
|---|---|---|
| What are we building? | A static, read-only React site + a Go data pipeline | `00-overview/01-project.md` |
| Which languages? | **Go** (backend/ETL), **TypeScript + React** (frontend), **TypeScript / AWS CDK** (infra), **YAML** (CI/CD), **SQL / SQLite** (data model) | `00-overview/04-tech-stack.md` |
| Do we need a backend? What for? | **Yes — as a scheduled job, not a server.** It downloads files too big for a browser (454 MB), joins three sources, applies the selection rule, records provenance, validates, and publishes JSON snapshots. Nothing runs between refreshes. | `00-overview/02-decisions.md` (D6), `00-overview/03-architecture.md` |
| Could it be done with no backend at all? | No: secrets in the bundle, CORS blocks, 454 MB downloads, cross-source joins on every visit. | D6, option A |
| Where does it run? | AWS (personal account `436812234154`, `us-east-1`): S3 + CloudFront. Pipeline on GitHub Actions with OIDC. | `05-infrastructure/`, `06-ci-cd/` |
| How much does it cost? | ≈ US$0.01–0.10 / month | `00-overview/05-aws-cost.md` |
| Design system? | **shadcn/ui** (Radix + Tailwind CSS 4), nothing else. | D8 |
| How is it tested? | Go unit + whole-pipeline integration tests; Vitest + Testing Library; Playwright e2e (Chromium + WebKit) with axe; CDK assertion tests. **85 % coverage gate** on backend and frontend. Every frontend step is also checked interactively with Claude in Chrome. All of it runs before every deploy. | D15, `01-foundation/05-testing-strategy.md` |
| Why GitHub Actions and not Lambda for the ingestion? | Same pipeline as the code (tests then ETL), visible to every member without AWS access, no extra infra to secure, no 15-minute limit. | D16 |
| How is it deployed? | Same design as the group's proven setup: **every push to `main`** runs CI and, if green, publishes the site. Data refreshes weekly, on demand, and when `backend/` changes. Infra is deployed manually. | D13, `06-ci-cd/` |
| In what order do we build it? | **`STEPS.md`** — 25 numbered steps, one PR each: "do step 1", "do step 2"… (calendar/ownership view: `07-delivery/01-implementation-order.md`) | `STEPS.md` |

## Tree

```
plan/
├── README.md                                   ← you are here
├── STEPS.md                                    ★ the execution order: "do step N"
├── 00-overview/                                WHAT and WHY
│   ├── 01-project.md                           scope, MVP goals, selection rule, out of scope, schedule
│   ├── 02-decisions.md                         decision log D1–D16 (backend or not, design system, LGPD, deploy, landing, tests, Actions vs Lambda)
│   ├── 03-architecture.md                      diagram, components, data flow, security model
│   ├── 04-tech-stack.md                        languages, libraries, pinned versions, rejected options
│   └── 05-aws-cost.md                          monthly estimate and guard rails
├── 01-foundation/                              CONTRACTS everything else depends on
│   ├── 01-prerequisites.md                     tools, GitHub roles, AWS profile safety
│   ├── 02-repository-layout.md                 monorepo tree, .gitignore, Makefile, conventions
│   ├── 03-data-model.md                        SQLite schema, source ids, result codes, slug rule
│   ├── 04-json-contract.md                     backend → frontend JSON files and TypeScript types
│   └── 05-testing-strategy.md                  test gates, coverage, synthetic fixtures, golden snapshot, browser checks
├── 02-data-sources/                            WHERE the data comes from
│   ├── 01-tse.md                               candidates, votes, assets, finance, social, photos
│   ├── 02-camara.md                            deputies, bills, roll-call votes, CEAP
│   ├── 03-portal-transparencia.md              parliamentary amendments (bulk, no key)
│   └── 04-alesc.md                             why link-only in the MVP
├── 03-backend-etl/                             HOW the Go pipeline is built (step by step)
│   ├── 01-go-project-setup.md                  module, config, CLI flags, orchestration
│   ├── 02-fetch-and-parse.md                   HTTP client, cache, remote zip, Latin-1 CSV, BR numbers
│   ├── 03-load-tse.md                          TSE loaders
│   ├── 04-select-candidates.md                 top-20 + born-in-Videira rule
│   ├── 05-load-camara.md                       Câmara loaders and CPF matching
│   ├── 06-load-amendments.md                   CGU amendments and author matching
│   ├── 07-export-json.md                       JSON export, photos, dataset
│   ├── 08-validate-and-publish.md              validation gates and S3 publish
│   └── 09-integration-tests.md                 fake upstream, synthetic fixtures, golden snapshot
├── 04-frontend/                                HOW the React app is built (step by step)
│   ├── 01-project-setup.md                     Vite, Tailwind, shadcn/ui, tokens, dev data server
│   ├── 02-data-access.md                       types, fetchers, loaders, formatters
│   ├── 03-landing-page.md                      "/" — what it is for, how it works, live figures
│   ├── 04-search-page.md                       "/candidatos" — search, filters, sorting, cards
│   ├── 05-profile-page.md                      "/candidatos/:slug" — profile sections and source notes
│   ├── 06-about-page.md                        "/sobre" — methodology, sources, dataset download
│   ├── 07-accessibility-responsiveness-neutrality.md
│   └── 08-testing.md                           Vitest, Playwright + axe, Claude in Chrome procedure
├── 05-infrastructure/                          AWS
│   ├── 01-cdk-stack.md                         S3, CloudFront, Function, headers, deploy role
│   └── 02-github-oidc-and-variables.md         OIDC trust, repository variables, branch protection
├── 06-ci-cd/                                   GitHub Actions
│   ├── 01-ci.md                                checks on every change
│   ├── 02-deploy-site.md                       deploy.yml: every push to main publishes the site
│   └── 03-refresh-data.md                      run the ETL and publish a snapshot
└── 07-delivery/                                WHEN and WHO
    ├── 01-implementation-order.md              master checklist mapped to the Modelo 4 dates
    └── 02-launch-checklist.md                  manual QA, security sign-off, rollback
```

## How to read it

1. `00-overview/` end to end (20 minutes). Everyone.
2. `01-foundation/` — the contracts. Everyone, before writing code.
3. Then only your area: `02` + `03` (backend), `04` (frontend), `05` + `06` (infra/CI).
4. Build with `STEPS.md`, one step (one PR) at a time. Track dates and owners in `07-delivery/01-implementation-order.md`.

## Ground rules

- Every value on screen comes from an official source and shows that source.
- Nothing is edited by hand. If data is wrong, it is wrong at the source — link to it.
- Every AWS command passes the project's profile explicitly (`--profile personal`). Never rely on a default profile.
- This folder is public: no personal data, e-mail addresses, credentials or machine-specific setup in it. People are referred to by role.
- If a decision changes, update `00-overview/02-decisions.md` first.
