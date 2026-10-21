# 03.04 — Selection rule ("linked to Videira")

## Goal

Implement decision **D1** exactly, deterministically, in one place:

> For each office (federal deputy, state deputy), the **20** candidates with
> the most **valid nominal votes in Videira**, **plus** every candidate for
> those offices **born in Videira** whose candidacy is `DEFERIDO`. Ties at
> position 20 are all included.

## Files to create

```
backend/internal/selection/selection.go
```

## Types and signature

```go
package selection

type Reason string
const (
	TopVotes             Reason = "top_votes"
	BornInMunicipality   Reason = "born_in_municipality"
)

type Selected struct {
	C                 *tse.Candidate // pointer into the loaded map
	VotesMunicipality int
	VotesState        int
	ShareMunicipality float64 // votes / office municipality total, rounded to 4 decimals
	ShareState        float64
	RankMunicipality  int     // competition rank among ALL candidates of the office in the municipality
	Reasons           []Reason // sorted: top_votes before born_in_municipality
}

type Result struct {
	List []*Selected // deterministic order (see below)
}

func Select(cands tse.Candidates, votes tse.Votes, topN int) (*Result, error)
func (r *Result) IDs() map[string]bool
func (r *Result) ForgetCPFs() // sets C.CPF = "" for every candidate it references
```

## Algorithm

1. For each office `o ∈ {federal_deputy, state_deputy}`:
   1. Build the ranking population: **every** candidate of office `o` that has
      rows in the municipality (from `votes.ByCandidate`, including those with
      0 valid votes).
   2. Sort by `VotesMunicipality` desc, then `BallotName` asc, then `TSEID` asc
      (the tie-breakers only make the order stable; they never decide
      inclusion).
   3. Assign **competition ranks** ("1224"): rank = 1 + number of candidates
      with strictly more votes.
   4. `cutoff` = votes of the candidate at position `topN` (1-based) in the
      sorted list; if the population is smaller than `topN`, everyone is in.
   5. `top_votes` = every candidate with `VotesMunicipality >= cutoff`
      **and** `VotesMunicipality > 0`. This includes everyone tied at the
      cutoff, so the group may exceed 20.
2. `born_in_municipality` = every candidate of offices 6/7 with
   `textnorm.Norm(BirthCity) == textnorm.Norm(config.MunicipalityName)`,
   `BirthUF == config.UF` and `CandidacyStatus == "DEFERIDO"` — regardless of
   votes (they get their real rank, possibly low).
3. Union by `TSEID`; merge reasons.
4. Shares: `ShareMunicipality = VotesMunicipality / OfficeTotals[o].Municipality`
   (0 if the total is 0), likewise for the state. Round to 4 decimals.
5. Output order: office (`federal_deputy` first), then `RankMunicipality`
   asc, then `BallotName` asc. This is also the order of `index.json`.

```go
func competitionRanks(sorted []*entry) {
	for i, e := range sorted {
		if i > 0 && e.votes == sorted[i-1].votes {
			e.rank = sorted[i-1].rank
		} else {
			e.rank = i + 1
		}
	}
}

func cutoff(sorted []*entry, topN int) int {
	if len(sorted) <= topN {
		return 1 // everyone with at least one vote
	}
	return sorted[topN-1].votes
}
```

## Expected result (data of 2026-10-21)

| Office | 20th place | 21st place | `top_votes` | `born_in_municipality` (DEFERIDO) |
|---|---|---|---|---|
| Federal deputy | 288 votes | 234 votes | 20 (no tie) | 1 |
| State deputy | 112 votes | 107 votes | 20 (no tie) | 2 |

Born in Videira across SC deputy candidates: 4 (2 federal, 2 state); one
federal candidate is `INDEFERIDO` and is therefore excluded. Expected total:
**40–43** profiles (fewer if a born-in-Videira candidate is also top 20).

## Why these exact choices

| Choice | Reason |
|---|---|
| Valid nominal votes (`QT_VOTOS_NOMINAIS_VALIDOS`) | The official count; annulled votes do not elect anyone. |
| Ties all included | Excluding one of two candidates with the same votes would be an arbitrary, non-neutral choice. |
| `> 0` votes for `top_votes` | In a tiny population, "top 20" must not pull in zero-vote candidates. |
| `DEFERIDO` for birthplace | A candidacy that was rejected is not a candidacy the citizen could vote for. |
| Rank among *all* candidates | "3rd most voted in Videira" must mean the same thing for every profile. |

## Pitfalls

- Ranking only the selected candidates (instead of the full population)
  makes born-in-Videira ranks wrong.
- Floating-point shares: compute from integers, round once at the end.
- `BirthCity` may be `nil` (`-3` in the file): never dereference blindly.

## Done when

```bash
DS=backend/dist/data/$(jq -r .version backend/dist/data/latest.json)/dataset.sqlite
sqlite3 "$DS" "SELECT c.office, COUNT(*) FROM candidate c GROUP BY c.office;"
sqlite3 "$DS" "SELECT c.office, v.rank, v.votes, s.reasons, c.ballot_name
               FROM candidate c
               JOIN vote_total v ON v.tse_id=c.tse_id AND v.scope='municipality'
               JOIN selection s ON s.tse_id=c.tse_id
               ORDER BY c.office, v.rank;"
# The 20th row per office must have votes >= the 21st candidate in the full file,
# and every born_in_municipality row must have candidacy_status = 'DEFERIDO'.
```
