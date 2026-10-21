# 04.05 — Candidate profile page (`/candidatos/:slug`)

## Goal

One page per candidate, built from `candidates/<slug>.json`, that shows every
official data block **in a fixed order, with the same template for everyone**,
and says for each block where the data came from and when.

Section order (from `00-overview/01-project.md` — do not reorder per
candidate):

| # | Section (h2, pt-BR) | Anchor | Contract field | Source id(s) |
|---|---|---|---|---|
| 1 | *(header: identification)* | — | `summary`, `officialLinks` | `tse_candidates`, `tse_photos` |
| 2 | Votação nas Eleições 2026 | `#votacao` | `votes`, `summary.result` | `tse_votes` |
| 3 | Dados pessoais declarados ao TSE | `#dados-pessoais` | `identity` | `tse_candidates` |
| 4 | Bens declarados | `#bens` | `assets` | `tse_assets` |
| 5 | Prestação de contas de campanha | `#contas` | `campaignFinance` | `tse_finance` |
| 6 | Propostas | `#propostas` | `chamber.bills` | `camara_bills` |
| 7 | Atuação na Câmara dos Deputados | `#atuacao-camara` | `chamber.rollCallVotes`, `chamber.expenses` | `camara_deputies`, `camara_votes`, `camara_expenses` |
| 8 | Emendas parlamentares destinadas a Videira | `#emendas` | `amendments` | `cgu_amendments` |
| 9 | Redes sociais e sites informados ao TSE | `#redes` | `social` | `tse_social` |
| 10 | Assembleia Legislativa de Santa Catarina (ALESC) | `#alesc` | `stateAssembly` | `tse_candidates` (2022 history); ALESC itself is link only |

Sections 2–9 render for **every** candidate. When a block does not apply or
has no data, the section still appears with a neutral sentence — so two
profiles always look structurally the same. Section 10 renders for **every
state-deputy candidate** and never for federal ones (it is about the state
assembly).

## Files to create

```
frontend/src/
├── pages/CandidatePage.tsx
├── components/
│   ├── SourceNote.tsx
│   ├── ExternalLink.tsx
│   ├── ValueBar.tsx
│   ├── EmptyState.tsx
│   └── profile/
│       ├── ProfileHeader.tsx
│       ├── ProfileToc.tsx
│       ├── ProfileSection.tsx
│       ├── VotesSection.tsx
│       ├── IdentitySection.tsx
│       ├── AssetsSection.tsx
│       ├── FinanceSection.tsx
│       ├── ProposalsSection.tsx
│       ├── ChamberSection.tsx
│       ├── AmendmentsSection.tsx
│       ├── SocialSection.tsx
│       └── StateAssemblySection.tsx
└── lib/safe-url.ts
```

---

## Step 1 — Shared building blocks

### `ProfileSection`

```tsx
interface ProfileSectionProps {
  id: string;                 // anchor, e.g. "bens"
  title: string;              // h2 text
  sources: readonly SourceId[];
  children: React.ReactNode;
}

export function ProfileSection({ id, title, sources, children }: ProfileSectionProps) {
  const headingId = `${id}-titulo`;
  return (
    <section id={id} aria-labelledby={headingId} className="scroll-mt-20 py-8">
      <h2 id={headingId} className="text-xl font-semibold">{title}</h2>
      <div className="mt-4 space-y-4">{children}</div>
      {sources.length > 0 && <SourceNote sourceIds={sources} />}
    </section>
  );
}
```

Sections are separated by shadcn `Separator` (decorative, `decorative` prop
true).

### `SourceNote`

Reads `index.sources` through `useDataIndex()`.

```tsx
export function SourceNote({ sourceIds }: { sourceIds: readonly SourceId[] }) {
  const { sources } = useDataIndex();
  return (
    <div className="mt-4 rounded-md bg-muted px-3 py-2 text-xs text-muted-foreground">
      {sourceIds.map((id) => {
        const s = sources[id];
        if (!s) return <p key={id}>Fonte não informada.</p>;
        return (
          <p key={id}>
            <span className="font-medium text-foreground">Fonte:</span> {s.publisher} – {s.name}.{" "}
            Dados gerados pela fonte em{" "}
            {s.generatedAt ? formatTimestamp(s.generatedAt) : "data não informada pela fonte"}
            {" · "}coletados em {formatTimestamp(s.fetchedAt)}.{" "}
            <ExternalLink href={s.datasetUrl}>Ver fonte oficial</ExternalLink>
          </p>
        );
      })}
    </div>
  );
}
```

