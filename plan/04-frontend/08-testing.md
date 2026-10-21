# 04.08 — Frontend testing

## Goal

How the frontend is tested, concretely. The **contract** (gates, pinned
versions, coverage rule, synthetic data, golden snapshot) is
`01-foundation/05-testing-strategy.md` and decision D15; this file is the
frontend's implementation of it. Every page doc (`01`–`07`) lists the tests
of its own step under "Tests for this step" and its interactive check under
"Browser check (Claude in Chrome)"; the shared machinery they use is defined
here.

Rules that never bend:

- tests are written **in the same PR** as the code they cover;
- **no test touches the network** or a real government site — data comes
  from synthetic builders (unit/component) or the committed golden snapshot
  (end-to-end);
- `make test-frontend` (coverage ≥ 85 % on lines, statements, functions and
  branches) and `make test-e2e` (Chromium + WebKit) must be green before a PR
  is opened, and CI re-runs both before any deploy;
- every step that changes the UI gets the interactive browser check before
  its PR.

## The pyramid

| Layer | Tools (pinned) | Scope | Speed | Where the data comes from |
|---|---|---|---|---|
| Unit | Vitest 5.0.5 | pure functions: formatters, text normalisation, URL params, filter/sort, safe URLs, fetch client, loaders, error mapping, token contrast, source-code guards | ms | literals and `src/test/fixtures.ts` builders; `fetch` stubbed |
| Component | Testing Library (`react` 16.3.3, `dom` 10.4.2, `user-event` 14.6.8, `jest-dom` 7.0.1) on jsdom 30.1.3 | each page and component rendered through the **real route table** with `createMemoryRouter`: sections, order, empty states, links, URL updates, focus | tens of ms | builders + stubbed `fetch` |
| End-to-end | `@playwright/test` 1.64.1 + `@axe-core/playwright` 4.14.0 | the **production build** served by `vite preview`, in Chromium (Desktop Chrome) and WebKit (iPhone 13): real navigation, history, downloads, layout, accessibility | seconds | the backend's golden snapshot at `/data`; per-test overrides with `page.route` |
| Interactive | Claude in Chrome (Playwright MCP as fallback) | what nobody thought to automate: look and feel, odd interactions, real console/network | minutes, per step | local `make ingest` snapshot or the golden snapshot |

Most cases live in unit and component tests; end-to-end specs cover each
route's main flows, the failure scenarios and the cross-cutting checks
(axe, reflow, neutrality).

## Commands

| Command | What |
|---|---|
| `make test-frontend` | `cd frontend && npm run test:coverage` — Vitest once, v8 coverage, fails under 85 % on any metric |
| `make test-e2e` | `cd frontend && npm run test:e2e` — builds, starts `vite preview` on port 4317 with `E2E_DATA_DIR=../backend/testdata/golden/data`, runs both projects |
| `npm run test:watch` | Vitest watch mode while coding |
| `npm run test:e2e:ui` | Playwright UI mode to debug a spec |
| `npx playwright show-trace test-results/<…>/trace.zip` | open the trace of a failed test (traces and screenshots are kept on failure) |
| `npx playwright install --with-deps chromium webkit` | once per machine (CI does it in the `e2e` job) |

Coverage report: `frontend/coverage/index.html`. Excluded (and only these,
per the contract): `src/components/ui/**`, `src/main.tsx`, `**/*.d.ts`, test
files and `src/test/**` helpers.

---

## Files

```
frontend/
├── src/test/
│   ├── setup.ts          # jest-dom matchers, cleanup, scroll stubs, memo reset (01-project-setup.md, Step 3)
│   ├── fixtures.ts       # synthetic builders typed by the JSON contract
│   ├── mock-fetch.ts     # mockFetch() + response helpers
│   ├── render.tsx        # renderRoute()
│   ├── copy-guard.test.ts
│   └── no-inline-html.test.ts
├── e2e/
│   ├── fixtures.ts       # the shared Playwright fixture: console guard + axe + data overrides
│   ├── golden.ts         # reads the golden snapshot from disk; findCandidate()
│   └── *.spec.ts         # smoke, data-middleware, data-states, landing, search, not-found,
│                         # profile, about, a11y, neutrality
testdata/
└── textnorm-vectors.json # repository root — shared with the Go suite
```

