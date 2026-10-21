# 04.02 — Data access layer

## Goal

One small, typed layer that every page uses to read the snapshot published by
the ETL:

- the **types** are a verbatim copy of `01-foundation/04-json-contract.md`;
- `latest.json` is always revalidated; versioned files are fetched once and
  cached by the browser forever (they are immutable);
- the index is fetched **once per page load** and shared by all routes,
  including the landing page's live figures;
- the **landing page (`/`) never depends on data**: it renders fully, and
  works, before any data snapshot exists (see `03-landing-page.md`);
- the data pages (`/candidatos`, `/candidatos/:slug`, `/sobre`) use React
  Router **loaders** that fetch before rendering, so they never render empty;
- failures become one of three typed errors with a dedicated screen;
- numbers, money, percentages and dates are formatted in **pt-BR** in one place;
- search normalisation is **identical** to the Go `internal/textnorm`.

## Routes

| Path | Page | Data | Doc |
|---|---|---|---|
| `/` | `LandingPage` | optional, non-blocking (live figures only) | `03-landing-page.md` |
| `/candidatos` | `SearchPage` | index (required) | `04-search-page.md` |
| `/candidatos/:slug` | `CandidatePage` | index + profile (required) | `05-profile-page.md` |
| `/sobre` | `AboutPage` | index (required) | `06-about-page.md` |
| `*` | `NotFoundPage` | none | `04-search-page.md`, Step 10 |

URL paths are Portuguese because users see them. Component and file names
stay English.

## Files to create

```
frontend/src/
├── router.tsx
├── data/
│   ├── types.ts          # the contract (copy)
│   ├── errors.ts         # DataError
│   ├── client.ts         # fetchLatest, getIndex (memo), getIndexOrNull, getProfile, dataUrl
│   ├── loaders.ts        # landingLoader, indexLoader, candidateLoader
│   ├── hooks.ts          # useDataIndex(), useOptionalDataIndex()
│   ├── labels.ts         # pt-BR labels for codes (office, result, reasons)
│   └── format.ts         # Intl formatters
├── pages/
│   ├── ErrorPage.tsx     # data-route error boundary (inside the layout)
│   ├── RootErrorPage.tsx # last-resort boundary (layout itself failed)
│   └── LoadingPage.tsx   # data-route HydrateFallback
└── lib/
    └── text.ts           # normalize(), matchesQuery()
```

---

## Step 1 — `src/data/types.ts`

Copy the whole TypeScript block from `01-foundation/04-json-contract.md`
**verbatim**, and add one header comment:

```ts
/**
 * Mirror of plan/01-foundation/04-json-contract.md (schemaVersion 1).
 * Change the contract first, then this file and backend/internal/export,
 * in the same pull request.
 */
```

Do not "improve" names or make fields optional here: the backend always
writes every key (absent = `null`, lists = `[]`), so the types say exactly
that.

## Step 2 — `src/data/errors.ts`

```ts
export type DataErrorKind = "network" | "unavailable" | "not_found";

/** Every failure of the data layer is one of these three. */
export class DataError extends Error {
  readonly kind: DataErrorKind;
  constructor(kind: DataErrorKind, message: string) {
    super(message);
    this.name = "DataError";
    this.kind = kind;
  }
}

export function isDataError(e: unknown): e is DataError {
  return e instanceof DataError;
}
```

| Kind | When | Screen on data pages | Landing page |
|---|---|---|---|
| `network` | `fetch` rejects (offline, DNS), HTTP 5xx, or the body is not JSON | `ErrorPage`: "Não foi possível carregar os dados." + "Tentar novamente" | strip shows "Dados em preparação"; page otherwise unchanged |
| `unavailable` | no snapshot published yet (`latest.json` 404/403), `schemaVersion !== 1`, or `index.json` missing/malformed | `ErrorPage`: "Os dados estão em preparação." + "Tentar novamente" | same |
| `not_found` | the profile JSON returns 404, or the slug is not in the index | `NotFoundPage` | — |

## Step 3 — `src/data/client.ts`

