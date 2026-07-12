## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in both its definition and all call sites.
**How I verified:**

## Comment 2 — Deduplication
**What I did:** Added deduplication logic to `add_to_watchlist()` by accounting for existing films in watchlists and adding a new exception `AlreadyInWatchlistError` to catch in the watchlist route.
**How I verified:**

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, following the same fixture and assertion structure as `test_add_to_collection_nonexistent_film_raises` in `test_collection.py`. Added `app`, `sample_user`, and `sample_film` fixtures and wrote `test_add_to_watchlist_nonexistent_film_raises`, which asserts that calling `add_to_watchlist()` with a nonexistent `film_id` raises `FilmNotFoundError`.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — 1 passed.

## Comment 4 — Default visibility
**My position:** Defaulting to public=True is the right choice in the context of this project. 
**Reasoning:** A film collection and watchlist app inherently contains a social aspect (being able to see what friends would like to watch and/or have already watched). Setting it to private by default would make it so that this feature is broken for the majority of users, unless they were to toggle it on in some settings.
**Tradeoff acknowledged:** Of course this does come with some privacy concerns for users who would like their collections private, but that shouldn't be much of an issue if users are allowed to easily toggle this back to their preference.

## Comment 5 — Sort order
**My position:** Keep alphabetical-by-title for the watchlist; collection stays date-added descending.
**Reasoning:** Collection is a record of the past: "when I watched it" is meaningful history, so newest-first makes sense. A watchlist, by contrast, has no priority field in the schema; recency of the *add* action is not the same thing as intent to watch soon, it's just an artifact of when a user happened to click. CineLog's watchlist is used more as a reference list — "did I already add Dune?" — which alphabetical serves better than a timeline would.
**Engagement with reviewer's point:** The maintainer's push for date-added likely rests on two arguments, and I want to engage with both directly rather than dismiss them:
1. *Recency signals priority.* This holds up for a single deliberate add, but breaks down under a realistic use case: a user adding ten films in one sitting from a "best sci-fi" list. In that case date-added order just reflects click order within that session, not any real preference — it's a weaker signal than it first appears, and the schema has no actual priority/ranking field to back it up.
2. *Consistency with `get_collection()`.* I accept this as a real cost, not a non-issue — the two endpoints will sort differently, so any future shared UI component has to accept sort order as a parameter rather than assuming identical behavior. I'm accepting that cost because collection and watchlist represent different semantic categories (a log of the past vs. a list to consult), so I don't think forcing identical ordering is worth the browsability we'd give up.

I'm also acknowledging two tradeoffs the original response didn't cover:
- **Performance:** alphabetical requires `.join(Film)` to sort on `Film.title`, which is more expensive than sorting directly on `WatchlistEntry.date_added`. I'm accepting this because a single user's watchlist is small in practice (nowhere near table-scan territory), so it's not the right place to optimize.

## Comment 6 — Rebase
**What conflicted:** `.gitignore` had a textual conflict (both branches added the file independently with different entries). But the more important issue was `models.py`, which did *not* show up as a conflict on the first rebase attempt: main's `refactor: migrate film IDs from integer to UUID` commit deleted the `WatchlistEntry` class entirely (it doesn't exist on main yet), while my branch's commits had never touched `models.py` since the shared root commit — `WatchlistEntry` was part of the original starter scaffold, not something I'd edited. Since my side had zero diff against the common ancestor for that file, git's 3-way merge silently took main's version with no conflict markers, quietly deleting `WatchlistEntry` from my branch. I only caught this by noticing `watchlist_service.py` still imported a class that no longer existed in `models.py`.
**How I resolved it:** First rebase pass: merged the `.gitignore` entries from both branches by hand. For `models.py`, since the deletion caused no real conflict to resolve against, I reset back to my pre-rebase branch tip (`git reset --hard ORIG_HEAD`) and made a small commit that edited a line inside the `WatchlistEntry` class before rebasing again — this turned main's deletion into an explicit modify/delete conflict that git had to stop and flag. On the second rebase attempt, I resolved that conflict by keeping the `WatchlistEntry` class and updating `film_id` from `db.Integer` to `db.String(36)` to match the UUID refactor (`Film.id` and `CollectionEntry.film_id` had already been migrated on main). I also updated stale docstrings/comments in `watchlist_service.py` and `routes/watchlist/watchlist.py` that still described `film_id` as an integer.
**How I verified no conflict remains:** Searched the repo for leftover conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) — none found. Ran `pytest tests/ -v` — all 5 tests pass, confirming `models.py` imports cleanly and the UUID types are consistent end-to-end. Checked `git log --merges --oneline HEAD` — the only merge commit reachable from HEAD is one already on `origin/main` (from a prior, unrelated PR), not one introduced by my rebase, so the branch's own commit history stays linear per `CONTRIBUTING.md`.

## AI Usage
Used Claude Code for a few specific debugging tasks during this project:
- Diagnosed a `fixture 'app' not found` pytest error in `test_watchlist.py`, tracing it to missing fixtures that `test_collection.py` had defined locally.
- Caught that the `watchlist` route wasn't catching `FilmNotFoundError`/`AlreadyInWatchlistError`, so both would have surfaced as 500s instead of 404/409.
- During the rebase, traced why `models.py`'s `WatchlistEntry` class silently disappeared with no conflict markers — it turned out my branch had never touched `models.py` since the shared root commit, so git's 3-way merge took main's (post-refactor, watchlist-less) version with nothing to flag. Used that diagnosis to redo the rebase in a way that forced a real, resolvable conflict instead of a silent deletion, then fixed the resulting `film_id` UUID type mismatch.


## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->