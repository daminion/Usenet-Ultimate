# Patches

This fork adds one fix on top of upstream `usenet-ultimate` v1.4.1: missing
season/episode verification on search results, which let wrong-episode
releases through to the final stream list.

## The bug

Requesting a specific episode (e.g. S32E08) could return releases for
completely different episodes of the same show (S08E02, S07E08, S32E01,
S32E04, etc.) mixed into the results. Confirmed reproducible on:

- *TVShowExample1* S32E08 (large episode count — many season/episode digit
  collisions)
- *TVShowExample2* S04E01 and S03E06 (small, 4-season show — ruled out "only happens
  on shows with hundreds of episodes" as an explanation)

## Root cause

Three separate search code paths filtered by **show title only** and never
checked whether a candidate release's parsed season/episode actually matched
the season/episode being requested:

1. **`src/parsers/usenetSearcher.ts`** — primary Newznab text search
   (`searchTV`, the `if (method === 'text')` branch). The only season-aware
   check (`seasonOk`) only ran for absolute-numbered (anime-style) queries —
   normal SxxExx shows had no season or episode check at all, just title.
2. **`src/parsers/usenetSearcher.ts`** — ID-based search (tvdb/tvmaze/imdb).
   `season` and `ep` were sent as query parameters to the indexer's API, but
   the results that came back were used completely unfiltered — no
   client-side check that the indexer actually honored those parameters.
3. **`src/searchers/easynewsSearcher.ts`** — primary EasyNews text search,
   same shape as #1: title-matched, season/episode not checked.

By contrast, the season-pack and multi-season-fanout logic in both files
*already* validated season numbers correctly via `tagSeasonPack` — that's
what tipped this off. The individual-episode search paths just never got
the equivalent check.

`src/nzbdav/librarySearch.ts` (the WebDAV library pre-check) was audited and
found to already do this correctly via `findVideoFile(..., strictEpisodeMatch=true)`
plus a season pre-filter — no changes needed there.

## The fix

Added `isEpisodeMatch(releaseTitle, season, episode)` to
`src/parsers/titleMatching.ts`. It uses the existing
`@viren070/parse-torrent-title` parser (already a project dependency, used
elsewhere for season-pack detection) to extract a release's actual season
and episode numbers, and checks them against what was requested:

```typescript
export function isEpisodeMatch(releaseTitle: string, season: number,
