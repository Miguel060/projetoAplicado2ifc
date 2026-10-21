# 03 — Data model (SQLite)

The ETL writes a **fresh** `dataset.sqlite` on every run (no migrations: the
file is rebuilt from the sources each time). It is the project's relational
model, the input of the JSON export, and a public open-data download.

The schema lives in `backend/internal/store/schema.sql` and is embedded with
`//go:embed`. Below is its full content.

## Principles

1. **Every row traces to a source.** Tables that hold source data have a
   `source_id` column referencing `source(id)`.
2. **No sensitive personal data.** No CPF, voter ID, e-mail, donor or supplier
   identities (D10). CPF exists only in memory during matching.
3. **Store raw labels, derive codes.** Keep the TSE's own text (e.g.
   `ELEITO POR QP`) next to our normalised code (`elected_by_quota`), so the
   UI can always show the official wording.
4. **Aggregates are tables too.** Campaign finance and CEAP are stored
   pre-aggregated, because the detail is not published by us.
5. **Money is `REAL` in BRL, dates are ISO-8601 `TEXT`.**

## Entity overview

```
source ─┬─< candidate ─┬─< vote_total
        │              ├── selection
        │              ├─< asset
        │              ├─< campaign_revenue_by_origin
        │              ├─< campaign_expense_by_type
        │              ├─< social_link
        │              └── chamber_deputy ─┬─< chamber_bill
        │                                  ├─< chamber_roll_call_vote
        │                                  ├─< chamber_expense_by_year
        │                                  ├─< chamber_expense_by_category
        │                                  └─< amendment ─< amendment_beneficiary
        └─ (every table above has source_id)
ingest_run (one row: metadata of the run)
```

## `schema.sql`

