# 04.07 — Accessibility, responsiveness and neutrality

## Goal

Three non-functional requirements from the Modelo 4 scope, turned into
concrete rules a developer can follow and a reviewer can check by hand:

1. **Accessibility:** WCAG 2.1 level **AA** (plus the WCAG 2.2 target-size
   rule, which costs nothing extra).
2. **Responsiveness:** works from a 320 px phone to a desktop, mobile first.
3. **Neutrality:** the UI itself never favours, ranks or judges a candidate.

Verification has three layers (D15, `08-testing.md`):

1. **Automated, in CI** — axe on every route (tags `wcag2a`, `wcag2aa`,
   `wcag21a`, `wcag21aa`), reflow, target size, landmarks, heading order,
   reduced motion, token contrast and neutrality checks ("Tests for this
   step" below). They gate every deploy.
2. **Interactive, per step** — the Claude in Chrome check of each step.
3. **Manual, before launch** — screen readers, real devices and a human
   neutrality review (Part 4), recorded in `07-delivery/02-launch-checklist.md`.
   Automated tools catch roughly a third to a half of accessibility
   problems; Part 4 is what catches the rest.

---

## Part 1 — WCAG 2.1 AA: success criteria → implementation rules

| WCAG | Criterion | Rule in this codebase | Where |
|---|---|---|---|
| 1.1.1 | Non-text content | Profile photo: `alt="Foto de {nome} divulgada pelo TSE"`. Card photo: `alt=""` (name is adjacent). Placeholder silhouette and `ValueBar`: `aria-hidden="true"`. Icons from lucide: `aria-hidden="true"` unless they are the only content of a button (then the button has `aria-label`). | `CandidatePhoto`, `ValueBar`, icons |
| 1.3.1 | Info and relationships | Real elements only: `header`/`nav`/`main`/`footer`, `h1`→`h2`→`h3` with no skipped levels, `ul`/`ol` for lists, `dl` for term/value pairs, `table` with `caption` + `th scope`, `fieldset`/`legend` for the filter group, `<label htmlFor>` for every control. No `div` buttons. | all pages |
| 1.3.2 | Meaningful sequence | DOM order = visual order. No CSS `order`/`flex-direction: row-reverse` that changes reading order. | layout |
| 1.3.4 | Orientation | Nothing locked to portrait/landscape. | — |
| 1.3.5 | Identify input purpose | Search box: `type="search"`, `autocomplete="off"`. No personal-data inputs exist. | `SearchForm` |
| 1.4.1 | Use of colour | Colour never carries meaning: results are words, not green/red; bars are one neutral colour with the value in text; links are underlined, not only coloured. | global |
| 1.4.3 | Contrast (minimum) | Text ≥ 4.5:1; large text (≥ 24 px, or ≥ 18.66 px bold) ≥ 3:1. Tokens in `01-project-setup.md` Step 6; `--muted-foreground` darkened to ~7:1. | `index.css` |
| 1.4.4 | Resize text | All sizes in `rem`; viewport meta never blocks zoom; layout works at 200 % zoom. | `index.html`, CSS |
| 1.4.5 | Images of text | None. The site name and headings are text. | — |
| 1.4.10 | Reflow | At 320 CSS px (= 1280 px at 400 % zoom) no horizontal page scroll. Exception allowed by WCAG: data tables scroll **inside** their own focusable region. | grid, `Table` |
| 1.4.11 | Non-text contrast | Input/select borders (`--input`), focus ring (`--ring`) and bars (`--bar`) ≥ 3:1 against adjacent colours. shadcn's default `--input` fails this and is overridden. | `index.css` |
| 1.4.12 | Text spacing | No fixed heights on text containers; cards grow with content. Test with a text-spacing bookmarklet. | cards, badges |
| 1.4.13 | Content on hover or focus | No tooltips, popovers or hover cards. All information is visible. | — |
| 2.1.1 | Keyboard | Every action reachable with Tab/Shift+Tab/Enter/Space/arrows. Wide tables: the scroll region has `tabIndex={0}` so arrow keys scroll it. | global, `Table` |
| 2.1.2 | No keyboard trap | No modals in the MVP; nothing captures focus. | — |
| 2.1.4 | Character key shortcuts | None defined. | — |
| 2.4.1 | Bypass blocks | First focusable element: "Pular para o conteúdo" → `#conteudo`, visible on focus (`sr-only focus:not-sr-only`). | `SiteLayout` |
| 2.4.2 | Page titled | Every route renders a unique `<title>` (React 19 hoisting): "<page> – Candidatos de Videira". | every page |
| 2.4.3 | Focus order | Follows DOM order. After a route change, focus moves to the new page's `h1` (`tabIndex={-1}`). Filter changes do **not** move focus. | `SiteLayout` |
| 2.4.4 | Link purpose (in context) | No "clique aqui"/"saiba mais" alone. Card link text = candidate name; external links say what they open + "(abre em nova aba)". | cards, `ExternalLink` |
| 2.4.5 | Multiple ways | Navigation menu + search + in-page "Nesta página" lists. | layout, profile, about |
| 2.4.6 | Headings and labels | Descriptive `h2` per section; labels describe the control ("Ordenar por"). | — |
| 2.4.7 | Focus visible | Global `:focus-visible { outline: 3px solid var(--ring); outline-offset: 2px }`. Never `outline: none` without a replacement. Card focus: `has-[a:focus-visible]:outline` on the card. | `index.css`, card |
| 2.5.3 | Label in name | Visible label text is contained in the accessible name (no `aria-label` that contradicts the visible text). | controls |
| 2.5.8 (2.2) | Target size (minimum) | Controls ≥ 24×24 px; we use **44 px** high buttons, inputs and selects (`h-11`), and ≥ 8 px spacing between inline links in lists. | shadcn edits |
| 3.1.1 | Language of page | `<html lang="pt-BR">`. | `index.html` |
| 3.2.1 / 3.2.2 | On focus / on input | Focus changes nothing. Changing a filter updates the results in place (not a change of context). | `SearchForm` |
| 3.2.3 / 3.2.4 | Consistent navigation / identification | Same header, nav, footer and the same section template on every profile. | layout |
| 3.3.2 | Labels or instructions | Search label explains what can be typed: "Buscar por nome, número ou partido". | `SearchForm` |
| 4.1.2 | Name, role, value | Native elements first (`button`, `a`, `select`, `input`); Radix only where shadcn generated it. `NavLink` sets `aria-current="page"`. | global |
| 4.1.3 | Status messages | Results count in `aria-live="polite"` (debounced 500 ms); loading status in `role="status"`. | `ResultsSummary`, `LoadingPage` |

Additional rules (good practice beyond AA):

- **Reduced motion:** the global `prefers-reduced-motion` block in
  `index.css` neutralises animations and smooth scrolling. Any transition the
  app adds uses Tailwind's `motion-safe:` prefix.
- **Forced colours (Windows high contrast):** do not rely on
  `background-color` alone for boundaries — cards and inputs keep a border;
  the focus outline uses `outline`, which forced-colours mode preserves.
- **Numbers:** `tabular-nums` so columns of values line up; values formatted
  with `Intl` in pt-BR so screen readers read "três mil cento e vinte".
- **Upper-case source text** (TSE/Câmara labels) is kept verbatim; it is not
  styled with `uppercase` CSS on purpose (screen readers may spell out
  CSS-uppercased short words).
- **Language of parts:** all content is Portuguese; no `lang` switches needed.

## Part 2 — Responsiveness

Mobile first: base styles are the phone layout; breakpoints add columns.

| Breakpoint (Tailwind) | Width | Search page | Profile page |
|---|---|---|---|
| base | < 640 px | 1 card per row; filters stacked | photo above name; TOC as a list under the header; tables scroll inside their region |
| `sm` | ≥ 640 px | 2 cards per row | photo beside name |
| `md` | ≥ 768 px | filters + sort in a 4-column row | tables usually fit without scrolling |
| `lg` | ≥ 1024 px | 3 cards per row | optional sticky TOC sidebar |

Landing page (`/`): hero CTAs stacked full-width on phones and side by side
from `sm`; "O que você encontra" grid 1 → 2 (`sm`) → 4 (`lg`) columns; text
sections capped at `max-w-3xl` for readable line length at every width.
| container | max 72 rem (`max-w-6xl`) list / 56 rem (`max-w-4xl`) profile | centred, `px-4` gutters | — |

Rules:

- No fixed pixel widths on content containers; images have intrinsic size +
  `max-w-full`.
- Long unbroken strings (URLs, SHA-256, e-mail) use `break-all` /
  `overflow-wrap:anywhere`.
- Touch: 44 px targets (Part 1, 2.5.8); no hover-only affordances.
- Performance budget for slow phones: total JS ≤ ~200 KB gzipped on first
  load (React + Router + app), no web fonts beyond the one Inter variable
  file, photos lazy-loaded with explicit dimensions. Check in Lighthouse
  (mobile, throttled).

## Part 3 — Neutrality rules for the UI

These rules are as binding as the WCAG ones. A pull request that breaks one
must not be merged.

1. **One template.** Every card has the same fields in the same order; every
   profile has the same sections in the same order (only the ALESC section
   differs, and it differs by office, not by candidate).
2. **No party colours, logos, symbols or numbers styled as branding.** The
   palette is neutral grey; party is plain text (acronym – name).
3. **No adjectives or evaluations** in UI copy: no "destaque", "melhor",
   "polêmico", "ficha limpa", "atuante". Copy describes the data source and
   the fact, nothing else.
4. **Objective ordering only:** votes in Videira (default), votes in SC,
   alphabetical. The current order is always stated in text, with "A ordem não
   representa recomendação nem avaliação."
5. **No highlighting of any candidate:** no "featured", no pinned cards, no
   badges other than the factual result and selection reason, no different
   card size, no animation on a specific candidate.
6. **Results look alike.** "Eleito(a)" and "Não eleito(a)" use the same badge
   style; only the words differ.
7. **Absence is neutral.** Missing data is stated with the neutral sentences
   defined in `05-profile-page.md`, never with warning colours or icons.
8. **No cross-candidate scales.** Bars compare values within one candidate's
   table only.
9. **No embeds or third-party content** (videos, posts, follower counts) from
   candidates' social media — links only.
10. **No user-generated content:** no comments, ratings, likes or polls.
11. **Same photo treatment:** same size, crop and placeholder for all.
12. **Verbatim sources:** official labels and texts are shown as published
    (ementas, TSE labels), never rewritten or summarised by us.
13. **Nobody on the front page.** The landing page shows no candidate names,
    photos, numbers or parties.
14. **No partners or sponsors.** No institution is presented as partner,
    sponsor or supporter; public bodies appear only as data sources and in
    the "not an official site" disclaimer.

## Part 4 — Manual verification procedure (complements the automated suite)

Run on the production build (`npm run build && npm run preview`) with real
ETL data, and again on the deployed CloudFront URL before launch. Record
each item as pass/fail with the date in `07-delivery/02-launch-checklist.md`.

### 4.1 Keyboard-only pass (≈ 10 min)

Unplug/ignore the mouse. On the landing `/`, `/candidatos` with filters, a
profile with Câmara data, a profile without it, `/sobre`, and a 404 URL:

- [ ] First Tab shows "Pular para o conteúdo"; Enter jumps to the main content.
- [ ] Every control and link is reachable, in visual order, with a clearly visible focus ring.
- [ ] Filters work with keyboard only (select with arrows, clear with Enter).
- [ ] Opening a profile moves focus to its `h1`; "Voltar" returns to the same filters.
- [ ] Wide tables on a narrow window: Tab focuses the table region; arrow keys scroll it.
- [ ] No focus is lost (focus never disappears to `body`) after any action.

### 4.2 Screen readers

- **VoiceOver on macOS** (Cmd+F5, Safari): headings rotor (VO+U) lists a
  logical outline; links rotor shows one link per card named after the
  candidate; result count is announced once after typing stops; tables are
  announced with caption, rows and columns.
- **VoiceOver on iOS** (Settings → Accessibility → VoiceOver, Safari): swipe
  through a card and a profile; the "(abre em nova aba)" hint is read on
  external links.
- **NVDA on Windows** (free, nvaccess.org; Firefox or Chrome): H/Shift+H
  through headings; T through tables; the filter fieldset legend "Filtros" is
  announced when entering the first select.

### 4.3 Tools (manual runs, not CI)

- [ ] **Lighthouse** (Chrome DevTools → Lighthouse, mobile): Accessibility
      100, Best Practices ≥ 95, Performance ≥ 90 on `/`, `/candidatos` and
      one profile.
- [ ] **axe DevTools** browser extension (free tier): zero violations on each
      page type.
- [ ] Contrast spot-check of every token pair with the DevTools colour picker.
- [ ] DevTools → Rendering: emulate `prefers-reduced-motion: reduce`,
      `forced-colors: active`, and vision deficiencies (protanopia,
      deuteranopia, achromatopsia) — information is still clear.

### 4.4 Zoom and reflow

- [ ] Browser zoom 200 %: nothing overlaps or is cut off.
- [ ] Browser zoom 400 % on a 1280 px window (= 320 px): no horizontal page
      scroll; only table regions scroll.
- [ ] Text-spacing bookmarklet (line height 1.5, paragraph spacing 2, letter
      spacing 0.12, word spacing 0.16): no clipped text.

### 4.5 Real devices

- [ ] An Android phone with Chrome and an iPhone with Safari, on mobile data:
      search, open two profiles, open an external link, come back.
- [ ] A low-end Android (or Chrome DevTools "Slow 4G" + 4× CPU throttling):
      the first screen shows data in under ~3 s.
- [ ] Landscape and portrait both work.

### 4.6 Neutrality review (two people who did not write the page)

- [ ] Compare three profiles of the same office (one elected, one substitute,
      one not elected): same sections, same order, same visual weight.
- [ ] Read all UI copy aloud: no adjectives or evaluations (Part 3, rule 3).
- [ ] Check every colour on the page: none associated with a party.

## Tests for this step

This step adds the **cross-cutting** suites; per-page specs already exist
from the earlier steps.

Unit (Vitest):

| File | Cases |
|---|---|
| `src/styles/contrast.test.ts` | Reads `src/index.css`, extracts the `:root` (and, if shipped, dark) `oklch(L C H)` tokens and checks the documented pairs: `foreground`/`background` ≥ 4.5, `muted-foreground` on `background` and on `muted` ≥ 4.5, `primary-foreground`/`primary` ≥ 4.5, `input`, `ring` and `bar` against `background` and `muted` ≥ 3. For achromatic tokens (`C = 0`) relative luminance is exactly `L³`, so the ratio is `(L₁³ + 0.05) / (L₂³ + 0.05)`; chromatic tokens (only `destructive`) are skipped and checked by hand. |
| `src/test/copy-guard.test.ts` | Scans every `src/**/*.tsx` (except `components/ui/**` and tests) for banned evaluative words in string literals and JSX text — `destaque`, `melhor`, `pior`, `polêmic`, `ficha limpa`, `atuante`, `recomendad` — and for "Câmara Municipal" / "Vereadores" / "parceir". Fails with file and line. |
| `src/test/no-inline-html.test.ts` | No source file uses `dangerouslySetInnerHTML`, `eval(`, `new Function(`, or `<style>` in JSX (CSP and XSS hygiene). |

End-to-end (Playwright, both projects; routes come from `e2e/golden.ts`: landing, `/candidatos`, a filtered `/candidatos`, an incumbent profile, a non-incumbent profile, a state re-election profile, `/sobre`, a 404 URL, and the landing + `/candidatos` with `latest.json` → 404):

| File | Cases (for every route above) |
|---|---|
| `e2e/a11y.spec.ts` | **axe:** zero violations for `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa` (the shared fixture's `expectNoAxeViolations()`). **Landmarks:** exactly one `header`, `nav[aria-label="Principal"]`, `main#conteudo`, `footer`. **Headings:** exactly one `h1`; no level skipped going down (h1→h3 without h2 fails). **Skip link:** first Tab focuses "Pular para o conteúdo"; Enter moves focus into `main`. **Focus visible:** for the first 15 Tab stops, the focused element's computed `outline-style` is not `none` (or it has a visible box-shadow ring). **Reflow:** at viewport 320 × 640, `document.documentElement.scrollWidth <= window.innerWidth`. **Target size:** every `button`, `input`, `select` and every link that is not inside running text has a bounding box ≥ 24 × 24 px (WCAG 2.5.8; inline links in paragraphs are exempt by the criterion). **Reduced motion:** with `page.emulateMedia({ reducedMotion: "reduce" })`, computed `transition-duration` and `animation-duration` of a card, a button and the progress bar are ≤ 0.01 ms. **Forced colours:** with `emulateMedia({ forcedColors: "active" })`, inputs keep a visible border and the focus outline is present. **Zoom:** `page.evaluate(() => document.body.style.zoom = "2")` on desktop → no element overlaps the header (bounding boxes) — complementary to the manual 200 % check. |
| `e2e/neutrality.spec.ts` | **Same template:** the ordered list of `h2` texts is identical for every golden profile of the same office (and differs between offices only by the ALESC section). **Same card:** on `/candidatos`, every card root and every result badge has the same `className` and computed colours, whatever the result. **Nobody on the front page:** no golden ballot name, number or party acronym appears in the landing text. **No partner:** "Câmara Municipal", "Vereadores", "parceir" absent from every route's text. **Banned words:** none of the copy-guard list in any route's rendered text (catches strings that come from data too). **Sort disclosure:** the "A ordem não representa recomendação nem avaliação" sentence is present on `/candidatos` for every `ordem`. |

## Browser check (Claude in Chrome)

Procedure: `08-testing.md`. Serve with `npm run build && npm run preview`
(local snapshot). This is the widest check of the project: run it on every
route listed above.

- **Keyboard:** the full Part 4.1 pass on every route, driven with
  `computer` key presses (Tab, Shift+Tab, Enter, Space, arrows); take a
  screenshot at each focus stop that looks doubtful.
- **Widths:** `resize_window` to 320 × 640, 360 × 800, 768 × 1024 and
  1440 × 900 on every route; at each width confirm no horizontal scroll with
  `javascript_tool` (`document.documentElement.scrollWidth <= innerWidth`) and
  no overlap.
- **Zoom:** with `javascript_tool`, set `document.body.style.zoom = "2"` on
  desktop width and look for clipped or overlapping text; reset afterwards.
- **Neutrality read-through:** open three profiles of the same office (one
  elected, one substitute, one not elected) side by side in turn and compare
  section lists, badge styles and visual weight.
- **Console:** `read_console_messages` with pattern `error|Uncaught|Warning|Content Security` → nothing on any route.
- **Network:** `read_network_requests` → only same-origin requests on every route.
- **Not possible in Claude in Chrome** — use Playwright (MCP or script) and
  say so in the PR: `prefers-reduced-motion` and `forced-colors` emulation,
  CPU/network throttling.
- **GIF:** `step-23-a11y.gif` (keyboard pass on `/candidatos` → a profile, then the 320 px resize).
- **Fallback:** Playwright MCP or a throw-away script for everything, noted in the PR.

## Done when

- [ ] Every row of Part 1 has been implemented as described.
- [ ] Parts 4.1–4.6 all pass on the deployed site, with results recorded in the launch checklist.
- [ ] A reviewer who did not build the page confirms the fourteen neutrality rules.
- [ ] `make test-frontend` is green with coverage ≥ 85 % on all four metrics, including the contrast, copy-guard and no-inline-html tests.
- [ ] `make test-e2e` is green on `chromium` and `webkit`, including `a11y.spec.ts` and `neutrality.spec.ts` on every route.
- [ ] The browser check above is done and its GIF is referenced in the PR description.
