# 03.02 — Fetching and parsing primitives

## Goal

Three small leaf packages every loader uses:

- `internal/fetch` — HTTP client (User-Agent, timeouts, retries, rate limit),
  on-disk cache, **remote zip reader over HTTP range requests**, and the
  provenance registry (`Source`).
- `internal/csvx` — CSV readers for Latin-1 / Windows-1252 / UTF-8-BOM files,
  header-name access, Brazilian numbers and dates, TSE null sentinels.
- `internal/textnorm` — the one normalisation function used for matching,
  slugs and search text.

## Files to create

```
backend/internal/fetch/client.go      # Client, Options, Get, GetJSON, retry, rate limit
backend/internal/fetch/cache.go       # URL → cache path, conditional reuse
backend/internal/fetch/remotezip.go   # OpenZip: range ReaderAt or full download fallback
backend/internal/fetch/source.go      # Source, Sources registry
backend/internal/csvx/reader.go       # Open(reader, Encoding) → *Reader with Get(col)
backend/internal/csvx/values.go       # Null, Money, Int, DateBR, TimestampBR
backend/internal/textnorm/textnorm.go # Norm, Slug
testdata/textnorm-vectors.json        # repository root: [{input, expected}], read by Go and TS tests
# tests (next to the code):
backend/internal/fetch/client_test.go
backend/internal/fetch/cache_test.go
backend/internal/fetch/remotezip_test.go
backend/internal/fetch/source_test.go
backend/internal/csvx/reader_test.go
backend/internal/csvx/values_test.go
backend/internal/textnorm/textnorm_test.go
```

## 1. `fetch.Client` and `fetch.Env`

```go
// Tuning holds the knobs tests shrink. Zero value = production behaviour.
type Tuning struct {
	MaxAttempts int                                          // 0 → config.MaxAttempts (3)
	BackoffBase time.Duration                                // 0 → 1s (waits 1s, 2s, 4s)
	APIRate     int                                          // requests/s for GetJSON; 0 → config.APIRequestsPerSec; <0 → unlimited
	Sleep       func(ctx context.Context, d time.Duration) error // nil → real timer; tests record the waits instead
	BlockSize   int64                                        // range-read block; 0 → 1 MiB
	Transport   http.RoundTripper                            // nil → http.DefaultTransport; the integration test
	                                                         // injects testsrv.Transport() (real URLs → fake upstream)
}

type Options struct {
	UserAgent string
	CacheDir  string // "" → no disk cache
	Offline   bool   // only serve from cache
	Refresh   bool   // ignore cache, always download
	Tuning    Tuning
	Now       func() time.Time // FetchedAt and cache-day decisions; injected by the pipeline
	Log       *slog.Logger
}

type Client struct {
	http *http.Client // Timeout: config.HTTPTimeoutSeconds
	opts Options
	rate *rateLimiter // used by GetJSON only
}

func New(o Options) *Client

// Get downloads url into the cache (if needed) and returns the local path and
// the response metadata used for provenance.
func (c *Client) Get(ctx context.Context, url string) (path string, meta Meta, err error)

// GetJSON performs a rate-limited API call (no disk cache) and decodes into v.
// Returns the links[rel=next] URL when present (Câmara API style).
func (c *Client) GetJSON(ctx context.Context, url string, v any) (next string, meta Meta, err error)

type Meta struct {
	FetchedAt    time.Time  // UTC, from Options.Now
	LastModified *time.Time // from the Last-Modified header, if any
	Size         int64
}

// HTTPError is returned for non-retryable 4xx answers; errors.Is(err, ErrNotFound) for 404.
type HTTPError struct{ URL string; Status int }
var ErrNotFound = errors.New("not found")
func (e *HTTPError) Is(target error) bool { return target == ErrNotFound && e.Status == http.StatusNotFound }

// Env is what every loader receives: one value instead of five parameters.
type Env struct {
	Client       *Client
	Sources      *Sources
	URLs         config.URLs
	ChamberYears []int
	Now          func() time.Time
	Log          *slog.Logger
}
```

Loaders build URLs **only** from `env.URLs` (never from literals), so the
integration test can point them at the fake upstream.

### Retry policy (in one helper)