Rendered output example:

> **Fonte:** TSE – Bens de candidatos – Eleições 2026. Dados gerados pela
> fonte em 21/10/2026 19:33 (horário de Brasília) · coletados em 09/11/2026
> 03:01 (horário de Brasília). Ver fonte oficial (abre em nova aba)

### `ExternalLink` and `safe-url.ts`

```ts
// src/lib/safe-url.ts — defence in depth: the ETL already keeps only http(s)
export function safeHref(url: string): string | null {
  try {
    const u = new URL(url);
    return u.protocol === "https:" || u.protocol === "http:" ? u.href : null;
  } catch {
    return null;
  }
}
```

```tsx
export function ExternalLink({ href, children }: { href: string; children: React.ReactNode }) {
  const safe = safeHref(href);
  if (safe === null) return <span>{children}</span>; // never render javascript:/data: links
  return (
    <a href={safe} target="_blank" rel="noopener noreferrer" className="underline underline-offset-4">
      {children}
      <span className="ml-1 text-muted-foreground">(abre em nova aba)</span>
      <ExternalLinkIcon aria-hidden="true" className="ml-0.5 inline size-3.5" />
    </a>
  );
}
```

The "(abre em nova aba)" hint is **visible** text inside the link, so sighted
and screen-reader users get the same warning (WCAG 3.2.5 good practice).
`noopener noreferrer` stops the opened site from controlling our tab and from
seeing which profile the visitor came from.

### `ValueBar`

```tsx
/** Decorative bar; the value is always printed as text in the same row. */
export function ValueBar({ value, max }: { value: number; max: number }) {
  const pct = max > 0 ? Math.max(0, Math.min(100, (value / max) * 100)) : 0;
  return (
    <div aria-hidden="true" className="h-2 w-full rounded-full bg-muted">
      <div className="h-2 rounded-full bg-bar" style={{ width: `${pct}%` }} />
    </div>
  );
}
```

- `max` is the largest value **in the same table, for the same candidate**.
  Bars are never scaled against other candidates — the page must not
  invite comparisons the data does not support.
- Same neutral colour (`--bar`) everywhere; no colour encodes meaning.
- `style={{ width }}` goes through the CSSOM and is allowed by the CSP.

### `EmptyState`

```tsx
export function EmptyState({ children }: { children: React.ReactNode }) {
  return <p className="text-muted-foreground">{children}</p>;
}
```

Plain paragraph, no icon, no colour: absence of data is a fact, not a warning.

### Tables

Use the edited shadcn `Table` (see `01-project-setup.md` Step 7) for every
tabular block:

- visible `<caption>` (`caption-top`, left aligned) — also passed as the
  scroll region's `label`;
- `<th scope="col">` for column headers, `<th scope="row">` for the first
  cell of each row;
- money and counts right-aligned (`text-right tabular-nums`);
- totals in `<tfoot>` with `<th scope="row">Total</th>`;
- labels from the sources shown **verbatim** (TSE and Câmara often publish in
  upper case; keep it — fidelity over aesthetics).

---

## Step 2 — Page shell (`CandidatePage.tsx`)

```tsx
export function CandidatePage() {
  const { index, profile } = useLoaderData() as CandidateLoaderData;
  const { summary } = profile;
  const location = useLocation();
  const from = (location.state as { from?: unknown } | null)?.from;
  const backTo = typeof from === "string" && from.startsWith("?") ? `/candidatos${from}` : "/candidatos";

  return (
    <article className="mx-auto max-w-4xl px-4 py-6">
      <title>{`${summary.ballotName} (${OFFICE_LABEL[summary.office]}) – ${SITE_SHORT_NAME}`}</title>
      <meta
        name="description"
        content={`Informações oficiais de ${summary.ballotName}, candidato(a) a ${OFFICE_LABEL[summary.office].toLowerCase()} nas Eleições ${index.election.year}: votos em Videira, bens, prestação de contas e atuação, com a fonte de cada dado.`}
      />
      <Link to={backTo} className="text-sm underline underline-offset-4">
        ← Voltar para a lista de candidatos
      </Link>
      <ProfileHeader profile={profile} version={index.version} />
      <ProfileToc office={summary.office} />
      <VotesSection profile={profile} index={index} />
      <IdentitySection identity={profile.identity} />
      <AssetsSection assets={profile.assets} />
      <FinanceSection finance={profile.campaignFinance} officialLinks={profile.officialLinks} />
      <ProposalsSection summary={summary} chamber={profile.chamber} />
      <ChamberSection chamber={profile.chamber} />
      <AmendmentsSection amendments={profile.amendments} chamber={profile.chamber} />
      <SocialSection social={profile.social} />
      {summary.office === "state_deputy" && (
        <StateAssemblySection summary={summary} stateAssembly={profile.stateAssembly} />
      )}
    </article>
  );
}
```

