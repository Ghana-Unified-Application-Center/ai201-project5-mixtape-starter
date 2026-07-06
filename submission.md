# Mixtape Bug Hunt — Submission

## Codebase Map

### Main files and their roles

- **`app.py`** — Flask application factory (`create_app`). Configures the SQLAlchemy
  `db` object, registers the four blueprints (`songs`, `playlists`, `users`, `feed`),
  and calls `db.create_all()`. This is the only entrypoint — the app must be started
  with `FLASK_APP=app:create_app flask run`, not `python app.py`, because `models.py`
  imports `db` from `app.py`, and running `app.py` directly re-imports the module
  under a different name (`__main__` vs `app`), creating two separate `SQLAlchemy`
  instances and duplicate table registrations.

- **`models.py`** — All SQLAlchemy models: `User`, `Song`, `Tag`, `ListeningEvent`,
  `Rating`, `Playlist`, `Notification`. Three association tables back the many-to-many
  relationships: `friendships` (symmetric user-to-user, inserted in both directions),
  `song_tags` (song-to-tag), and `playlist_entries` (playlist-to-song, extended with
  `position`, `added_by`, and `added_at` columns — songs in a playlist have an
  explicit order, not just insertion order). Every model has a `to_dict()` used
  directly by routes for JSON serialization — there's no separate serializer layer.

- **`routes/`** — Thin Flask blueprints. Every handler parses the request, calls
  exactly one service function, and wraps the result in `jsonify`. `ValueError`
  raised by a service becomes a 400/404 JSON error. No business logic lives here.
  - `songs.py` — search, get-by-id, rate, listen (this last one is what drives streaks).
  - `playlists.py` — create, get metadata, get songs, add song.
  - `users.py` — get user, get streak, get/mark-read notifications.
  - `feed.py` — friends-listening-now, activity feed.

- **`services/`** — All business logic. This is where the five open bugs live.
  - `streak_service.py` — increments/resets a user's `listening_streak` based on
    the gap between `last_listened_at` and the new listening event's date.
  - `feed_service.py` — builds "Friends Listening Now" (recent events within a
    rolling time window, deduplicated to one entry per friend) and a general,
    non-time-filtered activity feed.
  - `search_service.py` — searches songs by title/artist (case-insensitive
    substring match), joined against `song_tags` so tags can be attached to results.
  - `notification_service.py` — creates `Notification` rows when a friend interacts
    with a song a user shared, and lets a user list/mark-read their notifications.
  - `playlist_service.py` — creates playlists and returns a playlist's songs in
    position order.

- **`seed_data.py`** — Wipes and repopulates the DB with 5 users (with an explicit
  friendship graph), 25 songs (deliberately split into 0-tag / 1-tag / 3+-tag groups
  to exercise Issue #3), 3 playlists, a spread of `ListeningEvent`s from "10 minutes
  ago" out to "58 hours ago" (to exercise Issue #2's time window), and a couple of
  pre-existing streak values and notifications.

- **`tests/`** — `pytest` suites already exist for streaks, search, and playlists,
  and several of them assert the *fixed* behavior directly (with comments like
  `# Bug causes this to return 4`) — these tests currently fail against the
  unfixed code and act as an executable spec for the fix.

### Pattern I noticed

Every route delegates immediately to a service function named after the action
(`rate_song`, `add_to_playlist`, `get_playlist_songs`, ...); routes never touch
`db.session` directly except for the trivial `get_user` lookup in `users.py`. All
timestamps are created with `datetime.now(timezone.utc)` — the codebase is
consistently UTC-aware at creation time, though SQLite drops tzinfo on round-trip
(columns come back as naive datetimes), which matters when reasoning about any
date/time comparison bug.

### Data flow — a friend rates a shared song

1. Client calls `POST /songs/<song_id>/rate` with `{"user_id": ..., "score": ...}`
   (`routes/songs.py:rate`).
2. The route does input validation only (checks `user_id`/`score` are present,
   casts `score` to `int`) and calls `notification_service.rate_song(user_id,
   song_id, score)`.
3. `rate_song` (`services/notification_service.py`) validates the score is 1–5,
   loads the `Song` and rating `User`, and either updates an existing `Rating` row
   (enforced unique per `user_id`+`song_id` by a DB constraint) or inserts a new one.
4. The route returns the `Rating.to_dict()` as JSON with a 201.
5. **Compare to the sibling flow, `add_to_playlist`**: that function, after adding
   the song to the playlist, calls `create_notification(...)` to tell the song's
   original sharer (`song.shared_by`) that their song was added — *if* the adder
   isn't the sharer. `rate_song` has no equivalent call. This asymmetry (one
   write-path notifies, the structurally identical one doesn't) is Issue #4.

### Data flow — a song gets added to a friend's feed (listening)

1. Client calls `POST /songs/<song_id>/listen` with `{"user_id": ...}`
   (`routes/songs.py:listen`).
2. The route calls `streak_service.record_listening_event(user_id, song_id)`.
3. That creates a `ListeningEvent` row stamped `datetime.now(timezone.utc)`, then
   calls `update_listening_streak(user, now)` to adjust `User.listening_streak` /
   `last_listened_at`, then commits both in one transaction.
4. Separately, `feed_service.get_friends_listening_now(user_id)` (called from
   `GET /feed/<user_id>/listening-now`) queries all `ListeningEvent`s from the
   caller's friends within the last `RECENT_THRESHOLD` (24h), orders them
   newest-first, and keeps only the first (most recent) event per friend.

## The Five Issues — Investigation Notes

I read all five issue descriptions before starting and reproduced each with a
Python script against the seeded DB (not guesswork) before changing any code.
Full findings for the bugs I fixed are in the Root Cause Analysis section below.

For **Issue #2** (Friends Listening Now / stale "yesterday" entries) and
**Issue #3** (duplicate search results), I made a genuine reproduction attempt
before choosing to focus my three (plus stretch) fixes elsewhere — details below,
since the brief explicitly says "if you can't reproduce a bug after a genuine
attempt, try a different one."

*(RCA entries below will be filled in as each bug is fixed.)*