```go
func (c *Client) do(ctx context.Context, req *http.Request) (*http.Response, error) {
	req.Header.Set("User-Agent", c.opts.UserAgent)
	t := c.opts.Tuning
	var last error
	for attempt := 1; attempt <= t.MaxAttempts; attempt++ {
		resp, err := c.http.Do(req.Clone(ctx))
		var retryAfter time.Duration
		switch {
		case err == nil && resp.StatusCode < 300:
			return resp, nil
		case err == nil && (resp.StatusCode == 429 || resp.StatusCode >= 500):
			last = &HTTPError{URL: req.URL.String(), Status: resp.StatusCode} // Câmara API 504s happen
			retryAfter = parseRetryAfter(resp.Header.Get("Retry-After"))   // seconds form only; capped at 30s
			resp.Body.Close()
		case err == nil:
			resp.Body.Close()
			return nil, &HTTPError{URL: req.URL.String(), Status: resp.StatusCode} // 4xx: do not retry
		default:
			last = err
		}
		if attempt == t.MaxAttempts {
			break
		}
		wait := t.BackoffBase << (attempt - 1) // 1s, 2s, 4s in production
		if retryAfter > wait {
			wait = retryAfter
		}
		if err := t.Sleep(ctx, wait); err != nil { // returns ctx.Err() when cancelled
			return nil, err
		}
	}
	return nil, fmt.Errorf("after %d attempts: %w", t.MaxAttempts, last)
}
```

Rules:

- `User-Agent` on **every** request (`config.UserAgent(contact)`, step 01).
- `Accept: application/json` on every `GetJSON`.
- Rate limit only for the Câmara **API**; bulk file downloads are a handful
  of requests and are not throttled.
- Never log response bodies.

## 2. Cache (`cache.go`)

```go
// cachePath: <CacheDir>/<sha256(url)[:16]>-<basename(url)>
func (c *Client) cachePath(url string) string
```

| Mode | Behaviour |
|---|---|
| default | If the cached file exists **and was written in the current calendar day (UTC, per `Options.Now`)**, reuse it; otherwise download to `path.tmp`, `fsync`, rename. |
| `-offline` | Use cached file or fail with `"not cached: <url>"`. |
| `-refresh` | Always download. |

With `CacheDir == ""` (unit tests) nothing is written to disk: `Get`
downloads to a temp file removed when the run ends, and `OpenZipMember`
streams the member directly.

The cache is a developer convenience. CI starts empty, so every published
snapshot comes from fresh downloads. `backend/.cache/` is git-ignored.

Remote zips read with range requests are **not** cached as whole files: the
member that was extracted is cached instead, as `…-<member>.csv` (see §3).

## 3. Remote zip — read one member without downloading 454 MB

`archive/zip.NewReader` needs an `io.ReaderAt` and the total size. An HTTP
server that honours `Range:` gives exactly that. `cdn.tse.jus.br`,
`dadosabertos-download.cgu.gov.br` and `www.camara.leg.br/cotas` all honour
ranges (verified 2026-10-21: the SC votes member is 5.2 MB compressed inside
the 453.7 MB zip).

```go
// httpReaderAt implements io.ReaderAt with ranged GETs and a 1 MiB block cache.
type httpReaderAt struct {
	c     *Client
	url   string
	size  int64
	block map[int64][]byte // blockIndex → bytes (zip directory + member reads are sequential)
}

func (r *httpReaderAt) ReadAt(p []byte, off int64) (int, error) {
	// For each 1 MiB block covering [off, off+len(p)): fetch if missing with
	//   Range: bytes=<start>-<end>   → expect HTTP 206
	// then copy into p. Return io.EOF when off+len(p) > size.
}

// OpenZipMember returns a reader over one member of a remote zip.
// It caches the *decompressed member* on disk, keyed by url+member.
func (c *Client) OpenZipMember(ctx context.Context, url, member string) (io.ReadCloser, Meta, error) {
	// 0. cache hit (url+member) → open the cached file.
	// 1. HEAD url → Content-Length, Accept-Ranges, Last-Modified.
	// 2. If Accept-Ranges != "bytes" or a probe GET with Range returns 200 instead of 206:
	//      fall back: c.Get(url) (full download to cache) → zip.OpenReader(path).
	// 3. Else zip.NewReader(&httpReaderAt{…}, size).
	// 4. Find the member by exact name; error lists available names if missing.
	// 5. Stream-decompress to cache tmp file, rename, return it opened.
	//    Meta.LastModified = member's Modified time (zip header) if set, else HTTP Last-Modified.
}
```

Pitfalls:

- Range responses must be **206**; a 200 means the server ignored the range —
  switch to the fallback, never treat the full body as a block.
- The zip central directory is at the end of the file: the first reads hit
  the last block. Expect ~3–6 ranged requests for the directory, then
  sequential blocks for the member.
- Always look members up by name (`consulta_cand_2026_SC.csv`), never index.

## 4. Provenance registry (`source.go`)

