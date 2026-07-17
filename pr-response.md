# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used AI (Claude Code) at three distinct points in this PR, in three different ways:

1. **Codebase orientation before touching any code.** Before making any changes, I had it read `models.py`, `services/collection_service.py`, and `tests/test_collection.py` and summarize `add_to_collection()`'s control flow, how deduplication is enforced (app-level check + DB-level `UniqueConstraint`), and the fixture/assertion patterns used in the test suite. This was read-only orientation — no code was written at this stage — so I understood the existing conventions before asking for anything to be applied to the watchlist code.
2. **Mechanical execution of explicit instructions.** For the rename (Comment 1), test scaffolding (Comment 3), and rebase conflict resolution (Comment 6), I gave specific, scoped instructions — e.g. "rename `save_to_watchlist` to `add_to_watchlist`, find every call site, update them" — and had the tool carry out the mechanical steps: grepping for call sites, applying the same edit in each location, writing tests that mirror an existing test's structure, resolving the `models.py` merge conflict per the pattern already used in `7c37bcd`. In each case I reviewed the diff and the `pytest` output before moving on.
3. **Independent reasoning on the two design-decision comments.** For Comment 4 (default visibility) and Comment 5 (sort order), I decided my position first — private-by-default, and date-added sort — and only then used the tool to stress-test the reasoning and tradeoffs behind decisions I'd already made, rather than asking it to generate a position from scratch. The "My position" stated in each of those sections is mine; the tool's role there was pressure-testing, not deciding.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention used elsewhere (e.g. `add_to_collection()`). Before renaming, I ran a project-wide search (`grep -rn "save_to_watchlist" --include="*.py" .`) to find every call site. It turned up two other references, both in `routes/watchlist/watchlist.py`: the import statement and the call inside the `add_film` view. I updated both to use the new name. A follow-up grep confirmed zero remaining references to the old name anywhere in the project.
**How I verified:** Ran `pytest tests/ -v` after the rename to confirm nothing broke:

```
============================= test session starts ==============================
platform darwin -- Python 3.14.5, pytest-9.1.1, pluggy-1.6.0 -- .../ai201-project6-cinelog-starter/.venv/bin/python3.14
cachedir: .pytest_cache
rootdir: /Users/shivendrabhagat/ShivDon/Codepath Projects/ai201-project6-cinelog-starter
collecting ... collected 4 items

tests/test_collection.py::test_add_to_collection_creates_entry PASSED    [ 25%]
tests/test_collection.py::test_add_to_collection_duplicate_raises PASSED [ 50%]
tests/test_collection.py::test_add_to_collection_nonexistent_film_raises PASSED [ 75%]
tests/test_collection.py::test_get_collection_returns_newest_first PASSED [100%]

============================== 4 passed in 0.19s ===============================
```

All 4 collection tests still pass. Note: there are no watchlist-specific tests yet, so this only confirms the rename didn't regress the collection feature — it doesn't exercise `add_to_watchlist()` directly (see Comment 3).

## Comment 2 — Deduplication
**What I did:** Added deduplication to `add_to_watchlist()` in `services/watchlist_service.py`, following the two-layer pattern already used by `add_to_collection()`: an app-level check plus a DB-level constraint. At the app level, I added a new `AlreadyInWatchlistError` exception and, before inserting, a `WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()` lookup that raises it if a match is found. At the DB level, I added `UniqueConstraint("user_id", "film_id")` to the `WatchlistEntry` model in `models.py`, matching the one `CollectionEntry` already has. I didn't copy this verbatim, though: I gave it a distinct constraint name, `unique_user_film_watchlist`, instead of reusing `CollectionEntry`'s `unique_user_film_collection` — constraint names must be unique within the database, so reusing the same name across two different tables would collide once both exist.
**How I verified:** Ran `pytest tests/ -v` after making the change:

```
============================= test session starts ==============================
platform darwin -- Python 3.14.5, pytest-9.1.1, pluggy-1.6.0 -- .../ai201-project6-cinelog-starter/.venv/bin/python3.14
cachedir: .pytest_cache
rootdir: /Users/shivendrabhagat/ShivDon/Codepath Projects/ai201-project6-cinelog-starter
collecting ... collected 4 items

tests/test_collection.py::test_add_to_collection_creates_entry PASSED    [ 25%]
tests/test_collection.py::test_add_to_collection_duplicate_raises PASSED [ 50%]
tests/test_collection.py::test_add_to_collection_nonexistent_film_raises PASSED [ 75%]
tests/test_collection.py::test_get_collection_returns_newest_first PASSED [100%]

============================== 4 passed in 0.15s ===============================
```