```sql
PRAGMA foreign_keys = ON;

-- One row per official dataset / endpoint used in this run.
CREATE TABLE source (
  id            TEXT PRIMARY KEY,        -- e.g. 'tse_candidates' (see list below)
  publisher     TEXT NOT NULL,           -- 'TSE' | 'Câmara dos Deputados' | 'CGU – Portal da Transparência'
  name          TEXT NOT NULL,           -- human name, pt-BR, shown in the UI
  dataset_url   TEXT NOT NULL,           -- human landing page of the dataset
  file_url      TEXT NOT NULL,           -- exact file or endpoint fetched
  generated_at  TEXT,                    -- when the publisher generated it (ISO-8601), if known
  fetched_at    TEXT NOT NULL            -- when we downloaded it (ISO-8601, UTC)
);

CREATE TABLE ingest_run (
  version       TEXT PRIMARY KEY,        -- '20261109T030000Z'
  started_at    TEXT NOT NULL,
  finished_at   TEXT NOT NULL,
  git_sha       TEXT,                    -- commit of the ETL code
  election_year INTEGER NOT NULL,
  uf            TEXT NOT NULL,
  municipality  TEXT NOT NULL,           -- 'VIDEIRA'
  ibge_code     TEXT NOT NULL,           -- '4219309'
  tse_code      TEXT NOT NULL,           -- resolved at run time from the votes file
  top_n         INTEGER NOT NULL
);

-- Only the SELECTED candidates are stored (≈40–45 rows).
CREATE TABLE candidate (
  tse_id            TEXT PRIMARY KEY,    -- SQ_CANDIDATO
  slug              TEXT NOT NULL UNIQUE,
  office            TEXT NOT NULL CHECK (office IN ('federal_deputy','state_deputy')),
  office_code       INTEGER NOT NULL,    -- CD_CARGO: 6 federal, 7 state
  number            TEXT NOT NULL,       -- NR_CANDIDATO
  ballot_name       TEXT NOT NULL,       -- NM_URNA_CANDIDATO
  full_name         TEXT NOT NULL,       -- NM_CANDIDATO
  social_name       TEXT,                -- NM_SOCIAL_CANDIDATO
  birth_date        TEXT,                -- DT_NASCIMENTO → ISO
  age_at_inauguration INTEGER,           -- NR_IDADE_DATA_POSSE (complementary file)
  birth_city        TEXT,                -- NM_MUNICIPIO_NASCIMENTO (complementary file)
  birth_uf          TEXT,                -- SG_UF_NASCIMENTO
  occupation        TEXT,                -- DS_OCUPACAO
  education         TEXT,                -- DS_GRAU_INSTRUCAO
  party_number      INTEGER NOT NULL,    -- NR_PARTIDO
  party_acronym     TEXT NOT NULL,       -- SG_PARTIDO
  party_name        TEXT NOT NULL,       -- NM_PARTIDO
  federation_acronym TEXT,               -- SG_FEDERACAO
  federation_name   TEXT,                -- NM_FEDERACAO
  candidacy_status  TEXT NOT NULL,       -- DS_SITUACAO_CANDIDATO_TOT (complementary file), e.g. 'DEFERIDO'
                                         -- (DS_SITUACAO_CANDIDATURA is '#NE' for every 2026 SC deputy)
  result_code       TEXT NOT NULL,       -- normalised, see mapping below
  result_tse_label  TEXT NOT NULL,       -- DS_SIT_TOT_TURNO as published
  running_for_reelection INTEGER NOT NULL CHECK (running_for_reelection IN (0,1)), -- elected to the same office in 2022,
                                         -- from historico_candidatura_2026 (ST_REELEICAO is '#NE' for everyone)
  declared_assets   INTEGER NOT NULL CHECK (declared_assets IN (0,1)),             -- ST_DECLARAR_BENS = 'S'
  photo_path        TEXT,                -- 'photos/<tse_id>.jpg' or NULL
  tse_page_url      TEXT,                -- DivulgaCandContas deep link or NULL
  source_id         TEXT NOT NULL REFERENCES source(id)
);

CREATE TABLE vote_total (
  tse_id        TEXT NOT NULL REFERENCES candidate(tse_id),
  scope         TEXT NOT NULL CHECK (scope IN ('municipality','state')),
  votes         INTEGER NOT NULL,        -- sum of QT_VOTOS_NOMINAIS_VALIDOS
  office_valid_total INTEGER NOT NULL,   -- all valid nominal votes for this office in the scope
  share         REAL NOT NULL,           -- votes / office_valid_total
  rank          INTEGER,                 -- competition rank in the scope (1,2,2,4); NULL if not computed
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (tse_id, scope)
);

CREATE TABLE selection (
  tse_id        TEXT PRIMARY KEY REFERENCES candidate(tse_id),
  reasons       TEXT NOT NULL            -- comma list: 'top_votes', 'born_in_municipality'
);

CREATE TABLE asset (
  tse_id        TEXT NOT NULL REFERENCES candidate(tse_id),
  ord           INTEGER NOT NULL,        -- NR_ORDEM_BEM_CANDIDATO
  type          TEXT NOT NULL,           -- DS_TIPO_BEM_CANDIDATO
  description   TEXT NOT NULL,           -- DS_BEM_CANDIDATO
  value         REAL NOT NULL,           -- VR_BEM_CANDIDATO
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (tse_id, ord)
);

CREATE TABLE campaign_revenue_by_origin (
  tse_id        TEXT NOT NULL REFERENCES candidate(tse_id),
  origin        TEXT NOT NULL,           -- DS_ORIGEM_RECEITA
  total         REAL NOT NULL,           -- sum of VR_RECEITA
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (tse_id, origin)
);

CREATE TABLE campaign_expense_by_type (
  tse_id        TEXT NOT NULL REFERENCES candidate(tse_id),
  kind          TEXT NOT NULL,           -- DS_ORIGEM_DESPESA
  total         REAL NOT NULL,           -- sum of VR_DESPESA_CONTRATADA
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (tse_id, kind)
);

CREATE TABLE social_link (
  tse_id        TEXT NOT NULL REFERENCES candidate(tse_id),
  ord           INTEGER NOT NULL,        -- NR_ORDEM_REDE_SOCIAL
  url           TEXT NOT NULL,           -- DS_URL (only http/https kept)
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (tse_id, ord)
);

-- Selected candidates who were federal deputies in legislature 57 (2023–2027).
CREATE TABLE chamber_deputy (
  deputy_id     INTEGER PRIMARY KEY,     -- Câmara id
  tse_id        TEXT NOT NULL UNIQUE REFERENCES candidate(tse_id),
  parliamentary_name TEXT NOT NULL,      -- ultimoStatus.nome
  electoral_name TEXT,                   -- ultimoStatus.nomeEleitoral
  status        TEXT,                    -- ultimoStatus.situacao (e.g. 'Exercício')
  profile_url   TEXT NOT NULL,           -- https://www.camara.leg.br/deputados/<id>
  source_id     TEXT NOT NULL REFERENCES source(id)
);

CREATE TABLE chamber_bill (
  bill_id       INTEGER NOT NULL,        -- proposição id
  deputy_id     INTEGER NOT NULL REFERENCES chamber_deputy(deputy_id),
  type          TEXT NOT NULL,           -- siglaTipo: PL | PLP | PEC | PDL
  number        INTEGER NOT NULL,
  year          INTEGER NOT NULL,
  summary       TEXT NOT NULL,           -- ementa
  presented_at  TEXT NOT NULL,           -- dataApresentacao (date part)
  url           TEXT NOT NULL,           -- https://www.camara.leg.br/propostas-legislativas/<id>
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (bill_id, deputy_id)
);

CREATE TABLE chamber_roll_call_vote (
  voting_id     TEXT NOT NULL,           -- idVotacao, e.g. '2345719-45'
  deputy_id     INTEGER NOT NULL REFERENCES chamber_deputy(deputy_id),
  voted_at      TEXT NOT NULL,           -- dataHoraVoto
  vote          TEXT NOT NULL,           -- 'Sim' | 'Não' | 'Abstenção' | 'Obstrução' | 'Artigo 17' | …
  description   TEXT,                    -- votacoes.descricao
  url           TEXT NOT NULL,           -- proposition page if known, else the API uri
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (voting_id, deputy_id)
);

CREATE TABLE chamber_expense_by_year (
  deputy_id     INTEGER NOT NULL REFERENCES chamber_deputy(deputy_id),
  year          INTEGER NOT NULL,
  total         REAL NOT NULL,           -- sum of vlrLiquido
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (deputy_id, year)
);

CREATE TABLE chamber_expense_by_category (
  deputy_id     INTEGER NOT NULL REFERENCES chamber_deputy(deputy_id),
  category      TEXT NOT NULL,           -- txtDescricao
  total         REAL NOT NULL,
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (deputy_id, category)
);

-- Parliamentary amendments authored by a matched deputy whose money reached
-- beneficiaries in the configured municipality (CGU per-beneficiary file).
-- committed/liquidated/paid describe the WHOLE amendment (all localities);
-- received_in_municipality is what beneficiaries in Videira actually received.
CREATE TABLE amendment (
  code          TEXT PRIMARY KEY,        -- Código da Emenda
  deputy_id     INTEGER NOT NULL REFERENCES chamber_deputy(deputy_id),
  author_name   TEXT NOT NULL,           -- Nome do Autor da Emenda, as published
  year          INTEGER NOT NULL,        -- Ano da Emenda
  type          TEXT NOT NULL,           -- Tipo de Emenda
  number        TEXT NOT NULL,           -- Número da emenda
  function_name TEXT,                    -- Nome Função
  subfunction_name TEXT,                 -- Nome Subfunção
  committed     REAL NOT NULL,           -- Valor Empenhado (whole amendment)
  liquidated    REAL NOT NULL,           -- Valor Liquidado (whole amendment)
  paid          REAL NOT NULL,           -- Valor Pago (whole amendment)
  received_in_municipality REAL NOT NULL, -- SUM(Valor Recebido) for beneficiaries in the municipality
  url           TEXT,                    -- Portal da Transparência detail page (NULL if pattern unverified)
  source_id     TEXT NOT NULL REFERENCES source(id)
);

-- Who in the municipality received the money. Legal entities by name; natural
-- persons are never named: their amounts are summed under one fixed label (D10).
CREATE TABLE amendment_beneficiary (
  code          TEXT NOT NULL REFERENCES amendment(code),
  name          TEXT NOT NULL,           -- 'Favorecido' (legal entity) or 'Pessoas físicas (valores agregados)'
  received      REAL NOT NULL,
  source_id     TEXT NOT NULL REFERENCES source(id),
  PRIMARY KEY (code, name)
);

CREATE INDEX idx_bill_deputy   ON chamber_bill(deputy_id, presented_at DESC);
CREATE INDEX idx_vote_deputy   ON chamber_roll_call_vote(deputy_id, voted_at DESC);
CREATE INDEX idx_amend_deputy  ON amendment(deputy_id, year DESC);
```