## Synthetic builders (`src/test/fixtures.ts`)

Typed against `src/data/types.ts`, so a contract change that the frontend
has not absorbed fails `tsc` in the test code too. Values are invented and
obviously fake (public repository, D10): names like `CANDIDATA TESTE UM`,
party `PTX` (number 99), ballot numbers `9901`, `99001`, version
`20990101T000000Z`.

```ts
import type { CandidateProfile, CandidateSummary, DataIndex, Source, SourceId } from "@/data/types";

const VERSION = "20990101T000000Z";

export function makeSummary(over: Partial<CandidateSummary> = {}): CandidateSummary {
  const base: CandidateSummary = {
    slug: "candidata-teste-um-9901",
    tseId: "990000000001",
    ballotName: "CANDIDATA TESTE UM",
    number: "9901",
    office: "federal_deputy",
    party: { number: 99, acronym: "PTX", name: "PARTIDO TESTE X" },
    federation: null,
    result: { code: "substitute", tseLabel: "SUPLENTE" },
    votes: { municipality: 1200, municipalityShare: 0.05, municipalityRank: 3, state: 45000 },
    bornInMunicipality: false,
    selectionReasons: ["top_votes"],
    hasChamberRecord: false,
    runningForReelection: false,
    photo: null,
    searchText: "candidata teste um candidata teste um da silva 9901 ptx",
  };
  return { ...base, ...over };
}

export function makeSource(id: SourceId, over: Partial<Source> = {}): Source { /* publisher, name, URLs, dates */ }
export function makeIndex(over: Partial<DataIndex> = {}): DataIndex { /* 11 sources, 2 candidates, topN 20 */ }
export function makeProfile(over: Partial<CandidateProfile> = {}): CandidateProfile { /* chamber: null etc. */ }
export function makeChamber(over = {}) { /* bills, votes, CEAP */ }
export function makeAmendments(over = {}) {
  /* two amendments; beneficiaries include a legal entity and
     { name: "Pessoas físicas (valores agregados)", received: … };
     totals.receivedInMunicipality = sum of beneficiaries */
}
```

Builders accept overrides so each test states only what it cares about:
`makeProfile({ chamber: null, amendments: null })`.

## Mocking `fetch` (`src/test/mock-fetch.ts`)

```ts
import { vi } from "vitest";

type Reply = Response | Promise<Response> | (() => Response | Promise<Response>);

export const json = (body: unknown, status = 200) =>
  new Response(JSON.stringify(body), {
    status,
    headers: { "Content-Type": "application/json; charset=utf-8" },
  });
export const status = (code: number) => new Response("", { status: code });
export const html = () => new Response("<!doctype html>", { headers: { "Content-Type": "text/html" } });
export const never = () => new Promise<Response>(() => {});            // pending forever
export const networkError = () => Promise.reject(new TypeError("Failed to fetch"));

/**
 * Stubs global fetch with a path → reply table. Unknown paths answer 404,
 * so a test never silently hits something it did not declare.
 * Returns the spy: expect(spy).toHaveBeenCalledTimes(…), inspect URLs and init.
 */
export function mockFetch(routes: Record<string, Reply>) {
  const spy = vi.fn(async (input: RequestInfo | URL, _init?: RequestInit) => {
    const path = new URL(String(input), "http://test.local").pathname;
    const reply = routes[path];
    if (reply === undefined) return status(404);
    return typeof reply === "function" ? reply() : reply;
  });
  vi.stubGlobal("fetch", spy);
  return spy;
}

/** The happy path for a given index and profiles. */
export function mockSnapshot(index: DataIndex, profiles: CandidateProfile[] = []) {
  const routes: Record<string, Reply> = {
    "/data/latest.json": () => json({ schemaVersion: 1, version: index.version, generatedAt: index.generatedAt }),
    [`/data/${index.version}/index.json`]: () => json(index),
  };
  for (const p of profiles) {
    routes[`/data/${index.version}/candidates/${p.summary.slug}.json`] = () => json(p);
  }
  return mockFetch(routes);
}
```

