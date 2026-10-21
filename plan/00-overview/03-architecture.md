# 03 — Architecture

## The picture

```
                 ┌──────────────────────────── GitHub (Miguel060/projetoAplicado2ifc) ─────────────────────────┐
                 │                                                                                              │
                 │  PR / push ──► ci.yml (vet, build, typecheck, lint, cdk synth)                               │
                 │                                                                                              │
                 │  push to main                                                                                │
                 │        ├──► deploy.yml: verify (= ci.yml) ──► npm run build ──► s3 sync (site files)         │
                 │        │                                                                                     │
                 │  cron weekly / manual / backend change                                                       │
                 │        └──► refresh-data.yml ──► go run ./cmd/ingest ──► s3 sync (data/<version>/)           │
                 │                                     │                     + data/latest.json (last)         │
                 └─────────────────────────────────────┼──────────────────────────────┬───────────────────────┘
                                                       │ HTTPS downloads               │ OIDC → IAM role
          ┌────────────────────────────────────────────┼───────────────┐               │ (no stored keys)
          │ Official sources                           ▼               │               ▼
          │  TSE  cdn.tse.jus.br   (zips: candidates, votes, assets,   │   ┌──────── AWS 436812234154 / us-east-1 ───────┐
          │                         finance, social, photos)           │   │                                              │
          │  Câmara dadosabertos.camara.leg.br (API + bulk CSV)        │   │  S3 bucket (private, OAC only)                │
          │  Câmara www.camara.leg.br/cotas   (CEAP bulk)              │   │   ├── index.html, assets/*   ← site          │
          │  CGU   dadosabertos-download.cgu.gov.br (amendments bulk)  │   │   └── data/                  ← data          │
          └────────────────────────────────────────────────────────────┘   │        ├── latest.json                       │
                                                                           │        └── <version>/index.json, candidates/,│
                                                                           │             photos/, dataset.sqlite          │
                                                                           │              ▲                               │
                                                                           │  CloudFront  │ (HTTPS, security headers,     │
                                                                           │   + Function │  SPA route rewrite)           │
                                                                           └──────────────┼───────────────────────────────┘
                                                                                          │
                                                                        Citizen's browser: React SPA
                                                                        fetches /data/latest.json → /data/<version>/…
```

## Components

| # | Component | Language | Runs where | Does what |
|---|---|---|---|---|
| 1 | **Ingest (ETL)** — `backend/cmd/ingest` | Go | GitHub Actions runner (and locally) | Downloads, parses, joins, selects, aggregates, writes SQLite, exports JSON. |
| 2 | **Dataset** — `dataset.sqlite` | SQL (SQLite) | Produced by 1, published to S3 | The data model. Also offered as an open-data download on the About page. |
| 3 | **Data snapshot** — `data/<version>/…` | JSON | S3, served by CloudFront | What the site reads. Immutable once published. |
| 4 | **Pointer** — `data/latest.json` | JSON | S3 | Names the current `<version>`. Uploaded last; switching it is the atomic "go live". |
| 5 | **Web app** — `frontend/` | TypeScript, React | Visitor's browser | Search, filters, profiles, About page. Read-only. |
| 6 | **Infrastructure** — `infra/` | TypeScript, AWS CDK | Deployed manually by the AWS account owner (`make infra-deploy`) | S3, CloudFront, CloudFront Function, response-headers policy, GitHub OIDC deploy role. |
| 7 | **CI/CD** — `.github/workflows/` | YAML | GitHub Actions | Checks every change; deploys the site; refreshes the data. |

## Data flow, step by step

1. **Trigger.** `refresh-data.yml` starts (weekly cron, manual button, or a
   push that touches `backend/`).
2. **Download.** The Go program fetches each source. For the two very large
   TSE zips it reads **only the `_SC.csv` member** through HTTP range requests
   instead of downloading the whole national file (454 MB → a few tens of MB).
3. **Parse.** TSE and CGU files are Latin-1 with `;` separators and Brazilian
   decimals (`1.234,56`); Câmara files are UTF-8 with a BOM. Each parser
   converts to typed Go structs and maps TSE sentinels (`#NULO#`, `#NE#`,
   `-1`, `-3`, `-4`) to null.
