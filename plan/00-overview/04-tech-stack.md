# 04 — Tech stack

Tie-break rule for any close call: **what will a student understand when
maintaining this alone in six months?**

Version numbers below are the ones already proven to work together in the
group's other project deployed in the same AWS account (checked
2026-10-21). Use them, or the newest **patch** of the same line at setup time.
**Pin exact versions** — no `^` or `~` — and update deliberately in their own
commits.

## Languages per layer

| Layer | Language | Version | Why |
|---|---|---|---|
| Backend (ETL) | **Go** | 1.25 (`go 1.25` in `go.mod`; local toolchain is 1.25.1) | Chosen by the group. Streams huge CSVs with flat memory, single static binary, stdlib covers HTTP, zip, CSV, JSON. |
| Data model | **SQL (SQLite dialect)** | SQLite 3 via `modernc.org/sqlite` | A real relational model and an open-data download, with zero servers. |
| Frontend | **TypeScript** (strict) + **React** | TS 7.0.2, React 19.3.0 | Biggest ecosystem; shadcn/ui requires React. |
| Infrastructure | **TypeScript** (AWS CDK v2) | aws-cdk-lib 2.269.0, aws-cdk CLI 2.1141.0 | Same proven S3 + CloudFront + OIDC pattern already running in this account. |
| CI/CD | **YAML** (GitHub Actions) | — | Free for public repositories; OIDC into AWS. |
| Runtime for tooling | **Node.js** | 24 LTS (`.nvmrc` = `24`) | npm 11; runs CDK's TypeScript directly (`--experimental-strip-types`). |

**Identifiers, comments, commit messages and these docs are in English.**
Only user-visible UI text is in Portuguese (pt-BR).

## Backend (Go)

| Concern | Choice | Why |
|---|---|---|
| CLI | stdlib `flag` | One command, a handful of flags. No framework. |
| HTTP | stdlib `net/http` with a shared client (timeouts, retries, `User-Agent`) | Enough; retries are a 20-line helper. |
| Zip | stdlib `archive/zip` over an HTTP **range-request `io.ReaderAt`** | Reads only the `_SC.csv` member from 454 MB / 206 MB national zips. Falls back to a full download if the server ignores ranges. |
| CSV | stdlib `encoding/csv` (`Comma = ';'`, `LazyQuotes = true`) | Handles TSE/CGU/Câmara quirks. |
| Latin-1 | `golang.org/x/text/encoding/charmap` (`ISO8859_1`, or `Windows1252` for CGU) | TSE and CGU files are not UTF-8. |
| SQLite | `modernc.org/sqlite` | Pure Go, **no CGO** — builds identically on macOS and the Linux runner. |
| JSON | stdlib `encoding/json` | — |
| Logging | stdlib `log/slog` (text locally, JSON in CI) | Structured, zero deps. |
| Lint/format | `gofmt`, `go vet` (CI); `staticcheck` optional | Standard. |

Allowed third-party modules: `modernc.org/sqlite`, `golang.org/x/text`. Anything
else needs a reason written in this file.

## Frontend

| Concern | Choice | Version | Why |
|---|---|---|---|
| Build/dev server | **Vite** | 8.2.2 | Fast, minimal config. |
| React plugin | `@vitejs/plugin-react` | 6.1.1 | — |
| Routing | **React Router** (data mode: `createBrowserRouter` + loaders), **path-based** | 8.3.1 | Shareable URLs like `/candidatos/fulano-1234`. Loaders fetch data before render, so pages never flash empty. |
| Design system | **shadcn/ui** (copy-in components on Radix primitives) | CLI `shadcn` 4.21.0 (dev) | Accessible by default; we own the component files. The only component source (D8). |
| Primitives | `radix-ui` | 1.6.7 | Pulled by shadcn components. |
| Styling | **Tailwind CSS 4** + `@tailwindcss/vite` | 4.3.3 | Required by shadcn/ui. |
| Class helpers | `cn`, `class-variance-authority` | 0.2.6, 0.7.1 | Pulled by shadcn components. |
| Animation utils | `tw-animate-css` | 1.4.0 | Used by shadcn components; respects reduced motion via Tailwind's `motion-safe:`. |
| Icons | `lucide-react` | 1.43.0 | Tree-shaken SVG icons. |
| Font | `@fontsource-variable/inter` (self-hosted) | 5.3.0 | Self-hosted: no font CDN request leaking visitor IPs. Inter has excellent legibility for numbers and accents. |
| State | **none** (URL search params + loader data) | — | Filters live in the URL, data comes from loaders. No store library needed. |
| Charts | **none** — HTML tables with CSS bars | — | Tables are accessible to screen readers by nature; a chart lib would be the heaviest dependency for the least value. |
| Lint + format | **Biome** | 2.5.12 | One tool instead of ESLint + Prettier. |
| Type check | `tsc --noEmit` | 7.0.2 | Vite does not type-check during build. |

## Infrastructure

| Concern | Choice |
|---|---|
| IaC | AWS CDK v2, TypeScript, one stack `ProjetoAplicado2` |
| Hosting | S3 (private) + CloudFront (Origin Access Control) |
| SPA routing | CloudFront Function (viewer-request rewrite of extension-less paths) |
| Security headers | CloudFront Response Headers Policy |
| CI auth | GitHub OIDC → IAM role scoped to `main` of `Miguel060/projetoAplicado2ifc` (existing account-level OIDC provider is **imported**, not created) |
| Region | `us-east-1` (required for a future ACM certificate on CloudFront) |

## Deliberately rejected

| Rejected | Why |
|---|---|
| Next.js / SSR | Needs a server; static SPA + loaders is enough for ~45 profiles. |
| A permanent Go API, RDS, Lambda, DynamoDB | See D6: cost, attack surface, zero benefit for a read-only weekly dataset. |
| PostgreSQL for the ETL | A server for a batch job that runs 2 minutes a week. SQLite is a file. |
| CGO SQLite (`mattn/go-sqlite3`) | Needs a C toolchain in CI; `modernc.org/sqlite` does not. |
| Redux / Zustand / TanStack Query | No client state worth a library: filters are URL params, data is loader output. |
| Chart libraries (Recharts, Chart.js, D3) | Weight and accessibility cost; tables do the job. |
| Charcoal UI | Evaluated, not approved (D8): animated-effects catalogue, not a design system. |
| gov.br Design System (DSGov) | Using the federal government's visual identity would make a student project look like an official federal service — misleading. |
| Google Fonts / any CDN | Third-party requests leak visitor IPs; everything is self-hosted. |
| Docker | Nothing to containerise: a Go binary and static files. |
| Automated test suites | Out of scope by instruction (production code only). |

## Version policy

- Exact pins in `package.json`; commit `package-lock.json` and `go.sum`.
- Go toolchain from `go.mod` (`actions/setup-go` with `go-version-file`).
- Node from `.nvmrc` (`actions/setup-node` with `node-version-file`).
- Upgrades happen in their own commit, never mixed with features.
