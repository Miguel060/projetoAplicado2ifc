# 04.04 — Search page (`/candidatos`)

## Goal

The search page lists every candidate in the panel. It is reached from the
landing page's "Ver candidatos" button and from the "Candidatos" item in the
navigation. A citizen can:

- type a name, ballot number or party acronym;
- filter by office, result and party;
- sort by an **objective** criterion (votes in Videira, votes in SC, name);
- see why these candidates are listed (the selection rule, in one sentence);
- share or bookmark the exact view (everything lives in the URL);
- open any profile and come back to the same filtered list.

Every candidate is rendered by the **same** card, with the same fields, in the
same order.

## Files to create

```
frontend/src/
├── components/
│   ├── SearchForm.tsx        # search box + filters + sort
│   ├── NativeSelect.tsx      # styled native <select>
│   ├── CandidateCard.tsx
│   ├── CandidatePhoto.tsx    # photo or neutral placeholder
│   └── ResultsSummary.tsx    # count + aria-live
├── pages/
│   ├── SearchPage.tsx
│   └── NotFoundPage.tsx
└── lib/
    ├── search-params.ts      # parse/serialise the URL state
    └── search.ts             # applySearch() — filter + sort (Step 3)
```

---

## Step 1 — Layout

The page renders inside `SiteLayout` (header, navigation, skip link, footer,
focus management and scroll restoration), specified in `03-landing-page.md`,
Step 1. The search page itself starts at its `h1`.

## Step 2 — URL state (`src/lib/search-params.ts`)

The URL query string is the **single source of truth**. No `useState` copy
of the filters exists anywhere.

| Param | Values | Default (param absent) | Meaning |
|---|---|---|---|
| `q` | free text | `""` | matched against `searchText` |
| `cargo` | `deputado-federal`, `deputado-estadual` | all offices | office filter |
| `situacao` | `eleitos`, `suplentes`, `nao-eleitos`, `sem-resultado` | all | result group (`RESULT_GROUPS` in `labels.ts`) |
| `partido` | a party acronym present in the index, e.g. `MDB` | all | party filter |
| `ordem` | `votos-videira`, `votos-sc`, `nome` | `votos-videira` | sort |

```ts
import type { Office } from "@/data/types";
import { RESULT_GROUPS, type ResultGroup } from "@/data/labels";

export type SortKey = "votos-videira" | "votos-sc" | "nome";

export interface SearchState {
  q: string;
  office: Office | null;
  group: ResultGroup | null;
  party: string | null;
  sort: SortKey;
}

const OFFICE_PARAM: Record<string, Office> = {
  "deputado-federal": "federal_deputy",
  "deputado-estadual": "state_deputy",
};
export const OFFICE_TO_PARAM: Record<Office, string> = {
  federal_deputy: "deputado-federal",
  state_deputy: "deputado-estadual",
};
const SORTS: readonly SortKey[] = ["votos-videira", "votos-sc", "nome"];

/** Unknown values are ignored (treated as the default), never an error. */
export function parseSearch(params: URLSearchParams, parties: ReadonlySet<string>): SearchState {
  const cargo = params.get("cargo");
  const situacao = params.get("situacao");
  const partido = params.get("partido");
  const ordem = params.get("ordem");
  return {
    q: params.get("q") ?? "",
    office: cargo !== null ? (OFFICE_PARAM[cargo] ?? null) : null,
    group: situacao !== null && situacao in RESULT_GROUPS ? (situacao as ResultGroup) : null,
    party: partido !== null && parties.has(partido) ? partido : null,
    sort: SORTS.includes(ordem as SortKey) ? (ordem as SortKey) : "votos-videira",
  };
}

/** Writes only non-default values, so the plain URL "/candidatos" is the default view. */
export function toSearchParams(s: SearchState): URLSearchParams {
  const p = new URLSearchParams();
  if (s.q.trim() !== "") p.set("q", s.q);
  if (s.office) p.set("cargo", OFFICE_TO_PARAM[s.office]);
  if (s.group) p.set("situacao", s.group);
  if (s.party) p.set("partido", s.party);
  if (s.sort !== "votos-videira") p.set("ordem", s.sort);
  return p;
}
```

