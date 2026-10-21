# 07.02 — Launch checklist (manual QA) and rollback

**When:** milestone 5, 2026-11-19 — and again before any later public
announcement. **Who:** everyone; each section has a suggested role, and the
**neutrality review must be done by people who did not write the page**.

There are no automated test suites in this project (production code only).
This checklist is the quality gate: compilers, linters and the ETL's
validation catch the mechanical errors; this catches the rest.

Set these once in the terminal used for the checks:

```bash
SITE_URL=https://dxxxxxxxxxxxx.cloudfront.net    # repository variable SITE_URL
BUCKET=<SITE_BUCKET>
DIST=<SITE_DISTRIBUTION_ID>
V=$(curl -s "$SITE_URL/data/latest.json" | jq -r .version); echo "$V"
```

AWS commands below are read-only, and every one passes the profile
explicitly — always pass `--profile <your-profile>` (the project uses a
profile named `personal`).

---

## 1. Functional — [frontend lead + one other member]

### Landing page (`/`)

- [ ] Loads fast on a phone and explains, in neutral words, what the panel is and where the data comes from.
- [ ] The main call to action leads to `/candidatos`; the About link leads to `/sobre`.
- [ ] With no snapshot published (or `latest.json` blocked in DevTools) it shows the "Dados em preparação" fallback instead of failing.
- [ ] With data, any figure it shows (e.g. last update date) matches `index.json`.

### Search page (`/candidatos`)

- [ ] The page lists as many candidates as `index.json` has
      (`curl -s "$SITE_URL/data/$V/index.json" | jq '.counts'`).
- [ ] Search ignores accents and case: `joao` finds `JOÃO`, `sao` finds `SÃO`.
- [ ] Search by ballot number finds the candidate.
- [ ] Filters — office, result, party — work alone and combined; "clear filters" resets all.
- [ ] Filters and search are in the URL (`/candidatos?…`): copy the URL, open it in a private
      window → same results. Reload keeps them. The back button undoes the last change.
- [ ] Every sort option orders as labelled; the default is "most voted in Videira".
- [ ] The result count updates and a screen reader announces it.
- [ ] No result → neutral empty-state message, no blank area.

### Profile page (`/candidatos/<slug>`)

- [ ] Open at least one profile of each kind:
  - [ ] federal deputy **with** Câmara record (bills, votes, CEAP, amendments)
  - [ ] federal deputy **without** Câmara record (neutral "no official record" texts)
  - [ ] state deputy running for re-election (ALESC link present)
  - [ ] candidate selected because born in Videira
- [ ] Sections appear in the fixed order of `00-overview/01-project.md` for every candidate.
- [ ] Every block has its source note: dataset name, generation date, fetch date, link — and the link opens the official page.
- [ ] Money is formatted `R$ 1.234,56`, dates `dd/mm/aaaa`, percentages `10,4%`.
- [ ] Photo has a meaningful `alt`; a candidate without photo shows the same neutral placeholder as everyone else.
- [ ] `/candidatos/nao-existe` shows the not-found page (HTTP 200 from CloudFront, "not found" content from the app).

### About page (`/sobre`)

- [ ] The selection rule shows **top 20** (matches `index.json` → `selection.topN`).
- [ ] All source ids in `index.json.sources` are listed with their dates.
- [ ] The dataset download works and the file opens:

  ```bash
  curl -sO "$SITE_URL/data/$V/dataset.sqlite"
  sqlite3 dataset.sqlite 'SELECT office, COUNT(*) FROM candidate GROUP BY office;'
  sqlite3 dataset.sqlite '.schema' | grep -i cpf      # prints NOTHING (decision D10)
  ```

### Failure behaviour

- [ ] DevTools → Network → block `*/data/latest.json` → reload: the app shows
      its "dados em atualização" error state, not a blank page or a stack trace.
- [ ] DevTools → Network shows **only same-origin requests** on every page
      (no font CDN, no analytics, no third-party scripts).

### Personal data (decision D10)

- [ ] No key containing `cpf` anywhere in the published JSON:

  ```bash
  curl -s "$SITE_URL/data/$V/index.json" | jq '[paths | map(tostring) | join(".") | select(test("cpf";"i"))]'   # []
  SLUG=$(curl -s "$SITE_URL/data/$V/index.json" | jq -r '.candidates[0].slug')
  curl -s "$SITE_URL/data/$V/candidates/$SLUG.json" | jq '[paths | map(tostring) | join(".") | select(test("cpf|email|titulo";"i"))]'   # []
  ```

- [ ] Campaign finance shows aggregates only — no donor or supplier names.

## 2. Data spot-checks against the official sites — [backend lead + AWS account owner]

Pick **5 candidates**: 2 federal, 2 state, and the one closest to the rank-20
cut-off. For each, compare the panel with the official source and note any
difference (a difference is a bug until explained).

