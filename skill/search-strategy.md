# Search strategy: where your scout looks, and why

## Queries

```bash
# Traces to fit profile sentence 1 ("I prioritize C++ desktop applications, libraries, and systems/tooling projects...") and sentence 3 ("bounded bug fixes and small behavioral or accessibility improvements"): newcomer-labeled C++ issues across GitHub.
gh search issues --label "good first issue" --language "c++" --state open --limit 100 --json title,repository,labels,assignees,commentsCount,updatedAt,url

# Traces to fit profile sentence 1 (C++ priority) and sentence 3 (bounded bug fixes): maintainer-invited C++ bug reports that may use "help wanted" instead of "good first issue".
gh search issues --label "help wanted" --label "bug" --language "c++" --state open --limit 100 --json title,repository,labels,assignees,commentsCount,updatedAt,url

# Traces to fit profile sentence 1 (C++ Qt/desktop GUI tools) and sentence 3 ("small behavioral or accessibility improvements"): C++ accessibility issues with newcomer invitations.
gh search issues --label "good first issue" --label "accessibility" --language "c++" --state open --limit 50 --json title,repository,labels,assignees,commentsCount,updatedAt,url

# Traces to fit profile sentence 2 ("I am also comfortable in Python ... as a secondary track when C++ candidates are thin"): Python fallback pool.
gh search issues --label "good first issue" --language python --state open --limit 50 --json title,repository,labels,assignees,commentsCount,updatedAt,url
```

## Ranking filters

Apply these conditions and orderings in sequence using only the returned JSON fields (`title`, `repository`, `labels`, `assignees`, `commentsCount`, `updatedAt`, `url`) and the query track that surfaced each item:

1. **Drop assigned issues:** discard any result where `assignees` is non-empty (`len(assignees) > 0`).
2. **Drop stale issues:** discard any result whose `updatedAt` timestamp is more than 60 days older than the scan date.
3. **Drop crowded threads:** discard any result with `commentsCount > 8`, since long threads on newcomer issues usually indicate active claims or unresolved design debate.
4. **Drop non-behavioral polish:** discard any result whose `labels` contain `documentation`, `docs`, or `typo` (unless `labels` also contains `bug` or `accessibility`), matching the fit profile's avoidance of copy/styling-only work.
5. **Order by language fit:** rank results surfaced by the C++ queries ahead of results surfaced only by the Python fallback query.
6. **Order by scope/invitation label signals:** within the same language tier, count how many labels match `{"good first issue", "good-first-issue", "help wanted", "bug", "accessibility"}` and rank higher counts first.
7. **Order by quietness:** prefer fewer comments (`commentsCount` ascending) to reduce the chance of an unrecorded claim in the thread.
8. **Deterministic tie-breaker:** break remaining ties by most recent activity (`updatedAt` descending), and finally by `url` ascending.

## How many to grade

Grade the top **5** ranked results per scan.
