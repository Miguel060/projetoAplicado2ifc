# 02 — Repository layout and conventions

## Target tree

```
projetoAplicado2ifc/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # checks on every PR/push; reusable (workflow_call)
│       ├── deploy.yml                # every push to main: verify (ci.yml) + publish the React app
│       └── refresh-data.yml          # run the ETL + publish a data snapshot
├── plan/                             # this implementation plan (public, versioned)
├── backend/                          # Go module: the data pipeline
│   ├── go.mod                        # module github.com/Miguel060/projetoAplicado2ifc/backend
│   ├── go.sum
│   ├── cmd/
│   │   └── ingest/
│   │       └── main.go               # thin CLI: flags → pipeline.Options
│   ├── config/
│   │   └── name-overrides.json       # amendment author name → Câmara deputy id (only for misses)
│   ├── internal/
│   │   ├── pipeline/                 # Run(ctx, Options): the whole ETL, called by main and by the integration test
│   │   ├── testsrv/                  # in-process fake upstream for tests (imported only by *_test.go)
│   │   ├── config/                   # constants (year, UF, municipality, topN, offices, URLs)
│   │   ├── fetch/                    # HTTP client, retries, local cache, remote zip reader
│   │   ├── csvx/                     # Latin-1 / BOM readers, BR number/date parsing, sentinels
│   │   ├── tse/                      # candidates, complementary, votes, assets, finance, social, photos
│   │   ├── selection/                # top-N + born-in-municipality rule
│   │   ├── camara/                   # deputies (API), bills (API), votes + CEAP (bulk)
│   │   ├── amendments/               # CGU bulk file, filter by IBGE code, author matching
│   │   ├── store/                    # SQLite schema (embedded schema.sql) and writers
│   │   ├── export/                   # JSON writers, slugs, photo copy
│   │   ├── validate/                 # pre-publish sanity checks
│   │   └── textnorm/                 # accent/case normalisation shared by matching and search text
│   │       (every package has its *_test.go next to the code)
│   ├── testdata/
│   │   ├── sources/                  # SYNTHETIC fixtures in the real layouts/encodings (byte-exact)
│   │   └── golden/data/              # expected pipeline output = the frontend e2e fixture
│   ├── scripts/
│   │   └── publish-data.sh           # manual mirror of the publish job
│   ├── .cache/                       # downloaded files (git-ignored)
│   └── dist/                         # ETL output (git-ignored)
│       └── data/
│           ├── latest.json
│           └── <version>/…
├── frontend/                         # React SPA
│   ├── package.json
│   ├── package-lock.json
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── components.json               # shadcn/ui config
│   ├── public/
│   │   ├── favicon.svg
│   │   └── robots.txt
│   └── src/
│       ├── main.tsx
│       ├── router.tsx
│       ├── index.css                 # Tailwind + design tokens
│       ├── data/                     # contract types, fetchers, loaders, formatters
│       ├── pages/                    # LandingPage (/), SearchPage (/candidatos), CandidatePage, AboutPage, NotFoundPage, ErrorPage
│       ├── components/               # app components (CandidateCard, SourceNote, …)
│       ├── components/ui/            # shadcn/ui generated components (ours to edit)
│       ├── lib/                      # utils (cn, text normalisation, url params)
│       └── **/*.test.ts(x)           # unit/component tests next to the code
│   ├── e2e/                          # Playwright specs + fixtures.ts (axe, console guard)
│   ├── vitest.config.ts
│   └── playwright.config.ts
├── infra/                            # AWS CDK (TypeScript)
│   ├── package.json
│   ├── package-lock.json
│   ├── cdk.json
│   ├── tsconfig.json
│   ├── bin/app.ts
│   ├── lib/
│   │   ├── config.ts                 # account, region, repository ids — imported by app.ts and the tests
│   │   ├── site-stack.ts
│   │   └── spa-rewrite.js            # CloudFront Function source
│   └── test/                         # CDK assertion tests + function rewrite tests
├── testdata/
│   └── textnorm-vectors.json         # shared by the Go and TS text-normalisation tests
├── .editorconfig
├── .gitattributes                    # backend/testdata/** -text (fixtures are byte-exact)
├── .gitignore
├── .nvmrc                            # 24
├── biome.json                        # lint/format for frontend + infra
├── Makefile                          # the commands everyone uses
└── README.md                         # how to run (public, short)
```

Three independent toolchains (`backend/`, `frontend/`, `infra/`) instead of
npm workspaces: each has its own lockfile, so a frontend dependency bump can
never break the infra synth and vice versa. `biome.json` lives at the root
and covers both TypeScript folders.

