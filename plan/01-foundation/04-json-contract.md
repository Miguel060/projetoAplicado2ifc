# 04 — JSON contract (backend → frontend)

This file is **the** interface between the Go ETL and the React app. The Go
export structs (`backend/internal/export`) and the TypeScript types
(`frontend/src/data/types.ts`) are both written from it. Change this file
first, then both sides, in the same pull request, and bump `schemaVersion`
on any breaking change.

## Files and URLs

All paths are relative to the site origin.

| Path | Cache-Control | Content |
|---|---|---|
| `/data/latest.json` | `public, max-age=60` | Pointer to the live version. |
| `/data/<version>/index.json` | `public, max-age=31536000, immutable` | Run metadata, sources, all candidate summaries. |
| `/data/<version>/candidates/<slug>.json` | same, immutable | One full profile. |
| `/data/<version>/photos/<tseId>.jpg` | same, immutable | Official TSE photo. |
| `/data/<version>/dataset.sqlite` | same, immutable | The full dataset (open-data download). |

`<version>` is the UTC start time of the run, `YYYYMMDDTHHMMSSZ`
(e.g. `20261109T060000Z`). It sorts lexicographically = chronologically.

Rules:

- All JSON is UTF-8, `Content-Type: application/json; charset=utf-8`.
- Keys are `camelCase`. Absent values are `null`, never omitted (both sides
  can then rely on the shape).
- Money: number in BRL, 2 decimals. Shares: number in `[0, 1]`, 4 decimals.
- Dates: `YYYY-MM-DD`; timestamps: ISO-8601 with offset (`Z` or `-03:00`).
- Arrays are always present (empty `[]` rather than `null`) unless the whole
  block is `null` because it does not apply.

## TypeScript definition (the contract)