Updating: `setSearchParams(toSearchParams(next), { replace: true })`.
`replace: true` so typing "joão" does not create four history entries; the
back button leaves the page instead of undoing keystrokes.

## Step 3 — Filtering and sorting algorithm (`src/lib/search.ts`)

Pure function, runs on every render (≈45 items — no memoisation or debounce
needed for performance):

```ts
const collator = new Intl.Collator("pt-BR", { sensitivity: "base", numeric: true });

export function applySearch(all: readonly CandidateSummary[], s: SearchState): CandidateSummary[] {
  const groupCodes = s.group ? new Set<ResultCode>(RESULT_GROUPS[s.group].codes) : null;
  const filtered = all.filter(
    (c) =>
      (s.office === null || c.office === s.office) &&
      (groupCodes === null || groupCodes.has(c.result.code)) &&
      (s.party === null || c.party.acronym === s.party) &&
      matchesQuery(c.searchText, s.q),
  );
  const byName = (a: CandidateSummary, b: CandidateSummary) =>
    collator.compare(a.ballotName, b.ballotName);
  const compare: Record<SortKey, (a: CandidateSummary, b: CandidateSummary) => number> = {
    // Rank asc (1 = most voted in Videira); ties broken alphabetically.
    "votos-videira": (a, b) => a.votes.municipalityRank - b.votes.municipalityRank || byName(a, b),
    "votos-sc": (a, b) => b.votes.state - a.votes.state || byName(a, b),
    nome: byName,
  };
  return filtered.toSorted(compare[s.sort]);
}
```

**Grouping by office.** Votes for federal and state deputy are different
contests, so a single ranking mixing both would be meaningless. When no
`cargo` filter is set, results render as **two sections**, in fixed order:
"Deputados(as) federais" then "Deputados(as) estaduais", each sorted by the
chosen key. With a `cargo` filter, one section. A section with zero results
after filtering shows its heading and "Nenhum resultado neste cargo."

The party `<select>` options are derived from the index, unique acronyms,
sorted with the same collator: `Todos os partidos`, then `ABC`, `DEF`, …
Option text: `"{acronym} – {name}"` (the full name helps people who do not
know the acronym).

## Step 4 — `SearchForm`

```
<form role="search" aria-label="Buscar candidatos">  (onSubmit: preventDefault — results are live)
  Label: "Buscar por nome, número ou partido"
  [ input type="search", autocomplete="off", enterKeyHint="search" ]

  <fieldset> <legend>Filtros</legend>
    Cargo:     [ Todos os cargos ▾ | Deputado(a) federal | Deputado(a) estadual ]
    Situação:  [ Todas ▾ | Eleitos(as) | Suplentes | Não eleitos(as) | Sem resultado ]
    Partido:   [ Todos os partidos ▾ | ABC – … ]
  </fieldset>

  Ordenar por: [ Mais votados em Videira ▾ | Mais votados em SC | Nome (A–Z) ]

  [ Limpar filtros ]   ← shown only when any param differs from the default
</form>
```

- Every control has a visible `<Label htmlFor>`; no placeholder-as-label.
- `NativeSelect` = a real `<select>` with the input's classes (`h-11`, border
  `--input`, focus ring). Changing a select updates the list in place; that is
  not a "change of context" (WCAG 3.2.2), so no submit button is required.
- "Limpar filtros" → `setSearchParams(new URLSearchParams(), { replace: true })`
  and moves focus to the search input.
- Layout: one column on phones; from `md` the three filters + sort sit in a
  4-column grid under the search box.

## Step 5 — Selection rule sentence (always visible)

Directly above the results, plain text, built from the index (no hard-coded
numbers):