```go
type Source struct {
	ID          string     // one of the fixed ids in 01-foundation/03-data-model.md
	Publisher   string
	Name        string     // pt-BR, shown in the UI
	DatasetURL  string
	FileURL     string     // first file/endpoint fetched for this source
	GeneratedAt *time.Time // publisher's generation time, if known
	FetchedAt   time.Time  // earliest fetch of this source in the run (UTC)
}

type Sources struct{ m map[string]*Source }

func NewSources() *Sources
// Record creates the source on first call; later calls keep the first FileURL
// and the earliest FetchedAt, and the *latest* GeneratedAt.
func (s *Sources) Record(src Source)
func (s *Sources) All() []Source // sorted by ID
```

Each loader decides `GeneratedAt`:

| Source | `GeneratedAt` |
|---|---|
| TSE files | `DT_GERACAO` + `HH_GERACAO` of the first row, `-03:00` |
| Câmara bulk | HTTP `Last-Modified` |
| Câmara API | `nil` (live data; `fetchedAt` is the reference) |
| CGU zip | member modified time |

Names (pt-BR) and dataset URLs are constants in each loader package, e.g.
`tse_votes` → `"Votação nominal por município e zona – Eleições 2026"`,
`https://dadosabertos.tse.jus.br/dataset/resultados-2026`.

## 5. CSV reader (`csvx`)

```go
type Encoding int
const (
	Latin1     Encoding = iota // TSE
	Windows1252                // CGU
	UTF8BOM                    // Câmara bulk (BOM optional)
)

type Reader struct {
	r   *csv.Reader
	idx map[string]int // header name → column
	row []string
}

func Open(src io.Reader, enc Encoding) (*Reader, error) {
	var dec io.Reader = src
	switch enc {
	case Latin1:
		dec = transform.NewReader(src, charmap.ISO8859_1.NewDecoder())
	case Windows1252:
		dec = transform.NewReader(src, charmap.Windows1252.NewDecoder())
	case UTF8BOM:
		dec = skipBOM(src) // bufio.Reader; Peek(3) == EF BB BF → Discard(3)
	}
	cr := csv.NewReader(dec)
	cr.Comma = ';'
	cr.LazyQuotes = true      // TSE has stray quotes inside fields
	cr.FieldsPerRecord = -1   // tolerate short trailing rows; validate per row instead
	cr.ReuseRecord = true     // streaming 96 MB files with flat memory
	header, err := cr.Read()
	// build idx with strings.TrimSpace + strings.Trim(h, "\"﻿")
	…
}

func (r *Reader) Next() bool               // advances; false at EOF
func (r *Reader) Err() error
func (r *Reader) Get(col string) string    // "" if column missing; panics never
func (r *Reader) Require(cols ...string) error // call once after Open: fail fast on schema drift
```

`Require` is the guard against the publisher renaming a column: the run
aborts with `missing column X in <file>` instead of exporting empty data.

## 6. Values (`values.go`)

```go
// Null reports TSE/CGU sentinel values that mean "no value".
func Null(s string) bool {
	switch strings.TrimSpace(s) {
	case "", "#NULO", "#NULO#", "#NE", "#NE#", "-1", "-3", "-4", "Sem informação", "S/I":
		return true
	}
	return false
}

func Str(s string) *string // nil if Null(s), else trimmed

// emptyNumeric: the only sentinels allowed in numeric columns (never -1/-3/-4,
// which could be real values in money columns such as CEAP refunds).
func emptyNumeric(s string) bool {
	switch strings.TrimSpace(s) {
	case "", "#NULO", "#NULO#", "#NE", "#NE#":
		return true
	}
	return false
}

// Money parses both "1.234,56" / "1300000,00" (BR) and "1234.56" / "699.9" (Câmara).
func Money(s string) (float64, error) {
	s = strings.TrimSpace(s)
	if emptyNumeric(s) { return 0, nil }
	if strings.Contains(s, ",") { // BR: '.' thousands, ',' decimal
		s = strings.ReplaceAll(s, ".", "")
		s = strings.Replace(s, ",", ".", 1)
	}
	v, err := strconv.ParseFloat(s, 64)
	if err != nil { return 0, fmt.Errorf("money %q: %w", s, err) }
	return math.Round(v*100) / 100, nil
}

func Int(s string) (int, error)                  // "0012" → 12; emptyNumeric → 0
func DateBR(s string) (*time.Time, error)         // "04/10/2026" → date; Null → nil
func TimestampBR(date, hhmmss string) (*time.Time, error) // DT_GERACAO+HH_GERACAO in America/Sao_Paulo (fixed −03:00)
```

