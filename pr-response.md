# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
save_to_watchlist() should follow the project's naming convention. Compare with add_to_collection() — the pattern here is verb_to_noun. Please rename to add_to_watchlist() and update all call sites.
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` convention (consistent with `add_to_collection()`). Updated the only call site — the import and invocation in `routes/watchlist/watchlist.py`. A repo-wide grep confirms no other references existed.
**How I verified:** `grep -rn "save_to_watchlist"` returns nothing; both edited files compile (`python -m py_compile`) and import cleanly; the existing `tests/` suite still passes.

## Comment 2 — Deduplication
What happens if a user calls this with a film that's already on their watchlist? The current implementation would add a duplicate entry. Please handle this case.
**What I did:** Mirrored the two-layer dedup approach already used for collections. (1) Added a `UniqueConstraint("user_id", "film_id")` to the `WatchlistEntry` model — it previously lacked the constraint `CollectionEntry` has, so duplicates were possible at the DB level. (2) Added an `AlreadyInWatchlistError` exception and a pre-insert check in `add_to_watchlist()` so a duplicate raises a clean domain error instead of a DB IntegrityError. (3) Updated the `POST /watchlist/<user_id>/add` route to catch it and return HTTP 409, matching the collection route's behavior.
**How I verified:** Adding the same film twice now raises `AlreadyInWatchlistError` and only one row persists; the endpoint returns 409 on the second add. Confirmed the models/service/route compile and the existing suite still passes.

## Comment 3 — Missing test
Please add a test for the case where film_id doesn't exist in the database. Look at the existing tests in test_collection.py — the pattern is there. Create a new file tests/test_watchlist.py. Read tests/test_collection.py and find test_add_to_collection_nonexistent_film_raises — write the equivalent test for add_to_watchlist() following the same fixture and assertion structure.
**What I did:** Created `tests/test_watchlist.py` with the same `app`, `sample_user`, and `sample_film` fixtures as `test_collection.py`. Added `test_add_to_watchlist_nonexistent_film_raises`, the direct equivalent of `test_add_to_collection_nonexistent_film_raises`: it calls `add_to_watchlist()` with a film_id that was never inserted and asserts `FilmNotFoundError` is raised inside `pytest.raises(...)`. (Since film IDs are still integers on this branch, the fake id is `999999` rather than a UUID string — this becomes a UUID in Comment 6.)
**How I verified:** `pytest tests/test_watchlist.py` passes, and the full `tests/` suite stays green.

## Comment 4 — Default visibility
I notice watchlists default to public=True. We don't have a documented decision on default visibility for user lists. Before I can approve this, I need you to add a note to your PR description explaining your reasoning. I want to make sure we're being intentional here, not just inheriting a default.
**My position:** 
**Reasoning:** 
**Tradeoff acknowledged:** 

## Comment 5 — Sort order
I'd prefer watchlists to default to "date added" order rather than alphabetical. Most users want to see what they added recently. I'm open to discussion if you see it differently — but let's make a decision and document it.
**My position:** 
**Reasoning:** 
**Engagement with reviewer's point:** 

## Comment 6 — Rebase
A refactor merged to main that changed film IDs from integers to UUIDs. Your watchlist code still references integer IDs. Please rebase on main and update accordingly.
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->