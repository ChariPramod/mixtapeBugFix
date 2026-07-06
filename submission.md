## AI Usage

I used AI tools mainly to understand the codebase structure and trace how different files were connected before making bug fixes. I asked AI to help explain the purpose of the main files in `routes/`, `services/`, and `models.py`, especially where the route functions called service functions and where those service functions used database models.

For each bug, I used AI to help trace the related call chain. For example, I looked at how song rating starts in `routes/songs.py`, then moves into `notification_service.py`, and then uses the `Song`, `User`, `Rating`, and `Notification` models from `models.py`. I also used AI to compare similar code paths, such as the working playlist notification flow and the missing rating notification flow.

AI helped me understand what each suspicious function was doing, but I still verified the issue myself by reading the actual files, reproducing the bugs, checking the tests, and confirming the behavior after each fix. I did not rely only on AI guesses. I used the AI explanations as a guide, then confirmed the root cause from the code and made targeted fixes. Each bug fix was committed separately on the `bugfix/mixtape` branch.


## Codebase Map

Mixtape is a small Flask app for sharing and discovering music with friends. Users can share songs, listen to songs, rate them, create playlists, add songs to playlists, see what their friends are listening to, track listening streaks, and receive notifications when other people interact with their shared songs.

The main setup happens in `app.py`. This file creates the Flask app, connects SQLAlchemy to the database, registers the route files, and creates the database tables. It does not handle much feature logic directly. Its main job is to start the app and connect all the pieces together.

