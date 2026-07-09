# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used AI (Claude, via Claude Code) for two things on this project:

1. **Orientation.** Before touching any review comment, I had it summarize `models.py`, `services/collection_service.py`, and `tests/test_collection.py` so I could confirm the naming convention (`add_to_collection` / `remove_from_collection` / `get_collection`), the dedup pattern (query for an existing entry before insert, raise a dedicated `AlreadyIn...Error`), and the test fixture structure (`app` → `sample_user` → `sample_film`, one `app.app_context()` block per test). I verified all of this against the actual file contents rather than trusting the summary blindly — e.g., I caught that the AI's first read of `get_watchlist()` assumed it used `entry.film` correctly, when in fact the `Film` model was missing a `watchlist_entries` relationship (a pre-existing bug I fixed, see Comment 6/rebase notes).
2. **Stress-testing my Comment 5 reasoning.** After drafting my sort-order argument, I asked what counterargument a reviewer might raise. It correctly pointed out that switching to date-added changes existing client-facing ordering behavior for anyone already polling the endpoint expecting alphabetical order. I addressed this directly in my final response below rather than ignoring it.

I did not ask AI to write the dedup logic, the design responses, or the commit messages — those are my own work, checked against the codebase.

## Comment 1 — Rename

**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, matching the `add_to_collection()` / `add_to_watchlist()` naming convention used everywhere else in the service layer (`add_to_collection`, `remove_from_collection`, `get_collection`). Updated the one call site in `routes/watchlist/watchlist.py` (`add_film` route handler) and its import statement.

**How I verified:** Ran a project-wide search for `save_to_watchlist` after the rename (`grep -rn save_to_watchlist`) and confirmed zero remaining references. Ran the full test suite (`pytest tests/ -v`) to confirm nothing else depended on the old name.

## Comment 2 — Deduplication

**What I did:** Added the same duplicate-check pattern `add_to_collection()` uses: query for an existing `WatchlistEntry` with the same `(user_id, film_id)` pair before inserting, and raise a new `AlreadyInWatchlistError` if one exists. Added the matching exception class right above the function, following the same layout as `FilmNotFoundError` / `AlreadyInCollectionError` in `collection_service.py`. Wired the new exception into the route handler with a 409 response, mirroring `routes/collection.py`'s `add_film` error handling.

**How I verified:** Wrote `test_add_to_watchlist_duplicate_raises` (mirrors `test_add_to_collection_duplicate_raises`) confirming a second add raises `AlreadyInWatchlistError` and that only one row exists in the table afterward. Deduplication lives in the service layer (not the route or the DB schema) because that's where `add_to_collection()` puts it — keeping the check next to the business rule it enforces, and keeping routes as thin request/response translators. The `CollectionEntry` model backs this up with a DB-level `UniqueConstraint`; I did not add an equivalent constraint to `WatchlistEntry` since the review comment scoped this to the service-layer check and adding a schema constraint wasn't part of any comment — flagging it here as a follow-up worth a separate PR rather than sneaking in an out-of-scope migration.

## Comment 3 — Missing test

**What I did:** Created `tests/test_watchlist.py` using `test_add_to_collection_nonexistent_film_raises` from `tests/test_collection.py` as the template: same `app` / `sample_user` / `sample_film` fixtures, same `app.app_context()` block structure, same "fake UUID that isn't in the DB" approach (`"00000000-0000-0000-0000-000000000000"`) rather than an arbitrary integer, since film IDs are UUIDs on main. Wrote `test_add_to_watchlist_nonexistent_film_raises` asserting `FilmNotFoundError` is raised.

**How I verified:** `pytest tests/test_watchlist.py -v` — passes. Also ran the full suite to confirm the new file doesn't collide with anything in `test_collection.py` (separate fixtures, separate module).

## Comment 4 — Default visibility

**My position:** Keep `public=True` as the default, but add an explicit `public` parameter (stretch feature) so callers who want a private entry can opt out.

**Reasoning:** CineLog is presented as a social film-tracking app — the whole point of a watchlist feature next to a collection feature is to let people see what others want to watch, not just what they've already watched. A `public=False` default would mean every watchlist entry is invisible to other users unless the client explicitly flips a flag most callers won't know to set, which silences the feature's core social value for the vast majority of adds by default. The collection feature doesn't have a visibility flag at all — everything a user has logged is implicitly part of their profile — so `public=True` for the watchlist is the closer match to the platform's existing behavior, not a new precedent.