Replies are functions so every call gets a **fresh** `Response` (a body can
be read only once). `setup.ts` calls `vi.unstubAllGlobals()` and
`resetDataClientForTests()` after each test, so the index memo never leaks
between tests.

## Rendering routes (`src/test/render.tsx`)

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { createMemoryRouter, RouterProvider } from "react-router";
import { routes } from "@/router";

/** Renders the real route table at `path`; waits until the page's h1 is there. */
export async function renderRoute(path: string, opts: { state?: unknown; waitForH1?: boolean } = {}) {
  const url = new URL(path, "http://test.local");
  const router = createMemoryRouter(routes, {
    initialEntries: [{ pathname: url.pathname, search: url.search, hash: url.hash, state: opts.state ?? null }],
  });
  const user = userEvent.setup();
  const utils = render(<RouterProvider router={router} />);
  if (opts.waitForH1 !== false) await screen.findByRole("heading", { level: 1 });
  return { ...utils, router, user };
}
```

- Assert navigation with `router.state.location` (`pathname`, `search`,
  `state`), and history behaviour with `router.state.historyAction`
  (`"REPLACE"` for filter changes).
- Use `user` (user-event) for typing, clicking and `user.tab()`; never fire
  raw DOM events for interactions.
- Prefer queries by role and accessible name (`getByRole("link", { name:
  "Ver candidatos" })`) — they double as accessibility checks.
- Debounced code (`ResultsSummary`): `vi.useFakeTimers({ shouldAdvanceTime:
  true })` + `vi.advanceTimersByTime(500)`; `setup.ts` restores real timers.

## Shared text-normalisation vectors

`testdata/textnorm-vectors.json` at the **repository root** is read by both
`src/lib/text.test.ts` and the Go `internal/textnorm` tests, so the two
implementations cannot drift. Format (proposed — the file is shared, agree it
with the backend side before the first commit):

```json
[
  { "input": "  CANDIDATA  Teste   Um ", "expected": "candidata teste um" },
  { "input": "Conceição São João", "expected": "conceicao sao joao" },
  { "input": "ÁÉÍÓÚ âêô ãõ à ç Ü", "expected": "aeiou aeo ao a c u" },
  { "input": "PTX 9901", "expected": "ptx 9901" },
  { "input": "tab\there\nnewline", "expected": "tab here newline" },
  { "input": "", "expected": "" }
]
```

```ts
// src/lib/text.test.ts (excerpt)
import vectors from "../../../testdata/textnorm-vectors.json";
it.each(vectors)("normalize($input)", ({ input, expected }) => {
  expect(normalize(input)).toBe(expected);
});
```

(`resolveJsonModule` is on; Vitest reads files outside `frontend/` without
extra config. If `tsc` complains that the file is outside `include`, read it
with `fs.readFileSync` instead.)

---

## End-to-end

### Data for end-to-end tests

1. **Default — the golden snapshot.** `playwright.config.ts` starts the
   preview server with `E2E_DATA_DIR=../backend/testdata/golden/data`; the
   Vite plugin's `configurePreviewServer` middleware serves that folder at
   `/data` with real 404s (`01-project-setup.md`, Step 3). The snapshot is
   produced and committed by the backend's pipeline integration test
   (`go test ./internal/pipeline -run TestPipeline -update`), uses the fixed
   version `20990101T000000Z`, and contains only synthetic people. When the
   backend changes the JSON, the golden snapshot changes in the same PR and
   these specs run against it — that is the contract test.
2. **Before the golden snapshot exists** (early steps), every `/data` request
   answers 404 — the "no data yet" state. Specs that need data are added from
   the step where the snapshot exists (see each page doc).
3. **Per-test overrides** with `page.route`, exposed by the shared fixture
   (below): no data, schema mismatch, network failure, slow data.

`e2e/golden.ts`:

```ts
import { readFileSync } from "node:fs";
import path from "node:path";
import type { CandidateProfile, CandidateSummary, DataIndex, LatestPointer } from "../src/data/types";