- **Back link preserves filters:** the search page passes
  `state={{ from: location.search }}` on every card link; the back link
  rebuilds `/candidatos?…`. Only strings starting with `?` are accepted, so
  the state can never point outside the app. A visitor who arrived from a
  shared link (no state) goes to the unfiltered `/candidatos`. The browser back button works too
  (`ScrollRestoration` restores the list position).
- **`<title>` / `<meta>`** are hoisted to `<head>` by React 19.
- Wrapped in `<article>` inside the layout's `<main>`.

### `ProfileToc` ("Nesta página")

`<nav aria-label="Seções do perfil">` with an ordered list of anchor links to
sections 2–9 (+ 10 for state deputies). Same list for every candidate of the
same office. On `lg` screens it can be a sticky sidebar; on phones it is a
plain list under the header.

---

## Step 3 — Section 1: header (identification)

```
┌────────┐  FULANO DE TAL                         ← h1 (tabIndex -1)
│  foto  │  Deputado(a) federal · número 1234
│  3:4   │  Partido: ABC – PARTIDO EXEMPLO (12)
│        │  Federação: XYZ – FEDERAÇÃO EXEMPLO     (only when present)
└────────┘  Resultado: Suplente (TSE: SUPLENTE)
            Incluído(a) no painel por: entre os mais votados em Videira
            Página oficial no DivulgaCandContas (TSE) (abre em nova aba)
Fonte: TSE – Candidatos – Eleições 2026 … · Fonte: TSE – Fotos de candidatos …
```

- Photo: `alt={`Foto de ${ballotName} divulgada pelo TSE`}` (informative
  here — it is the official photo). `width`/`height` set, `aspect-[3/4]`,
  `w-32 sm:w-40`.
- No photo: the same neutral placeholder as the cards, plus visible small text
  "Foto não divulgada pelo TSE".
- Facts as a `<dl>` (term/definition), not loose text.
- Result: `resultLabel(summary.result)` followed by the verbatim TSE label in
  parentheses, so the official wording is always visible.
- Selection reasons: `REASON_LABEL` joined with " e ".
- DivulgaCandContas link only when `officialLinks.tseCandidatePage` is not
  null.
- `SourceNote` with `["tse_candidates", "tse_photos"]`.

## Step 4 — Section 2: Votação nas Eleições 2026

Content (all numbers from `profile.votes`):

> **Em Videira:** {formatVotes(m.votes)} — {formatShare(m.share)} dos votos
> válidos para {office lowercase} no município — {formatOrdinal(m.rank)}
> lugar entre {m.candidatesInOffice} candidatos ao cargo.
>
> **Em Santa Catarina:** {formatVotes(s.votes)} — {formatShare(s.share)} dos
> votos válidos para o cargo no estado.
>
> **Resultado:** {resultLabel} (TSE: {tseLabel}).
>
> Votos nominais válidos do 1º turno ({formatDate(election.date)}), somados em
> todas as zonas eleitorais de Videira.

Source: `tse_votes`. This section is never empty (every selected candidate
has a votes record — the ETL validates it).

## Step 5 — Section 3: Dados pessoais declarados ao TSE

`<dl>` with, in this order:

| Term | Value |
|---|---|
| Nome completo | `identity.fullName` |
| Nome social | `identity.socialName` — row omitted when null |
| Data de nascimento | `formatDate(birthDate, "long")` |
| Idade na data da posse | `{ageAtInauguration} anos` |
| Naturalidade | `{birthCity} – {birthUf}` |
| Ocupação declarada | `occupation` |
| Grau de instrução | `education` |
| Situação da candidatura | `candidacyStatus`, verbatim (e.g. "DEFERIDO") |

Any other null → "Não informado na fonte". Source: `tse_candidates`.

Deliberately absent (D10): CPF, voter ID, e-mail, marital status, race/colour,
gender — they are not in the contract and must not be added in the UI.

