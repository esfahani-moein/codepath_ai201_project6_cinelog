# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI tools in three specific ways during this project:

1. **Codebase orientation**: I asked an AI assistant to summarize the responsibilities of `models.py` and `services/collection_service.py` and to walk through `add_to_collection()` before implementing the watchlist deduplication logic. I verified the summaries against the actual code before writing anything.
2. **Stress-testing design arguments**: After drafting my responses to Comments 4 (default visibility) and 5 (sort order), I asked the AI to play devil's advocate and identify counterarguments or tradeoffs I might have missed. It surfaced that privacy-first users could be surprised by `public=True`, so I strengthened the "Tradeoff acknowledged" section and explicitly documented the `public: false` endpoint escape hatch.
3. **Commit-format verification**: Before finalizing, I shared my `git log --oneline` output with the AI and asked whether the messages followed conventional commit format and whether any commit bundled multiple logical changes. It confirmed the prefixes were correct and suggested no splits were needed.

The design arguments in Comments 4 and 5 are my own reasoning; AI was used only to probe for gaps, not to write the responses.

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated the single call site in `routes/watchlist/watchlist.py` (both the import and the function call). This aligns with the `verb_to_noun` convention documented in `CONTRIBUTING.md` and already used by `add_to_collection()`, `remove_from_collection()`, and `get_collection()`.

**How I verified:**
Ran a project-wide search for `save_to_watchlist` and confirmed zero remaining references. Ran the full test suite (`pytest tests/ -v`) to confirm no import or call-site errors were introduced.

## Comment 2 — Deduplication
**What I did:**
Added an `AlreadyInWatchlistError` exception and a pre-insert check in `add_to_watchlist()` that queries for an existing `(user_id, film_id)` pair before creating a new `WatchlistEntry`, mirroring the pattern in `add_to_collection()`. I also added a database-level `UniqueConstraint("user_id", "film_id")` on `WatchlistEntry` so the data model itself enforces the invariant. The route now catches `AlreadyInWatchlistError` and returns HTTP 409, matching the collection endpoint's conflict handling.

**How I verified:**
Added `test_add_to_watchlist_duplicate_raises`, which adds the same film twice and asserts that the second call raises `AlreadyInWatchlistError` and that only one database row exists. The test passes, and the full suite remains green.

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py` and included `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`. It uses a fake UUID film ID and asserts that `add_to_watchlist()` raises `FilmNotFoundError` rather than a database integrity error.

**How I verified:**
Ran `pytest tests/test_watchlist.py -v`; the targeted test passes. The full `pytest tests/ -v` run also passes.

## Comment 4 — Default visibility
**My position:**
Keep the default `public=True`, but expose it as an explicit endpoint parameter so callers can opt into `public=False`.

**Reasoning:**
CineLog is a community film-tracking app. The primary value proposition is sharing what you want to watch with friends and discovering films through others' lists. Defaulting to public supports that social, discovery-oriented use case out of the box — a new user's watchlist is immediately visible to the community, which encourages engagement. Making `public` an explicit request parameter (`POST /watchlist/<user_id>/add` body: `{ "film_id": "...", "public": false }`) preserves user agency without requiring a second API call or setting change. This mirrors how many social platforms treat lists/activity as public-by-default while letting power users toggle privacy.

**Tradeoff acknowledged:**
Privacy-first users may be surprised that their watchlist is public by default. To mitigate this, the API now lets clients set `public: false` at creation time, and a future privacy-settings endpoint could let users change their account-wide default. I considered flipping the default to `False`, but that would make the watchlist feature essentially private-by-default and undercut the community-sharing angle that distinguishes CineLog from a personal note-taking app. The explicit parameter is the compromise: safe-by-choice for privacy-conscious users, social-by-default for everyone else.

## Comment 5 — Sort order
**My position:**
Accept the reviewer's preference: `get_watchlist()` should return entries sorted by `date_added` descending (newest first), replacing the original alphabetical sort.

**Reasoning:**
A watchlist is a working queue. Users add films they intend to watch, and the most recently added items are usually the ones they're most excited about or planning to watch next. Sorting by date added surfaces those films immediately, which matches user mental models and matches the existing `get_collection()` behavior (also newest-first). The implementation is also simpler: it removes the unnecessary `join(Film)` and `order_by(Film.title.asc())`, relying on the `WatchlistEntry.date_added` column we already store.

**Engagement with reviewer's point:**
The reviewer noted that "most users want to see what they added recently." That aligns with how watchlists are typically used, so I adopted it rather than proposing a third option like alphabetical-with-recent-pinned. Alphabetical order is defensible for large, stable libraries, but a watchlist is inherently temporal. By switching to newest-first, the watchlist endpoint is now consistent with the collection endpoint's sort semantics, giving CineLog a uniform "recently added first" pattern across user lists.

## Comment 6 — Rebase
**What conflicted:**
Two conflicts arose when rebasing `feature/watchlist` onto `origin/main`:

1. `.gitignore`: main already included a `.gitignore` that added `.pytest_cache/` to the starter's original list. My branch added the starter list without `.pytest_cache/`, creating an add/add conflict.
2. `models.py`: main had refactored `Film.id` and `CollectionEntry.film_id` from `Integer` to `String(36)` UUIDs. My branch's `WatchlistEntry` still used `db.Integer` for `film_id` and added a unique constraint, so Git could not auto-merge the model changes.

**How I resolved it:**
- For `.gitignore`, I kept main's version (which includes `.pytest_cache/`) since it is a strict superset of what I had added.
- For `models.py`, I kept main's UUID-based `Film` and `CollectionEntry` models and added `WatchlistEntry` with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), nullable=False)`. I also added the `watchlist_entries` relationship on `Film` so `entry.film` resolves correctly in `get_watchlist()`.