**Tradeoff acknowledged:** The real cost of `True`-by-default is that a user who wants to keep an embarrassing or spoiler-heavy watchlist entry private has to remember to pass `public=False` on every add, and a client that forgets exposes something the user may not have wanted shared. That's a real privacy footgun. I mitigated it the only way available at this layer — by adding the `public` parameter to `add_to_watchlist()` and the `/watchlist/<user_id>/add` endpoint (previously the field existed on the model but nothing ever set it) — so the choice is at least possible per-entry instead of requiring a follow-up "update visibility" endpoint. If usage data later showed users frequently wanted private-by-default, that's a product decision to revisit, but nothing in the current codebase (no privacy settings, no follower/blocking model) suggests users expect private-by-default here.

## Comment 5 — Sort order

**My position:** Switch `get_watchlist()` from alphabetical (`Film.title.asc()`) to date-added descending (`WatchlistEntry.date_added.desc()`), matching `get_collection()`'s sort order exactly.

**Reasoning:** `get_collection()` already sorts newest-first, and the watchlist is the more time-sensitive of the two lists — it's a queue of "what to watch next," and the thing you most likely want to look at first is what you just added, not "A" through "Z". Alphabetical sort actively works against that use case: adding a film named "Alien" makes it disappear to the top of an alphabetical list forever, disconnected from when you actually decided you wanted to watch it, while a newly-added "Zootopia" sinks to the bottom where it's easy to forget about — the opposite of what a watchlist queue should do.

**Engagement with reviewer's point:** If the maintainer's argument for alphabetical was scanability — a user hunting for one specific title in a long list benefits from A–Z order — that's a real UX need, but it belongs in a client-side or query-param sort option, not baked as the only server-side default, because it actively fights the "what did I just add" use case. I checked whether this is a breaking change for any existing consumer: this is a new, unreleased endpoint (this whole feature is still on an open PR, not on main), so there's no live client depending on the current alphabetical order — this is the right time to fix it before it ships, not after.

## Comment 6 — Rebase

