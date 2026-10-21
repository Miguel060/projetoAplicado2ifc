# 04.06 — About and methodology page (`/sobre`)

## Goal

One page that makes the panel **auditable**: anyone can read exactly who is
listed and why, where every value comes from and when it was collected, what
is deliberately not shown, how to download the full dataset, and who made the
site. It also states plainly that this is **not an official site of any
public body**.

Everything that can come from `index.json` (numbers, dates, sources, dataset
size and hash) is rendered from it, never typed by hand, so the page can never
drift from the data.

## Files to create

```
frontend/src/pages/AboutPage.tsx
frontend/src/components/about/SourcesTable.tsx
frontend/src/components/about/DatasetDownload.tsx
```

Uses `useDataIndex()` (root loader data), `ProfileSection`-like plain
sections (no SourceNote here — the whole page is about sources),
`ExternalLink`, the edited `Table`, and the formatters from
`02-data-access.md`.

## Page structure

```
h1  Sobre o painel e metodologia
nav "Nesta página" (anchors)

h2  Sobre o projeto                         #projeto
h2  Critério de seleção dos candidatos      #criterio
h2  O que é mostrado — e o que não é        #o-que-e-mostrado
h2  Fontes oficiais                         #fontes
h2  Atualização dos dados                   #atualizacao
h2  Imparcialidade                          #imparcialidade
h2  Privacidade                             #privacidade
h2  Dados abertos: baixe o conjunto completo #dados
h2  Créditos                                #creditos
h2  Contato e correções                     #contato
```

`<title>Sobre e metodologia – {SITE_SHORT_NAME}</title>`.

Anchors are stable: the search page links to `#criterio`, the footer to
`#fontes`, the Proposals section to `#o-que-e-mostrado`. Every `h2` has
`scroll-mt-20` so anchored headings are not hidden under the header.

---

## Section copy (pt-BR) and rendering rules

Values in `{…}` come from the index.

### Sobre o projeto (`#projeto`)

> Este painel reúne, em um só lugar, informações oficiais sobre os candidatos
> a deputado(a) federal e deputado(a) estadual ligados a Videira/SC nas
> Eleições {election.year}. Cada informação é exibida com a indicação da
> fonte oficial, da data em que a fonte gerou os dados e da data em que foram
> coletados. O painel não emite opiniões, notas ou recomendações.

### Critério de seleção dos candidatos (`#criterio`)

> Para cada cargo — deputado(a) federal e deputado(a) estadual — são exibidos:
>
> 1. os **{selection.topN}** candidatos com mais votos nominais válidos em
>    Videira no 1º turno das Eleições {election.year}
>    ({formatDate(election.date, "long")}), somando todas as zonas eleitorais
>    do município. Se houver empate na {selection.topN}ª posição, todos os
>    empatados são incluídos;
> 2. todos os candidatos a esses cargos **nascidos em Videira** com
>    candidatura apta, independentemente da votação.
>
> O mesmo critério é aplicado automaticamente a todos os candidatos. Nenhum
> nome é incluído ou excluído manualmente. Hoje o painel exibe
> {counts.federal_deputy} candidatos a deputado(a) federal e
> {counts.state_deputy} a deputado(a) estadual.
>
> Município: {election.municipality.name}/{election.uf} (código IBGE
> {election.municipality.ibgeCode}).

### O que é mostrado — e o que não é (`#o-que-e-mostrado`)

Two short lists.

**Mostrado, para todos os candidatos:** votação em Videira e em Santa
Catarina e resultado; dados pessoais publicados pelo TSE (ocupação, grau de
instrução, idade, naturalidade); bens declarados; totais da prestação de
contas de campanha; redes sociais informadas ao TSE.

**Mostrado para quem exerceu mandato de deputado(a) federal na legislatura
2023–2027:** projetos de autoria ou coautoria, votações nominais, gastos da
cota parlamentar (CEAP) e emendas parlamentares com beneficiários em Videira.

Then three fixed explanations:

> **Propostas.** O TSE publica propostas de governo apenas de candidatos a
> cargos do Poder Executivo (presidente e governador). Candidatos a deputado
> não registram propostas no TSE. Por isso, para quem já exerceu mandato de
> deputado(a) federal, a seção "Propostas" mostra os projetos apresentados na
> Câmara dos Deputados; para os demais, informa que não há proposta
> registrada em fonte oficial.
>
> **Deputados estaduais e a ALESC.** A Assembleia Legislativa de Santa
> Catarina não disponibiliza seus dados por meio de uma API de dados abertos.
> Por isso a atuação na ALESC não é exibida; o perfil de quem foi eleito(a)
> deputado(a) estadual em 2022 e concorre à reeleição traz um link para a
> página da ALESC.
>
> **Não exibimos**, mesmo quando publicados pelo TSE: CPF, número do título de
> eleitor, e-mail, estado civil, cor/raça, gênero, certidões criminais e nomes
> ou documentos de doadores e fornecedores de campanha. Esses dados não são
> necessários para conhecer a atuação e as propostas dos candidatos, e vários
> deles se referem a pessoas que não são candidatas.

