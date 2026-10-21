# 03.05 — Load Câmara dos Deputados data

## Goal

For the selected candidates who were **federal deputies for SC in
legislature 57**, collect: identity in the Câmara, authored bills
(PL/PLP/PEC/PDL), roll-call votes and CEAP totals. Details of each endpoint
and file are in `02-data-sources/02-camara.md`.

## Files to create

```
backend/internal/camara/sources.go   # URLs, source ids/names/dataset URLs
backend/internal/camara/deputies.go  # MatchDeputies (API, CPF join)
backend/internal/camara/bills.go     # LoadBills (API, paginated)
backend/internal/camara/votes.go     # LoadRollCallVotes (bulk)
backend/internal/camara/expenses.go  # LoadExpenses (bulk CEAP)
```

## Types

```go
package camara

type Deputy struct {
	ID                int
	TSEID             string // the selected candidate it matched
	ParliamentaryName string // ultimoStatus.nome
	ElectoralName     *string
	CivilName         string // nomeCivil — kept in memory for amendment matching only
	Status            *string
	ProfileURL        string // https://www.camara.leg.br/deputados/<id>
}
type Deputies map[int]*Deputy // by Câmara id

type Bill struct {
	ID int; Type string; Number, Year int
	Summary string; PresentedAt string // "YYYY-MM-DD"
	URL string
}

type RollCallVote struct {
	VotingID, VotedAt, Vote string
	Description *string
	URL string
}

type Expenses struct {
	ByYear     map[int]float64
	ByCategory map[string]float64
}
```

## Step A — `MatchDeputies` (CPF join, then forget)

1. `GetJSON(CamaraAPI + "/deputados?idLegislatura=57&siglaUf=SC")`; follow
   `links[rel=next]` if present. Expect ~26 deputies.
2. For each id: `GetJSON(CamaraAPI + "/deputados/{id}")` (rate-limited).
3. Build `cpfToDeputy := map[string]*detail` with `digitsOnly(dados.cpf)`.
4. For each selected candidate with a non-empty CPF: if
   `cpfToDeputy[c.CPF]` exists → create `Deputy{ID, TSEID: c.TSEID, …}`.
5. Discard `cpfToDeputy` (let it go out of scope). The caller then runs
   `sel.ForgetCPFs()` (see `01-go-project-setup.md`).
6. Record `camara_deputies` (`GeneratedAt = nil`, `FileURL` = the list URL,
   `DatasetURL = https://dadosabertos.camara.leg.br/swagger/api.html`).
7. Log `matched=<n>` — **never** log CPFs or which CPF matched.

Fallback when a candidate has no CPF in the TSE file: match by
`Norm(nomeCivil) == Norm(FullName)` **and** same birth date; log
`matched_by=name_birthdate`. Do not match by name alone.

## Step B — `LoadBills` (API, paginated)

For each matched deputy:

```go
url := fmt.Sprintf("%s/proposicoes?idDeputadoAutor=%d&siglaTipo=%s&dataApresentacaoInicio=%s&ordem=DESC&ordenarPor=id&itens=100",
	config.CamaraAPI, d.ID, config.BillTypes, config.LegislatureStart)
for url != "" {
	var page struct {
		Dados []struct {
			ID int `json:"id"`; SiglaTipo string `json:"siglaTipo"`
			Numero int `json:"numero"`; Ano int `json:"ano"`
			Ementa string `json:"ementa"`; DataApresentacao string `json:"dataApresentacao"`
		} `json:"dados"`
	}
	next, _, err := c.GetJSON(ctx, url, &page) // next = links[rel=next].href or ""
	// append Bill{…, PresentedAt: dataApresentacao[:10], URL: "https://www.camara.leg.br/propostas-legislativas/<id>"}
	url = next
}
```

- Keep **all** bills in SQLite; the export caps the list at 50 most recent.
- `byType` counts are computed at export from the stored rows.
- Deduplicate by `(bill_id, deputy_id)` (pages can shift if a bill is
  registered during pagination).
- Record `camara_bills` (`GeneratedAt = nil`).

## Step C — `LoadRollCallVotes` (bulk, per year)

For `Y` in `config.ChamberYears`:

1. `votacoesProposicoes-Y.csv` → `map[idVotacao]{proposicaoID, titulo}` (first
   proposition per voting; ~5 MB/year).
2. `votacoes-Y.csv` → `map[id]{data, descricao}` (~5–9 MB/year).
3. Stream `votacoesVotos-Y.csv` (18–58 MB/year), `csvx.UTF8BOM`:
   keep rows where `deputado_id ∈ deps` **and** `deputado_idLegislatura == "57"`.
   - `VotedAt = dataHoraVoto` (as published, ISO without zone);
   - `Vote = voto` verbatim;
   - `Description = votacoes[id].descricao` or `proposicao_titulo` or nil;
   - `URL = https://www.camara.leg.br/propostas-legislativas/<proposicaoID>` if
     linked, else `uriVotacao`.
4. Record `camara_votes` with `GeneratedAt` = HTTP `Last-Modified` of the
   votes file of the first year, `FileURL` = that file,
   `DatasetURL = https://dadosabertos.camara.leg.br/swagger/api.html#staticfile`.

These are downloaded with `c.Get` (cached per day), not range-read: they are
plain CSVs, not zips.

Memory: only the two small lookup maps are held; the big votes file is
streamed and filtered row by row.

## Step D — `LoadExpenses` (CEAP)

For `Y` in `config.ChamberYears`:

1. `OpenZipMember(CamaraCotas+"/Ano-Y.csv.zip", "Ano-Y.csv")` (4–7.5 MB zips).
2. `csvx.UTF8BOM`; `Require("ideCadastro","numAno","txtDescricao","vlrLiquido")`.
3. Keep rows where `Int(ideCadastro) ∈ deps` (**not** `nuDeputadoId`).
4. `ByYear[numAno] += Money(vlrLiquido)`; `ByCategory[txtDescricao] += …`
   (negative values included).
5. Round after summing. Record `camara_expenses` (`GeneratedAt` = zip member mtime).

Never read supplier/passenger columns.

## Pitfalls

- 504 from the API is normal — the retry helper handles it; a deputy whose
  bills still fail after retries **fails the run** (no partial profiles).
- `ordenarPor=id` + `ordem=DESC` gives "most recent first" by id; sort by
  `PresentedAt` desc, then id desc, at export anyway.
- The January part of the 2023 votes file belongs to legislature 56.
- A deputy may have left office (`situacao` ≠ `Exercício`): still a valid
  match; the UI shows the status verbatim.

## Done when

```bash
DS=backend/dist/data/$(jq -r .version backend/dist/data/latest.json)/dataset.sqlite
sqlite3 "$DS" "SELECT d.deputy_id, c.ballot_name, d.status,
  (SELECT COUNT(*) FROM chamber_bill b WHERE b.deputy_id=d.deputy_id) AS bills,
  (SELECT COUNT(*) FROM chamber_roll_call_vote v WHERE v.deputy_id=d.deputy_id) AS votes,
  (SELECT ROUND(SUM(total),2) FROM chamber_expense_by_year e WHERE e.deputy_id=d.deputy_id) AS ceap
  FROM chamber_deputy d JOIN candidate c USING (tse_id);"
# Spot-check one deputy against https://www.camara.leg.br/deputados/<id>:
# bill count (filter by the four types since 2023) and CEAP yearly totals should match.
```
