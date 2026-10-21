# 02 — Câmara dos Deputados

Publisher: **Câmara dos Deputados – Dados Abertos**
(<https://dadosabertos.camara.leg.br>). No authentication, no key. Two access
modes, both used:

| Mode | Base | Used for |
|---|---|---|
| REST API v2 (JSON) | `https://dadosabertos.camara.leg.br/api/v2` | deputies (`camara_deputies`), authored bills (`camara_bills`) |
| Bulk files (CSV) | `https://dadosabertos.camara.leg.br/arquivos/...` and `https://www.camara.leg.br/cotas/` | roll-call votes (`camara_votes`), parliamentary quota CEAP (`camara_expenses`) |

Scope: **legislature 57** (2023-02-01 → 2027-01-31). Only deputies whose
`siglaUf = SC` matter, because SC candidates can only match SC deputies.

Cadence: bulk files are regenerated **daily** (all `Last-Modified:
Wed, 21 Oct 2026 06:3x–06:5x GMT` on the day checked). The API is live.

## REST API

Always send `Accept: application/json` (otherwise the API may answer XML)
and the project `User-Agent`.

### Deputies of SC in legislature 57

```
GET /api/v2/deputados?idLegislatura=57&siglaUf=SC
```

- Returns **26** deputies (16 seats + substitutes who took office), fields:
  `id, uri, nome, siglaPartido, uriPartido, siglaUf, idLegislatura, urlFoto, email`.
- Fits in one page; links only `self/first/last`. Still implement pagination
  generically (see bills).
- Observed: the same call with `&itens=100` returned **HTTP 504
  "upstream request timeout"** once and succeeded without it. → retries with
  backoff are mandatory for this API.

### Deputy detail (for CPF and names)

```
GET /api/v2/deputados/{id}
```

`dados` keys: `id, uri, nomeCivil, ultimoStatus, cpf, sexo, urlWebsite,
redeSocial, dataNascimento, dataFalecimento, ufNascimento,
municipioNascimento, escolaridade`.
`ultimoStatus` keys: `id, uri, nome, siglaPartido, uriPartido, siglaUf,
idLegislatura, urlFoto, email, data, nomeEleitoral, gabinete, situacao,
condicaoEleitoral, descricaoStatus`.

| Field | Use |
|---|---|
| `cpf` | **join to TSE `NR_CPF_CANDIDATO`** — in memory only, never stored (D10) |
| `ultimoStatus.nome` | `chamber_deputy.parliamentary_name`; also amendment-author matching |
| `ultimoStatus.nomeEleitoral` | `electoral_name`; amendment-author matching |
| `ultimoStatus.situacao` | `status` (e.g. `Exercício`, `Licença`, `Fim de Mandato`) |
| `id` | `deputy_id`; profile URL `https://www.camara.leg.br/deputados/{id}` |

**Ignored:** `email`, `gabinete`, `sexo`, `redeSocial` (we show the TSE's
social links only, one source per block), birth fields.

### Authored bills

```
GET /api/v2/proposicoes
      ?idDeputadoAutor={id}
      &siglaTipo=PL,PLP,PEC,PDL          ← comma-separated list works
      &dataApresentacaoInicio=2023-02-01
      &ordem=DESC&ordenarPor=id
      &itens=100                         ← maximum; larger values are capped to 100
      &pagina=N
```

Verified 2026-10-21 with a SC deputy:

- the type filter works (339 results with the four types vs 2,273 without —
  without it you get hundreds of `REQ`, `RPD`, `EMC`, `RIC`…);
- `X-Total-Count` header = total rows;
- body `links[]` has `rel: self | next | first | last`; **follow `next`
  until absent** (do not compute pages yourself);
- each item: `id, uri, siglaTipo, codTipo, numero, ano, ementa,
  dataApresentacao` (`"2023-02-02T11:42"` — no seconds, no zone).

`idDeputadoAutor` returns authorship **and co-authorship**; the UI says
"autoria ou coautoria". Bill page: `https://www.camara.leg.br/propostas-legislativas/{id}`.

## Bulk files

All Câmara CSVs: **UTF-8 with BOM**, `;` separator, every field quoted,
decimal point `.` with no thousands separator.

### Roll-call votes — `camara_votes`

For each year `Y` in 2023…2026:

| File | Size (2025) | Columns used |
|---|---|---|
| `/arquivos/votacoesVotos/csv/votacoesVotos-Y.csv` | 58 MB (2023: 43 MB, 2026: 18 MB) | `idVotacao`, `uriVotacao`, `dataHoraVoto`, `voto`, `deputado_id`, `deputado_idLegislatura` |
| `/arquivos/votacoes/csv/votacoes-Y.csv` | 8.8 MB | `id`, `data`, `descricao`, `siglaOrgao` |
| `/arquivos/votacoesProposicoes/csv/votacoesProposicoes-Y.csv` | 5 MB | `idVotacao`, `proposicao_id`, `proposicao_titulo` (e.g. `PL 1234/2025`) |

- Filter `votacoesVotos` by `deputado_id ∈ matched deputies` **and**
  `deputado_idLegislatura = 57` (the 2023 file starts in January, still
  legislature 56).
- Only **nominal (roll-call)** votes exist in these files; symbolic votes have
  no per-deputy record. The UI says so.
- `voto` values are kept verbatim (`Sim`, `Não`, `Abstenção`, `Obstrução`,
  `Artigo 17`, …).
- URL per vote: if `votacoesProposicoes` links the voting to a proposition,
  `https://www.camara.leg.br/propostas-legislativas/{proposicao_id}`; else the
  API `uriVotacao`.
- Description: `votacoes.descricao`; fallback `proposicao_titulo`.

### Parliamentary quota (CEAP) — `camara_expenses`

`https://www.camara.leg.br/cotas/Ano-Y.csv.zip` (2025: 6.9 MB zip → 75 MB
`Ano-2025.csv`; 2023: 7.5 MB; 2026: 4.4 MB). One member, `Ano-Y.csv`.

| Column | Use |
|---|---|
| `ideCadastro` | **deputy id = API `id`** (e.g. `98148`). Join on this. |
| `nuDeputadoId` | a *different* internal id (e.g. `3704`) — **do not join on it** |
| `numAno` | year |
| `txtDescricao` | expense category (e.g. `MANUTENÇÃO DE ESCRITÓRIO DE APOIO À ATIVIDADE PARLAMENTAR`) |
| `vlrLiquido` | net value, decimal point: `699`, `699.9`, `699.99`. Can be **negative** (refunds/compensations) — sum as is. |

**Ignored:** `cpf`, `txtCNPJCPF`, `txtFornecedor`, `txtPassageiro`,
`txtTrecho` (supplier and passenger identities, D10), `urlDocumento`.
Leadership rows (`txNomeParlamentar` like `LID.GOV-CD`, empty `ideCadastro`)
are skipped naturally by the join.

## Pitfalls

- API **504s happen**; retry (3 attempts, exponential backoff 1 s → 2 s → 4 s).
- Respect a polite rate: ≤ 4 requests/second to the API.
- `dataApresentacao` has no seconds/zone: parse with layout `2006-01-02T15:04`, store the date part.
- Bulk votes per year are tens of MB; stream them, never load whole files in memory.
- The BOM is on the **first header** (`﻿"idVotacao"`) — strip it or the first column name never matches.
- Leading zeros: `cpf` from the API is a string of 11 digits; compare as strings after removing non-digits.
