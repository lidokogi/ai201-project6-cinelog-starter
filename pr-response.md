# PR Response Doc — CineLog Watchlist Feature

## AI Usage
Used AI mainly to debug stuff and double check patterns, not to make decisions for me.
- Checked the dedup pattern in `add_to_collection()` before writing the same thing for `add_to_watchlist()`.
- Used it to help debug a few things that broke along the way — a function that ended up in the wrong file, an undefined `AlreadyInWatchlistError` that I had to trace back to find where the sibling error was actually defined, a random stray character that broke an import and threw a `NameError` on every test, and a model that silently got dropped during the rebase.
- For comments 4 and 5 I came up with my own answer first (private by default, sort by date added) and just used AI to help me get the reasoning written out clearly. The actual decisions are mine.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, updated the one call site in `routes/watchlist/watchlist.py`.

**How I verified:** Searched the whole project for `save_to_watchlist` to make sure nothing still referenced the old name.

## Comment 2 — Deduplication
**What I did:** Added a dedup check to `add_to_watchlist()` — same idea as `add_to_collection()`, check if there's already an entry for that user/film, raise an error if so instead of just creating a duplicate. Had to add a new `AlreadyInWatchlistError` since nothing like it existed yet, put it right next to `AlreadyInCollectionError` since that's where those errors live.

**How I verified:** This one had a bunch of small issues. First I accidentally pasted the function into `collection_service.py` instead of `watchlist_service.py` and got undefined variable errors until I moved it. Then I had to actually go find where `AlreadyInCollectionError` was defined (turns out it's just declared at the top of `collection_service.py`, not in some separate errors file) before I could add my own error next to it. Then a random stray character got left in the import line somehow and broke every single test with a `NameError` until I found and deleted it. Ran `pytest tests/ -v` after each fix until everything passed.

## Comment 3 — Missing test
**What I did:** Made `tests/test_watchlist.py`, wrote `test_add_to_watchlist_nonexistent_film_raises` copying the structure of `test_add_to_collection_nonexistent_film_raises` — same fixtures, same `pytest.raises` pattern, just calling `add_to_watchlist()` instead.

**How I verified:** The fixtures (`app`, `sample_user`, `sample_film`) were only defined inside `test_collection.py` so my new test couldn't see them. Moved them into a shared `tests/conftest.py` so both test files can use them. Ran the full suite and got all 5 tests passing.

## Comment 4 — Default visibility
**My position:** Private by default.

**Reasoning:** Collection and watchlist aren't really the same thing to expose. Collection is just a record of what you watched, low stakes. Watchlist shows what you want to watch, which can honestly say more about you — could be something tied to a breakup, a guilty pleasure genre, whatever. Making that public by default means people's intent gets shown before they actually chose to share it. Private by default means sharing is something you decide to do, not something that just happens.

**Tradeoff acknowledged:** Downside is discovery — a chunk of what makes CineLog social is seeing what your friends want to watch. If it's private by default and nobody bothers turning it on, that social piece basically doesn't get used. I still think that's worth it because a public toggle on the endpoint makes it a one-line thing for anyone who does want to share, so the cost of defaulting private is pretty low.

## Comment 5 — Sort order
**My position:** Sort by date added, not alphabetically — agree with the reviewer here.

**Reasoning:** A watchlist is more of a queue than something you're browsing like a catalog. Most of the time you're checking "what did I just add" or "what's next," not looking for one specific title by name. Alphabetical makes more sense for something you're searching through, not really for a watchlist.

**Engagement with reviewer's point:** This also matches how collection is already sorted (`test_get_collection_returns_newest_first` — newest first is already the pattern in this codebase), so keeping watchlist the same way keeps things consistent instead of having two different sorting rules for two similar features.

**Tradeoff acknowledged:** If someone has a really long watchlist, alphabetical would make it faster to check if they already added something. Could eventually support both (date added as default, alphabetical as an option) but for this PR I'm just matching what collection already does.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` then `git rebase origin/main` to pull in the UUID refactor that merged into main while my PR was open. Two things came up:
1. `.gitignore` had an add/add conflict — both main and my branch added one separately.
2. `WatchlistEntry` went missing from `models.py` entirely after the rebase applied with no conflict flagged. The refactor rewrote enough of the file that my earlier addition just didn't get reapplied and nothing warned me about it.

**How I resolved it:** Merged both `.gitignore` versions by hand (kept everything from both, deleted the conflict markers), `git add`ed it and ran `git rebase --continue`. For the missing model, I re-added `WatchlistEntry` to `models.py` matching the same UUID pattern as `Film`/`CollectionEntry` (`db.Column(db.String(36), primary_key=True, default=generate_uuid)`), added the foreign keys, a unique constraint on user/film like `CollectionEntry` has, and added the `watchlist_entries` relationship back on `User` and `Film`.

**How I verified no conflict remains:** `git log --oneline` shows a clean line with no merge commits. Ran `pytest tests/ -v` after and got all 5 tests passing again, which confirmed the model was actually back and everything still worked.

## PR Description

**What it does:** Adds a watchlist feature — lets users track films they want to watch, separate from their collection of films they've already seen. Includes `add_to_watchlist()` with duplicate protection, a `WatchlistEntry` model, the `POST /watchlist/<user_id>/add` endpoint, and a test for the nonexistent-film case.

**Design decisions:**
- Watchlist entries default to private (`public=False`) — full reasoning under Comment 4.
- Watchlist is sorted by date added, newest first, matching how collection already works — full reasoning under Comment 5.

**How to test it manually:**
1. Run the app: `python app.py`
2. Create a user and film (through existing collection endpoints or a seed script)
3. Add a film to the watchlist:
```bash
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_id>"}'
```
4. Check it shows up sorted newest first
5. Try adding the same film again — should get `AlreadyInWatchlistError`, not a duplicate
6. Try a fake `film_id` — should get `FilmNotFoundError`
7. Run `pytest tests/ -v` to confirm everything passes