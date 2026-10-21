# 04.03 — Landing page (`/`) and site layout

## Goal

The first page a visitor sees. In a few seconds it must say **what the
panel is, what it is for, how it works and who appears in it**, then send
the visitor to `/candidatos`. Sober, neutral, text first.

> **No hard dependency on the ETL.** The landing page and the site layout
> render completely, and work, **before any data snapshot exists**. Live
> figures are an enhancement: when `latest.json` is missing (404/403), fails
> to load, or has an unsupported `schemaVersion`, the page still ships every
> section and the figures strip reads "Dados em preparação". This is why the
> landing can be one of the **first things deployed** — `deploy.yml` can
> publish it before `refresh-data.yml` has ever run.

How that is guaranteed (`02-data-access.md`, Step 4):

- the root route (layout) has **no loader**;
- the landing route's `landingLoader` returns `{ index: getIndexOrNull() }`
  **without awaiting** it — navigation completes at once and the promise
  never rejects;
- the page reads it with `<Suspense>` + `<Await>`, so only the strip waits.

The landing shows **no candidate names or photos** — nothing on the front
page may feature, or seem to feature, anyone.

## Files to create

```
frontend/src/
├── components/
│   ├── SiteLayout.tsx           # header, nav, skip link, footer, <Outlet/> (Step 1)
│   └── landing/
│       ├── Hero.tsx
│       ├── StatsStrip.tsx
│       ├── WhatItIsFor.tsx
│       ├── WhatYouFind.tsx
│       ├── HowItWorks.tsx
│       ├── WhoAppears.tsx
│       ├── Impartiality.tsx
│       └── OfficialSources.tsx
└── pages/
    └── LandingPage.tsx
```

Uses shadcn `button`, `card` and `separator`; `ExternalLink`
(`05-profile-page.md`, Step 1); formatters and `getIndexOrNull` from
`02-data-access.md`; constants from `src/lib/site.ts` (`SITE_NAME`,
`SITE_SHORT_NAME`, `ELECTION_YEAR`, `MUNICIPALITY`, `UF`).

---

## Step 1 — `SiteLayout` (shared by every page)

```
┌────────────────────────────────────────────────────────────────────────┐
│ [Pular para o conteúdo]  (visible on focus only)                       │
│ Candidatos de Videira – Eleições 2026   Início  Candidatos  Sobre e metodologia │
├────────────────────────────────────────────────────────────────────────┤
│ <main id="conteudo" tabIndex={-1}>  <Outlet/>                          │
├────────────────────────────────────────────────────────────────────────┤
│ Dados atualizados em 09/11/2026 03:02 (horário de Brasília) · Fontes oficiais │  ← only on data pages
│ Projeto acadêmico de estudantes do IFC. Não é um site oficial de nenhum órgão público. │
│ Sobre e metodologia · Código-fonte                                      │
└────────────────────────────────────────────────────────────────────────┘
```

Rules:

- **No data dependency.** The layout never calls `useDataIndex()`. The
  freshness line uses `useOptionalDataIndex()`: rendered when a data page
  (`/candidatos`, `/candidatos/:slug`, `/sobre`) is active, omitted otherwise
  (the landing has its own figures strip).
- `<header>`: the site name as a `Link` to `/` (not an `h1` — each page owns
  its `h1`).
- `<nav aria-label="Principal">` with three `NavLink`s: "Início" → `/`
  (with `end`, so it is not "current" everywhere), "Candidatos" →
  `/candidatos` (no `end`, so it stays current on profiles), "Sobre e
  metodologia" → `/sobre`. `NavLink` sets `aria-current="page"`. On phones
  the three items wrap under the site name — no hamburger menu (three links
  fit, and a menu would add a focus-management case).
- `<main id="conteudo" tabIndex={-1}>` — the skip-link target. Pages render
  `div`/`article` inside it, never another `main`.