**How I verified no conflict remains:**
- Ran `git status` and confirmed no remaining conflict markers.
- Ran `git log --oneline --merges origin/main..HEAD` and confirmed zero merge commits.
- Ran the full test suite (`pytest tests/ -v`) on the rebased branch; all 11 tests pass, including the watchlist tests that create and query films by UUID.

## PR Description

**Feature overview:**
This PR adds a watchlist feature to CineLog so users can save films they want to watch and retrieve them in a social, community-friendly list. It introduces a new `WatchlistEntry` model, service-layer functions (`add_to_watchlist`, `remove_from_watchlist`, `get_watchlist`), and REST endpoints under `/watchlist/<user_id>`.

**Design decisions:**
1. **Default visibility (`public=True`)**: Watchlist entries default to public to support CineLog's community-sharing use case, but callers can opt into privacy by passing `"public": false` in the POST body.
2. **Sort order (date-added descending)**: `GET /watchlist/<user_id>` returns the most recently added films first, matching the existing collection endpoint behavior.

**Stretch additions:**
- `remove_from_watchlist(user_id, film_id)` and `DELETE /watchlist/<user_id>/remove`
- Explicit `public` parameter on `POST /watchlist/<user_id>/add`
- Additional tests for the public parameter, duplicate prevention, and remove behavior

**How to manually test:**
1. Start the app: `python app.py` (runs on `http://127.0.0.1:5000`).
2. Create a film:
   ```bash
   curl -X POST http://127.0.0.1:5000/films/ -H "Content-Type: application/json" -d '{"title":"Inception","year":2010,"genre":"Sci-Fi"}'
   ```
3. Create a user (via Flask shell or directly in the database) and note the user UUID.
4. Add a film to the watchlist:
   ```bash
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add -H "Content-Type: application/json" -d '{"film_id":"<film-uuid>"}'
   ```
5. Add a second film, then verify `GET /watchlist/<user_id>` returns the newest first.
6. Add with `"public": false` and confirm the entry has `public: false`.
7. Try adding the same film again and confirm HTTP 409.
8. Remove a film:
   ```bash
   curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove -H "Content-Type: application/json" -d '{"film_id":"<film-uuid>"}'
   ```

**Final commit history (`git log --oneline origin/main..HEAD`):**
```
537cd7f docs: add pr-response.md with design decisions
fb0c0c2 test: add watchlist service tests
fcff143 fix: change get_watchlist sort order to date-added descending
993b64c feat: add remove_from_watchlist endpoint
728ec10 feat: add public parameter to add_to_watchlist endpoint
78e76e9 fix: add deduplication check to prevent duplicate watchlist entries
4d45e16 fix: rename save_to_watchlist to add_to_watchlist per naming convention
1c1dbac feat: add watchlist model and add_to_watchlist endpoint
```