### Fontes oficiais (`#fontes`) — `SourcesTable`

Rendered from `Object.values(index.sources)`, grouped by `publisher` in this
fixed order: TSE, Câmara dos Deputados, CGU – Portal da Transparência. Within a
group, sorted by `name` (pt-BR collator).

Table — caption "Fontes utilizadas nesta versão dos dados":

| Publicador | Conjunto de dados | Gerado pela fonte em | Coletado em |
|---|---|---|---|
| `publisher` | `ExternalLink` (`datasetUrl`) with text `name` | `formatTimestamp(generatedAt)` or "não informado pela fonte" | `formatTimestamp(fetchedAt)` |

Below the table:

> Os dados são reproduzidos das fontes acima sem edição manual. Em caso de
> divergência, **a fonte oficial prevalece**.

### Atualização dos dados (`#atualizacao`)

> Os dados são coletados automaticamente uma vez por semana e sempre que o
> código de coleta é alterado. Versão atual: {version}, publicada em
> {formatTimestamp(generatedAt)}. Se uma das fontes estiver fora do ar ou com
> erro, a versão anterior continua no ar até a próxima coleta bem-sucedida.

### Imparcialidade (`#imparcialidade`)

> Este é um painel não-partidário. Todos os candidatos são apresentados com o
> mesmo modelo de página, as mesmas seções e na mesma ordem. Não usamos cores,
> logotipos ou símbolos de partidos, não atribuímos notas, não destacamos
> nenhum candidato e não publicamos opiniões. A ordenação padrão da lista é
> pelo número de votos em Videira — um dado objetivo — e pode ser trocada por
> ordem alfabética ou por votos em Santa Catarina. Nenhuma ordenação
> representa recomendação.

### Privacidade (`#privacidade`)

> Este site não usa cookies, não exige cadastro e não utiliza ferramentas de
> análise de audiência ou de rastreamento. Todos os arquivos (páginas, fotos,
> fontes tipográficas e dados) são servidos pelo próprio site; nenhuma
> informação sobre sua visita é enviada a terceiros. Links externos (TSE,
> Câmara dos Deputados, redes sociais) levam a sites com políticas de
> privacidade próprias.
>
> Em respeito à Lei Geral de Proteção de Dados (LGPD), o painel exibe apenas
> dados públicos necessários à finalidade de informar o eleitor e não
> reproduz dados pessoais sensíveis ou de terceiros (veja "O que é mostrado —
> e o que não é").

Keep this statement true: it is a commitment that the frontend (no trackers,
no external origins) and the infrastructure must honour. If CloudFront
access logging is ever enabled, this paragraph must be updated.

### Dados abertos (`#dados`) — `DatasetDownload`

```tsx
const href = dataUrl(index.version, index.dataset.path);
// <Button asChild><a href={href} download>Baixar o conjunto de dados (SQLite, {formatBytes(index.dataset.bytes)})</a></Button>
```

Copy:

> Todo o conteúdo do painel está disponível em um único arquivo de banco de
> dados SQLite, com a mesma versão exibida no site. O arquivo pode ser aberto
> com programas gratuitos como o DB Browser for SQLite ou com a ferramenta de
> linha de comando `sqlite3`.
>
> Verificação de integridade (SHA-256):
> `{index.dataset.sha256}`

- The hash in a `<code>` element with `break-all` so it wraps on phones.
- The link is same-origin (`/data/<version>/dataset.sqlite`), so the CSP is
  unaffected and the `download` attribute works.
- One line pointing to the source code: "O código-fonte do painel é aberto:"
  + `ExternalLink` to `REPOSITORY_URL` (from `src/lib/site.ts`).

### Créditos (`#creditos`)

> Projeto acadêmico desenvolvido por estudantes do Instituto Federal
> Catarinense (IFC) — <NOMES DO GRUPO> — na disciplina Projeto Aplicado II.
>
> **Este não é um site oficial de nenhum órgão público.** Ele não pertence
> nem fala em nome do Tribunal Superior Eleitoral, da Câmara dos Deputados, da
> Controladoria-Geral da União ou de qualquer outra instituição. As
> informações exibidas são reproduzidas das fontes oficiais citadas nesta
> página.

`<NOMES DO GRUPO>` is a placeholder the group fills in before launch.

Rules that keep the site from implying official status or sponsorship:

- **No institution is named as partner, sponsor or supporter** anywhere on
  the site (landing, this page, footer, metadata). Public bodies are named
  only as **data sources** (TSE, Câmara dos Deputados, CGU – Portal da
  Transparência) and in the disclaimer above. The municipal legislative body
  of Videira is **not mentioned** at all.
- **No coat of arms, logo or visual identity** of any public body, of the
  municipality or of the federal government anywhere on the site (header,
  favicon, footer, social preview). Text mentions only.
- Never use wording such as "portal oficial", "site da …", "em nome de".
- The short disclaimer "Projeto acadêmico de estudantes do IFC. Não é um site
  oficial de nenhum órgão público." appears in the footer of every page (see
  `03-landing-page.md`, Step 1).

### Contato e correções (`#contato`)