- `<footer>`: freshness line (data pages only) with a link "Fontes oficiais"
  → `/sobre#fontes`; the disclaimer "Projeto acadêmico de estudantes do IFC.
  Não é um site oficial de nenhum órgão público." (wording owned by
  `06-about-page.md`); links "Sobre e metodologia" and "Código-fonte"
  (`ExternalLink` to `REPOSITORY_URL`). No institution is named as partner
  or supporter.
- `<ScrollRestoration />` from React Router, so "back" returns to the same
  scroll position.
- Navigation pending state (`aria-busy`, progress bar): `02-data-access.md`,
  Step 4.
- **Focus on navigation:** when the **pathname** changes (not the query
  string), move focus to the new page's `h1` (every page renders
  `<h1 tabIndex={-1}>`), so screen-reader users hear the new page instead of
  staying on the link they clicked.

```tsx
const { pathname } = useLocation();
const first = useRef(true);
useEffect(() => {
  if (first.current) { first.current = false; return; } // not on first load
  document.querySelector<HTMLElement>("main h1")?.focus();
}, [pathname]);
```

## Step 2 — `LandingPage` structure

```tsx
export function LandingPage() {
  const { index } = useLoaderData() as LandingLoaderData;
  return (
    <div>
      <title>{SITE_NAME}</title>
      <meta
        name="description"
        content="Informações oficiais dos candidatos a deputado federal e estadual ligados a Videira/SC nas Eleições 2026, reunidas em um só lugar e com a fonte de cada dado."
      />
      <Hero />
      <Suspense fallback={<StatsStrip state={{ status: "loading" }} />}>
        <Await resolve={index} errorElement={<StatsStrip state={{ status: "unavailable" }} />}>
          {(data: DataIndex | null) => (
            <StatsStrip state={data ? { status: "ready", index: data } : { status: "unavailable" }} />
          )}
        </Await>
      </Suspense>
      <WhatItIsFor />
      <WhatYouFind />
      <HowItWorks />          {/* id="como-funciona" */}
      <WhoAppears index={index} />
      <Impartiality />
      <OfficialSources />
    </div>
  );
}
```

Every section is a `<section aria-labelledby>` with an `h2`, separated by
generous vertical space (`py-12 md:py-16`), single column of text
(`max-w-3xl`) inside the `max-w-6xl` container. No background images, no
illustrations of people, no animation.

## Step 3 — Hero

```
h1   Informações oficiais sobre os candidatos a deputado ligados a Videira
p    Votação, bens, contas de campanha e atuação parlamentar dos candidatos
     a deputado federal e estadual nas Eleições 2026, reunidos em um só
     lugar e com a fonte oficial de cada dado.
     [ Ver candidatos ]   [ Como funciona ]
```

- `h1` with `tabIndex={-1}`; text built from constants
  (`MUNICIPALITY`, `ELECTION_YEAR`), never from data.
- Primary CTA: `<Button asChild><Link to="/candidatos">Ver candidatos</Link></Button>`.
- Secondary CTA: `<Button variant="outline" asChild><a href="#como-funciona">Como funciona</a></Button>`
  — a same-page anchor; the target section has `scroll-mt-20`. With reduced
  motion the jump is instant (global CSS).
- Both 44 px tall; stacked full-width on phones, side by side from `sm`.

## Step 4 — Live figures strip (`StatsStrip`)

```ts
type StripState =
  | { status: "loading" }
  | { status: "ready"; index: DataIndex }
  | { status: "unavailable" };
```

| State | Visible text |
|---|---|
| `loading` | "Carregando informações dos dados…" (muted) |
| `ready` | "{N} candidatos · {M} fontes oficiais · atualizado em {DD/MM/AAAA}" |
| `unavailable` | "Dados em preparação" + small text "Os dados oficiais ainda estão sendo publicados. O restante desta página já está disponível." |

Values in `ready`:

- `N = index.candidates.length` (`formatInteger`);
- `M = Object.keys(index.sources).length`;
- date = `formatDayOf(index.generatedAt)` (Brasília calendar day).