```ts
import type { CandidateProfile, DataIndex, LatestPointer } from "./types";
import { DataError } from "./errors";

const DATA_BASE = "/data";
const SUPPORTED_SCHEMA = 1;

/** Builds a URL inside a published version: dataUrl(v, "photos/123.jpg"). */
export function dataUrl(version: string, relativePath: string): string {
  return `${DATA_BASE}/${version}/${relativePath}`;
}

async function getJson<T>(url: string, init?: RequestInit): Promise<T> {
  let res: Response;
  try {
    res = await fetch(url, { ...init, headers: { Accept: "application/json" } });
  } catch {
    throw new DataError("network", `Request failed: ${url}`);
  }
  if (res.status === 404 || res.status === 403) {
    // S3 answers 403 for a missing key when the caller cannot list the bucket.
    throw new DataError("not_found", `Not found: ${url}`);
  }
  if (!res.ok) throw new DataError("network", `HTTP ${res.status}: ${url}`);
  // Guard against a misconfigured SPA fallback returning index.html with 200.
  const type = res.headers.get("Content-Type") ?? "";
  if (!type.includes("application/json")) {
    throw new DataError("network", `Expected JSON, got "${type}": ${url}`);
  }
  try {
    return (await res.json()) as T;
  } catch {
    throw new DataError("network", `Invalid JSON: ${url}`);
  }
}

/** A missing pointer or index means "no snapshot yet", not "page not found". */
function asUnavailable(e: unknown): never {
  if (e instanceof DataError && e.kind === "not_found") {
    throw new DataError("unavailable", e.message);
  }
  throw e;
}

/**
 * latest.json is the only mutable file. `cache: "no-cache"` makes the browser
 * revalidate it on every call (a cheap conditional request answered with 304
 * when unchanged). "no-store" would also work but throws away the ETag and
 * always downloads the body; "no-cache" gives the same freshness for less.
 * CloudFront itself may serve it up to 60 s stale (Cache-Control max-age=60).
 */
export async function fetchLatest(): Promise<LatestPointer> {
  const latest = await getJson<LatestPointer>(`${DATA_BASE}/latest.json`, {
    cache: "no-cache",
  }).catch(asUnavailable);
  if (latest.schemaVersion !== SUPPORTED_SCHEMA || typeof latest.version !== "string") {
    throw new DataError("unavailable", `Unsupported schemaVersion ${String(latest.schemaVersion)}`);
  }
  return latest;
}

/** Memoised for the lifetime of the page; reset on failure so a retry refetches. */
let indexPromise: Promise<DataIndex> | null = null;

export function getIndex(): Promise<DataIndex> {
  if (indexPromise === null) {
    indexPromise = (async () => {
      const latest = await fetchLatest();
      // Versioned files are immutable: default caching (the CDN sends
      // max-age=31536000, immutable), so a revisit costs nothing.
      const index = await getJson<DataIndex>(dataUrl(latest.version, "index.json")).catch(
        asUnavailable,
      );
      if (index.schemaVersion !== SUPPORTED_SCHEMA || !Array.isArray(index.candidates)) {
        throw new DataError("unavailable", "index.json does not match schemaVersion 1");
      }
      return index;
    })();
    indexPromise.catch(() => {
      indexPromise = null;
    });
  }
  return indexPromise;
}

/**
 * For the landing page only: never rejects. Shares the same memo, so going
 * from the landing to /candidatos does not download the index twice.
 */
export function getIndexOrNull(): Promise<DataIndex | null> {
  return getIndex().catch(() => null);
}

export async function getProfile(version: string, slug: string): Promise<CandidateProfile> {
  const profile = await getJson<CandidateProfile>(
    dataUrl(version, `candidates/${encodeURIComponent(slug)}.json`),
  );
  if (profile.schemaVersion !== SUPPORTED_SCHEMA || profile.version !== version) {
    throw new DataError("unavailable", `Profile ${slug} does not match version ${version}`);
  }
  return profile;
}
```

Why the profile URL uses the **index's** version: a visitor who opened the
site before a refresh keeps reading one consistent snapshot. Old versions stay
on S3 (the publish step keeps the last 5), so their profile links keep working
for weeks. A full page reload picks up the new `latest.json`.

No runtime schema library (zod etc.): the backend is ours and validated
before publishing (`03-backend-etl/08-validate-and-publish.md`). The checks
above catch the only realistic failures — no snapshot yet, a version bump, or
a wrong file.

## Step 4 — Router and loaders

### Route tree

