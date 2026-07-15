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
**My position:** New watchlist entries should default to `public=True`. This is a deliberate choice, not an inherited default.
**Reasoning:** The watchlist is a discovery/social feature — its value comes from being shareable (seeing what friends plan to watch, recommending films). Defaulting to public keeps the primary use case friction-free: a user who wants to share their list doesn't have to flip a setting first. This mirrors how comparable "want to watch" lists work on platforms like Letterboxd, where lists are shareable by default. The `public` field is per-entry and writable, so a user who wants privacy can already set it to `False`; the default only decides the starting point, not the ceiling on control.
**Tradeoff acknowledged:** The real cost is privacy-by-default: a user's watchlist is visible before they make any explicit sharing choice, which can surprise people who assume personal lists start private. I judged that acceptable because a watchlist is low-sensitivity data (films someone intends to watch, not viewing history or ratings) and the feature is fundamentally about sharing. If we later add higher-sensitivity lists, or if user research shows people expect privacy-first, the honest move is to flip the default to `public=False` (opt-in sharing) rather than rely on users discovering the toggle. Flagging this so the default is a recorded decision we can revisit, not an accident.

## Comment 5 — Sort order
I'd prefer watchlists to default to "date added" order rather than alphabetical. Most users want to see what they added recently. I'm open to discussion if you see it differently — but let's make a decision and document it.
**My position:** Keep the default sort alphabetical by title (the current `get_watchlist` behavior).
**Reasoning:** A watchlist is something users return to in order to *find a specific film to watch* — "what do I want to put on tonight?" Alphabetical order makes a title predictable to locate: you scan to roughly where it should be and it's there. Date-added order optimizes for a different task (seeing what you just added), but for finding a known title it means scrolling the whole list, since position depends on when it was added rather than anything the user can predict. As a watchlist grows, alphabetical keeps lookup roughly constant while date-added gets progressively harder to scan.
**Engagement with reviewer's point:** The reviewer's point is fair — recent-first is genuinely better for the "what did I just add" moment, and that's a real use case. Where we differ is which task the *default* should optimize for: I'm weighting find-a-title (repeated, happens every time you pick something to watch) over review-recent-additions (occasional, right after adding). Since the field and query are easy to change, this isn't a one-way door — a good follow-up would be to let the client pass a sort parameter (`?sort=recent`) so both are supported and the default stops being a forced choice. Happy to switch the default to date-added if we'd rather align with the collection view, but documenting alphabetical as the intentional decision here.

## Comment 6 — Rebase
A refactor merged to main that changed film IDs from integers to UUIDs. Your watchlist code still references integer IDs. Please rebase on main and update accordingly.
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->