4. **Select.** Apply D1 (top 20 per office in Videira + born in Videira).
5. **Enrich.** For the selected candidates only: match to Câmara deputies by
   CPF, fetch authored bills, roll-call votes and CEAP; match amendments to
   deputies by normalised author name (+ a small override file).
6. **Store.** Write everything to a fresh `dataset.sqlite` (CPF is not stored).
7. **Export.** Produce `index.json`, one `candidates/<slug>.json` per
   candidate, and copy the selected photos.
8. **Validate.** Sanity checks (counts, required fields, sources present, no
   big drop compared to the live version). Any failure aborts the run; the
   previous snapshot stays live.
9. **Publish.** Upload `data/<version>/` with immutable caching, then
   `data/latest.json`, then invalidate that one path, then prune old versions
   (keep the last 5).
10. **Read.** The browser fetches `latest.json` (short cache), then the
    versioned files (cached for a year — they never change).

## Why versioned snapshots + a pointer

- **Atomic switch.** A visitor never sees half of an old snapshot and half of
  a new one: until `latest.json` changes, everyone reads the old version in
  full.
- **Caching is trivial.** Versioned files are immutable, so they get
  `max-age=31536000, immutable`. Only `latest.json` is short-lived.
- **Rollback is one upload.** Point `latest.json` back at the previous version.
- **Provenance.** Every snapshot is a dated, reproducible artefact.

Old versions are pruned by the publish step (keep 5), **not** by an S3
lifecycle rule: if the weekly cron ever stops, a lifecycle rule would
eventually delete the version that is still live.

## Why SPA routes need a CloudFront Function (and not error pages)

The common trick — "map 403/404 to `/index.html`" — applies to the whole
distribution. A missing `/data/<version>/candidates/x.json` would then return
**200 with HTML**, and the app would fail with a confusing JSON parse error.
Instead, a tiny **viewer-request CloudFront Function** rewrites only paths
**without a file extension** (`/`, `/candidatos`, `/candidatos/fulano-1234`, `/sobre`) to
`/index.html`. Real files, including data files, keep real 404s.

## Security model

| Threat | Mitigation |
|---|---|
| Server compromise | There is no server. |
| Database breach / SQL injection | There is no database at runtime; SQLite is a static, read-only download of public data. |
| Bucket exposure | Bucket is private with Block Public Access; only CloudFront reads it, via Origin Access Control. |
| Leaked AWS credentials | None exist in the repo or in GitHub secrets. GitHub OIDC mints short-lived credentials for a role that trusts **only** `main` of this repository. |
| Over-privileged CI | The deploy role can only list/put/delete objects in this bucket and invalidate this distribution. Infrastructure changes are deployed by a human with `--profile personal`. |
| Deploying into the wrong AWS account | `--profile personal` on every command; the CDK stack pins account `436812234154`; every workflow aborts if `sts get-caller-identity` returns another account. |
| XSS / clickjacking / MIME sniffing | Response-headers policy: CSP (`script-src 'self'`), `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, HSTS, strict referrer policy. React escapes text by default; the app never uses `dangerouslySetInnerHTML`. |
| Poisoned / broken upstream data | ETL validation refuses to publish; the last good snapshot stays live. |
| Personal-data leakage | CPF never persisted; donors/suppliers only aggregated (D10). |
| Supply chain | Exact version pins (no `^`), lockfiles committed, minimal dependencies, Dependabot optional. |
| Traffic spike / DDoS | CloudFront absorbs it; static files cost almost nothing to serve; no compute to exhaust. |

## What would change for phase 2

- **Senator / governor:** add office codes 5 and 3 to the config; TSE
  `proposta_governo_2026_SC.zip` feeds a real "proposals" block for governors.
- **ALESC scraping:** a new `internal/alesc` package and a new source id; the
  JSON contract gains an optional block. Nothing else moves.
- **Custom domain:** ACM certificate in `us-east-1` + alias on the distribution
  + DNS record. Additive; nothing is rebuilt.