| Check | Official source | Panel |
|---|---|---|
| Votes in Videira and in SC | TSE results portal (resultados.tse.jus.br), municipality Videira | Votes block |
| Result (eleito / suplente / não eleito) | TSE results portal | Header + Votes block |
| Declared assets total | TSE DivulgaCandContas (divulgacandcontas.tse.jus.br) | Assets block |
| Campaign revenue / contracted expenses totals | DivulgaCandContas → prestação de contas | Campaign finance block |
| Number of authored PL/PLP/PEC/PDL since 2023-02-01 | camara.leg.br/deputados/<id> → Propostas | Bills block |
| CEAP total per year | camara.leg.br/deputados/<id> → Cota parlamentar | Expenses block |
| Amendments to Videira (committed / paid) | portaldatransparencia.gov.br → Emendas, filter municipality Videira/SC | Amendments block |

And the selection rule itself:

```bash
sqlite3 dataset.sqlite "
  SELECT c.office, v.rank, v.votes, c.ballot_name, s.reasons
  FROM candidate c
  JOIN vote_total v ON v.tse_id = c.tse_id AND v.scope = 'municipality'
  JOIN selection s  ON s.tse_id = c.tse_id
  WHERE v.rank >= 18 OR s.reasons LIKE '%born%'
  ORDER BY c.office, v.rank;"
```

- [ ] Ranks 18–20 (and ties at 20) match the TSE results portal ordering for Videira.
- [ ] Everyone listed with reason `born_in_municipality` really has birthplace Videira on DivulgaCandContas.

## 3. Responsive — [frontend lead]

Procedure details: `04-frontend/07-accessibility-responsiveness-neutrality.md` Part 2 and §4.5.

- [ ] Browser widths 320, 360, 390, 768, 1024, 1440 px: no horizontal scroll, nothing cut off, tables usable.
- [ ] **Real phones**: one low-end Android (Chrome) and one iPhone (Safari), portrait and landscape.
- [ ] DevTools throttling "Slow 4G": the first page is usable within a few seconds; loading states are visible.
- [ ] Touch targets are comfortable (filters, links in tables, the back link).

## 4. Accessibility, manual pass — [frontend lead + one other member]

Procedure details: `04-frontend/07-accessibility-responsiveness-neutrality.md` Part 4.

- [ ] Keyboard only: skip link first, every control reachable in a logical order, focus always visible, no trap.
- [ ] Screen reader: VoiceOver (macOS/iOS) or NVDA (Windows) or TalkBack (Android) reads a card, a profile header and one data table sensibly.
- [ ] Contrast and automated hints with browser tools (Lighthouse / axe DevTools, run by hand): no serious issue left open.
- [ ] Zoom 200% and text-only zoom: content reflows, nothing overlaps.
- [ ] OS "reduce motion" on: no animation remains.
- [ ] `<html lang="pt-BR">`; each page has a distinct `<title>`; headings form a proper outline.

## 5. Neutrality review — [two members who did not write the pages]

Rules: `04-frontend/07-accessibility-responsiveness-neutrality.md` Part 3.

- [ ] Same template, same order, same wording for every candidate.
- [ ] No adjectives, scores, rankings beyond the published vote ranking, or party colours/logos.
- [ ] Empty states are worded the same way for everyone ("não há registros nesta fonte").
- [ ] The About page explains the selection rule and the sort order plainly.
- [ ] The site does not name or suggest endorsement by any public institution, party or candidate (the APL partnership belongs in the course deliverables, not on the site).
- [ ] Ideally, someone at the APL partner reads two profiles and the About page.

## 6. Security — [AWS account owner]

```bash
# Response headers on the HTML and on a data file
curl -sI "$SITE_URL/" | grep -iE '^(content-security-policy|strict-transport-security|x-frame-options|x-content-type-options|referrer-policy|permissions-policy):'
curl -sI "$SITE_URL/data/latest.json" | grep -iE '^(content-type|cache-control):'

# HTTP is redirected to HTTPS
curl -sI "http://${SITE_URL#https://}/" | sed -n 1p                       # 301

# The bucket is not reachable directly, and is fully private
curl -s -o /dev/null -w '%{http_code}\n' "https://$BUCKET.s3.amazonaws.com/index.html"   # 403
aws s3api get-public-access-block --bucket "$BUCKET" --profile personal  # all four true

# Missing files are real 404s
curl -s -o /dev/null -w '%{http_code}\n' "$SITE_URL/data/nao-existe.json"                # 404

# The deploy role trusts exactly two subjects of main
aws iam get-role --role-name ProjetoAplicado2Deploy --profile personal \
  --query 'Role.AssumeRolePolicyDocument.Statement[0].Condition'

# Account hygiene: no root access keys, MFA on root
aws iam get-account-summary --profile personal \
  --query 'SummaryMap.{RootAccessKeys:AccountAccessKeysPresent,RootMFA:AccountMFAEnabled}'
#   RootAccessKeys 0, RootMFA 1
```