All 4 existing tests still pass, but I want to be explicit: none of these tests exercise `add_to_watchlist()` or the new `AlreadyInWatchlistError` path directly — there was no watchlist-specific test file yet. This pass only confirms the change didn't regress the collection feature. Comment 3 addresses adding real coverage for this dedup logic.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled directly on the fixture and assertion patterns already established in `tests/test_collection.py`. It reuses the same `app`, `sample_user`, and `sample_film` fixture structure (isolated in-memory DB per test, ids returned rather than ORM objects). I added `test_add_to_watchlist_nonexistent_film_raises`, mirroring `test_add_to_collection_nonexistent_film_raises` — calls `add_to_watchlist()` with a fake UUID and asserts `FilmNotFoundError` is raised via `pytest.raises`. I also added `test_add_to_watchlist_duplicate_raises`, mirroring `test_add_to_collection_duplicate_raises` — adds a film to the watchlist, then adds it again and asserts `AlreadyInWatchlistError` is raised, followed by a `WatchlistEntry.query.filter_by(...).count() == 1` check proving the failed second call had no side effect. This gives the Comment 2 dedup logic (both the app-level check and the `AlreadyInWatchlistError` exception) direct test coverage for the first time.
**How I verified:** Ran the new file in isolation, then the full suite:

```
============================= test session starts ==============================
platform darwin -- Python 3.14.5, pytest-9.1.1, pluggy-1.6.0 -- .../ai201-project6-cinelog-starter/.venv/bin/python3.14
cachedir: .pytest_cache
rootdir: /Users/shivendrabhagat/ShivDon/Codepath Projects/ai201-project6-cinelog-starter
collecting ... collected 2 items

tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises PASSED [ 50%]
tests/test_watchlist.py::test_add_to_watchlist_duplicate_raises PASSED   [100%]

============================== 2 passed in 0.25s ===============================
```

```
============================= test session starts ==============================
platform darwin -- Python 3.14.5, pytest-9.1.1, pluggy-1.6.0 -- .../ai201-project6-cinelog-starter/.venv/bin/python3.14
cachedir: .pytest_cache
rootdir: /Users/shivendrabhagat/ShivDon/Codepath Projects/ai201-project6-cinelog-starter
collecting ... collected 6 items

tests/test_collection.py::test_add_to_collection_creates_entry PASSED    [ 16%]
tests/test_collection.py::test_add_to_collection_duplicate_raises PASSED [ 33%]
tests/test_collection.py::test_add_to_collection_nonexistent_film_raises PASSED [ 50%]
tests/test_collection.py::test_get_collection_returns_newest_first PASSED [ 66%]
tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises PASSED [ 83%]
tests/test_watchlist.py::test_add_to_watchlist_duplicate_raises PASSED   [100%]

============================== 6 passed in 0.19s ===============================
```

Both new tests pass in isolation and the full suite (6 tests total) passes with no regressions.

## Comment 4 — Default visibility
**My position:** Private by default.
**Reasoning:** A watchlist represents intent, not achievement, it's the "things I haven't watched yet" list, not a curated showcase like the collection. People are more self-conscious about unfinished or aspirational items than about things they've already watched and can vouch for.
**Tradeoff acknowledged:** Private-by-default weakens the social discovery angle, since most users never go back to toggle visibility settings after signup, so fewer watchlists end up publicly visible than would under a public default. The tradeoff is accepted because avoiding accidental exposure of embarrassing or aspirational picks outweighs the loss in social reach for this specific feature.

