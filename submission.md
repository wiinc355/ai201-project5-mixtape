# Mixtape Bug Hunt Submission

## AI Usage

- I used AI tooling primarily for codebase navigation and explanation, not blind bug diagnosis.
- During orientation, I used AI to trace route-to-service call chains and summarize what each service function was responsible for.
- During debugging, I used AI to compare similar code paths (for example, working notification flow versus missing notification flow) and to flag likely edge-case conditions to inspect.
- I verified all AI-assisted conclusions by directly reading the referenced files/functions and running tests before committing any fix.
- In places where AI guidance was broad, I narrowed scope by reproducing behavior with project tests and then validating the exact line-level root cause myself.

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

## Bug Reproduction Notes

- I reproduced each issue before patching by running the focused tests tied to that service area:
	- `pytest tests/test_streaks.py`
	- `pytest tests/test_search.py`
	- `pytest tests/test_playlists.py`
- For issue #2 and issue #4 (not covered by provided tests), I reproduced by tracing the route -> service call chain and validating behavior against service logic before implementing changes.
- After each fix, I reran relevant tests and then executed `pytest tests` as a regression check.

## Initial Triage Plan (First Three Issues)

- First: Issue #1 (`services/streak_service.py`) because it has focused tests and a user-facing correctness bug in core streak logic.
- Second: Issue #3 (`services/search_service.py`) because duplicate search rows are easy to reproduce and isolate at query level.
- Third: Issue #5 (`services/playlist_service.py`) because the last-song omission is deterministic and covered by playlist tests.
- Follow-up after the first three: address Issue #2 and Issue #4, then rerun full regression tests.

## Root Cause Analysis Entries

### Bug 1: My listening streak keeps resetting

- **Issue/Symptom:** Consecutive listening across Saturday -> Sunday reset streak instead of incrementing.
- **How Reproduced (before fix):** In `tests/test_streaks.py`, ran the Saturday/Sunday path (`test_streak_increments_on_sunday`) using `pytest tests/test_streaks.py`. Input state: `last_listened_at` on Saturday UTC, then call `update_listening_streak` on Sunday UTC.
- **How I Found the Root Cause:** Navigation path: `README.md` issue list -> `routes/songs.py` (`POST /<song_id>/listen`) -> `services/streak_service.py` (`record_listening_event` -> `update_listening_streak`). In `update_listening_streak`, the consecutive-day branch had an extra `today.weekday() != 6` condition, which directly explained why Sunday consecutive listens failed.
- **Root Cause:** `update_listening_streak()` had a hard-coded `today.weekday() != 6` guard that blocked Sunday increments even when `days_since_last == 1`.
- **Fix and Side-Effect Check:** Removed the Sunday-specific condition and incremented streak for any consecutive day. Side-effect checks: confirmed same-day listens still do not double-count and skipped-day behavior still resets via `pytest tests/test_streaks.py`.
- **Commit:** `62f46cc` — `fix: allow streak increments on Sunday transitions`

### Bug 2: Friends Listening Now shows people from yesterday

- **Issue/Symptom:** Listening-now feed included events from yesterday if they were within the last 24 hours.
- **How Reproduced (before fix):** Triggered feed logic with one friend event from yesterday evening and one from today morning. Yesterday event still appeared in listening-now because it was inside a rolling 24-hour window.
- **How I Found the Root Cause:** Navigation path: `README.md` issue list -> `routes/feed.py` (`/<user_id>/listening-now`) -> `services/feed_service.py` (`get_friends_listening_now`). The key moment was seeing `cutoff = datetime.now(timezone.utc) - timedelta(hours=24)`, which encodes recency-by-hours, not calendar-day “now”.
- **Root Cause:** `get_friends_listening_now()` used a rolling 24-hour window (`timedelta(hours=24)`) instead of a calendar-day boundary expected by product behavior.
- **Fix and Side-Effect Check:** Replaced rolling threshold with start-of-current-UTC-day cutoff (`00:00:00` UTC). Side-effect checks: retained ordering and per-friend deduplication behavior, then ran `pytest tests` to ensure no regressions in adjacent features.
- **Commit:** `5fe7bde` — `fix: limit listening-now feed to today's events`

### Bug 3: The same song keeps showing up twice in search

- **Issue/Symptom:** Search returned duplicate songs when matched songs had multiple tags.
- **How Reproduced (before fix):** Ran `pytest tests/test_search.py` and targeted the multi-tag condition (`test_search_no_duplicates_multi_tag_song`). Trigger condition: a matching song with multiple rows in `song_tags` from the outer join path.
- **How I Found the Root Cause:** Navigation path: `README.md` issue list -> `routes/songs.py` (`/search`) -> `services/search_service.py` (`search_songs`). I checked query composition and confirmed it outer-joined `song_tags` but returned raw `Song` rows without deduplication, matching the duplicate-on-multi-tag symptom exactly.
- **Root Cause:** Query outer-joined `song_tags`, producing one row per matching tag; results were returned directly without deduplication.
- **Fix and Side-Effect Check:** Added `.distinct()` to the song query before `.all()`. Side-effect checks: ensured no-tag and single-tag songs still return once and that no-match queries still return empty via `pytest tests/test_search.py`.
- **Commit:** `84e8f10` — `fix: deduplicate song rows in tag-joined search`

### Bug 4: Missing notification when a friend rated my song

- **Issue/Symptom:** Song owner received playlist-add notifications but not rating notifications.
- **How Reproduced (before fix):** Compared side effects of adding to playlist vs rating the same shared song. Playlist action created a notification for the owner; rating action only stored rating data and produced no notification.
- **How I Found the Root Cause:** Navigation path: `README.md` issue list -> `routes/playlists.py` and `routes/songs.py` -> `services/notification_service.py`. Line-by-line comparison of `add_to_playlist()` and `rate_song()` showed only playlist flow called `create_notification` for owner interactions.
- **Root Cause:** `rate_song()` persisted rating changes but never triggered notification creation for the song owner.
- **Fix and Side-Effect Check:** Added notification creation after rating commit when rater is not the song owner. Side-effect checks: preserved score validation and upsert behavior for existing ratings, and prevented self-notification for owner self-ratings.
- **Commit:** `7052601` — `fix: notify song owner when friend rates shared song`

### Bug 5: The last song in a playlist never shows up

- **Issue/Symptom:** Playlist song list omitted final track.
- **How Reproduced (before fix):** Ran `pytest tests/test_playlists.py` and used a seeded 5-song playlist from fixture data (`test_playlist_returns_all_songs`). Trigger condition: retrieval path in `get_playlist_songs` always slicing the final element.
- **How I Found the Root Cause:** Navigation path: `README.md` issue list -> `routes/playlists.py` (`/<playlist_id>/songs`) -> `services/playlist_service.py` (`get_playlist_songs`). In the return statement, `songs[:-1]` directly explained why one item was always dropped.
- **Root Cause:** `get_playlist_songs()` returned `[song.to_dict() for song in songs[:-1]]`, which always sliced off the last result.
- **Fix and Side-Effect Check:** Returned full result list (`songs`) without slicing. Side-effect checks: verified ordering remained position-ascending and empty playlists still returned `[]` via `pytest tests/test_playlists.py`.
- **Commit:** `be28132` — `fix: include final playlist entry in song retrieval`