> São exibidos os **{topN}** candidatos mais votados em Videira para cada
> cargo — deputado(a) federal e deputado(a) estadual — nas Eleições
> {election.year} (votos nominais válidos, 1º turno de
> {formatDate(election.date)}), além dos candidatos a esses cargos nascidos em
> Videira. Empates na {topN}ª posição são todos incluídos.
> [Como os dados são reunidos](/sobre#criterio)

And one line stating the current order, so the sort is never mistaken for a
recommendation:

> Ordenação atual: {sortLabel}. A ordem não representa recomendação nem
> avaliação.

where `sortLabel` is "mais votados em Videira" / "mais votados em Santa
Catarina" / "nome, em ordem alfabética".

## Step 6 — Results count (`ResultsSummary`)

```tsx
<p aria-live="polite" aria-atomic="true" className="text-sm text-muted-foreground">
  {announced}
</p>
```

- Text: `"{n} candidato(s) encontrado(s)"`; when no filter is active:
  `"{n} candidatos no painel"`.
- The **visible** count updates immediately; the **announced** text is the
  same string but updated 500 ms after the user stops typing (a `useEffect`
  with `setTimeout`), so screen readers are not flooded on every keystroke.
  This is the only `useState` on the page and it holds a message, not filter
  state.

## Step 7 — `CandidateCard` (identical for every candidate)

```
┌─────────────────────────────────────┐
│ ┌──────┐  FULANO DE TAL             │  ← h3 > Link (the only link in the card)
│ │ foto │  Deputado(a) federal · 1234│
│ │ 3:4  │  ABC · Federação XYZ       │
│ └──────┘  [Suplente]                │  ← Badge variant="outline", text only
│ 3.120 votos em Videira (10,4%) · 1º │
│ 101.234 votos em Santa Catarina     │
│ Incluído(a) por: entre os mais votados em Videira │
└─────────────────────────────────────┘
```

Rules:

- **Markup:** `<article>` (shadcn `Card`) with an `h3` containing the single
  `<Link to={`/candidatos/${slug}`} state={{ from: location.search }}>`. The
  link is stretched over the whole card with a CSS pseudo-element
  (`after:absolute after:inset-0`, card `relative`), so the entire card is
  clickable but the accessibility tree contains **one** link whose name is the
  candidate's name — not a card full of nested links.
- **Link text:** the ballot name. Add `aria-describedby` pointing to the
  office/number line so screen readers say "FULANO DE TAL, link, Deputado(a)
  federal, 1234".
- **Photo:** `CandidatePhoto` with `alt=""` (decorative here: the name is
  right next to it). Fixed 3:4 box (`aspect-[3/4] w-20`), `object-cover`,
  `loading="lazy"`, `decoding="async"`, explicit `width`/`height` to avoid
  layout shift.
- **Placeholder** when `photo === null`: the **same** neutral silhouette for
  everyone (inline SVG, `fill="currentColor"` in `text-muted-foreground`,
  `aria-hidden="true"`), on `bg-muted`. Never initials (they would vary in
  visual weight), never a party colour.
- **Fields, always in this order:** ballot name; office + number; party
  acronym (+ federation acronym when present); result badge; votes in Videira
  with share and rank; votes in SC; selection reason(s) joined with " e ".
- **Result badge:** `variant="outline"`, neutral colours for **every** result.
  Elected and not-elected look identical except for the words.
- No hover effects that change emphasis beyond a uniform border/background
  shift applied to all cards; focus shows the global focus ring around the
  card (`has-[a:focus-visible]:outline`).

## Step 8 — Grid and responsiveness

| Viewport | Columns |
|---|---|
| < 640 px (default) | 1 |
| `sm` ≥ 640 px | 2 |
| `lg` ≥ 1024 px | 3 |

Container: `mx-auto max-w-6xl px-4`. Results are a list: `<ul role="list">`
with each card in an `<li>` (the list semantics give "list, 20 items" to
screen readers). At 320 px width nothing scrolls horizontally.

## Step 9 — Empty state

When the filtered list is empty (across all sections):

```
Nenhum candidato encontrado com os filtros atuais.
Verifique a grafia ou tente buscar só pelo número.  [Limpar filtros]
```

Neutral copy; it never suggests "did you mean <candidate>".

## Step 10 — `NotFoundPage`

```
h1: Página não encontrada
p:  O endereço pode estar incorreto ou o candidato pode não fazer parte do
    painel. Os candidatos exibidos seguem o critério descrito em
    "Sobre e metodologia".
[Ver candidatos]  [Página inicial]  [Sobre e metodologia]
```

Links: `/candidatos`, `/`, `/sobre`. `<title>Página não encontrada –
{SITE_SHORT_NAME}</title>`.

`NotFoundPage` renders a `<div>` with its `h1`, not a `<main>`: it always
appears inside the layout (as the `*` route, or from `ErrorPage` for an
unknown candidate). It needs no data, so it also works before any snapshot
exists.

## Page `<title>` and metadata

React 19 hoists `<title>` and `<meta>` rendered anywhere to `<head>`:

```tsx
<title>{`Candidatos – ${SITE_NAME}`}</title>
<meta name="description" content="Busque e compare informações oficiais dos candidatos a deputado ligados a Videira/SC." />
```

The page `h1`: "Candidatos a deputado ligados a Videira" (visible).

## Tests for this step

Unit tests (Vitest):

| File | Cases |
|---|---|
| `src/lib/search-params.test.ts` | `parseSearch` of an empty query → all defaults; each valid param parsed (`cargo=deputado-federal` → `federal_deputy`, `situacao=suplentes`, `partido=PTX`, `ordem=nome`); unknown values (`cargo=foo`, `situacao=x`, `ordem=xyz`, a party not in the index) fall back to defaults without throwing. `toSearchParams` writes only non-default values (default state → empty string). **Round trip:** for a table of states, `parseSearch(toSearchParams(state)) ≡ state`; for a table of query strings, `toSearchParams(parseSearch(qs))` is the canonical form (e.g. `?ordem=votos-videira` → empty). |
| `src/lib/search.test.ts` | with a fixture of ~8 synthetic summaries across both offices: text query (accent-insensitive, multi-token, number, party acronym); each filter alone and combined; each result group maps to the right codes (including `other` under "sem-resultado"); sort `votos-videira` by rank asc with alphabetical tie-break; `votos-sc` by state votes desc; `nome` with the pt-BR collator ("Álvaro" before "Bruno", "Ana 2" before "Ana 10"); input array not mutated. |

Component tests (Testing Library, `renderRoute("/candidatos?…")`):

| File | Cases |
|---|---|
| `src/pages/SearchPage.test.tsx` | without `cargo`: two sections "Deputados(as) federais" then "Deputados(as) estaduais"; with `cargo`: one section; typing in the search box (`userEvent.type`) filters the list and updates the router location to `?q=…` with `replace` (history length unchanged); changing each `<select>` updates the URL; "Limpar filtros" appears only when a param is set, clears the URL and focuses the search input; selection-rule sentence uses `topN` and the election date from the fixture; "Ordenação atual: …" sentence follows the sort; empty state text and its "Limpar filtros" button; a section with zero results shows "Nenhum resultado neste cargo." |
| `src/components/ResultsSummary.test.tsx` | the visible count updates immediately; the `aria-live` text updates only after 500 ms (`vi.useFakeTimers()`); singular/plural wording. |
| `src/components/CandidateCard.test.tsx` | exactly one link, named by the ballot name, pointing to `/candidatos/<slug>` with `state.from` = current search; fields in the documented order; badge text = `resultLabel`; placeholder rendered (and `aria-hidden`) when `photo` is null; photo `alt=""` otherwise; the same classes for an elected and a not-elected candidate (neutrality). |
| `src/components/NativeSelect.test.tsx` | renders a native `select` associated with its `label`; forwards `value`/`onChange`. |
| `src/pages/NotFoundPage.test.tsx` | heading, the three links (`/candidatos`, `/`, `/sobre`), no `main` element of its own. |

End-to-end (Playwright, golden snapshot, both projects, shared fixture):

| File | Scenario |
|---|---|
| `e2e/search.spec.ts` | `/candidatos` lists the golden candidates in two sections; **URL round trips:** type a query, pick office/result/party/sort → the URL holds exactly those params; reload → same view; open the URL in a new page → same view; `?ordem=xyz&cargo=foo` → default view without errors. **Back navigation:** filter, scroll, open a profile, press "Voltar para a lista de candidatos" → same filters; repeat with `page.goBack()` → same filters and scroll position. **Typing history:** after typing 5 characters, one `goBack()` leaves `/candidatos`. **Keyboard:** Tab order search box → Cargo → Situação → Partido → Ordenar por → (Limpar filtros) → first card link; Enter on a card opens the profile. **Screen-reader text:** the live region's text equals the count after typing stops. Zero axe violations on the default and on a filtered view; zero console errors. On `webkit` (iPhone 13): no horizontal scroll (`document.documentElement.scrollWidth <= innerWidth`). |
| `e2e/not-found.spec.ts` | `/qualquer-coisa` → "Página não encontrada" with its three links; zero axe violations. |

## Browser check (Claude in Chrome)

Procedure: `08-testing.md`. Serve with `npm run build && npm run preview`
using a local `make ingest` snapshot (real data shapes) — fall back to the
golden snapshot if the ETL is not available.

- **Controls to exercise:** the search box (a name with accents typed without
  them, a ballot number, a party acronym); each `<select>` — Cargo, Situação,
  Partido, Ordenar por — through every option; "Limpar filtros"; a card link;
  "Voltar para a lista de candidatos" on the profile; the browser back
  button; the "Como os dados são reunidos" link.
- **URL:** after each change, read the address bar (`tabs_context_mcp` or a
  screenshot) and confirm the params; reload and confirm the same view.
- **Keyboard:** full pass from the skip link through the filters to the first
  three cards; Enter opens a profile; focus lands on the profile `h1`.
- **Widths:** 360 × 800 (one column, filters stacked) and 1440 × 900 (three
  columns, filters in one row); no horizontal scroll at 360.
- **Console:** `read_console_messages` with pattern `error|Uncaught|Warning|Content Security` → nothing.
- **Network:** `read_network_requests` → no repeated `latest.json`/`index.json` while filtering; photos only for visible cards (lazy loading).
- **GIF:** `step-19-search.gif` (type → filter → sort → open profile → back).
- **Fallback:** Playwright MCP or a throw-away script, noted in the PR.

## Done when

- [ ] `/candidatos` is reachable from the landing page's "Ver candidatos" button and from the navigation; the "Candidatos" nav item shows as current (`aria-current="page"`) on `/candidatos` and on profiles.
- [ ] Typing "joao" finds "JOÃO …"; typing a ballot number finds that candidate; typing a party acronym lists that party.
- [ ] Every filter and sort change is reflected in the URL; copying the URL into a new tab reproduces the same view.
- [ ] Back button after typing leaves the page (no keystroke history).
- [ ] Opening a profile and pressing "Voltar para a lista" (or the browser back button) returns to the same filters **and** scroll position.
- [ ] `/candidatos?ordem=xyz&cargo=foo` shows the default view, no crash.
- [ ] With no `cargo` filter, federal and state candidates appear in two separate, headed sections.
- [ ] VoiceOver/NVDA announce the result count once after typing stops, not per keystroke.
- [ ] Each card is one link in the screen reader's links list, named after the candidate.
- [ ] At 320 px wide and at 200 % zoom, no horizontal scrolling and nothing overlaps.
- [ ] A candidate without a photo shows the same placeholder as every other one without a photo.
- [ ] `make test-frontend` is green with coverage ≥ 85 % on all four metrics.
- [ ] `make test-e2e` is green on `chromium` and `webkit`.
- [ ] The browser check above is done and its GIF is referenced in the PR description.
