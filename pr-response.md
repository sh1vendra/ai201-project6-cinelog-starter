# PR Response Doc — CineLog Watchlist Feature

## AI Usage

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
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