`models.py` defines the database structure. The main models are `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, and `Notification`. There are also join tables for relationships between users, songs, tags, and playlists. One important table is `playlist_entries`, which connects songs to playlists. It does more than just link a song and playlist because it also stores the song’s position, who added it, and when it was added. That means playlist order is stored directly in the database instead of depending on random insertion order.

The `routes/` folder contains the API endpoints. These files are mostly responsible for receiving requests, checking required input, calling the correct service function, and returning JSON. `routes/songs.py` handles song search, song details, song ratings, and listening events. `routes/playlists.py` handles creating playlists, getting playlist details, getting songs in a playlist, and adding songs to a playlist. `routes/users.py` handles user profiles, streaks, notifications, and marking notifications as read. `routes/feed.py` handles the listening-now feed and the general activity feed.

The `services/` folder is where most of the app logic lives. `streak_service.py` handles listening streak updates when a user listens to a song. `feed_service.py` builds the friend activity feeds. `search_service.py` searches songs by title or artist. `notification_service.py` handles notifications, ratings, and adding songs to playlists. `playlist_service.py` handles playlist creation and returning playlist songs in the correct order.

The `tests/` folder contains tests for the main bug areas. `test_streaks.py` checks how listening streaks behave across different dates. `test_search.py` checks that search results work correctly and do not return duplicate songs. `test_playlists.py` checks that playlists return all songs in the right order, including empty playlists.

`seed_data.py` creates sample data for local testing. It adds users, friendships, songs, tags, listening events, playlists, playlist entries, ratings, and notifications. This is useful because it gives the app enough realistic data to reproduce bugs without manually creating everything from scratch.

One data flow I traced is rating a song. When a user rates a song, the request goes to `POST /songs/<song_id>/rate` in `routes/songs.py`. The route reads the `user_id` and `score` from the request body, checks that they exist, and then calls `rate_song()` from `notification_service.py`. Inside that service function, the app checks that the score is valid, loads the song and user from the database, and then checks whether the user has already rated that song. If a rating already exists, it updates the score. If not, it creates a new `Rating` record. After that, the change is committed to the database and the route returns the rating as JSON.

Another data flow I traced is listening to a song. When a user listens to a song, the request goes to `POST /songs/<song_id>/listen` in `routes/songs.py`. That route calls `record_listening_event()` in `streak_service.py`. The service creates a new `ListeningEvent`, then calls `update_listening_streak()` to decide whether the user’s streak should start, stay the same, increase, or reset. The streak logic compares today’s date with the user’s last listening date.

A third data flow I looked at is adding a song to a playlist. The request goes to `POST /playlists/<playlist_id>/songs` in `routes/playlists.py`. The route gets the `song_id` and `added_by` user ID, then calls `add_to_playlist()` in `notification_service.py`. That function loads the song, playlist, and user from the database. If the song is not already in the playlist, it adds it. Then, if the person adding the song is not the same person who originally shared it, the app creates a notification for the original sharer.

The main pattern I noticed is that the project keeps routes and business logic separate. The route files are mostly thin wrappers around the service functions. They handle request and response details, while the service files handle the real app behavior and database changes. Another pattern is that models use `to_dict()` methods, which makes it easier for routes and services to return clean JSON responses. Overall, the codebase is small, but the features are connected: listening events affect streaks and feeds, playlist actions can trigger notifications, and songs connect to tags, ratings, playlists, and users.


## Root Cause Analysis

### Issue #1: My listening streak keeps resetting

**How I reproduced it:**
I reproduced this by checking the streak behavior for a user who listened on Saturday and then listened again on Sunday. Since those are consecutive calendar days, the streak should increase by 1. Instead, the streak reset back to 1 on Sunday.

**How I found the root cause:**
I started from the listening route in `routes/songs.py`, where `POST /songs/<song_id>/listen` calls `record_listening_event()` from `streak_service.py`. From there, I traced the logic into `update_listening_streak()`, which decides whether the streak should start, stay the same, increment, or reset. The suspicious line was the condition that checked `today.weekday() != 6`.

**The root cause:**
The bug was caused by the streak logic treating Sunday as a special reset case. Python’s `weekday()` returns `6` for Sunday. The code only incremented the streak when `days_since_last == 1` and the current day was not Sunday. That meant Saturday to Sunday, even though it is a normal consecutive-day streak, went into the reset path instead of incrementing.

**Your fix and side-effect check:**
I removed the unnecessary Sunday check so that any true consecutive calendar day increments the streak. The updated logic increments when `days_since_last == 1`, does nothing when the user already listened today, and resets only when more than one day was skipped. I checked the related streak cases: new user starts at 1, same-day listening does not double count, consecutive days increment, skipped days reset, and Saturday-to-Sunday now increments correctly.

---

### Issue #2: Friends Listening Now shows people from yesterday

**How I reproduced it:**
I reproduced this by checking the “Friends Listening Now” feed with seeded listening events. The seed data includes very recent listening events and older events from hours or days earlier. The feed was showing events that were too old to reasonably count as “listening now.”

**How I found the root cause:**
I started from `routes/feed.py`, where `GET /feed/<user_id>/listening-now` calls `get_friends_listening_now()` in `feed_service.py`. In that service file, I found the recency cutoff was controlled by `RECENT_THRESHOLD`.

**The root cause:**
The root cause was that `RECENT_THRESHOLD` was set to 24 hours. That makes sense for a general activity feed, but not for a “Listening Now” feature. Because the cutoff was too broad, friends who listened many hours ago or yesterday could still appear in the listening-now feed.

**Your fix and side-effect check:**
I changed the listening-now threshold from 24 hours to a much shorter recent window. This makes the feature only return friends who listened recently instead of everyone who listened within the last day. I checked that the general activity feed was not affected because it uses a separate function, `get_activity_feed()`, which is supposed to return older recent activity regardless of the listening-now threshold.

---

### Issue #3: The same song keeps showing up twice in search

**How I reproduced it:**
I reproduced this by searching for a song that has multiple tags. Songs with no tags or one tag appeared once, but a song with multiple tags could appear more than once in the search results.

**How I found the root cause:**
I started from `routes/songs.py`, where `GET /songs/search` calls `search_songs()` from `search_service.py`. In `search_service.py`, I saw that the query was searching the `Song` table while also doing an outer join through the `song_tags` association table.

**The root cause:**
The search only matches against song title and artist, but the query joined against `song_tags`. For songs with multiple tags, that join can produce multiple matching rows for the same song because there is one join row per tag. Since the tag join was not needed for title/artist search, it introduced duplicate-result behavior without adding useful filtering.

**Your fix and side-effect check:**
I removed the unnecessary join to `song_tags` and searched directly on the `Song` model using the title and artist filters. The song’s tags are still included in the returned dictionary through the model relationship and `to_dict()`, so the response still contains tag data. I checked that songs with no tags, one tag, and multiple tags still appear in search results, but each matching song appears only once.

---

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:**
I reproduced this by having one user rate a song shared by another user, then checking the original sharer’s notifications. A notification was created when a song was added to a playlist, but no notification was created when a song was rated.

**How I found the root cause:**
I traced the rating flow from `routes/songs.py`, where `POST /songs/<song_id>/rate` calls `rate_song()` in `notification_service.py`. Then I compared that function with `add_to_playlist()` in the same service file. `add_to_playlist()` had a notification creation step, but `rate_song()` only saved the rating and returned it.

**The root cause:**
The root cause was missing notification logic in the rating path. The app already had a working pattern for notifying the original song sharer when someone else added their song to a playlist. However, the rating flow did not follow that same pattern. It created or updated the `Rating` record, committed the database transaction, and returned without creating a `Notification`.

**Your fix and side-effect check:**
I added notification creation to `rate_song()` when the rater is not the same user who originally shared the song. The notification is sent to the original sharer and includes the rater’s username, the song title, and the rating score. I also checked that a user rating their own shared song does not create a notification for themselves, matching the pattern used in the playlist notification flow.

---

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:**
I reproduced this by retrieving songs from a playlist that had five songs. The service returned only four songs, and the missing song was always the last one in playlist order.

**How I found the root cause:**
I started from `routes/playlists.py`, where `GET /playlists/<playlist_id>/songs` calls `get_playlist_songs()` from `playlist_service.py`. The database query correctly joined songs through `playlist_entries`, filtered by playlist ID, and ordered by playlist position. The issue was not in the query. It was in the return statement.

**The root cause:**
The function used `songs[:-1]` when building the response. In Python, `songs[:-1]` means “all songs except the last one.” So even when the database query returned the correct full playlist, the function intentionally sliced off the final song before returning the result.

**Your fix and side-effect check:**
I changed the return statement to iterate over `songs` instead of `songs[:-1]`. This returns every song from the ordered query. I checked that playlists now return all songs, that the song order is still preserved by `playlist_entries.position`, and that an empty playlist still returns an empty list without error.

---

### Issue #6: Adding a song to a playlist can fail silently on required columns

**How I reproduced it:**
I reproduced this by tracing what actually happens at the database level when a song is added to a playlist through `add_to_playlist()`. The `playlist_entries` table is not a simple many-to-many association table — it has extra non-nullable columns, `position` and `added_by`, in addition to `playlist_id` and `song_id`.

**How I found the root cause:**
I compared the ORM relationship append in `notification_service.py` (`playlist.songs.append(song)`) against how `seed_data.py` populates the same table. `seed_data.py` never uses the `songs` relationship to add entries — it always inserts directly into `playlist_entries` with explicit `position`, `added_by`, and `added_at` values.

**The root cause:**
`playlist.songs` is a SQLAlchemy `secondary` relationship over `playlist_entries`. Appending to it only knows how to set `playlist_id` and `song_id` on the association row. It has no way to populate `position` or `added_by`, which are declared `nullable=False` on the table. Depending on the database backend, this can raise an integrity error or otherwise insert an incomplete row.

**Your fix and side-effect check:**
I replaced `playlist.songs.append(song)` with a direct insert into `playlist_entries`, computing `position` as one more than the current max position for that playlist (falling back to 1 for an empty playlist), and setting `added_by` to the user performing the add and `added_at` to the current UTC time — mirroring the pattern already used in `seed_data.py`. I verified this by adding a song to a seeded playlist through the real `POST /playlists/<playlist_id>/songs` endpoint and confirming the new row had a correct incremented `position`, the correct `added_by`, and a populated `added_at`, with the playlist's song count increasing as expected and no errors raised.