```
root  "/"            SiteLayout          no loader → never blocks, never fails on data
├── index            LandingPage         landingLoader: returns an UNAWAITED promise (non-blocking)
├── id "data" (pathless layout)          indexLoader: awaits the index (throws on failure)
│   ├── "candidatos"         SearchPage
│   ├── "candidatos/:slug"   CandidatePage   candidateLoader
│   └── "sobre"              AboutPage
└── "*"              NotFoundPage
```

The root route has **no loader**. That is the property that lets the landing
page (and the layout: header, nav, footer) render before any data exists.
All data dependencies sit below the pathless `data` route, whose error
boundary and loading fallback render **inside** the layout.

`src/data/loaders.ts`:

```ts
import type { LoaderFunctionArgs } from "react-router";
import { getIndex, getIndexOrNull, getProfile } from "./client";
import { DataError } from "./errors";
import type { CandidateProfile, DataIndex } from "./types";

/**
 * Landing: returns the promise WITHOUT awaiting it. The route renders at
 * once and <Await> fills the figures in when (if) they arrive. The promise
 * never rejects (getIndexOrNull), so the landing cannot fail on data.
 */
export interface LandingLoaderData {
  index: Promise<DataIndex | null>;
}
export function landingLoader(): LandingLoaderData {
  return { index: getIndexOrNull() };
}

/** Data pages: block until the index is there; failures go to ErrorPage. */
export async function indexLoader(): Promise<DataIndex> {
  return getIndex();
}

export interface CandidateLoaderData {
  index: DataIndex;
  profile: CandidateProfile;
}

export async function candidateLoader({ params }: LoaderFunctionArgs): Promise<CandidateLoaderData> {
  const slug = params["slug"] ?? "";
  const index = await getIndex();
  // Unknown slug: answer from the index without a network round-trip.
  if (!index.candidates.some((c) => c.slug === slug)) {
    throw new DataError("not_found", `Unknown candidate ${slug}`);
  }
  const profile = await getProfile(index.version, slug);
  return { index, profile };
}
```

`src/router.tsx`:

```tsx
import { createBrowserRouter } from "react-router";
import { SiteLayout } from "@/components/SiteLayout";
import { candidateLoader, indexLoader, landingLoader } from "@/data/loaders";
import { AboutPage } from "@/pages/AboutPage";
import { CandidatePage } from "@/pages/CandidatePage";
import { ErrorPage } from "@/pages/ErrorPage";
import { LandingPage } from "@/pages/LandingPage";
import { LoadingPage } from "@/pages/LoadingPage";
import { NotFoundPage } from "@/pages/NotFoundPage";
import { RootErrorPage } from "@/pages/RootErrorPage";
import { SearchPage } from "@/pages/SearchPage";

/**
 * Path-based routes (shareable URLs). CloudFront rewrites extension-less
 * paths to /index.html (05-infrastructure), so deep links work.
 */
export const router = createBrowserRouter([
  {
    id: "root",
    path: "/",
    Component: SiteLayout,
    ErrorBoundary: RootErrorPage, // only for unexpected render errors in the layout
    children: [
      {
        index: true,
        loader: landingLoader,
        Component: LandingPage,
        shouldRevalidate: () => false,
      },
      {
        id: "data",
        loader: indexLoader,
        // The index never changes during a visit; filters only touch the URL
        // query, which must not re-run the loader.
        shouldRevalidate: () => false,
        HydrateFallback: LoadingPage,
        ErrorBoundary: ErrorPage,
        children: [
          { path: "candidatos", Component: SearchPage },
          {
            path: "candidatos/:slug",
            loader: candidateLoader,
            Component: CandidatePage,
            ErrorBoundary: ErrorPage,
          },
          { path: "sobre", Component: AboutPage },
        ],
      },
      { path: "*", Component: NotFoundPage },
    ],
  },
]);
```

The pathless `data` route has no `Component`; React Router renders an
`<Outlet />` for it by default.

`src/data/hooks.ts`:

```ts
import { useRouteLoaderData } from "react-router";
import type { DataIndex } from "./types";

/** The index loaded by the "data" route. Use only inside /candidatos, /candidatos/:slug, /sobre. */
export function useDataIndex(): DataIndex {
  const index = useRouteLoaderData("data") as DataIndex | undefined;
  if (index === undefined) throw new Error("useDataIndex used outside the data routes");
  return index;
}

/** For shared chrome (the footer): the index when a data page is active, else undefined. */
export function useOptionalDataIndex(): DataIndex | undefined {
  return useRouteLoaderData("data") as DataIndex | undefined;
}
```

