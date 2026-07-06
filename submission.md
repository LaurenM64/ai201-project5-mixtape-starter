---
# AI201 Project 5: Mixtape Bug Hunt Submission

## AI Usage

During this project, I used AI as an interactive sounding board and architectural guide. When first orienting myself, I used it to quickly map out the responsibilities of the `services/` directory and explain how data flows from the routes to the database. During debugging, I used the AI to help break down complex concepts, such as visualizing the `LEFT OUTER JOIN` which helped me understand the 1-to-many fan-out bug in the search feature.

I did not use AI to write the fixes for me blindly. Instead, I traced the code manually to find the suspicious lines and formulated my own hypotheses first. For example, when investigating the streak bug, I researched that Python's `today.weekday()` evaluates to `6` for Sunday, which allowed me to pinpoint the exact logical flaw blocking the updates. For every bug, I independently verified the AI's suggestions by successfully reproducing the issues via terminal `curl` commands, isolating the specific application state, and confirming the fixes worked as intended without breaking related features.

---

## Codebase Map

### Core Architecture

The Mixtape backend follows a clean, separated architecture:

* **`app.py`:** The application factory that initializes the Flask app and the SQLAlchemy database connection.
* **`models.py`:** Defines the database schema using SQLAlchemy ORM (e.g., `User`, `Song`, `Playlist`, `ListeningEvent`, `Notification`). It also defines relationships and handles data serialization (via `to_dict()` methods).
* **`routes/` (Controllers):** These files (e.g., `feed.py`, `songs.py`) define the API endpoints. They handle incoming web requests, parse parameters, and immediately delegate all heavy lifting to the service layer.
* **`services/` (Business Logic):** These files (e.g., `feed_service.py`, `streak_service.py`) contain the actual logic of the application. They execute database queries, perform calculations, and format the final data.

### Example Data Flow: "Listening Now" Feed

1. A `GET` request is made to `/feed/<user_id>/listening-now`.
2. The route in `routes/feed.py` captures the `user_id` and calls `get_friends_listening_now(user_id)` from `services/feed_service.py`.
3. The service function queries the `ListeningEvent` table, filtering for events belonging to the user's friends that occurred after a specific time cutoff (`RECENT_THRESHOLD`).
4. The function deduplicates the results to ensure only the most recent song per friend is shown, fetches the corresponding `User` and `Song` objects, and formats them into a dictionary to be returned as JSON.

---

## Root Cause Analyses

### Issue #1: Listening streak keeps resetting on Sundays

* **How I reproduced it:** I reproduced the issue by overriding the time on my system to simulate a Sunday and sending a `POST` request to create a listening event for a user with an active streak. I verified via a `GET` request that the streak incorrectly reset to 1 instead of incrementing.
* **How I found the root cause:** I investigated `services/streak_service.py` to see how the streak increment logic was calculated. Knowing the bug was tied to a specific day, I looked for conditional logic involving days of the week.
* **The root cause:** On line 73, the code explicitly checked `elif days_since_last == 1 and today.weekday() != 6`. Because Python's `today.weekday()` returns `6` for Sunday, this hardcoded condition actively prevented streaks from updating on Sundays, forcing the code into the `else` block which reset the streak to 1.
* **My fix and side-effect check:** I deleted the `and today.weekday() != 6` condition entirely. The `days_since_last == 1` check inherently handles date boundaries perfectly. I reran the test command on a simulated Sunday and confirmed the streak successfully incremented.

### Issue #2: Friends Listening Now shows people from yesterday

* **How I reproduced it:** I fetched the `listening-now` endpoint for a sample user ID and reviewed the JSON response for their friends' activity. I observed listening events with timestamps from the previous day appearing in what should be a real-time feed.
* **How I found the root cause:** Based on my codebase map, I knew this feed was handled by `services/feed_service.py`. I investigated the query logic in that file to see how it was filtering timestamps to define "now."
* **The root cause:** On line 13, a constant named `RECENT_THRESHOLD` was set to `timedelta(hours=24)`. This threshold was far too large for a status meant to represent current activity, which inadvertently allowed yesterday's listening events to pass the cutoff filter and display in the feed.
* **My fix and side-effect check:** I changed `RECENT_THRESHOLD` to `timedelta(minutes=15)`. This made the feed an accurate representation of what a user would consider listening "now." I verified that older, seeded database events no longer appeared in the output.

### Issue #3: Same song keeps showing up twice in search

* **How I reproduced it:** I ran a search using the `/songs/search?q=Harlem` endpoint. This targeted a seeded song ("Harlem Renaissance") that I knew had multiple genre tags associated with it, which exposed the duplication.
* **How I found the root cause:** I reviewed `services/search_service.py` to see how the search query was constructed. I noticed the SQLAlchemy query was using an `.outerjoin()` on the `song_tags` table, even though the `.filter()` was only checking the `title` and `artist` text.
* **The root cause:** The `LEFT OUTER JOIN` created a 1-to-many "fan-out" effect. Because the target song had 3 distinct tags in the database, the database was forced to duplicate the parent song row 3 times to display each tag on a flat grid. In environments without automatic ORM deduplication, this causes the song to appear in the search results multiple times.
* **My fix and side-effect check:** I deleted the `.outerjoin(song_tags, ...)` line entirely. The search still works perfectly and includes the tags in the final JSON response because the `Song` model (in `models.py`) already uses a SQLAlchemy `db.relationship` to securely fetch and bundle the tags during the `song.to_dict()` serialization.

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

* **How I reproduced it:** I triggered a rating event for a song that was originally shared by a different user. I then checked the original sharer's feed using the `get_notifications` endpoint and confirmed that no new notification was generated.
* **How I found the root cause:** I knew from the issue description that playlist notifications worked but rating notifications did not. I opened `services/notification_service.py` and compared the working `add_to_playlist` function to the broken `rate_song` function line-by-line to look for architectural differences.
* **The root cause:** The `rate_song` function was missing the notification dispatch logic entirely. While it successfully committed the new `Rating` object to the database, it never called the `create_notification` helper function afterward.
* **My fix and side-effect check:** I added a conditional block at the end of `rate_song` that checks if the rater is someone other than the original sharer (`song.shared_by != user_id`). If true, it calls `create_notification` using the `"song_rated"` type. I verified the logic mirrors the working implementation in `add_to_playlist` to ensure consistency.

### Issue #5: The last song in a playlist never shows up

* **How I reproduced it:** I hit the `GET /playlists/<PLAYLIST_ID>/songs` endpoint using the ID for a seeded playlist ("Late Night Vibes"). I knew from the seed data that this playlist contained exactly 7 songs, but the JSON response only returned 6.
* **How I found the root cause:** I knew this feature was handled in `services/playlist_service.py`, so I checked the `get_playlist_songs` function. The database query itself was correct, so I traced the data down to the final `return` statement that formatted the output.
* **The root cause:** The return statement was using Python list slicing: `songs[:-1]`. In Python, this specific slice tells the code to return elements from the beginning of the list up to, but *excluding*, the very last element. This intentionally dropped the final song from the output every single time.
* **My fix and side-effect check:** I deleted the `[:-1]` slice entirely, changing the comprehension to `[song.to_dict() for song in songs]`. To verify the fix and ensure I didn't introduce an index error, I re-ran the `GET` request for the playlist and confirmed all 7 songs successfully appeared in the correct order.