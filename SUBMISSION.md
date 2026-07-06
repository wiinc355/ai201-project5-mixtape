# Mixtape Bug Hunt Submission

## Codebase Map

- `app.py`: Flask app factory (`create_app`), SQLAlchemy initialization, and blueprint registration.
- `models.py`: Core SQLAlchemy entities (`User`, `Song`, `Playlist`, `ListeningEvent`, `Rating`, `Notification`, `Tag`) plus association tables (`friendships`, `song_tags`, `playlist_entries`).
- `routes/songs.py`: Song endpoints for search, song detail, rating, and listen events.
- `routes/playlists.py`: Playlist endpoints for creation, details, listing songs, and adding songs.
- `routes/users.py`: User profile, streak, and notification endpoints.
- `routes/feed.py`: Friends listening-now and activity feed endpoints.
- `services/streak_service.py`: Streak update logic and listen event recording.
- `services/feed_service.py`: Listening-now and activity feed query logic.
- `services/search_service.py`: Song search and single-song lookup.
- `services/notification_service.py`: Notification creation/retrieval plus rating and playlist side effects.
- `services/playlist_service.py`: Playlist creation and ordered playlist-song retrieval.
- `tests/test_streaks.py`: Streak behavior tests.
- `tests/test_search.py`: Search behavior tests, including duplicate-result coverage.
- `tests/test_playlists.py`: Playlist retrieval and ordering tests.

### Data Flow Example: Rating a Song

1. Client calls `POST /songs/<song_id>/rate` with `user_id` and `score`.
2. Route handler in `routes/songs.py` validates payload and calls `services.notification_service.rate_song()`.
3. `rate_song()` validates user/song/score, upserts a `Rating` row, commits, and conditionally creates a `Notification` for the song owner.
4. The route returns `rating.to_dict()` as JSON.

## Root Cause Analysis Entries

### Bug 1: My listening streak keeps resetting

- **Issue/Symptom:** Consecutive listening across Saturday -> Sunday reset streak instead of incrementing.
- **Root Cause:** `update_listening_streak()` had a hard-coded `today.weekday() != 6` guard that blocked Sunday increments even when `days_since_last == 1`.
- **Fix Implemented:** Removed the Sunday-specific condition and incremented streak for any consecutive day.
- **Verification:** Ran `pytest tests/test_streaks.py`; Sunday coverage test now passes.
- **Commit:** `62f46cc` — `fix: allow streak increments on Sunday transitions`

### Bug 2: Friends Listening Now shows people from yesterday

- **Issue/Symptom:** Listening-now feed included events from yesterday if they were within the last 24 hours.
- **Root Cause:** `get_friends_listening_now()` used a rolling 24-hour window (`timedelta(hours=24)`) instead of a calendar-day boundary expected by product behavior.
- **Fix Implemented:** Replaced rolling threshold with start-of-current-UTC-day cutoff (`00:00:00` UTC).
- **Verification:** Ran full test suite (`pytest tests`); no regressions. Manual logic check confirms yesterday entries are excluded.
- **Commit:** `5fe7bde` — `fix: limit listening-now feed to today's events`

### Bug 3: The same song keeps showing up twice in search

- **Issue/Symptom:** Search returned duplicate songs when matched songs had multiple tags.
- **Root Cause:** Query outer-joined `song_tags`, producing one row per matching tag; results were returned directly without deduplication.
- **Fix Implemented:** Added `.distinct()` to the song query before `.all()`.
- **Verification:** Ran `pytest tests/test_search.py`; multi-tag duplicate test now passes.
- **Commit:** `84e8f10` — `fix: deduplicate song rows in tag-joined search`

### Bug 4: Missing notification when a friend rated my song

- **Issue/Symptom:** Song owner received playlist-add notifications but not rating notifications.
- **Root Cause:** `rate_song()` persisted rating changes but never triggered notification creation for the song owner.
- **Fix Implemented:** Added notification creation after rating commit when rater is not the song owner.
- **Verification:** Ran full test suite (`pytest tests`) to confirm no regressions; route/service flow now includes notification side effect.
- **Commit:** `7052601` — `fix: notify song owner when friend rates shared song`

### Bug 5: The last song in a playlist never shows up

- **Issue/Symptom:** Playlist song list omitted final track.
- **Root Cause:** `get_playlist_songs()` returned `[song.to_dict() for song in songs[:-1]]`, which always sliced off the last result.
- **Fix Implemented:** Returned full result list (`songs`) without slicing.
- **Verification:** Ran `pytest tests/test_playlists.py`; all playlist count/order tests pass.
- **Commit:** `be28132` — `fix: include final playlist entry in song retrieval`