## Comment 5 — Sort order
**My position:** Date-added (matching the maintainer's preference).
**Reasoning:** A watchlist answers "what should I watch next," and the most recently added item is usually the one a user is most excited about right now, often because they just heard about it, saw a trailer, or got a recommendation. Timeline reflects intent better than alphabetical order does for this use case.
**Engagement with reviewer's point:** Agreed with dev-lead's push for date-added over alphabetical. Alphabetical is a lookup structure, useful when you already know the title and want to find it fast, but a watchlist is meant to be browsed for what's next, not searched by name, so recency serves the feature's actual purpose better.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` then `git rebase origin/main`. Two conflicts came up. First, `.gitignore` — an add/add conflict, since main had already added its own `.gitignore` (via `718a9a8`) that included `.pytest_cache/`, which our branch's version didn't have. Second, and expected, `models.py` — main's `07ca580` migrated `Film.id` from integer to UUID (`db.String(36)`), but `WatchlistEntry.film_id` (added on this branch) was still declared as `db.Integer`, so it no longer matched the foreign key it pointed to.
**How I resolved it:** For `.gitignore`, merged both versions so `.pytest_cache/` is retained alongside the rest of our entries. For `models.py`, changed `WatchlistEntry.film_id` from `db.Integer` to `db.String(36)`, following the exact same pattern already applied to `CollectionEntry.film_id` in commit `7c37bcd`. After the rebase finished, I also grepped for lingering integer-era references and found two stale docstring/comment spots that predated the migration: the `film_id (int): ... (Note: integer — pre-refactor)` docstring line in `services/watchlist_service.py`, and the `Body: { "film_id": <int> }` comment in `routes/watchlist/watchlist.py`. Updated both to describe `film_id` as a UUID string, consistent with the actual schema.
**How I verified no conflict remains:**

```
============================= test session starts ==============================
platform darwin -- Python 3.14.5, pytest-9.1.1, pluggy-1.6.0 -- .../ai201-project6-cinelog-starter/.venv/bin/python3.14
cachedir: .pytest_cache
rootdir: /Users/shivendrabhagat/ShivDon/Codepath Projects/ai201-project6-cinelog-starter
collecting ... collected 6 items

tests/test_collection.py::test_add_to_collection_creates_entry PASSED    [ 16%]
tests/test_collection.py::test_add_to_collection_duplicate_raises PASSED [ 33%]
tests/test_collection.py::test_add_to_collection_nonexistent_film_raises PASSED [ 50%]
tests/test_collection.py::test_get_collection_returns_newest_first PASSED [ 66%]
tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises PASSED [ 83%]
tests/test_watchlist.py::test_add_to_watchlist_duplicate_raises PASSED   [100%]

============================== 6 passed in 0.19s ===============================
```

```
$ git log --oneline --merges origin/main..HEAD
(no output)
```

All 6 tests pass post-rebase, and the empty output from `git log --oneline --merges origin/main..HEAD` confirms the rebase replayed every commit linearly with no merge commits introduced.

## PR Description

### What this feature does

Adds a watchlist to CineLog — a list of films a user wants to watch, separate from their collection (films already watched). The watchlist supports:

- `add_to_watchlist(user_id, film_id)` — adds a film to a user's watchlist. Raises `FilmNotFoundError` if the film doesn't exist, and `AlreadyInWatchlistError` if the film is already on that user's watchlist (enforced both at the application level and via a DB-level `UniqueConstraint` on `(user_id, film_id)`).
- `get_watchlist(user_id)` — returns all films on a user's watchlist as a list of dicts, each with `date_added` and `public` attached, sorted newest-added first.
- `GET /watchlist/<user_id>` and `POST /watchlist/<user_id>/add` endpoints exposing the above.

### Design decisions

- **Visibility should default to private.** A watchlist is aspirational ("things I haven't watched yet"), not a curated showcase like the collection, so it should default to `public=False` rather than exposing it by default. **Note:** `WatchlistEntry.public` in `models.py` is currently still `default=True` — this decision is documented here but the model default has not yet been flipped to match. Flagging as a follow-up before merge.
- **Sort order is date-added, newest first.** A watchlist answers "what should I watch next," and the most recently added film is usually the one the user is most excited about right now, so recency-first serves that purpose better than an alphabetical listing.

### Manual testing instructions

Run the automated suite:

```
pytest tests/ -v
```

To exercise the feature manually against a running instance:

```bash
# start the app
python app.py

# create a user and a film first (via existing endpoints/DB), then:

# add a film to a user's watchlist
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'
# → 201, returns the new WatchlistEntry as JSON

# try adding the same film again to confirm dedup
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'
# → should fail with AlreadyInWatchlistError (raised, not yet mapped to a specific HTTP status/handler)

# view the watchlist and confirm sort order
curl http://127.0.0.1:5000/watchlist/<user_id>
# → JSON array of films; add a second film after the first and confirm it appears first (newest-added-first)
```

To exercise dedup and the nonexistent-film case directly via pytest:

```
pytest tests/test_watchlist.py -v
```