## Source ids (fixed list)

| id | Publisher | What |
|---|---|---|
| `tse_candidates` | TSE | `consulta_cand_2026` + `consulta_cand_complementar_2026` + `historico_candidatura_2026` (SC members) |
| `tse_votes` | TSE | `votacao_candidato_munzona_2026` (SC member) |
| `tse_assets` | TSE | `bem_candidato_2026` (SC member) |
| `tse_finance` | TSE | `prestacao_de_contas_eleitorais_candidatos_2026` (SC members: revenue + contracted expenses) |
| `tse_social` | TSE | `rede_social_candidato_2026` (SC member) |
| `tse_photos` | TSE | `foto_cand2026_SC_div.zip` |
| `camara_deputies` | Câmara dos Deputados | API `/deputados` + `/deputados/{id}` |
| `camara_bills` | Câmara dos Deputados | API `/proposicoes?idDeputadoAutor=…` |
| `camara_votes` | Câmara dos Deputados | bulk `votacoesVotos-<year>.csv` + `votacoes-<year>.csv` |
| `camara_expenses` | Câmara dos Deputados | bulk `cotas/Ano-<year>.csv.zip` |
| `cgu_amendments` | CGU – Portal da Transparência | bulk `EmendasParlamentares.zip` (members `EmendasParlamentares.csv` + `EmendasParlamentares_PorFavorecido.csv`) |