## Step 6 — Section 4: Bens declarados

| Case | Rendering |
|---|---|
| `assets.declared === false` | EmptyState: "O(a) candidato(a) informou ao TSE que não possui bens a declarar." |
| declared, `items` empty | EmptyState: "Não há bens listados na fonte até a data de coleta." |
| items present | Sentence "Total declarado: {formatBRL(total)} em {n} bem(ns)." + table |

Table — caption "Bens declarados ao TSE":

| Nº | Tipo | Descrição | Valor (R$) |
|---|---|---|---|
| `order` | `type` | `description` (wraps) | `formatBRL(value)` |
| **Total** | | | `formatBRL(total)` |

Note under the table: "Valores informados pelo(a) próprio(a) candidato(a) no
registro de candidatura." Source: `tse_assets`.

## Step 7 — Section 5: Prestação de contas de campanha

Two tables, each with a `ValueBar` column:

1. caption "Receitas por origem" — columns: Origem | Valor | (bar). Total row.
2. caption "Despesas contratadas por tipo" — columns: Tipo | Valor | (bar). Total row.

Empty cases (independently per table):

- revenue: "Não há receitas declaradas na fonte até a data de coleta."
- expenses: "Não há despesas declaradas na fonte até a data de coleta."

Note (always):

> A prestação de contas é declarada pelo(a) candidato(a) à Justiça Eleitoral
> e pode ser retificada. Este painel mostra apenas totais por origem e por
> tipo; nomes de doadores e fornecedores não são exibidos. O detalhamento
> completo está no DivulgaCandContas do TSE.

…followed by an `ExternalLink` to `officialLinks.tseCandidatePage` when not
null, otherwise to the `tse_finance` source's `datasetUrl`. Source:
`tse_finance`.

## Step 8 — Section 6: Propostas (D2)

### When `chamber` is not null

Intro:

> Como {ballotName} exerceu mandato de deputado(a) federal na legislatura
> 2023–2027, esta seção mostra os projetos que apresentou ou dos quais foi
> coautor(a) na Câmara dos Deputados — o registro oficial de propostas
> disponível para o cargo.

Then:

- counts by type as a list, with the type spelled out (no hover-only `abbr`):
  "PL – Projeto de Lei: 12", "PLP – Projeto de Lei Complementar: 2",
  "PEC – Proposta de Emenda à Constituição: 1",
  "PDL – Projeto de Decreto Legislativo: 3";
- the bills (`items`, already most-recent-first) as an `<ol>`; each item:
  - `<ExternalLink href={bill.url}>{type} {number}/{year}</ExternalLink>` · "apresentado em {formatDate(presentedAt)}";
  - the `summary` (ementa) verbatim below, as a paragraph;
- when `total > items.length`: "Mostrando os {items.length} mais recentes de
  {total}. A lista completa está na página do(a) deputado(a) na Câmara." +
  `ExternalLink` to `chamber.profileUrl`.

Empty (`chamber.bills.total === 0`):

> Não há projetos de lei, de lei complementar, de emenda à Constituição ou de
> decreto legislativo de autoria deste(a) parlamentar registrados na fonte
> desde 01/02/2023.

Source: `camara_bills`.

### When `chamber` is null

> Não há proposta registrada em fonte oficial para este cargo. O TSE publica
> propostas de governo apenas de candidatos a cargos do Poder Executivo
> (presidente e governador); candidatos a deputado não registram propostas no
> TSE. Para quem exerceu mandato de deputado(a) federal na legislatura
> 2023–2027, este painel mostra os projetos apresentados na Câmara dos
> Deputados — não há registro desse mandato para este(a) candidato(a).

No `SourceNote` in this case (there is no data shown); link "Por que isso?"
→ `/sobre#o-que-e-mostrado`.

## Step 9 — Section 7: Atuação na Câmara dos Deputados

### When `chamber` is null

> Não há registro de mandato de deputado(a) federal deste(a) candidato(a) na
> legislatura 2023–2027 na Câmara dos Deputados.

### When `chamber` is not null

Top line: "Situação na Câmara: {status ?? 'não informada na fonte'}." +
`ExternalLink` "Página oficial do(a) deputado(a) na Câmara" → `profileUrl`.

**h3 "Votações nominais"**

- Sentence: "Participou de {formatInteger(total)} votações nominais
  registradas desde 2023."
