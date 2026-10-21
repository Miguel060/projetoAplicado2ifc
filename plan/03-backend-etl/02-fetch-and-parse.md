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
```

## 1. `fetch.Client`

```go
type Options struct {
	UserAgent string
	CacheDir  string
	Offline   bool // only serve from cache
	Refresh   bool // ignore cache, always download
	Log       *slog.Logger
}

type Client struct {
	http    *http.Client // Timeout: config.HTTPTimeoutSeconds
	opts    Options
	limiter <-chan time.Time // time.Tick(time.Second / config.APIRequestsPerSec), used by GetJSON only
}

func New(o Options) *Client

// Get downloads url into the cache (if needed) and returns the local path and
// the response metadata used for provenance.
func (c *Client) Get(ctx context.Context, url string) (path string, meta Meta, err error)

// GetJSON performs a rate-limited API call (no disk cache unless -offline was
// requested and a cached copy exists) and decodes into v. Returns the Link
// "next" URL when the body has links[rel=next] (Câmara API style).
func (c *Client) GetJSON(ctx context.Context, url string, v any) (next string, meta Meta, err error)

type Meta struct {
	FetchedAt    time.Time  // UTC
	LastModified *time.Time // from the Last-Modified header, if any
	Size         int64
}
```

### Retry policy (in one helper)

```go
func (c *Client) do(ctx context.Context, req *http.Request) (*http.Response, error) {
	req.Header.Set("User-Agent", c.opts.UserAgent)
	var last error
	for attempt := 1; attempt <= config.MaxAttempts; attempt++ {
		resp, err := c.http.Do(req.Clone(ctx))
		switch {
		case err == nil && resp.StatusCode < 300:
			return resp, nil
		case err == nil && (resp.StatusCode == 429 || resp.StatusCode >= 500):
			last = fmt.Errorf("%s: HTTP %d", req.URL, resp.StatusCode) // Câmara API 504s happen
			resp.Body.Close()
		case err == nil:
			resp.Body.Close()
			return nil, fmt.Errorf("%s: HTTP %d", req.URL, resp.StatusCode) // 4xx: do not retry
		default:
			last = err
		}
		wait := time.Duration(1<<(attempt-1)) * time.Second // 1s, 2s, 4s
		if ra := resp; ra != nil { /* honour Retry-After seconds if present */ }
		select {
		case <-time.After(wait):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, fmt.Errorf("after %d attempts: %w", config.MaxAttempts, last)
}
```

Rules:

- `User-Agent` on **every** request (`config.UserAgent(log)`).
- `Accept: application/json` on every `GetJSON`.
- Rate limit (`<-c.limiter`) only for the Câmara **API**; bulk file
  downloads are a handful of requests and are not throttled.
- Never log response bodies.

## 2. Cache (`cache.go`)

```go
// cachePath: <CacheDir>/<sha256(url)[:16]>-<basename(url)>
func (c *Client) cachePath(url string) string
```

| Mode | Behaviour |
|---|---|
| default | If the cached file exists **and was written in this calendar day (UTC)**, reuse it; otherwise download to `path.tmp`, `fsync`, rename. |
| `-offline` | Use cached file or fail with `"not cached: <url>"`. |
| `-refresh` | Always download. |

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

## Pitfalls

- Do not read CSVs with `bufio.Scanner` line by line: quoted fields may
  contain `;` and, in TSE asset descriptions, even line breaks.
- Do not `strings.ToUpper` before decoding Latin-1 — decode first.
- `encoding/csv` with `ReuseRecord` reuses the slice: copy strings you keep
  (`strings.Clone`) when storing them past the next `Next()`.

## Done when

```bash
cd backend
go vet ./internal/fetch ./internal/csvx ./internal/textnorm
# Manual smoke run from a tiny throwaway main or `go run` snippet:
#   OpenZipMember(TSEBase+"bem_candidato/bem_candidato_2026.zip", "bem_candidato_2026_SC.csv")
# → logs ~4 ranged requests, writes the member to .cache, prints its first header line
ls -la .cache | head
```
