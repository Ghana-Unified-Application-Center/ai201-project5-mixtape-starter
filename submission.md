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

## Chosen Bugs & Reproduction (Milestone 2)

I read all five issue descriptions before starting, then reproduced each one
against a freshly-seeded DB — either through the live HTTP API (`curl`) or by
calling the service function directly with controlled inputs — before writing
any fix code. The three I'm fixing (with a stretch goal of a 4th) are **#1, #4,
and #5**, all reproduced below. I also made a genuine attempt at #2 and #3 and
could not trigger the reported behavior with the code as written — details in
their own section at the end, rather than skipped silently.

### Issue #1 — Listening streak keeps resetting (Sunday boundary)

**How I reproduced it:** I first tried triggering it through the real `/listen`
endpoint using the server's actual wall-clock time, expecting "today" to be a
Sunday — but the server's `datetime.now(timezone.utc)` had already rolled over
to Monday UTC even though it was still Sunday night locally, so that route
didn't hit the buggy branch. Rather than wait for a real Sunday, I reproduced it
deterministically the same way the existing (currently failing) test
`tests/test_streaks.py::test_streak_increments_on_sunday` does: called
`update_listening_streak()` directly with two explicit, one-day-apart
timestamps, Saturday 2026-07-04 and Sunday 2026-07-05, on a fresh user with no
prior streak.

```python
from services.streak_service import update_listening_streak
saturday = datetime(2026, 7, 4, 20, 0, 0, tzinfo=timezone.utc)  # weekday()==5
sunday   = datetime(2026, 7, 5, 20, 0, 0, tzinfo=timezone.utc)  # weekday()==6
update_listening_streak(nova, saturday)   # streak -> 1
update_listening_streak(nova, sunday)     # streak stays 1 (bug) instead of -> 2
```

Result: streak stayed at `1` after the Sunday listen instead of incrementing to
`2`, even though Sunday is one calendar day after Saturday — a genuine
consecutive-day streak. Confirmed independently by running
`pytest tests/test_streaks.py -v`, where `test_streak_increments_on_sunday`
fails with `assert 1 == 2`.

### Issue #4 — No notification when a friend rates your song

**How I reproduced it:** Seeded fresh data, took a song shared by `nova` and
rated it as her friend `darius` via the live API, then diffed `nova`'s
notification list before and after:

```bash
curl http://127.0.0.1:5050/users/<nova_id>/notifications         # count: 1 (the seeded playlist-add notification)
curl -X POST http://127.0.0.1:5050/songs/<song_id>/rate \
  -H "Content-Type: application/json" \
  -d '{"user_id": "<darius_id>", "score": 5}'                    # 201, rating created
curl http://127.0.0.1:5050/users/<nova_id>/notifications         # count: still 1
```

Result: the rating succeeds (201, `Rating` row created), but `nova`'s
notification count doesn't change — no `song_rated` notification is ever
created, confirming the reported behavior.

### Issue #5 — Last song in a playlist never shows up

**How I reproduced it:** Queried the `playlist_entries` association table
directly for a seeded playlist to get the ground truth, then hit the real API
for the same playlist and compared counts:

```python
# ground truth: 7 rows in playlist_entries for this playlist, positions 1-7
```
```bash
curl http://127.0.0.1:5050/playlists/<playlist_id>/songs
# {"count": 6, "songs": [...]}   <- position 7 ("Golden Hour"'s successor) is missing
```

Result: the DB has 7 songs in the playlist but the API returns only 6 — the
song at the highest position (the last one added) is silently dropped every
time, regardless of playlist size.

### Issues #2 and #3 — attempted but not reproducible here

- **Issue #2 (Friends Listening Now shows people from yesterday):** I checked
  every friend pair in the seeded data and manually swept the 24-hour cutoff
  boundary (inserted events at 23h59m and 24h01m old) — `get_friends_listening_now`
  correctly includes/excludes on both sides of the boundary, and correctly
  picks each friend's *most recent* event when they have several. I could not
  construct a case where a >24h-old event leaked into the results.
- **Issue #3 (duplicate search results):** The `search_songs` query does
  `outerjoin` against `song_tags` without `.distinct()`, which *does* fan out
  into duplicate rows at the raw-SQL level for a song with 3+ tags (confirmed
  by inspecting the compiled SQL and running it directly). But the installed
  SQLAlchemy version (2.0.51) deduplicates full-entity results by identity in
  legacy `Query.all()`, so `search_songs("Crown Heights")` — the seed data's
  own 3-tag song — still returns exactly one result. I could not get a
  duplicate to actually surface through the service function or the live
  `/songs/search` endpoint.

Per the brief's own guidance ("if you can't reproduce a bug after a genuine
attempt, try a different one from the list"), I'm proceeding with #1, #4, #5,
and will revisit #2/#3 for the stretch goals if time allows.

*(RCA entries below will be filled in as each bug is fixed, per Milestone 3.)*