const ROOT = path.resolve(import.meta.dirname, "../../backend/testdata/golden/data");
const latest = JSON.parse(readFileSync(path.join(ROOT, "latest.json"), "utf8")) as LatestPointer;
export const goldenIndex = JSON.parse(
  readFileSync(path.join(ROOT, latest.version, "index.json"), "utf8"),
) as DataIndex;

export function goldenProfile(slug: string): CandidateProfile {
  return JSON.parse(readFileSync(path.join(ROOT, latest.version, "candidates", `${slug}.json`), "utf8"));
}

/** Pick candidates by property, never by hard-coded slug. Throws if the golden lacks one. */
export function findCandidate(pred: (c: CandidateSummary) => boolean): CandidateSummary {
  const c = goldenIndex.candidates.find(pred);
  if (!c) throw new Error("Golden snapshot has no candidate matching this predicate — extend the backend fixtures.");
  return c;
}
```

The golden snapshot must contain, at minimum (requirement on the backend
fixtures): an incumbent federal deputy with bills, roll-call votes, CEAP and
amendments (one amendment with a legal-entity beneficiary and one with
"Pessoas físicas (valores agregados)"); a non-incumbent federal candidate; a
state candidate elected in 2022 and one not elected in 2022; a candidate
without a photo; a candidate with `assets.declared === false`; a candidate
born in Videira outside the top N; a federation.

### The shared fixture (`e2e/fixtures.ts`)

Every spec imports `test` and `expect` from here, never from
`@playwright/test` directly.

```ts
import AxeBuilder from "@axe-core/playwright";
import { test as base, expect, type Page } from "@playwright/test";

const WCAG_TAGS = ["wcag2a", "wcag2aa", "wcag21a", "wcag21aa"];

type Fixtures = {
  /** Console errors matching these patterns are tolerated (e.g. the expected 404 in no-data tests). */
  allowedConsoleErrors: RegExp[];
  /** Auto fixture: collects console errors and page errors, fails the test at the end. */
  consoleGuard: void;
  expectNoAxeViolations: (opts?: { include?: string }) => Promise<void>;
  data: {
    none: () => Promise<void>;            // latest.json → 404
    schemaMismatch: () => Promise<void>;  // latest.json → { schemaVersion: 2, … }
    offline: () => Promise<void>;         // every /data request aborted
    slow: (ms: number) => Promise<void>;  // latest.json delayed
    reset: () => Promise<void>;           // remove the overrides
  };
};

export const test = base.extend<Fixtures>({
  allowedConsoleErrors: [[], { option: true }],

  consoleGuard: [
    async ({ page, allowedConsoleErrors }, use) => {
      const errors: string[] = [];
      page.on("console", (msg) => {
        if (msg.type() === "error" && !allowedConsoleErrors.some((re) => re.test(msg.text()))) {
          errors.push(msg.text());
        }
      });
      page.on("pageerror", (err) => errors.push(`pageerror: ${err.message}`)); // never allowed
      await use();
      expect(errors, "console errors").toEqual([]);
    },
    { auto: true },
  ],

  expectNoAxeViolations: async ({ page }, use) => {
    await use(async (opts) => {
      let builder = new AxeBuilder({ page }).withTags(WCAG_TAGS);
      if (opts?.include) builder = builder.include(opts.include);
      const { violations } = await builder.analyze();
      expect(
        violations.map((v) => `${v.id}: ${v.nodes.map((n) => n.target.join(" ")).join(", ")}`),
        "axe violations",
      ).toEqual([]);
    });
  },

  data: async ({ page }, use) => {
    const latest = "**/data/latest.json";
    await use({
      none: () => page.route(latest, (r) => r.fulfill({ status: 404, body: "" })),
      schemaMismatch: () =>
        page.route(latest, (r) =>
          r.fulfill({ json: { schemaVersion: 2, version: "20990101T000000Z", generatedAt: "2099-01-01T00:00:00Z" } }),
        ),
      offline: () => page.route("**/data/**", (r) => r.abort("internetdisconnected")),
      slow: (ms) =>
        page.route(latest, async (r) => {
          await new Promise((res) => setTimeout(res, ms));
          await r.continue();
        }),
      reset: () => page.unrouteAll({ behavior: "ignoreErrors" }),
    });
  },
});