- Table — caption "Como votou": Voto | Quantidade | (bar). Votes verbatim
  ("Sim", "Não", "Abstenção", "Obstrução", "Artigo 17").
- Table — caption "Votações mais recentes": Data | Descrição | Voto | Link.
  Description verbatim (CSS `line-clamp-3` visually; the full text stays in
  the DOM for screen readers). Link: `ExternalLink` "Ver na Câmara".
- Fixed note: "A fonte registra apenas os votos computados em votações
  nominais; ela não indica presença ou ausência em sessões."
- Empty (`total === 0`): "Não há votos nominais registrados na fonte para
  este(a) parlamentar desde 2023."

**h3 "Cota para o Exercício da Atividade Parlamentar (CEAP)"**

- Sentence: "Total reembolsado (valor líquido) desde 2023:
  {formatBRL(expenses.total)}."
- Table — caption "CEAP por ano": Ano | Valor | (bar).
- Table — caption "CEAP por categoria": Categoria | Valor | (bar).
- Fixed note: "A CEAP é a verba com que a Câmara dos Deputados reembolsa
  despesas do mandato (como passagens, manutenção de escritório e divulgação),
  dentro de limites definidos por estado. Os valores são publicados pela
  própria Câmara."
- Empty (`total === 0`): "Não há despesas da cota parlamentar registradas na
  fonte desde 2023."

Sources: `camara_deputies`, `camara_votes`, `camara_expenses`.

## Step 10 — Section 8: Emendas parlamentares destinadas a Videira

An amendment often funds several localities at once. Its committed /
liquidated / paid values are for the **whole amendment, all localities** —
not for Videira. The only figure that measures money that reached Videira is
what **beneficiaries located in Videira received** (`receivedInMunicipality`).
The section is built so the two can never be confused.

| Case | Rendering |
|---|---|
| `amendments === null` | "Emendas parlamentares ao orçamento federal são apresentadas por deputados(as) federais e senadores(as) em exercício. Não há registro deste(a) candidato(a) como deputado(a) federal na legislatura 2023–2027." (no SourceNote) |
| `items` empty | "Não foram encontradas, no Portal da Transparência, emendas de autoria deste(a) parlamentar com beneficiários em Videira." + SourceNote |
| items present | headline + per-amendment list + glossary + SourceNote |

### Headline (the figure that matters)

```
Recebido por beneficiários em Videira
R$ 1.234.567,89                         ← large text: formatBRL(totals.receivedInMunicipality)
em {items.length} emenda(s) de autoria deste(a) parlamentar.
```

Directly below, smaller and explicitly labelled:

> Valor total das mesmas emendas (todas as localidades): empenhado
> {formatBRL(totals.committed)} · liquidado {formatBRL(totals.liquidated)} ·
> pago {formatBRL(totals.paid)}. Esses valores se referem às emendas inteiras,
> que podem beneficiar vários municípios, e **não** ao valor destinado a
> Videira.

### Per-amendment list

`<ol>` of amendments (already ordered: year desc, then
`receivedInMunicipality` desc). Each item is a small block — not one wide
table, because the beneficiary list varies per amendment:

```
h3  Emenda {number} ({year}) · {type}
    Área: {functionName} / {subfunctionName}          (omit a null part; both null → "Área não informada na fonte")

    Recebido por beneficiários em Videira: {formatBRL(receivedInMunicipality)}

    Table — caption "Beneficiários em Videira da emenda {number}/{year}":
    | Beneficiário | Valor recebido |
    | {name}       | formatBRL(received) |
    | Total        | formatBRL(receivedInMunicipality) |

    Valor total da emenda (todas as localidades):
    empenhado {BRL} · liquidado {BRL} · pago {BRL}
```

Rules:

- Beneficiaries are shown **exactly as in the contract**: legal entities by
  name; natural persons only under the aggregated label
  "Pessoas físicas (valores agregados)", which the ETL already produces. The
  UI never tries to split or name individuals (D10).
- Every committed / liquidated / paid value carries the words
  "valor total da emenda (todas as localidades)" in its visible label — never
  a bare "Pago: R$ …" next to the Videira figure.
- No bars in this section: the two kinds of values have different meanings
  and a bar would invite comparing them.
- `url` is `null` in practice (the Portal da Transparência blocks scripted
  verification of per-amendment pages). Render `ExternalLink` to `url` only
  when it is non-null; otherwise each item shows nothing extra and the
  SourceNote links to the CGU dataset page.

