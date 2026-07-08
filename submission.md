# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude (Claude Code) throughout this project, mainly for codebase orientation and for helping me understand code I had located to break down how to fix the bug. 

- **Orientation phase**: I asked it to read README.md, models.py, everything in routes and services, and essentially give me a summary on what eah function does and its responsibility. It returned a file-by-file summary and, what the impact would be if that did not work. 
- **verifying claude**: for Issue #3, I was really struggling to figure out what was going on so i asked claude for an initial analysis and suggestion. Claude said the join was causing duplicate rows, which made sense reading the code, but when I ran search_songs("Anthem") myself it gave me exactly 1 result, not 3 like the bug report said. I didn't think claude was right so i ran the same join as a raw SQLAlchemy 2.0 and saw that it gave me 3 duplicate rows. The specific version of SQLAlchemy installed in this project happens to automatically clean up (deduplicate) results when you use the old-style query() method, but it doesn't do that automatic cleanup with the new-style method. So the bug was still an issue it just couldnt be seen outright so i fixed it. 

## Codebase Map

**Structure**: It's a pretty standard Flask app. app.py is just the app factory and SQLAlchemy setup, models.py has all the database tables, routes/ is where the HTTP endpoints live, and services/ is where the actual logic happens. seed_data.py fills a fresh SQLite database with 5 users, a bunch of songs with different tag counts, some playlists, listening events, and streaks already in progress. There are 3 test files in tests/: test_streaks.py, test_search.py, test_playlists.py.

**Data flow — rating a song and getting notified** 
1. Someone hits POST /songs/song_id/rate with a user_id and score and lands in routes/songs.py::rate().
2. That calls rate_song() in notification_service.py, which saves (or updates) a Rating row and commits.
3. Compare that to the path that actually works: adding a song to a playlist goes through add_to_playlist() in the same file, which adds the song AND then calls create_notification() for the original sharer, as long as they're not the one who added it.
4. a sharer gets notified through GET /users/id/notifications if someone added their song to a playlist, never if someone rated it which seems to be a bug. 

## Root Cause Analysis

### Issue #1 — My listening streak keeps resetting

**How I reproduced it**: I set a user's last_listened_at to a Saturday and then ran the streak-update function as if they'd listened again the next day, Sunday. Streak went from 12 down to 1 instead of up to 13.

**How I found the root cause**: All the streak math lives in streak_service.py so I double checked nothing else touches listening_streak. The function has three branches: same day (do nothing), one day later (increment), or anything else (reset to 1). The line that jumped out immediately was elif days_since_last == 1 and today.weekday() != 6 which didnt make any sense.

**The root cause**: In Python,weekday() returns 6 for Sunday. So the increment branch was quietly requiring "one day passed AND it's not Sunday." Which means if your second day in a row happened to land on a Sunday, that condition was false, and it went into the else statement instead resetting instead of incrementing.

**My fix and side-effect check**: Just deleted the and today.weekday() != 6 part. Now it's simply "one day passed to increment." I checked that Saturday→Sunday now goes 12→13 like it should, that a real 2-day gap ending on Sunday still resets to 1 (didn't want to break that), that a normal weekday increment still works, and ran the existing test suite including the test test_streak_increments_on_sunday` and everything passed.

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it**: I added a listening event for a friend timestamped about 11 hours ago (basically simulating "listened at 9pm, it's 8am now") and called the feed function. That friend showed up as "listening now" even though, to any normal person, that's clearly yesterday.

**How I found the root cause**: The feed logic is all in one short function in feed_service.py. Right away I saw RECENT_THRESHOLD = timedelta(hours=24) and a cutoff calculated as `now minus 24 hours`. The bug report is specifically about a day boundary (things showing up "the next morning"), not about a fixed duration, so this was the obvious place to look.

**The root cause**: "Recent" was defined as "within the last 24 hours" instead of "today." it counted a day as the most recent 24 hours instead of calendar day so it just kept showing up as current for almost the entire next day instead of resetting when the day actually changed.

**My fix and side-effect check**: Swapped the rolling 24-hour cutoff for "the start of today" (midnight UTC), and cleaned up the now-unused RECENT_THRESHOLD constant since it wasn't needed anymore. I tested three scenarios: 11pm yesterday (correctly excluded now), 12:30am today (correctly included), and 2 days ago (still excluded, as before). I also left get_activity_feed() alone since that one's not supposed to filter by recency in the first place, and confirmed it still returns everything.

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it**: covered in the AI Usage section above,  it didn't actually duplicate for me through the normal function call, but I confirmed the underlying join really does produce 3 duplicate rows once I manually did the join query. 

**How I found the root cause**: search_songs() joins Song to song_tags and then filters on title/artist  but it never actually uses the tags table for anything, not for filtering, not for loading data (tags get loaded separately elsewhere through a different relationship). Since a song can have multiple rows in song_tags (one per tag), joining without deduplicating means a song with 3 tags shows up 3 times in the raw result.

**The root cause**: it's a pointless join that multiplies matching songs by however many tags they have. Whether or not it actually shows duplicates to a user turned out to depend on which SQLAlchemy query style is used under the hood (see AI Usage section), but that's an accident of this specific setup the underlying query is still wrong regardless.

**My fix and side-effect check**: I just removed the join entirely since it wasn't doing anything useful Ran tests/test_search.py and all 5 tests still pass, including the ones that specifically check for no duplicates across songs with 0, 1, and multiple tags, and a plain search by artist name still works fine.

### Issue #4 — I got notified for playlist adds but not for ratings

**How I reproduced it**: Called rate_song() directly for a song owned by one user, rated by their friend, then checked the owner's notifications before and after. Count stayed at 0 even though the rating itself saved fine.

**How I found the root cause**: I put rate_song() side by side with add_to_playlist() since the bug report basically says "one works, one doesn't, they should behave the same." add_to_playlist() ends with a check, if the person adding isn't the original sharer, notify them. rate_song() has nothing like that at all. It saves the rating and just returns.

**The root cause**: there's no shared/automatic system for "notify someone when their song gets interacted with" every interaction (playlist add, rating, etc.) has to remember to call create_notification() on its own. its included in add_to_playlist but not in rate_song.

**My fix and side-effect check**: Added the same pattern used in add_to_playlist() to rate_song. Checked that rating a friend's song now creates exactly one notification, that rating your own song does NOT notify yourself, and that the playlist-add notifications still work fine since I didn't touch that function.

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it**: Tso I inserted playlist rows directly the same way the existing tests do ,3 songs at positions 1, 2, 3. Fetching the playlist back only gave me 2 songs. The one at the highest position (the most recently added) was always the one missing.

**How I found the root cause**: get_playlist_songs() is the only function that reads a playlist's songs. the return line was return [song.to_dict() for song in songs[:-1]] which was wrong.

**The root cause**: songs[:-1] chops off the last item in the list. Since the list is already sorted by position, the last item is always the newest song. So with N songs you only ever get N-1 back, and every time you add a new one, the previous "last" song becomes visible while the brand new one takes its place as the hidden one. That's exactly the "adding a song frees the previous one" behavior from the bug report.

**My fix and side-effect check**: Removed the[:-1] slice so it just returns everything. Reran the playlist tests and the two that were failing now pass. I also specifically checked a playlist with just 1 song, since [:-1] on a single-item list gives you an empty list that edge case now correctly returns the 1 song instead of nothing. The empty-playlist test still passes too, so that path wasn't affected.

