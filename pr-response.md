# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude to help with summarizing what each file is responsible for, who the models and functions interact with. I asked question on places where i thought the code didnt make sense, or i was confused with how it interacts with other files. I also asked questions about different formats like that of the int vs UUID. At the end I verified the finding with gemini.

## Comment 1 — Rename
**What I did:** Renames _save_to_watchlist() to add_to_watchlist() in services/watchlist_service.py to match the verb_to_noun naming convention used by add_to_collection(). Updated the import and call site in routes/watchlist/watchlist.py.
**How I verified:** an `git grep -n "save_to_watchlist"` across the repo 
to confirm no references remained. Ran `pytest tests/ -v` — all 4 existing 
tests passed.

## Comment 2 — Deduplication
**What I did:** Added a query in add_to_watchlist() that checks for an existing WatchlistEntry with the same user_id and film_id before creating a new one, raising a new AlreadyInWatchlistError if found — mirroring the same pattern used in add_to_collection() (application-level check via filter_by().first(), not a DB constraint).
**How I verified:** Ran pytest tests/ -v to confirm the existing test suite still passes with no regressions.


## Comment 3 — Missing test
**What I did:** Created tests/test_watchlist.py, mirroring the fixture and structure of test_collection.py. Added test_add_to_watchlist_nonexistent_film_raises, modeled directly on test_add_to_collection_nonexistent_film_raises.
**How I verified:** Ran pytest tests/test_watchlist.py -v to confirm the 
new test passes, then pytest tests/ -v to confirm no regressions across 
the full suite. Note: used a fake integer film_id (99999) rather than a 
UUID string, since Film.id is still an integer on this branch pre-rebase; 
this will need revisiting after Comment 6.

## Comment 4 — Default visibility
**My position:** my position is that default should be false.
**Reasoning:**  currently GET /watchlist/<user_id> has no authentication check, so public isnt doing anything today. anyone can see anybody's list regardless fo the flag. 'False' would be safer right now as the tradeoff in case it fails is that less than desired information is shared; rather than more than desired info incase of 'true'
**Tradeoff acknowledged:** currently defaulting to 'false' has no tradeoff. In future, if a "friends current watchlist" or similar feature is added, it will show nothing until the flag is changed again.

## Comment 5 — Sort order
**My position:** my position is date-added, newest-first.
**Reasoning:**  I agree with the maintainer's position that date added newest first, makes the most sense. People add movies to the list based on their current interest. theres a recency bias. So sorting movies in the watchlist according to that makes the most sense to me.
**Engagement with reviewer's point:** it is consistent with get_collection() which already uses date_added.desc(). I also considered alphabetic or maybe date_added.asc() because users might want to start from the beginning and finish up old movies or movies added a long time ago, but that likely wont be the most or base scenario, rather a once in a while thing.

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

## Additional test (stretch)
**What I did:** Added test_get_watchlist_returns_newest_first, mirroring 
test_get_collection_returns_newest_first, to verify the Comment 5 sort 
order change.
**Why this case:** This test is the first one to actually exercise 
get_watchlist() with real data. It uncovered a pre-existing bug: Film had 
a relationship for collection_entries but none for watchlist_entries, so 
entry.film raised an AttributeError. Added the missing 
watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True) 
line to Film in models.py to fix it — this bug was unrelated to the sort 
order change itself, just uncovered by writing a test that finally called 
get_watchlist() end-to-end.