`-1`/`-3`/`-4` are sentinels only in **code/text** columns (`Null`, `Str`).
Never call `Null` on vote counts or money columns, where a negative value can
be real (CEAP refunds) — `Money`/`Int` use `emptyNumeric`, which treats only
`""` and the `#NULO`/`#NE` variants as zero.

Use a fixed `time.FixedZone("BRT", -3*3600)` rather than the tz database:
Brazil has had no DST since 2019 and the runner image may lack tzdata.

## 7. Text normalisation (`textnorm`)

```go
// Norm: lowercase → NFD → drop combining marks → collapse whitespace → trim.
func Norm(s string) string {
	t := transform.Chain(norm.NFD, runes.Remove(runes.In(unicode.Mn)), norm.NFC)
	out, _, _ := transform.String(t, strings.ToLower(s))
	return strings.Join(strings.Fields(out), " ")
}

// Slug: Norm, then every run of non [a-z0-9] → "-", trim "-".
func Slug(s string) string
```

Identical behaviour to `frontend/src/lib/text.ts` (same steps, same order) so
`searchText` built in Go matches what the browser normalises from user input.
The guarantee is mechanical, not a promise: both test suites read the same
vectors file.

`testdata/textnorm-vectors.json` (repository root; created in this step) —
**an array of `{ "input", "expected" }` objects**, the exact shape the
frontend suite also reads (`04-frontend/08-testing.md`). It covers `Norm`
only; slug cases live in the Go table test (the frontend never builds slugs).

```json
[
  { "input": "JOSÉ DA CONCEIÇÃO", "expected": "jose da conceicao" },
  { "input": "  Maria   das  Graças ", "expected": "maria das gracas" },
  { "input": "ÁÀÂÃÄ éêë íï óôõö úü ç ñ", "expected": "aaaaa eee ii oooo uu c n" },
  { "input": "D'ÁVILA-SOUZA", "expected": "d'avila-souza" },
  { "input": "TESTE\tCOM\nQUEBRA", "expected": "teste com quebra" },
  { "input": "PTX 9901", "expected": "ptx 9901" },
  { "input": "", "expected": "" }
]
```

Add a vector whenever a real-world name surprises either implementation.

## Pitfalls

- Do not read CSVs with `bufio.Scanner` line by line: quoted fields may
  contain `;` and, in TSE asset descriptions, even line breaks.
- Do not `strings.ToUpper` before decoding Latin-1 — decode first.
- `encoding/csv` with `ReuseRecord` reuses the slice: copy strings you keep
  (`strings.Clone`) when storing them past the next `Next()`.

## Tests for this step

All table-driven, no network (an `httptest.Server` or the shared `testsrv`
from step 09), `t.Parallel()` where there is no shared state.

### `internal/fetch`

| File | Test | Cases |
|---|---|---|
| `client_test.go` | `TestRetry` | server answers `504, 504, 200` → success after 3 attempts; waits recorded by the injected `Tuning.Sleep` are `[1s, 2s]` (with `BackoffBase = 1s`, nothing actually sleeps); `429` with `Retry-After: 3` → recorded wait `3s`; `500 ×3` → error wrapping `*HTTPError{Status: 500}` after exactly `MaxAttempts` requests; `404` → **one** request, `errors.Is(err, ErrNotFound)`; `403` → one request, not `ErrNotFound`; network error (closed listener) → retried |
| | `TestRetryHonoursContext` | cancelled context during the backoff → returns `context.Canceled` without another request |
| | `TestUserAgentAndAccept` | every request carries the configured `User-Agent`; `GetJSON` sends `Accept: application/json` |
| | `TestGetJSONNextLink` | body with `links[{rel:"next"}]` → returns its `href`; without → `""`; invalid JSON → error naming the URL |
| | `TestRateLimit` | `APIRate: 2` and 5 calls → injected clock/sleep shows ≥ 2 s spread; `APIRate: -1` → no waits |
| `cache_test.go` | `TestCacheHitSecondRun` | two `Get` calls for the same URL with the same `Now` day → server sees **one** request; next day (`Now` + 24 h) → two |
| | `TestOffline` | `Offline: true` with an empty cache → error `not cached: <url>`; after a normal `Get`, offline `Get` succeeds with zero new requests |
| | `TestRefresh` | `Refresh: true` → always downloads |
| | `TestCachePathStable` | same URL → same path; different query strings → different paths; path stays inside `CacheDir` |
| | `TestAtomicWrite` | server drops the connection mid-body → no file left at the final cache path (only `*.tmp` cleaned up) |
| `remotezip_test.go` | `TestOpenZipMemberRange` | zip built in memory with 3 members served by `http.ServeContent` → member bytes equal the original; request log shows only `Range` GETs (and the HEAD), total bytes transferred < zip size when the wanted member is small and the others are large |
| | `TestRangeIgnoredFallback` | server ignores `Range` (always 200 with the full body) → falls back to a full download, still returns the right member, logs the fallback once |
| | `TestNoAcceptRanges` | HEAD without `Accept-Ranges: bytes` → fallback path |
| | `TestMissingMember` | asks for `x_SC.csv` not in the zip → error lists available member names |
| | `TestMemberModifiedTime` | `Meta.LastModified` = zip header time when set, else HTTP `Last-Modified` |
| | `TestCachedMember` | second `OpenZipMember` (same day, `CacheDir` set) → zero requests |
| `source_test.go` | `TestSourcesRecord` | first `FileURL` kept; earliest `FetchedAt` kept; latest `GeneratedAt` kept; `All()` sorted by ID |