export { expect };

/** Tolerate only the resource-load message browsers log for an expected failed request. */
export const EXPECTED_DATA_FAILURE = [/Failed to load resource/i];
```

Usage in a failure scenario:

```ts
test.describe("no data yet", () => {
  test.use({ allowedConsoleErrors: EXPECTED_DATA_FAILURE });
  test("landing still renders", async ({ page, data, expectNoAxeViolations }) => {
    await data.none();
    await page.goto("/");
    await expect(page.getByText("Dados em preparação")).toBeVisible();
    await expectNoAxeViolations();
  });
});
```

`pageerror` (an uncaught exception) is never tolerated, in any scenario.

### Projects

| Project | Device | Why |
|---|---|---|
| `chromium` | Desktop Chrome | the most common desktop engine; keyboard and layout at desktop width |
| `webkit` | iPhone 13 (390 × 844, touch, mobile UA) | Safari's engine at phone width — catches iOS-only layout and date/number formatting differences |

A spec that only makes sense at one width checks `testInfo.project.name` and
skips the other (`test.skip(project === "chromium", "phone-only check")`).

### What every route's spec must include

- it loads (main `h1` visible) and its key interactions work;
- keyboard: the documented Tab order reaches every control, with a visible
  focus outline;
- `await expectNoAxeViolations()` after the page settles and again after the
  main interaction (e.g. after filtering);
- zero console errors (automatic, via `consoleGuard`);
- the failure scenarios that apply to it: no data (`data.none()`), schema
  mismatch (`data.schemaMismatch()`).

### CI

The `frontend` job runs `npm ci`, `npm run typecheck`, `npm run lint`,
`npm run test:coverage`; the `e2e` job runs `npm ci`,
`npx playwright install --with-deps chromium webkit`, `npm run test:e2e` and
uploads `playwright-report/` (and `test-results/` traces) as artifacts on
failure. Workflow details are owned by `06-ci-cd/01-ci.md`.

---

## Claude in Chrome verification

Run this **before opening the PR of any step that changes the frontend**.
It is written as a checklist an agent (or a person driving the agent) can
follow literally. The step-specific controls, widths and GIF name are in the
step's own "Browser check (Claude in Chrome)" section.

### 0. Prepare

- [ ] `cd frontend && npm run build && npm run preview` → note the URL
      (`http://localhost:4173/`). Data: a local `make ingest` snapshot, or
      the golden snapshot (`E2E_DATA_DIR=../backend/testdata/golden/data npm run preview`),
      or an empty folder for no-data checks (`E2E_DATA_DIR=/tmp/empty-data`).
- [ ] Check it answers: `curl -s -o /dev/null -w "%{http_code}" http://localhost:4173/` → `200`.

### 1. Load the browser tools — one call

- [ ] One `ToolSearch` call with all the tools the check needs:

```
select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__find,mcp__claude-in-chrome__form_input,mcp__claude-in-chrome__resize_window,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__read_console_messages,mcp__claude-in-chrome__read_network_requests,mcp__claude-in-chrome__gif_creator,mcp__claude-in-chrome__tabs_close_mcp
```

### 2. Open a fresh tab

- [ ] `tabs_context_mcp` **first**, to see the current browser state. Never
      reuse a tab id from another session.
- [ ] `tabs_create_mcp` → a new tab; use only that tab id from here on.

### 3. Start recording

- [ ] `gif_creator` → start recording; capture a couple of extra frames
      before the first action.

### 4. Navigate and exercise