Glossary, always visible under the list (standard budget terms, factual):

> **Recebido por beneficiários em Videira:** soma dos valores que
> beneficiários localizados em Videira receberam por meio da emenda.
> **Empenhado:** valor reservado no orçamento para a despesa.
> **Liquidado:** valor cuja entrega do bem ou serviço foi verificada.
> **Pago:** valor efetivamente pago ou transferido.
> Os três últimos se referem à emenda inteira, em todas as localidades.

Source: `cgu_amendments`.

## Step 11 — Section 9: Redes sociais e sites informados ao TSE

- `<ul>` of `ExternalLink`s. Link text: `host` in medium weight, then the full
  URL in small muted text (`break-all`) — two links to the same host stay
  distinguishable.
- Fixed note: "Endereços informados pelo(a) candidato(a) ao TSE. O conteúdo
  desses perfis e sites é de responsabilidade de seus titulares."
- Empty: "Não há redes sociais ou sites informados ao TSE."
- No embeds, no follower counts, no previews (CSP and neutrality).

Source: `tse_social`.

## Step 12 — Section 10: ALESC (state-deputy candidates only — D3)

`stateAssembly` is non-null when the candidate was **elected state deputy in
2022** according to the TSE candidacy history — i.e. is running for
re-election to the same office (`summary.runningForReelection`).

| Case | Rendering |
|---|---|
| `stateAssembly !== null` | "Segundo o histórico de candidaturas do TSE, {ballotName} foi eleito(a) deputado(a) estadual em 2022 e concorre à reeleição. A Assembleia Legislativa de Santa Catarina não disponibiliza seus dados por meio de uma API de dados abertos, por isso a atuação na ALESC não é exibida neste painel. Consulte diretamente:" + `ExternalLink` "Deputados na página da ALESC" → `stateAssembly.profileUrl` |
| `stateAssembly === null` | "Segundo o histórico de candidaturas do TSE, este(a) candidato(a) não foi eleito(a) deputado(a) estadual em 2022; portanto, não há mandato atual na ALESC a consultar." |

SourceNote with `["tse_candidates"]` (the history file is part of that
source); no data from ALESC itself is shown.

---|---|
| `stateAssembly !== null` (running for re-election) | "Segundo o TSE, {ballotName} concorre à reeleição para deputado(a) estadual. A Assembleia Legislativa de Santa Catarina não disponibiliza seus dados por meio de uma API de dados abertos, por isso a atuação na ALESC não é exibida neste painel. Consulte diretamente:" + `ExternalLink` "Deputados na página da ALESC" → `stateAssembly.profileUrl` |
| `stateAssembly === null` | "Segundo o TSE, este(a) candidato(a) não concorre à reeleição para deputado(a) estadual." |

No SourceNote (no data from ALESC is shown).

---

## Done when

- [ ] Two profiles of the same office (one incumbent, one not) have identical section lists, headings and order; the non-incumbent shows the neutral sentences in sections 6–8.
- [ ] Every section that shows data ends with a SourceNote with both dates and a working link.
- [ ] Every table has a visible caption, column headers, and scrolls horizontally inside its own focusable region on a 320 px screen (Tab reaches it; arrow keys scroll it).
- [ ] Every value shown by a bar is also printed as text in the same row.
- [ ] Every external link shows "(abre em nova aba)" and opens a new tab; DevTools shows `rel="noopener noreferrer"`.
- [ ] The tab title reads "<NOME> (Deputado(a) federal) – Candidatos de Videira".
- [ ] From a filtered list, open a profile, click "Voltar para a lista de candidatos": the same filters are restored. Opening a profile from a pasted URL and clicking it goes to `/candidatos`.
- [ ] A state-deputy candidate shows section 10; a federal-deputy candidate does not.
- [ ] In the amendments section of an incumbent with amendments, the large headline is "Recebido por beneficiários em Videira", and every committed/liquidated/paid value on the page is labelled "valor total da emenda (todas as localidades)".
- [ ] Natural-person beneficiaries appear only as "Pessoas físicas (valores agregados)"; the beneficiary totals of each amendment add up to its "Recebido por beneficiários em Videira".
- [ ] A state-deputy candidate elected in 2022 shows the ALESC link; one not elected in 2022 shows the neutral sentence instead.
- [ ] No CPF, e-mail, campaign donor or supplier name, or name of an individual amendment beneficiary appears anywhere (search the page text for "@" and for 11-digit numbers).
