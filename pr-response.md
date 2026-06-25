# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude to help with environment setup and branch configuration, 
understanding the existing codebase patterns (add_to_collection, test 
structure), and step-by-step implementation guidance. For Comments 4 and 5, 
I provided my own reasoning and Claude helped shape it into written form. 
All design decisions reflect my own thinking.

## Comment 1 — Rename
**What I did:** Renamed save_to_watchlist() to add_to_watchlist() in 
services/watchlist_service.py and updated the import and call site in 
routes/watchlist/watchlist.py. Also updated the docstring to reflect 
film_id as a UUID string instead of int.

**How I verified:** Used project-wide search to confirm no remaining 
references to save_to_watchlist. Ran pytest tests/ -v — all 4 existing 
tests passed.

## Comment 2 — Deduplication
**What I did:** Added an AlreadyInWatchlistError exception class and a 
duplicate check in add_to_watchlist() following the same pattern as 
add_to_collection(). Updated the route to catch AlreadyInWatchlistError 
and return a 409 response.

**How I verified:** Ran pytest tests/ -v — all tests passed. The 
UniqueConstraint on the WatchlistEntry model provides a second layer of 
protection at the database level.

## Comment 3 — Missing test
**What I did:** Created tests/test_watchlist.py with 
test_add_to_watchlist_nonexistent_film_raises, following the same fixture 
and assertion structure as test_add_to_collection_nonexistent_film_raises 
in test_collection.py. Used the same fake UUID pattern and pytest.raises 
assertion.

**How I verified:** Ran pytest tests/test_watchlist.py -v — 1 passed. 
Ran full pytest tests/ -v — all 5 passed.

## Comment 4 — Default visibility
**My position:** public should default to False.

**Reasoning:** CineLog is a community app, but a watchlist is inherently 
personal — it's a list of films a user intends to watch, not films they've 
already seen and want to share. When I add something to a watchlist on any 
platform, my expectation is that it's mine until I choose to share it. 
Defaulting to private means users are in control from the start and can 
opt in to sharing when they're ready.

**Tradeoff acknowledged:** The argument for public=True is that CineLog is 
a social platform and visibility drives engagement — if watchlists are 
private by default, fewer people share them. That's a real tradeoff. But 
the cost of getting it wrong in the other direction is worse: a user who 
didn't realize their watchlist was public has had their data exposed without 
consent. Privacy as default is the safer choice.

## Comment 5 — Sort order
**My position:** Keeping alphabetical sort for now, but acknowledging this 
warrants a configurable option eventually.

**Reasoning:** A watchlist serves two purposes at once — it's a to-do list 
of films to watch next, but it's also a browsable catalog you scan when 
deciding what to watch. Alphabetical order serves the catalog use case well: 
if I remember a film title, I can find it instantly. Date-added serves the 
to-do use case: my most recent additions are probably what I'm most excited 
about right now.

**Engagement with reviewer's point:** The reviewer's preference for 
date-added makes sense if we treat the watchlist primarily as a queue. But 
unlike a collection — where newest-first shows your recent activity — a 
watchlist doesn't have a natural "done" state that makes recency the obvious 
priority. I'd push back gently here and suggest we either default to 
date-added with a documented sort parameter, or revisit once we have user 
data on how people actually use the watchlist.

## Comment 6 — Rebase
**What conflicted:** The rebase onto upstream/main completed cleanly with 
no conflicts. However, the UUID refactor on main meant WatchlistEntry was 
missing from models.py entirely — it had only existed on the feature branch 
with integer IDs. I added WatchlistEntry to models.py with UUID-typed 
film_id (String(36)) and public field defaulting to False.

**How I resolved it:** Added WatchlistEntry model to models.py following 
the same UUID pattern as CollectionEntry.

**How I verified no conflict remains:** Ran pytest tests/ -v — all 5 tests 
passed. Confirmed no merge commits with git log --oneline.

## PR Description

The watchlist feature allows users to save films they want to watch in the 
future. Unlike the collection (films already watched), the watchlist is a 
forward-looking list with a public/private visibility toggle. Users can add 
films by UUID, view their full watchlist, and entries are deduplicated so 
the same film cannot be added twice.

**Design decisions:**
- Visibility defaults to private (public=False) — users must explicitly opt 
  in to sharing their watchlist rather than having it public by default.
- Sort order is alphabetical by title — the watchlist serves as a browsable 
  catalog as much as a queue, and alphabetical order makes individual films 
  easier to locate by name.

**Manual testing steps:**

1. Start the app:
   python app.py

2. Add a film to a user's watchlist (replace UUIDs with real values from your db):
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d "{\"film_id\": \"<film_uuid>\"}"
   Expected: 201 with the new WatchlistEntry as JSON.

3. Try adding the same film again:
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d "{\"film_id\": \"<film_uuid>\"}"
   Expected: 409 with error message about duplicate entry.

4. Try adding a nonexistent film:
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d "{\"film_id\": \"00000000-0000-0000-0000-000000000000\"}"
   Expected: 404 with film not found error.

5. View the watchlist:
   curl http://127.0.0.1:5000/watchlist/<user_id>
   Expected: 200 with list of films sorted alphabetically by title.