Rendering rules:

- A `<p>` inside an `aside aria-label="Números do painel"`. The three figures
  are separated by "·" in the text itself (no icons carrying meaning).
- **Not** an `aria-live` region: the figures settle once, right after load;
  announcing them would interrupt the hero for screen-reader users. The
  loading text is plain text; `aria-busy="true"` while loading.
- Reserve the strip's height (`min-h-12`) in every state so the page does not
  jump when the figures arrive (no layout shift).
- Neutral styling: `bg-muted`, foreground text, no colour change between
  states — "Dados em preparação" is information, not an error.
- In `ready`, a small link "Ver candidatos" is **not** repeated (the hero has
  it); the strip is information only.

## Step 5 — "Para que serve"

```
h2  Para que serve
p   As informações oficiais sobre quem disputa uma vaga de deputado estão
    espalhadas por diferentes sites públicos: o Tribunal Superior Eleitoral
    (TSE) publica candidaturas, votação, bens e prestação de contas; a Câmara
    dos Deputados publica projetos, votações e gastos dos parlamentares; e o
    Portal da Transparência publica as emendas parlamentares. Consultar tudo
    isso exige saber onde procurar e como ler cada base.
p   Este painel reúne essas informações em um só lugar, para os candidatos
    ligados a Videira, e indica junto de cada dado a fonte oficial e a data
    em que foi coletado.
```

## Step 6 — "O que você encontra"

`h2` "O que você encontra no perfil de cada candidato", then a grid of eight
shadcn `Card`s (1 column on phones, 2 from `sm`, 4 from `lg`). Each card: a
lucide icon (`aria-hidden`), an `h3` and one sentence. Same visual weight for
all; no links inside (the CTA leads to the list).

| `h3` | Sentence |
|---|---|
| Votação | Votos recebidos em Videira e em Santa Catarina e o resultado da eleição. |
| Dados pessoais | Ocupação, grau de instrução, idade e naturalidade informados ao TSE. |
| Bens declarados | A lista de bens e valores declarada no registro de candidatura. |
| Contas de campanha | Totais de receitas por origem e de despesas por tipo. |
| Propostas | Para quem exerceu mandato de deputado(a) federal, os projetos apresentados na Câmara dos Deputados. |
| Atuação na Câmara | Votações nominais e gastos da cota parlamentar de deputados(as) federais. |
| Emendas em Videira | Valores de emendas parlamentares recebidos por beneficiários em Videira. |
| Redes sociais | Perfis e sites informados pelo(a) candidato(a) ao TSE. |

Below the grid, one muted line: "Cada informação aparece com a indicação da
fonte oficial e da data de coleta."

## Step 7 — "Como funciona" (`id="como-funciona"`)

`h2` "Como funciona", then an ordered list (`<ol>`) of three steps,
numbered visibly by the list itself (`list-decimal` or a styled counter —
the number is text, not an image):

1. **Coleta automática de dados oficiais.** Um programa baixa
   periodicamente os dados abertos do TSE, da Câmara dos Deputados e do Portal
   da Transparência. Nada é digitado ou editado à mão.
2. **Critério objetivo de seleção.** O mesmo critério, descrito abaixo, é
   aplicado a todos os candidatos para definir quem aparece no painel.
3. **Exibição com fonte e data.** Cada bloco de informação mostra de qual
   base oficial veio, quando a fonte gerou os dados e quando foram
   coletados, com link para a fonte.

Link at the end: "Detalhes da metodologia" → `/sobre`.

## Step 8 — "Quem aparece aqui"

`h2` "Quem aparece aqui". `topN` comes from the data when available; the
section **never waits** for it and never shows a number it does not have.

```tsx
<Suspense fallback={<WhoAppearsText topN={null} />}>
  <Await resolve={index} errorElement={<WhoAppearsText topN={null} />}>
    {(data: DataIndex | null) => <WhoAppearsText topN={data?.selection.topN ?? null} />}
  </Await>
</Suspense>
```

