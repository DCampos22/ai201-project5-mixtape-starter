# Mixtape Bug Hunt — Submission

---

## AI Usage

I used Claude to help orient myself in the codebase. I pasted each service file
and asked it to explain what the module was responsible for and what each function
does. This helped me build a mental model of the app faster than reading cold.
For each bug, I read the relevant service file myself first, formed a hypothesis,
then used Claude to verify my understanding of specific Python behavior (like
datetime.weekday() return values). I wrote all root cause analyses myself based
on my own reading of the code.

---

## Codebase Map

### Main Files

**app.py** — Flask application factory. Creates the app, configures the database,
and registers the four blueprints (songs, playlists, users, feed). Also initializes
SQLAlchemy. Never run directly — always use FLASK_APP=app:create_app flask run.

**models.py** — Defines all SQLAlchemy models: User, Song, Playlist, Rating,
ListeningEvent, Notification, Tag. Also defines two association tables:
`playlist_entries` (links songs to playlists with a position column) and
`song_tags` (links songs to tags). The User model has a self-referential
friends relationship.

**routes/songs.py** — Handles song search, song detail, rating, and listen
events. All business logic is delegated to service functions immediately.

**routes/playlists.py** — Handles playlist creation, retrieving playlist songs,
and adding songs to playlists.

**routes/users.py** — Handles user profiles, streaks, and notifications.

**routes/feed.py** — Handles the Friends Listening Now feed and activity feed.

**services/streak_service.py** — Manages listening streak logic. Tracks
consecutive calendar days a user has listened. Increments, resets, or holds
the streak depending on when the user last listened.

**services/feed_service.py** — Manages the Friends Listening Now feed. Filters
friends' listening events by a recency threshold and deduplicates to show only
the most recent song per friend.

**services/search_service.py** — Handles song search by title or artist using
case-insensitive LIKE queries. Also provides single song lookup by ID.

**services/notification_service.py** — Creates and retrieves notifications.
Notifications are generated when friends interact with a user's shared songs
(e.g. adding to a playlist or rating). Also handles the rate_song logic.

**services/playlist_service.py** — Handles playlist creation and retrieval.
Songs in a playlist are ordered by a position column in the playlist_entries
join table.

---

### Data Flow — User Rates a Song

1. Client sends `POST /songs/<song_id>/rate` with `user_id` and `score`
2. `routes/songs.py` parses the request and calls `notification_service.rate_song()`
3. `rate_song()` validates the score (1–5), looks up the song and user, checks
   for an existing rating, creates or updates the Rating record, commits to DB
4. Returns the Rating instance to the route, which serializes it as JSON

Pattern noticed: every route immediately delegates to a service function. Routes
handle input parsing and response formatting only — all business logic lives in
services/. This makes bugs easy to locate: if an endpoint is broken, the bug
is in the service it calls.

---

## Bug Fixes

### Bug 1 — Listening streak keeps resetting on Sundays

**How I reproduced it:**
Listened to a song on a Saturday (days_since_last = 1), then listened again the
next day (Sunday). Expected the streak to increment. Instead it reset to 1.

**How I found the root cause:**
Read `streak_service.py` and found `update_listening_streak()`. The elif branch
that increments the streak had an extra condition: `and today.weekday() != 6`.
I knew weekday() returns 6 for Sunday, so this condition blocks the increment
on Sundays specifically.

**Root cause:**
Python's `datetime.weekday()` returns 6 for Sunday. The streak increment condition
was `days_since_last == 1 and today.weekday() != 6` — which means "listened
yesterday AND today is not Sunday." Any user who listened on Saturday and then
again on Sunday would hit this branch, fail the weekday check, and fall through
to the else which resets the streak to 1. Sunday was the only day of the week
where a valid consecutive listen would reset instead of increment.

**Fix and side-effect check:**
Removed the `and today.weekday() != 6` condition entirely. The correct logic is
simply: if the user listened yesterday, increment the streak — no day-of-week
exception needed. Verified the else branch (streak reset) still fires correctly
when days_since_last > 1.

---

### Bug 4 — No notification when a friend rates your song

**How I reproduced it:**
Submitted a `POST /songs/<song_id>/rate` request with a different user_id than
the song's sharer. Checked notifications for the sharer — none appeared.
Compared to adding a song to a playlist, which does generate a notification.

**How I found the root cause:**
Read `notification_service.py` and compared `add_to_playlist()` to `rate_song()`.
`add_to_playlist()` calls `create_notification()` after doing its work.
`rate_song()` saves the rating and commits — but never calls `create_notification()`.
The notification block was simply missing.

**Root cause:**
The `rate_song()` function handles saving ratings correctly but has no notification
logic. Unlike `add_to_playlist()` which notifies the song's original sharer,
`rate_song()` never calls `create_notification()`. The feature was either never
implemented or was accidentally deleted.

**Fix and side-effect check:**
Added a notification block to `rate_song()` after the db.session.commit(), matching
the pattern from `add_to_playlist()`: check that the rater is not the original
sharer, then call `create_notification()` with type `"song_rated"` and a message
like `"{rater.username} rated your song '{song.title}' {score}/5."` Verified
that rating your own song does not generate a notification.

---

### Bug 5 — Last song in a playlist never shows up

**How I reproduced it:**
Created a playlist with 3 songs, then called `GET /playlists/<id>/songs`.
Only 2 songs returned — the last one was always missing regardless of which
song was added last.

**How I found the root cause:**
Read `playlist_service.py` and found `get_playlist_songs()`. The return statement
was `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops
the last element of any Python list.

**Root cause:**
The list slice `songs[:-1]` removes the last song from the results before
returning. This is a Python slicing error — `[:-1]` means "all elements except
the last one." A playlist with 5 songs returns 4. A playlist with 1 song returns
0. The docstring even says "This function returns all songs in the playlist"
which confirms the slice is unintentional.

**Fix and side-effect check:**
Changed `return [song.to_dict() for song in songs[:-1]]` to
`return [song.to_dict() for song in songs]`. Verified that playlist ordering
(by position column) is unaffected since the fix only removes the incorrect slice.