- [ ] `navigate` to the preview URL + the route under test.
- [ ] For **each control listed in the step's browser-check section**:
      locate it (`find` or `read_page`), act on it (`computer` click/type/key;
      `form_input` for `<select>`), then confirm the visible result
      (`read_page`, or a `computer` screenshot) and, when the step says so,
      the URL (`tabs_context_mcp`).
- [ ] External links: confirm a new tab opens, then close that tab with
      `tabs_close_mcp`; do not browse the external site.
- [ ] Never activate `mailto:` links or anything that opens a native dialog —
      inspect their `href` with `read_page`/`javascript_tool` instead.

### 5. Keyboard pass

- [ ] From the top of the page, press `Tab` repeatedly with `computer`
      (key action); after each stop take a screenshot or read
      `document.activeElement` with `javascript_tool`; confirm the order
      matches the step's list and the focus ring is visible.
- [ ] Activate the main control with `Enter`/`Space`; for selects use the
      arrow keys.

### 6. Widths

- [ ] `resize_window` to **360 × 800**; check
      `javascript_tool`: `document.documentElement.scrollWidth <= window.innerWidth`
      → `true`; screenshot.
- [ ] `resize_window` to **1440 × 900**; same check; screenshot.
- [ ] Extra widths only when the step asks (e.g. 320 and 768 in the
      accessibility step).

### 7. Console and network

- [ ] `read_console_messages` with a **pattern** —
      `error|Uncaught|Warning|Content Security|Refused` — expect nothing
      (except the expected `latest.json` 404 in a no-data run).
- [ ] `read_network_requests` — every request same-origin; no unexpected
      4xx/5xx; request counts as the step expects (e.g. one `latest.json`
      and one `index.json` per visit).

### 8. Finish

- [ ] `gif_creator` → stop and export with the step's name
      (`step-NN-<topic>.gif`, e.g. `step-19-search.gif`), with a couple of
      extra frames after the last action.
- [ ] `tabs_close_mcp` the test tab.
- [ ] Stop the preview server.

### 9. Report in the PR description

```
### Browser check (Claude in Chrome)
- Data: golden snapshot | local snapshot | empty (no-data)
- Controls exercised: … (all ✅ / list failures and fixes)
- Keyboard pass: ✅
- 360 px / 1440 px: ✅ no horizontal scroll
- Console: 0 errors · Network: same-origin only, no unexpected 4xx/5xx
- GIF: step-19-search.gif
- Fallback used: none | Playwright MCP | script (why)
```

### When something fails

- A defect found here is fixed **in the same step**; anything worth guarding
  against permanently becomes a Playwright spec in the same PR.
- If a browser tool fails two or three times in a row, or the extension does
  not respond, stop retrying and use the fallback.

### Fallback: Playwright

When Claude in Chrome is not connected, or a check needs scripting
(throttling, `prefers-reduced-motion`/`forced-colors` emulation, many
repetitions):

- the **Playwright MCP** server (`@playwright/mcp`, run `--isolated`), with
  the same checklist: navigate, click/type, snapshot the accessibility tree,
  resize, read console and network; or
- a throw-away script (not committed) using the already-installed
  `@playwright/test` — e.g. `npx playwright test e2e/search.spec.ts --headed
  --project=chromium` for a visual run of the existing spec, or a short
  `.mjs` with `chromium.launch({ headless: false })` and
  `page.video()`/screenshots instead of a GIF.

State in the PR that the fallback was used, and why.

## Done when

- [ ] `src/test/{setup.ts,fixtures.ts,mock-fetch.ts,render.tsx}`, `e2e/fixtures.ts` and `e2e/golden.ts` exist as described and are used by every test.
- [ ] `testdata/textnorm-vectors.json` exists at the repository root and both the Go and the TypeScript suites read it.
- [ ] `make test-frontend` is green with coverage ≥ 85 % on lines, statements, functions and branches.
- [ ] `make test-e2e` is green on `chromium` and `webkit`, with zero axe violations and zero console errors on every route.
- [ ] Every frontend PR description contains the browser-check report and its GIF name.