The API names above (`createBrowserRouter`, `loader`, `Component`,
`HydrateFallback`, `ErrorBoundary`, `shouldRevalidate`, `useRouteLoaderData`,
`useLoaderData`, `Await`, `useRouteError`, `isRouteErrorResponse`,
`ScrollRestoration`, `useNavigation`) are React Router's data-mode API.
`createBrowserRouter` + `RouterProvider` imported from `"react-router"` are
known to work with 8.3.1. Before coding, confirm against the **8.3**
reference two behaviours this design relies on:

1. **Returning an un-awaited promise from a loader** (inside a plain object)
   streams it to the component, which reads it with `<Await>` in
   `<Suspense>` — no `defer()` helper is needed.
2. **Partial hydration:** on a first visit to `/candidatos`, the router
   renders the root layout immediately and shows the `data` route's
   `HydrateFallback` inside the layout's `<Outlet>` while `indexLoader` runs.

If either differs in 8.3, the fallback is: no loader on the landing, and a
`useEffect` in the stats strip that calls `getIndexOrNull()`; and/or
`HydrateFallback` on the root rendering a skeleton of the layout.

### `ErrorPage` (data-route error boundary, rendered inside the layout)

```tsx
import { isRouteErrorResponse, useRouteError } from "react-router";
import { isDataError } from "@/data/errors";
import { NotFoundPage } from "./NotFoundPage";

export function ErrorPage() {
  const error = useRouteError();
  if ((isDataError(error) && error.kind === "not_found") ||
      (isRouteErrorResponse(error) && error.status === 404)) {
    return <NotFoundPage />;
  }
  const unavailable = isDataError(error) && error.kind === "unavailable";
  return (
    <div className="mx-auto max-w-2xl px-4 py-10">
      <title>{`Dados indisponíveis – ${SITE_SHORT_NAME}`}</title>
      <h1 tabIndex={-1} className="text-2xl font-semibold">
        {unavailable ? "Os dados estão em preparação" : "Não foi possível carregar os dados"}
      </h1>
      <p className="mt-3 text-muted-foreground">
        {unavailable
          ? "Os dados oficiais ainda estão sendo publicados ou atualizados. Tente novamente em alguns minutos."
          : "Verifique sua conexão com a internet e tente novamente."}
      </p>
      <div className="mt-6 flex flex-wrap gap-3">
        <Button onClick={() => window.location.reload()}>Tentar novamente</Button>
        <Button variant="outline" asChild>
          <Link to="/">Página inicial</Link>
        </Button>
      </div>
    </div>
  );
}
```

(Imports of `Button`, `Link` and `SITE_SHORT_NAME` omitted.) A full reload is
the simplest correct retry: it refetches `latest.json` and resets the memo.

`ErrorPage` renders a `<div>`, **not** `<main>`: it sits inside the layout's
`<main id="conteudo">` (one `main` per page).

### `RootErrorPage` (last resort)

Only reached if the layout itself throws while rendering (a bug, not a data
problem — the root has no loader). It renders its own minimal
`<main id="conteudo">` with "Ocorreu um erro inesperado.", a "Tentar
novamente" button (reload) and a plain `<a href="/">Página inicial</a>`.

### `LoadingPage` (`HydrateFallback` of the data route)

Shown inside the layout while `indexLoader` runs on a first visit to a data
page. Skeleton cards (shadcn `skeleton`) plus a visually hidden status:

```tsx
<div aria-busy="true" className="mx-auto max-w-6xl px-4 py-8">
  <p role="status" className="sr-only">Carregando dados dos candidatos…</p>
  {/* 6 skeleton cards in the same grid as the results */}
</div>
```

### Navigation pending state

Between pages (e.g. opening a profile on a slow connection) the previous page
stays on screen while the loader runs. In `SiteLayout`:

```tsx
const navigation = useNavigation();
const busy = navigation.state === "loading";
// <main aria-busy={busy}> and a thin top progress bar (motion-safe only)
// plus <p role="status" className="sr-only">{busy ? "Carregando…" : ""}</p>
```

Navigating to `/` is instant (its loader returns without awaiting), so the
bar never shows for the landing page.

## Step 5 — `src/lib/text.ts` (must match Go `internal/textnorm`)