Copy with `topN` known:

> Para cada cargo — deputado(a) federal e deputado(a) estadual — aparecem os
> **{topN} candidatos mais votados em Videira** nas Eleições
> {ELECTION_YEAR}, além de todos os candidatos a esses cargos **nascidos em
> Videira**. Empates na última posição são todos incluídos. Nenhum nome é
> incluído ou retirado manualmente.

Copy with `topN` unknown (no data yet) — identical except the first bold
phrase becomes "**os candidatos mais votados em Videira**" (no number).

Link: "Veja o critério completo" → `/sobre#criterio`.

## Step 9 — "Compromisso de imparcialidade"

`h2` "Compromisso de imparcialidade", then a `<ul>`:

- O painel não emite opiniões, notas, rankings de desempenho nem
  recomendações de voto.
- Todos os candidatos são apresentados com o mesmo modelo de página, as mesmas
  seções e na mesma ordem.
- Não são usadas cores, logotipos ou símbolos de partidos.
- A ordem da lista segue critérios objetivos (votos ou ordem alfabética) e
  pode ser trocada por quem consulta.
- As informações são reproduzidas das fontes oficiais, sem edição.

## Step 10 — "Fontes oficiais"

`h2` "Fontes oficiais". A **text** list — no logos or visual identity of any
body. Static (it must work with no data); the per-dataset list with dates is
on `/sobre#fontes`.

| Item (text) | Link (`ExternalLink`) |
|---|---|
| Tribunal Superior Eleitoral (TSE) — Portal de Dados Abertos | `https://dadosabertos.tse.jus.br/` |
| Câmara dos Deputados — Dados Abertos | `https://dadosabertos.camara.leg.br/` |
| Controladoria-Geral da União (CGU) — Portal da Transparência | `https://portaldatransparencia.gov.br/` |

Followed by: "Em caso de divergência, a fonte oficial prevalece. Veja a lista
completa de bases utilizadas, com datas, em Sobre e metodologia." (link →
`/sobre#fontes`).

## Step 11 — Copy and visual rules specific to the landing

- No candidate names, photos, numbers or parties anywhere on the landing.
- No institution named as partner, sponsor or supporter; public bodies appear
  only as data sources (Step 10) and in the footer disclaimer.
- No stock photos, mascots, flags, coats of arms or party-like colours; the
  neutral tokens from `01-project-setup.md` only.
- No adjectives that judge candidates or politics ("bons", "ruins",
  "polêmicos"); the copy describes the panel, not the people in it.
- Sentence case headings, plain Portuguese, short paragraphs (aim for text a
  first-time voter understands).

## Done when

- [ ] **Before any data exists** (empty `backend/dist/data` locally, or a freshly created bucket with only the site deployed): `/` renders every section, the strip says "Dados em preparação", "Quem aparece aqui" shows the no-number wording, and the browser console shows no uncaught error.
- [ ] Same result with a `latest.json` whose `schemaVersion` is `2`, and with DevTools → Network → "Offline" after the page's first load.
- [ ] With data: the strip shows "{N} candidatos · {M} fontes oficiais · atualizado em DD/MM/AAAA" with values matching `index.json`; the topN in "Quem aparece aqui" matches `selection.topN`.
- [ ] On a throttled connection ("Slow 4G"), the hero and all text sections appear before the figures; the page does not jump when the figures arrive.
- [ ] "Ver candidatos" opens `/candidatos`; "Como funciona" jumps to the section, whose heading is not hidden under the header.
- [ ] Going from `/` to `/candidatos` after the figures loaded does not request `latest.json` or `index.json` again.
- [ ] The landing page contains no candidate name or photo, no logo of any body, and no mention of any institution as partner.
- [ ] Nav: "Início" is current only on `/`; "Candidatos" is current on `/candidatos` and on profiles.
- [ ] Keyboard: skip link, hero buttons, nav and every link reachable in order with a visible focus ring; one `h1`, then `h2`s, then `h3`s.