- [ ] All six security headers present; CSP has `script-src 'self'` and `frame-ancestors 'none'`.
- [ ] `latest.json` is `application/json; charset=utf-8` with `max-age=60`.
- [ ] HTTP → HTTPS redirect works.
- [ ] Direct S3 access is `403`; Block Public Access is fully on.
- [ ] Trust policy shows only `repo:Miguel060/projetoAplicado2ifc:ref:refs/heads/main` and its immutable-id form.
- [ ] Infrastructure is deployed with a **least-privilege IAM user**, never root access keys; the account has **no root access keys** and **MFA is on** for the root user.
- [ ] No secrets in the repository or in GitHub:

  ```bash
  git grep -nE 'AKIA[0-9A-Z]{16}|aws_secret_access_key|BEGIN (RSA|EC|OPENSSH) PRIVATE KEY' || echo "clean"
  gh secret list --repo Miguel060/projetoAplicado2ifc          # empty: the project uses variables only
  ```

- [ ] The built site contains no credentials or internal URLs (`grep -rE 'AKIA|secret|localhost' frontend/dist` → nothing relevant).

## 7. Operations and cost — [AWS account owner]

- [ ] `refresh-data.yml` is enabled and its last run is green; the next scheduled run is a Monday.
- [ ] Someone who reads e-mail is the last editor of the `cron` line (failure notifications go to them).
- [ ] `deploy.yml`'s last run is green.
- [ ] Budgets exist:

  ```bash
  aws budgets describe-budgets --account-id 436812234154 --profile personal \
    --query 'Budgets[].[BudgetName,BudgetLimit.Amount,BudgetLimit.Unit]' --output table
  ```

- [ ] A week after launch, Cost Explorer shows ≈ US$0 for the project (filter by tag `project = projetoAplicado2ifc` once the cost allocation tag is activated).
- [ ] The S3 `data/` prefix holds at most 5 versions plus `latest.json`:
      `aws s3 ls "s3://$BUCKET/data/" --profile personal`.

## Sign-off

| Section | Checked by (role) | Date | Notes / issues opened |
|---|---|---|---|
| 1. Functional | | | |
| 2. Data spot-checks | | | |
| 3. Responsive | | | |
| 4. Accessibility | | | |
| 5. Neutrality | | | |
| 6. Security | | | |
| 7. Operations and cost | | | |

The MVP is handed to the APL partner only when every **blocker** in this
table is resolved. Non-blocking findings become GitHub issues.

---

## Rollback

### Bad data snapshot → point `latest.json` at the previous version

Takes about a minute; nothing is rebuilt. The previous versions are still in
the bucket (the refresh keeps the newest 5).

```bash
# 0. Preflight: must print 436812234154
aws sts get-caller-identity --profile personal --query Account --output text

# 1. See the versions (newest last) and pick the previous good one
aws s3 ls "s3://$BUCKET/data/" --profile personal
PREV=20261102T090000Z

# 2. Build the pointer from that version's own index.json (same shape as latest.json)
aws s3 cp "s3://$BUCKET/data/$PREV/index.json" - --profile personal \
  | jq '{schemaVersion, version, generatedAt}' > latest.json
cat latest.json

# 3. Switch the pointer and invalidate it
aws s3 cp latest.json "s3://$BUCKET/data/latest.json" --profile personal \
  --content-type "application/json; charset=utf-8" --cache-control "public, max-age=60"
aws cloudfront create-invalidation --profile personal \
  --distribution-id "$DIST" --paths "/data/latest.json"

# 4. Confirm
curl -s "$SITE_URL/data/latest.json"
```

**Then stop the next refresh from undoing it:** if the cause is an ETL change,
revert that commit on `main` first; otherwise disable the workflow
(Actions → Refresh data → ⋯ → Disable workflow) until the cause is fixed. The
next refresh would otherwise publish a new snapshot and switch the pointer
again.

Note: the prune step never deletes the version `latest.json` points to, so a
rolled-back version survives later refreshes until it is no longer live.

### Bad site deploy → revert the commit

The deploy role only trusts `main`, so the rollback goes through `main` too:

```bash
git switch -c revert/<short-name> origin/main
git revert <bad-commit-sha>
git push -u origin revert/<short-name>      # open the PR, get it approved, merge
```

The merge runs `deploy.yml` with the previous code. Old hashed assets are
kept in the bucket, so the previous build works as soon as `index.html` is
replaced. Data is not affected.

### Bad infrastructure change → redeploy the previous stack definition

```bash
git switch --detach <last-good-commit>
make infra-diff        # preflight + cdk diff --profile personal
make infra-deploy      # preflight + cdk deploy --profile personal
git switch main
```

The bucket is `RETAIN`, so no content is lost even if a resource is replaced.
Check the four stack outputs afterwards; if one changed, update the
repository variables (`05-infrastructure/02-github-oidc-and-variables.md`).
