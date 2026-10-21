# 03 — Portal da Transparência (CGU): parliamentary amendments

Publisher: **Controladoria-Geral da União – Portal da Transparência**.
Source id: `cgu_amendments`.

## Why the bulk file, not the API

| Access | Status (2026-10-21) | Decision |
|---|---|---|
| REST API `https://api.portaldatransparencia.gov.br/api-de-dados/emendas` | `401` without the personal `chave-api-dados`; CORS preflight `403` | **Not used** — it would need a secret (D9). |
| Website pages (`portaldatransparencia.gov.br/emendas/...`) | `405` "Human Verification" page for scripted clients | Not usable by the ETL; detail-page URL pattern **unverified**. |
| **Bulk download** | `200`, no key, no captcha | **Used.** |

## File

| | |
|---|---|
| Landing page | <https://portaldatransparencia.gov.br/download-de-dados/emendas-parlamentares> |
| Stable URL | `https://portaldatransparencia.gov.br/download-de-dados/emendas-parlamentares/UNICO` → **302** to the file below |
| Direct file | `https://dadosabertos-download.cgu.gov.br/PortalDaTransparencia/saida/emendas-parlamentares/EmendasParlamentares.zip` (32.5 MB) |
| Members | `EmendasParlamentares.csv` (47 MB; 3.9 MB compressed), `EmendasParlamentares_PorFavorecido.csv` (183 MB; **24.2 MB compressed**), `EmendasParlamentares_Convenios.csv` (26 MB) |
| Coverage | amendment years 2014 → 2026 |
| Cadence | regenerated periodically (member mtime on 2026-10-21: 2026-10-09). Use the zip member's modified time as `source.generated_at`. |
| Encoding | **Windows-1252** (UTF-8 decoding fails at byte 2), **no BOM** |
| Separator | `;` |
| Decimal | comma, no thousands separator: `1300000,00` |
| Sentinels | `Sem informação` (author name, amendment code), `S/I` (author code) |

Request the direct URL first; if it fails with 404 (the CGU has moved files
before), request the `/UNICO` URL and follow the redirect.

## ⚠️ Which member actually answers "amendments destined to Videira"

Verified 2026-10-21:

| Member | Filter | Rows for Videira |
|---|---|---|
| `EmendasParlamentares.csv` | `Código Município IBGE = 4219309` | **3** (2014, 2017, 2020) — most amendments are booked at state level (`SANTA CATARINA (UF)`) or "multiple localities", so this file misses almost everything |
| `EmendasParlamentares_PorFavorecido.csv` | `UF Favorecido = 'SC'` and `Município Favorecido = 'VIDEIRA'` | **84** rows, R$ 13,237,939.69 received, years 2015–2026 |
| `EmendasParlamentares_Convenios.csv` | `Localidade do gasto = 'VIDEIRA - SC'` | 3 |

**Use `EmendasParlamentares_PorFavorecido.csv` to decide *which* amendments
reached Videira and *how much* Videira received**, and
`EmendasParlamentares.csv` only to enrich each amendment code with year,
function/subfunction and the whole-amendment committed/liquidated/paid totals.

This needs a small change to the foundation contract (the current
`amendment` table and `Amendment` type only have whole-amendment values) —
see "Contract change required" in `03-backend-etl/06-load-amendments.md`.

### `EmendasParlamentares_PorFavorecido.csv` columns

| Column | Use |
|---|---|
| `Código da Emenda` | key (e.g. `202543010014`; first 4 digits = amendment year) |
| `Código do Autor da Emenda` | SIOP author code (e.g. `4301`) — **override key** for matching |
| `Nome do Autor da Emenda` | author name, upper case (e.g. `FULANO DE TAL`) — matched to Câmara names |
| `Número da emenda` | number |
| `Tipo de Emenda` | e.g. `Emenda Individual - Transferências com Finalidade Definida`, `Emenda Individual - Transferências Especiais`, `Emenda de Bancada`, `Emenda de Comissão`, `Emenda de Relator` |
| `Ano/Mês` | `YYYYMM` of the payment |
| `Tipo Favorecido` | `Pessoa Jurídica` / `Pessoa Física` |
| `Favorecido` | beneficiary name (e.g. `MUNICIPIO DE VIDEIRA`, `FUNDO MUNICIPAL DE SAUDE`) |
| `Natureza Jurídica` | e.g. `Associação Privada` |
| `UF Favorecido` | `SC` (two letters here) |
| `Município Favorecido` | `VIDEIRA` |
| `Valor Recebido` | amount received |

**Ignored:** `Código do Favorecido` (CNPJ/CPF). Rows with
`Tipo Favorecido = 'Pessoa Física'` are **summed but never named** (D10).

Authors seen for Videira: `RELATOR GERAL` (34 rows), `BANCADA DE SANTA
CATARINA` (8), individual parliamentarians (e.g. 6, 6, 4, 3, 2 rows),
`Sem informação` (4). Only **individual** amendments
(`Tipo de Emenda` starting with `Emenda Individual`) can be attributed to a
deputy; relator, committee and caucus amendments are not attributed to any
candidate (the About page explains this).

### `EmendasParlamentares.csv` columns (enrichment only)

`Código da Emenda`, `Ano da Emenda`, `Tipo de Emenda`, `Código do Autor da
Emenda`, `Nome do Autor da Emenda`, `Número da emenda`, `Nome Função`,
`Nome Subfunção`, `Valor Empenhado`, `Valor Liquidado`, `Valor Pago`.
Note `UF` here is the **full name** (`SANTA CATARINA`), unlike the
per-beneficiary file.

## Join to candidates

Amendment author → Câmara deputy (only deputies matched to a selected
candidate, see `02-camara.md`):

1. `config/name-overrides.json` by **author code** (explicit, wins);
2. else normalised author name equals normalised `ultimoStatus.nomeEleitoral`
   or `ultimoStatus.nome` or `nomeCivil`;
3. else unmatched → logged as a warning with the author code and name, so the
   group can add an override.

Amendments from earlier legislatures by the same person are included (they
are facts about the same parliamentarian); the UI groups by year.

## Pitfalls

- Windows-1252, not Latin-1: the difference matters for `–` and quotes in
  function names. Use `charmap.Windows1252`.
- Amount strings use comma decimals; a naive `strconv.ParseFloat` fails.
- The same amendment code appears in many beneficiary rows (one per payment
  month and beneficiary) — aggregate before storing.
- The author name is the *parliamentary* name in upper case, without accents
  in some rows and with accents in others: always normalise before comparing.