## `.gitignore` (final content)

```gitignore
# Go
/backend/.cache/
/backend/dist/

# Node
node_modules/
/frontend/dist/
/infra/cdk.out/

# Test output
/backend/coverage.out
/frontend/coverage/
/frontend/playwright-report/
/frontend/test-results/
/infra/coverage/

# OS / editors
.DS_Store
.vscode/*
!.vscode/extensions.json
.idea/

# Env
.env
.env.*
```

## `.gitattributes`

```gitattributes
# Fixtures reproduce the real files byte for byte (Latin-1, CRLF, BOM).
backend/testdata/** -text
```

## `Makefile` (the shared vocabulary)

```makefile
AWS := aws --profile personal
ACCOUNT := 436812234154

.PHONY: ingest dev build-site test test-backend test-frontend test-e2e test-infra check preflight publish-data-local infra-diff infra-deploy

ingest:            ## Run the ETL locally (uses backend/.cache)
	cd backend && go run ./cmd/ingest -out dist/data -cache .cache

dev:               ## Frontend dev server; serves backend/dist/data at /data
	cd frontend && npm run dev

build-site:
	cd frontend && npm run build

test-backend:      ## Go tests, race detector, 85% coverage gate
	cd backend && go test -race -covermode=atomic -coverpkg=./internal/... -coverprofile=coverage.out ./... \
	  && go tool cover -func=coverage.out | awk '/^total:/ { sub("%","",$$3); if ($$3+0 < 85) { print "coverage " $$3 "% < 85%"; exit 1 } else print "coverage " $$3 "%" }'

test-frontend:     ## Vitest with coverage thresholds (85%)
	cd frontend && npm run test:coverage

test-e2e:          ## Production build + Playwright (Chromium + WebKit) against the golden snapshot
	cd frontend && npm run test:e2e

test-infra:        ## CDK assertion tests
	cd infra && npm test

test: test-backend test-frontend test-e2e test-infra

check:             ## What CI runs, locally
	cd backend && test -z "$$(gofmt -l .)" && go vet ./... && go build ./...
	cd frontend && npm run typecheck && npm run lint && npm run build
	cd infra && npm run typecheck && npx cdk synth -q
	$(MAKE) test

preflight:         ## Refuse to continue unless the personal account is selected
	@test "$$($(AWS) sts get-caller-identity --query Account --output text)" = "$(ACCOUNT)" \
	  || (echo "Wrong AWS account. Stop." && exit 1)

infra-diff: preflight
	cd infra && npx cdk diff --profile personal

infra-deploy: preflight
	cd infra && npx cdk deploy --profile personal
```

(`publish-data-local` is defined in `03-backend-etl/08-validate-and-publish.md`.)

## Conventions

| Topic | Rule |
|---|---|
| Language | Code, comments, commits, docs in English. UI text in pt-BR. |
| Commits | Conventional Commits: `feat(etl): …`, `fix(web): …`, `chore(infra): …`, `ci: …`. Scopes: `etl`, `web`, `infra`, `ci`, `repo`. |
| Branches | `feat/<short-name>`, `fix/<short-name>`; PR into `main`; CI must pass. |
| Go style | `gofmt`; packages are nouns; errors wrapped with `%w` and context (`fmt.Errorf("tse votes: %w", err)`); no panics outside `main`. |
| TS style | Biome defaults (2 spaces, 100 cols); `import type` for types; no `any`; no non-null `!`. |
| Files | Go: `snake_case.go`. React components: `PascalCase.tsx`. Other TS: `kebab-case.ts`. |
| Money | Stored and exported as **numbers in BRL** (float64 / JS number) rounded to 2 decimals; formatted only in the UI with `Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' })`. |
| Dates | Exported as ISO 8601 (`2026-10-04`, `2026-11-09T03:00:00Z`); formatted only in the UI. |
| Text normalisation | One function per side, identical behaviour: lowercase → NFD → strip combining marks → collapse whitespace. Go: `internal/textnorm`; TS: `src/lib/text.ts`. |

## First commit sequence (2026-11-09)

1. `chore(repo): add gitignore, editorconfig, nvmrc, biome, makefile`
2. `feat(etl): scaffold go module and ingest command`
3. `feat(web): scaffold vite react app with tailwind and shadcn`
4. `feat(infra): add cdk stack for bucket, distribution and deploy role`
5. `ci: add ci workflow`