When a source has several files (e.g. four years of votes), `file_url` stores
the first one and `dataset_url` the landing page; the About page lists the
landing pages.

## Result code mapping (`DS_SIT_TOT_TURNO` → `result_code`)

| TSE label | `result_code` | UI label (pt-BR) |
|---|---|---|
| `ELEITO POR QP` | `elected_by_quota` | Eleito(a) por quociente partidário |
| `ELEITO POR MÉDIA` | `elected_by_average` | Eleito(a) por média |
| `ELEITO` | `elected` | Eleito(a) |
| `SUPLENTE` | `substitute` | Suplente |
| `NÃO ELEITO` | `not_elected` | Não eleito(a) |
| `#NULO`, `#NULO#`, empty | `no_result` | Sem resultado divulgado |
| anything else | `other` | the TSE label itself |

Filter groups in the UI: **Eleitos(as)** = `elected*`; **Suplentes**; **Não eleitos(as)**; **Sem resultado** = `no_result` + `other`.

## Slug rule

`slug = kebab(textnorm(ballot_name)) + "-" + number` → `fulano-de-tal-1234`.
Numbers are unique per office within SC, and federal (4 digits) and state
(5 digits) numbers never collide, so slugs are unique. The ETL still asserts
uniqueness and aborts on a collision.

Every slug must match `^[a-z0-9-]+$` — **no dots**. The CloudFront SPA
rewrite function treats any path containing a `.` as a file, so a dotted slug
would 404 instead of opening the profile. The export step asserts the pattern.

## Handy queries (for the group and for the About page)

```sql
-- Who was selected and why
SELECT c.ballot_name, c.office, v.votes, v.rank, s.reasons
FROM candidate c JOIN vote_total v ON v.tse_id = c.tse_id AND v.scope = 'municipality'
JOIN selection s ON s.tse_id = c.tse_id
ORDER BY c.office, v.rank;

-- Amendments to Videira per deputy
SELECT d.parliamentary_name, SUM(a.paid) AS paid
FROM amendment a JOIN chamber_deputy d USING (deputy_id)
GROUP BY d.deputy_id ORDER BY paid DESC;
```