```ts
export type SchemaVersion = 1;

export type Office = "federal_deputy" | "state_deputy";

export type ResultCode =
  | "elected_by_quota"
  | "elected_by_average"
  | "elected"
  | "substitute"
  | "not_elected"
  | "no_result"
  | "other";

export type SelectionReason = "top_votes" | "born_in_municipality";

export type SourceId =
  | "tse_candidates"
  | "tse_votes"
  | "tse_assets"
  | "tse_finance"
  | "tse_social"
  | "tse_photos"
  | "camara_deputies"
  | "camara_bills"
  | "camara_votes"
  | "camara_expenses"
  | "cgu_amendments";

// ---------- /data/latest.json ----------
export interface LatestPointer {
  schemaVersion: SchemaVersion;
  version: string;          // "20261109T060000Z"
  generatedAt: string;      // ISO timestamp when the run finished
}

// ---------- shared ----------
export interface Source {
  id: SourceId;
  publisher: string;        // "TSE" | "Câmara dos Deputados" | "CGU – Portal da Transparência"
  name: string;             // pt-BR, e.g. "Candidatos – Eleições 2026"
  datasetUrl: string;       // human landing page
  fileUrl: string;          // exact file/endpoint fetched
  generatedAt: string | null; // when the publisher generated the data, if known
  fetchedAt: string;        // when the ETL downloaded it
}

export interface CandidateSummary {
  slug: string;             // "fulano-de-tal-1234"
  tseId: string;            // SQ_CANDIDATO
  ballotName: string;       // NM_URNA_CANDIDATO
  number: string;           // "1234" (federal) / "12345" (state)
  office: Office;
  party: { number: number; acronym: string; name: string };
  federation: { acronym: string; name: string } | null;
  result: { code: ResultCode; tseLabel: string };
  votes: {
    municipality: number;       // valid nominal votes in Videira
    municipalityShare: number;  // of all valid nominal votes for the office in Videira
    municipalityRank: number;   // competition rank among all candidates for the office in Videira
    state: number;              // valid nominal votes in SC
  };
  bornInMunicipality: boolean;
  selectionReasons: SelectionReason[];
  hasChamberRecord: boolean;    // was a federal deputy in legislature 57 (2023–2027)
  runningForReelection: boolean;
  photo: string | null;         // relative to the version dir: "photos/<tseId>.jpg"
  searchText: string;           // normalised: ballotName + fullName + number + party acronym
}

// ---------- /data/<version>/index.json ----------
export interface DataIndex {
  schemaVersion: SchemaVersion;
  version: string;
  generatedAt: string;
  election: {
    year: number;             // 2026
    round: number;            // 1
    date: string;             // "2026-10-04"
    uf: string;               // "SC"
    municipality: { name: string; ibgeCode: string; tseCode: string }; // "Videira", "4219309", "<resolved>"
  };
  selection: { topN: number; offices: Office[] };
  counts: Record<Office, number>;
  sources: Record<SourceId, Source>;
  dataset: { path: string; bytes: number; sha256: string }; // "dataset.sqlite"
  candidates: CandidateSummary[]; // sorted: office (federal first), then municipalityRank asc, then ballotName
}

// ---------- /data/<version>/candidates/<slug>.json ----------
export interface CandidateProfile {
  schemaVersion: SchemaVersion;
  version: string;
  summary: CandidateSummary;

  identity: {
    fullName: string;
    socialName: string | null;
    birthDate: string | null;
    ageAtInauguration: number | null;
    birthCity: string | null;
    birthUf: string | null;
    occupation: string | null;
    education: string | null;
    candidacyStatus: string;          // DS_SITUACAO_CANDIDATO_TOT: "DEFERIDO", …
    source: "tse_candidates";
  };

  votes: {
    municipality: {
      votes: number;
      share: number;
      rank: number;
      officeValidTotal: number;
      candidatesInOffice: number;     // how many candidates the rank is out of
    };
    state: { votes: number; share: number; officeValidTotal: number };
    source: "tse_votes";
  };

  assets: {
    declared: boolean;                // ST_DECLARAR_BENS
    total: number;
    items: { order: number; type: string; description: string; value: number }[]; // by order
    source: "tse_assets";
  };

  campaignFinance: {
    revenue: { total: number; byOrigin: { label: string; total: number }[] };   // desc by total
    expenses: { contractedTotal: number; byType: { label: string; total: number }[] }; // desc
    source: "tse_finance";
  };

  social: {
    links: { url: string; host: string }[];   // host = "instagram.com", for the visible label
    source: "tse_social";
  };

  /** null when the candidate has no record as federal deputy in 2023–2027. */
  chamber: ChamberActivity | null;

  /** null when chamber is null (only deputies author amendments in this scope). */
  amendments: Amendments | null;

  /** non-null only for state deputies running for re-election (elected state deputy in 2022). */
  stateAssembly: { profileUrl: string } | null;

  officialLinks: {
    tseCandidatePage: string | null;  // DivulgaCandContas deep link
  };
}

export interface ChamberActivity {
  deputyId: number;
  profileUrl: string;                 // https://www.camara.leg.br/deputados/<id>
  status: string | null;              // e.g. "Exercício"
  bills: {
    total: number;                    // all PL/PLP/PEC/PDL authored or co-authored since 2023-02-01
    byType: { type: string; count: number }[];
    items: Bill[];                    // most recent first, capped at 50
    source: "camara_bills";
  };
  rollCallVotes: {
    total: number;                    // number of roll-call votes the deputy took part in
    byVote: { vote: string; count: number }[]; // desc by count
    recent: RollCallVote[];           // most recent first, capped at 20
    source: "camara_votes";
  };
  expenses: {
    total: number;                    // CEAP net (vlrLiquido) since 2023
    byYear: { year: number; total: number }[];       // asc by year
    byCategory: { category: string; total: number }[]; // desc by total
    source: "camara_expenses";
  };
}

export interface Bill {
  id: number;
  type: string;           // "PL"
  number: number;
  year: number;
  summary: string;        // ementa, verbatim
  presentedAt: string;    // "2023-02-02"
  url: string;            // https://www.camara.leg.br/propostas-legislativas/<id>
}

export interface RollCallVote {
  votingId: string;
  date: string;           // "2025-06-10"
  description: string | null;
  vote: string;           // verbatim from the Câmara: "Sim", "Não", …
  url: string;
}

export interface Amendments {
  totals: {
    receivedInMunicipality: number;  // what beneficiaries in Videira received — the headline figure
    committed: number;               // whole amendments, all localities
    liquidated: number;
    paid: number;
  };
  items: Amendment[];     // desc by year, then by receivedInMunicipality
  source: "cgu_amendments";
}

export interface Amendment {
  code: string;
  year: number;
  type: string;           // verbatim "Tipo de Emenda"
  number: string;
  functionName: string | null;
  subfunctionName: string | null;
  committed: number;
  liquidated: number;
  paid: number;
  receivedInMunicipality: number;    // SUM(Valor Recebido) for beneficiaries in Videira
  beneficiaries: { name: string; received: number }[]; // desc by received; legal entities by name,
                                                        // natural persons only as "Pessoas físicas (valores agregados)"
  url: string | null;     // Portal da Transparência detail page; null (pattern not verifiable — bot protection)
}
```

## Example `latest.json`

```json
{ "schemaVersion": 1, "version": "20261109T060000Z", "generatedAt": "2026-11-09T06:02:41Z" }
```

## Example index entry (values illustrative)

```json
{
  "slug": "fulano-de-tal-1234",
  "tseId": "240002543385",
  "ballotName": "FULANO DE TAL",
  "number": "1234",
  "office": "federal_deputy",
  "party": { "number": 12, "acronym": "ABC", "name": "PARTIDO EXEMPLO" },
  "federation": null,
  "result": { "code": "elected_by_quota", "tseLabel": "ELEITO POR QP" },
  "votes": { "municipality": 3120, "municipalityShare": 0.1043, "municipalityRank": 1, "state": 101234 },
  "bornInMunicipality": false,
  "selectionReasons": ["top_votes"],
  "hasChamberRecord": true,
  "runningForReelection": true,
  "photo": "photos/240002543385.jpg",
  "searchText": "fulano de tal fulano de tal da silva 1234 abc"
}
```

## Compatibility rules

- The frontend checks `schemaVersion === 1` on `latest.json`; on mismatch it
  shows the generic error page ("dados em atualização") instead of crashing.
- Adding a **nullable** field or a new `SourceId` is non-breaking.
- Renaming/removing a field or changing a type is breaking → `schemaVersion: 2`,
  and deploy the frontend that understands v2 **before** publishing v2 data.