**What conflicted:** While this PR was open, `main` merged a refactor (`07ca580 refactor: migrate film IDs from integer to UUID`) that changed `Film.id` from `db.Integer` to `db.String(36)` (UUID), and updated `CollectionEntry.film_id` to match. `WatchlistEntry` didn't exist on `main` at all — it only existed on this branch — so rebasing surfaced two problems, not one:
1. A textual conflict in `.gitignore` (add/add — both branches added one independently; trivial to merge).
2. A structural conflict: after the rebase replayed my commits on top of the UUID refactor, `models.py` had lost the `WatchlistEntry` class entirely (main's refactor commit rewrote the file without it, since it never had a watchlist feature), and `services/watchlist_service.py` / `routes/watchlist/watchlist.py` still referenced integer-shaped film IDs in docstrings and a stray test fixture (`fake_film_id = 999999`).

**How I resolved it:** Ran `git rebase origin/main`, resolved the `.gitignore` conflict by keeping both ignore entries. Then, since the rebase mechanically dropped `WatchlistEntry` (Git can't know a model class needs to be re-added when the surrounding file was independently rewritten upstream), I manually re-added the `WatchlistEntry` model with `film_id` as `db.String(36)` matching `CollectionEntry`'s post-refactor shape, and updated `add_to_watchlist()` / `remove_from_watchlist()` docstrings and the test's fake film ID (`999999` → `"00000000-0000-0000-0000-000000000000"`, matching `test_collection.py`'s convention for a UUID that doesn't exist) to stop assuming integer IDs.

**How I verified no conflict remains:** `git status` shows a clean rebase with no unmerged paths. `git log --oneline origin/main..HEAD` shows a linear sequence of commits with no merge commits. `pytest tests/ -v` passes all 12 tests (4 collection, 8 watchlist) against the rebased, UUID-based schema.

## Stretch — `remove_from_watchlist()`

Implemented `remove_from_watchlist(user_id, film_id)` in `services/watchlist_service.py`, mirroring `remove_from_collection()`: look up the entry by `(user_id, film_id)`, raise a new `NotInWatchlistError` if it's not found, otherwise delete and commit. Wired it to `DELETE /watchlist/<user_id>/remove`, matching `DELETE /collection/<user_id>/remove`'s route shape and 404 error handling. Tests: `test_remove_from_watchlist_deletes_entry` and `test_remove_from_watchlist_not_present_raises`.

## Stretch — second test (edge case)

Added `test_get_watchlist_returns_newest_first`, mirroring `test_get_collection_returns_newest_first`. I chose this case because Comment 5 changed the sort order in code — that behavior change needed its own regression test, the same way `test_collection.py` has a dedicated sort-order test, rather than relying on the (now-updated) docstring as the only source of truth for expected ordering. I also added `test_add_to_watchlist_defaults_to_public` and `test_add_to_watchlist_respects_explicit_visibility` to cover the Comment 4 / visibility-toggle behavior.

## Stretch — visibility toggle

Added an optional `public` parameter to `add_to_watchlist(user_id, film_id, public=True)` and to the `POST /watchlist/<user_id>/add` endpoint (`{"film_id": ..., "public": ...}`, `public` optional). Default stays `True` per the Comment 4 decision above; callers who want a private entry now have a way to say so at creation time instead of the field being permanently stuck at whatever the model's column default produces.

## PR Description

**What this feature does:** Adds a watchlist to CineLog — a list of films a user wants to watch, distinct from their collection (films already watched). Endpoints:

- `GET /watchlist/<user_id>` — returns the user's watchlist, newest-added first.
- `POST /watchlist/<user_id>/add` — body `{"film_id": "<uuid>", "public": <bool, optional, defaults true>}`. Returns 201 with the entry, 404 if the film doesn't exist, 409 if it's already on the watchlist.
- `DELETE /watchlist/<user_id>/remove` — body `{"film_id": "<uuid>"}`. Returns 200, or 404 if the film isn't on the watchlist.

**Design decisions:**
- **Default visibility:** watchlist entries default to `public=True`, consistent with the app's existing social/collection behavior, with an explicit `public` param for callers who want to opt a specific entry into privacy (see Comment 4).
- **Sort order:** watchlist entries are returned newest-added-first (`date_added` descending), matching `get_collection()`'s convention, instead of alphabetical — the watchlist is a "what's next" queue, and recency is more useful than alphabetization for that use case (see Comment 5).

**Manual testing steps:**

```bash
python -m venv .venv && source .venv/Scripts/activate  # or .venv/bin/activate on macOS/Linux
pip install -r requirements.txt
python app.py  # starts on http://127.0.0.1:5000
```

In another terminal, seed a user and a film via the existing endpoints (or use a script), then:

```bash
# Add a film to the watchlist (defaults to public)
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'

# Add a film as private
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid_2>", "public": false}'

# View the watchlist — confirm newest-added film appears first
curl http://127.0.0.1:5000/watchlist/<user_id>

# Adding the same film twice returns 409
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'

# Remove a film from the watchlist
curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'
```

Or run the automated suite: `pytest tests/ -v` (12 tests, covering add/dedup/remove/visibility/sort-order/nonexistent-film cases for both collection and watchlist).

## git log --oneline

<!-- Replace this with an actual screenshot of `git log --oneline main..HEAD` before submitting. -->

```
43cbcae docs: add pr-response.md with review responses and design decisions
617cc38 fix: migrate watchlist service and model to UUID film IDs after main rebase
b8ca396 fix: add missing Film.watchlist_entries relationship
0ffd755 test: add watchlist service tests
49af4de feat: add explicit visibility parameter to add_to_watchlist
50b72cb feat: implement remove_from_watchlist
22edfbd fix: sort watchlist by date added (newest first) instead of alphabetically
084c7eb fix: add deduplication check to add_to_watchlist
793c15c fix: rename save_to_watchlist to add_to_watchlist per service naming convention
45f9933 fix: update film retrieval method to use db.session.get in collection and watchlist services
3eef9ae feat: add watchlist model and add_to_watchlist endpoint
```

11 commits ahead of `main`, all conventional format, no merge commits.