### `internal/csvx`

| File | Test | Cases |
|---|---|---|
| `reader_test.go` | `TestOpenEncodings` | the same text `"JOÃO";"AÇÃO – TESTE"` encoded as Latin-1, as Windows-1252 and as UTF-8+BOM decodes correctly with the matching `Encoding`; byte `0x96` decodes to `–` under Windows-1252 and to U+0096 under Latin-1 (proves the two are not interchangeable) |
| | `TestBOM` | BOM present → first column name is `idVotacao` (not `\ufeffidVotacao`); BOM absent → same result |
| | `TestQuotedFields` | `;` inside quotes, CRLF and a line break inside a quoted field, stray quote (`LazyQuotes`) |
| | `TestRequire` | missing column → error `missing column X`; all present → nil |
| | `TestGetUnknownColumn` | returns `""`, never panics |
| | `TestReuseRecordClone` | values kept across `Next()` calls stay intact (guards the `strings.Clone` rule) |
| `values_test.go` | `TestNull` | `""`, `#NULO`, `#NULO#`, `#NE`, `#NE#`, `-1`, `-3`, `-4`, `Sem informação`, `S/I`, `"  #NULO  "` → true; `"0"`, `"VIDEIRA"`, `"-10"` → false |
| | `TestMoney` | `"1.234,56"` → 1234.56; `"1300000,00"` → 1300000; `"399688,80"` → 399688.8; `"699"` → 699; `"699.9"` → 699.9; `"699.99"` → 699.99; `"-150.5"` → −150.5 (refund); `"-1"` → −1 (**not** a sentinel here); `""`/`#NULO` → 0; `"abc"` → error; rounding `"1,239"` → 1.24 |
| | `TestInt` | `"0012"` → 12; `#NE` → 0; `"1.5"` → error |
| | `TestDateBR` | `"04/10/2026"` → 2026-10-04; `#NULO` → nil; `"2026-10-04"` → error |
| | `TestTimestampBR` | `"21/10/2026"`, `"16:31:26"` → `2026-10-21T16:31:26-03:00` (= `19:31:26Z`) |

### `internal/textnorm`

| File | Test | Cases |
|---|---|---|
| `textnorm_test.go` | `TestNormVectors` | decodes `../../../testdata/textnorm-vectors.json` into `[]struct{ Input string \`json:"input"\`; Expected string \`json:"expected"\` }` (`DisallowUnknownFields`; fails if the file is empty) and checks `Norm(input) == expected` for each |
| | `TestSlug` | table: `JOSÉ DA CONCEIÇÃO TESTE` → `jose-da-conceicao-teste`; `D'ÁVILA-SOUZA` → `d-avila-souza`; `DR. FULANO` → `dr-fulano`; `--A  B--` → `a-b`; every output matches `^[a-z0-9-]*$` |
| | `TestNormIdempotent` | `Norm(Norm(x)) == Norm(x)` for every vector input |

Fixtures: in-memory only (zips built with `archive/zip` in the test,
encodings produced with `charmap.*.NewEncoder()`), plus the shared vectors
file.

## Done when

```bash
cd backend
go vet ./internal/fetch ./internal/csvx ./internal/textnorm
go test -race ./internal/fetch/... ./internal/csvx/... ./internal/textnorm/...
cd .. && make test-backend      # all tests pass, total coverage >= 85%
# Optional manual smoke against the real CDN (not part of CI):
#   OpenZipMember(DefaultURLs().TSEBase+"bem_candidato/bem_candidato_2026.zip", "bem_candidato_2026_SC.csv")
#   → logs ~4 ranged requests, writes the member to .cache, prints its first header line
```