The ETL builds `searchText` with Go's `textnorm.Normalize`; the browser must
normalise the user's query **the same way**, or searches silently miss.

The shared rule (`01-foundation/02-repository-layout.md`): lowercase → NFD →
strip combining marks → collapse whitespace (and trim).

```ts
/** Identical to backend/internal/textnorm.Normalize. Change both together. */
export function normalize(input: string): string {
  return input
    .toLowerCase()
    .normalize("NFD")
    .replace(/\p{M}/gu, "") // combining marks: "São" → "sao", "Ç" → "c"
    .replace(/\s+/g, " ")
    .trim();
}

/**
 * Every whitespace-separated token of the query must appear in searchText.
 * "joao 12" matches "joao da silva 1234 abc".
 */
export function matchesQuery(searchText: string, query: string): boolean {
  const tokens = normalize(query).split(" ").filter((t) => t.length > 0);
  return tokens.every((t) => searchText.includes(t));
}
```

Go side, for reference (owned by `03-backend-etl`):
`strings.ToLower` → `norm.NFD` → drop runes in `unicode.Mn` → `strings.Fields`
+ `strings.Join(…, " ")`. For Portuguese text both produce identical output.
Sanity samples to compare manually once both exist:

| Input | Expected |
|---|---|
| `"  JOÃO  da Silva "` | `"joao da silva"` |
| `"Conceição"` | `"conceicao"` |
| `"MDB 15123"` | `"mdb 15123"` |

## Step 6 — `src/data/labels.ts` (pt-BR labels for codes)

Labels follow `01-foundation/03-data-model.md`. Gender-neutral forms with
"(a)" because the contract deliberately carries no gender.

```ts
import type { Office, ResultCode, SelectionReason } from "./types";

export const OFFICE_LABEL: Record<Office, string> = {
  federal_deputy: "Deputado(a) federal",
  state_deputy: "Deputado(a) estadual",
};

export const OFFICE_PLURAL: Record<Office, string> = {
  federal_deputy: "Deputados(as) federais",
  state_deputy: "Deputados(as) estaduais",
};

/** `other` shows the TSE label itself — see resultLabel(). */
export const RESULT_LABEL: Record<Exclude<ResultCode, "other">, string> = {
  elected_by_quota: "Eleito(a) por quociente partidário",
  elected_by_average: "Eleito(a) por média",
  elected: "Eleito(a)",
  substitute: "Suplente",
  not_elected: "Não eleito(a)",
  no_result: "Sem resultado divulgado",
};

export function resultLabel(result: { code: ResultCode; tseLabel: string }): string {
  return result.code === "other" ? result.tseLabel : RESULT_LABEL[result.code];
}

/** Filter groups (URL value → codes), per 01-foundation/03-data-model.md. */
export const RESULT_GROUPS = {
  eleitos: { label: "Eleitos(as)", codes: ["elected_by_quota", "elected_by_average", "elected"] },
  suplentes: { label: "Suplentes", codes: ["substitute"] },
  "nao-eleitos": { label: "Não eleitos(as)", codes: ["not_elected"] },
  "sem-resultado": { label: "Sem resultado", codes: ["no_result", "other"] },
} as const satisfies Record<string, { label: string; codes: readonly ResultCode[] }>;

export type ResultGroup = keyof typeof RESULT_GROUPS;

export const REASON_LABEL: Record<SelectionReason, string> = {
  top_votes: "Entre os mais votados em Videira",
  born_in_municipality: "Nascido(a) em Videira",
};
```

## Step 7 — `src/data/format.ts`

All formatting happens here, never inline in components.