> Encontrou uma informação diferente da fonte oficial? Escreva para
> {CONTACT_EMAIL} informando o candidato e a seção. Verificaremos se a coleta
> reproduz corretamente a fonte. Não alteramos dados manualmente: correções
> nos dados de origem devem ser solicitadas ao órgão responsável por eles.

`CONTACT_EMAIL` comes from `src/lib/site.ts` (`<CONTACT_EMAIL>` placeholder
until the group defines it). Render it as a `mailto:` link.

---

## Tests for this step

Component tests (`renderRoute("/sobre")` with a fixture index):

| File | Cases |
|---|---|
| `src/pages/AboutPage.test.tsx` | the ten `h2`s in order, each with its anchor id (`projeto`, `criterio`, `o-que-e-mostrado`, `fontes`, `atualizacao`, `imparcialidade`, `privacidade`, `dados`, `creditos`, `contato`); **every dynamic value comes from the fixture** — change `topN`, `counts`, `election.date`, `version`, `generatedAt` in the fixture and assert the new values appear; the credits contain `<NOMES DO GRUPO>` (or the real names once filled) and the disclaimer "Este não é um site oficial de nenhum órgão público"; the strings "Câmara Municipal", "Vereadores", "parceir" do not occur anywhere in the rendered text; the contact link is `mailto:` + `CONTACT_EMAIL`. |
| `src/components/about/SourcesTable.test.tsx` | caption; one row per source, grouped TSE → Câmara dos Deputados → CGU, sorted by name inside a group; `generatedAt: null` → "não informado pela fonte"; dataset name is an `ExternalLink`. |
| `src/components/about/DatasetDownload.test.tsx` | link `href` = `dataUrl(version, dataset.path)` with the `download` attribute; label includes `formatBytes(dataset.bytes)`; the SHA-256 is rendered in a `code` element; repository link present. |

End-to-end (Playwright, golden snapshot, both projects):

| File | Scenario |
|---|---|
| `e2e/about.spec.ts` | the sources table has one row per key of the golden `index.sources`; **download link:** `page.waitForEvent("download")` after clicking "Baixar o conjunto de dados" → the saved file's SHA-256 (Node `crypto`) equals the hash shown on the page and the golden `index.dataset.sha256`; **anchors:** `/sobre#criterio`, `#fontes`, `#o-que-e-mostrado`, `#dados` put their heading in the viewport below the header; **no data:** with `latest.json` fulfilled as 404, `/sobre` shows "Os dados estão em preparação" inside the layout. Zero axe violations, zero console errors. |

## Browser check (Claude in Chrome)

Procedure: `08-testing.md`. Serve with `npm run build && npm run preview`
(local snapshot, golden as fallback).

- **Controls to exercise:** every link of the "Nesta página" list; every
  dataset link in the sources table (opens a new tab — close it); the
  download button (confirm a `.sqlite` download starts; compare its size with
  the label); the repository link; the `mailto:` link (only hover/inspect —
  do not send); the links arriving here from other pages (`/candidatos` →
  "Como os dados são reunidos", footer "Fontes oficiais", profile "Por que
  isso?") land on the right heading.
- **Read the page** against the data: `topN`, counts, dates and version match
  `index.json`; no institution is presented as partner; the disclaimer is
  present here and in the footer.
- **Keyboard:** full Tab pass; the download link is reachable and activates
  with Enter; the sources table region scrolls with arrows at 360 px.
- **Widths:** 360 × 800 (hash wraps, table scrolls inside its region) and 1440 × 900.
- **Console:** `read_console_messages` with pattern `error|Uncaught|Content Security` → nothing.
- **Network:** `read_network_requests` → the `dataset.sqlite` request answers 200 from the same origin.
- **GIF:** `step-22-about.gif`.
- **Fallback:** Playwright MCP or a throw-away script, noted in the PR.

## Done when

- [ ] Every number, date, count, source and the SHA-256 on the page come from `index.json` (change a value in the local `index.json` and see it change on the page).
- [ ] The sources table lists all 11 source ids, each with a working link and both dates.
- [ ] The download button saves the `.sqlite` file of the current version; its SHA-256 (`shasum -a 256 dataset.sqlite`) matches the one shown.
- [ ] `/sobre#criterio`, `#fontes`, `#o-que-e-mostrado` and `#dados` scroll to the right heading, not hidden under the header.
- [ ] No logo or coat of arms of any public body appears anywhere on the site.
- [ ] The "não é um site oficial de nenhum órgão público" sentence appears on this page and in the footer of every page.
- [ ] Searching the built site's text (landing, about, footer, `index.html`) for "Câmara Municipal", "Vereadores", "parceir" and "apoio" finds nothing.
- [ ] The placeholders `<NOMES DO GRUPO>`, `<CONTACT_EMAIL>` and `<REPOSITORY_URL>` have been replaced before launch (listed in `07-delivery/02-launch-checklist.md`).
- [ ] `make test-frontend` is green with coverage ≥ 85 % on all four metrics.
- [ ] `make test-e2e` is green on `chromium` and `webkit`.
- [ ] The browser check above is done and its GIF is referenced in the PR description.