```ts
const TZ = "America/Sao_Paulo";

const integer = new Intl.NumberFormat("pt-BR");
const brl = new Intl.NumberFormat("pt-BR", { style: "currency", currency: "BRL" });
const percent = new Intl.NumberFormat("pt-BR", {
  style: "percent",
  minimumFractionDigits: 1,
  maximumFractionDigits: 1,
});
const dateFmt = new Intl.DateTimeFormat("pt-BR", { dateStyle: "short", timeZone: "UTC" });
const dateLong = new Intl.DateTimeFormat("pt-BR", { dateStyle: "long", timeZone: "UTC" });
const timestampFmt = new Intl.DateTimeFormat("pt-BR", {
  dateStyle: "short",
  timeStyle: "short",
  timeZone: TZ,
});

export const formatInteger = (n: number) => integer.format(n);         // 3.120
export const formatBRL = (n: number) => brl.format(n);                 // R$ 1.234,56
export const formatShare = (share: number) => percent.format(share);   // 0.1043 → 10,4%
export const formatOrdinal = (rank: number) => `${rank}º`;             // 1º

/**
 * "2026-10-04" is a calendar date, not an instant. Parsing it with
 * `new Date("2026-10-04")` gives UTC midnight, which in São Paulo is the
 * previous evening — so format it in UTC to keep the same day.
 */
export function formatDate(isoDate: string, style: "short" | "long" = "short"): string {
  const [y, m, d] = isoDate.split("-").map(Number);
  if (!y || !m || !d) return isoDate;
  const date = new Date(Date.UTC(y, m - 1, d));
  return (style === "long" ? dateLong : dateFmt).format(date);
}

/** Instants (generatedAt, fetchedAt) are shown in Brasília time. */
export function formatTimestamp(iso: string): string {
  const date = new Date(iso);
  return Number.isNaN(date.getTime()) ? iso : `${timestampFmt.format(date)} (horário de Brasília)`;
}

const dayOfFmt = new Intl.DateTimeFormat("pt-BR", { dateStyle: "short", timeZone: TZ });

/** Calendar day of an instant in Brasília: "2026-11-09T06:02:41Z" → "09/11/2026". */
export function formatDayOf(iso: string): string {
  const date = new Date(iso);
  return Number.isNaN(date.getTime()) ? iso : dayOfFmt.format(date);
}

export function formatBytes(bytes: number): string {
  const mb = bytes / (1024 * 1024);
  return mb >= 1
    ? `${new Intl.NumberFormat("pt-BR", { maximumFractionDigits: 1 }).format(mb)} MB`
    : `${new Intl.NumberFormat("pt-BR", { maximumFractionDigits: 0 }).format(bytes / 1024)} KB`;
}

/** Vote counts with correct singular/plural: "1 voto", "3.120 votos". */
export function formatVotes(n: number): string {
  return `${formatInteger(n)} ${n === 1 ? "voto" : "votos"}`;
}
```

Brazil has had no daylight saving time since 2019, so "horário de Brasília"
is unambiguous.

## Step 8 — URLs for photos and files

Every relative path in the contract (`summary.photo`, `index.dataset.path`)
is relative to the **version directory**. Always build URLs with
`dataUrl(index.version, path)`:

```ts
const photoSrc = summary.photo ? dataUrl(index.version, summary.photo) : null;
const datasetHref = dataUrl(index.version, index.dataset.path);
```

Never hard-code `/data/latest/...` — there is no such path.

## Done when

- [ ] **No data published yet** (empty `backend/dist/data`, or a fresh bucket): `/` renders completely, its strip says "Dados em preparação", and the console shows no uncaught error; `/candidatos`, `/candidatos/<x>` and `/sobre` show "Os dados estão em preparação" inside the normal header and footer.
- [ ] With the local ETL output, visiting `/` and then `/candidatos` shows exactly one request each for `latest.json` and `index.json` in DevTools → Network, no matter how many filters are changed afterwards.
- [ ] A first visit straight to `/candidatos` shows the header, the skeleton (`LoadingPage`) inside the layout, then the results.
- [ ] Opening a profile fetches only `candidates/<slug>.json` (and its photo); going back to the list fetches nothing.
- [ ] Reloading: `latest.json` is revalidated (304 when unchanged in the deployed site); `index.json` comes from the browser cache.
- [ ] `/candidatos/nao-existe` shows the not-found page **without** a network request for a profile.
- [ ] Temporarily editing the local `latest.json` to `"schemaVersion": 2`: data pages show "Os dados estão em preparação"; the landing still renders, with "Dados em preparação" in the strip.
- [ ] DevTools → Network → "Offline", then reload a data page: the network error screen appears and "Tentar novamente" works once back online.
- [ ] Formatting spot checks: `formatBRL(1234.5)` → `R$ 1.234,50`; `formatShare(0.1043)` → `10,4%`; `formatDate("2026-10-04")` → `04/10/2026`; `formatDayOf("2026-11-09T02:00:00Z")` → `08/11/2026` (still the 8th in Brasília).
