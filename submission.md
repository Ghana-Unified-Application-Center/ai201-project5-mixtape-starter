# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude Code (an AI coding assistant) throughout this project, in the
codebase-navigation and debugging phases specifically, not just for writing
the final fix diffs.

**Codebase orientation (Milestone 1):** I had it read every file in `app.py`,
`models.py`, `routes/`, and `services/` up front and summarize each module's
responsibility, then trace the two data flows that ended up in the codebase
map (rating a song, and listening → streak → friends feed). This was genuinely
useful for getting oriented fast — it correctly noticed the routes-delegate-
to-services pattern and the UTC-datetime convention on its own, which saved
time I'd otherwise have spent forming that picture file-by-file.

**Where it helped during debugging:** Once a suspicious line was identified
(e.g., `today.weekday() != 6` in `streak_service.py`, or `songs[:-1]` in
`playlist_service.py`), it was useful for quickly confirming factual details I
could otherwise get wrong from memory — e.g., confirming that Python's
`datetime.weekday()` maps Monday→0...Sunday→6 (vs. `isoweekday()`'s
Monday→1...Sunday→7), which mattered for stating the Issue #1 root cause
precisely rather than just gesturing at "a weekday bug."

**Where I had to verify or override it — this is the important part:**
- For Issue #1, my first attempt was to reproduce the Sunday bug through the
  live `/listen` endpoint using the *actual* server clock, on the assumption
  that "today" was a Sunday. It wasn't a hypothesis error exactly, but an
  unverified assumption: the server's `datetime.now(timezone.utc)` had already
  rolled over to Monday UTC while it was still Sunday night in local time. I
  caught this only by actually printing the weekday computed inside the
  request instead of trusting the assumed date, and switched to a
  deterministic repro with explicit timestamps instead — the same technique
  the pre-existing test suite already used.
- For Issues #2 and #3, my working hypotheses going in (a naive/aware
  datetime mismatch breaking the 24-hour feed cutoff; a missing `.distinct()`
  causing duplicate search rows) were both *plausible from reading the code*
  but turned out to be **not actually reproducible** once I ran real queries
  against the seeded data. For #3 specifically, I confirmed with the AI's help
  that the raw SQL really does fan out into duplicate rows for a multi-tag
  song — but then verified directly (not by asking the AI, by executing it)
  that this SQLAlchemy version's `Query.all()` deduplicates full ORM entities
  by identity before they ever reach `to_dict()`, which is exactly the kind of
  environment-specific behavior an AI's static reading of the code can't be
  expected to know. I did not accept "this looks like the bug" as sufficient —
  in both cases I only wrote up a bug as reproduced after running code and
  observing the actual output, and documented the two I couldn't reproduce
  honestly rather than fixing a plausible-but-unverified cause.
- More generally, I used AI to explain and trace code I had already located
  myself, and to verify a hypothesis I had already formed by reading the
  source — not to search for "what's the bug" blind. Every root cause claim
  below was confirmed by running the actual function with controlled inputs
  and observing the output, not by taking an explanation at face value.

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

## Root Cause Analyses

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** Called `update_listening_streak()` directly (bypassing
HTTP entirely, since the bug depends on the calendar date rather than any input
a route accepts) with two explicit UTC timestamps one calendar day apart:
Saturday 2026-07-04 and Sunday 2026-07-05, on a user with no prior listening
history. After the Saturday call the streak was `1` as expected. After the
Sunday call — one consecutive day later — the streak was still `1` instead of
`2`. I also ran the pre-existing `tests/test_streaks.py::test_streak_increments_on_sunday`,
which failed with `assert 1 == 2` against the unfixed code, corroborating the
manual repro.

**How I found the root cause:** Started at `routes/songs.py:listen`, which
calls `streak_service.record_listening_event()`, which in turn calls
`update_listening_streak()` — the docstring right above it states the rule
plainly: "If the user listened yesterday: streak increments by 1." Reading the
function body line by line, the branch that's supposed to implement that rule is:
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```
The `and today.weekday() != 6` clause is what caught my eye — it isn't
mentioned anywhere in the docstring's stated rules, and there's no comment
explaining why a week boundary would matter for a *consecutive-day* streak. I
confirmed the exact index by checking Python's own mapping
(`datetime(2026, 7, 5).weekday()` → `6`), which matched the Sunday I'd used
to reproduce the bug — that was the moment I was confident this specific
comparison, not just "something in the streak logic," was the cause.

**The root cause:** Python's `datetime.weekday()` returns `6` for Sunday
(Monday is `0`). `update_listening_streak` added `and today.weekday() != 6` to
the one-day-gap branch, so whenever the *new* listening event's date happens to
be a Sunday, that condition evaluates to `False` even though `days_since_last
== 1` is `True`. Control falls through to the `else` clause, which sets
`listening_streak = 1` — the same behavior used for a genuinely skipped day.
The visible symptom ("keeps resetting") is exactly this: any user whose streak
would otherwise continue into a Sunday gets treated as if they'd missed a day,
every single week.

**Fix and side-effect check:** Removed the `and today.weekday() != 6` clause
entirely, leaving `elif days_since_last == 1: user.listening_streak += 1`. There
is no rule anywhere in the spec (docstring or issue) that a week boundary
should interrupt a consecutive-day streak, so the smallest correct fix is to
delete the erroneous condition rather than replace it with different weekday
arithmetic. To check for side effects I ran the full `tests/test_streaks.py`
suite (all 5 pass now, including the same-day no-op and skipped-day-reset
cases) and additionally swept all 7 possible weekday transitions
(Mon→Tue, Tue→Wed, ..., Sun→Mon) plus a "skip a day and land on Sunday" case
directly against `update_listening_streak` — every consecutive-day transition
now increments regardless of which day of the week it lands on, and a truly
skipped day still resets to 1 even when the skip lands on a Sunday, so the
fix doesn't weaken the reset behavior the bug was conflated with.

### Issue #4 — Notified when a friend adds my song to a playlist, but not when they rate it

**How I reproduced it:** Seeded fresh data, took a song shared by `nova`, and
recorded her notification count via `GET /users/<nova_id>/notifications`
(`count: 1`, the seeded playlist-add notification). Then rated that song as her
friend `darius` via `POST /songs/<song_id>/rate` with `{"user_id": "<darius_id>",
"score": 5}` — got back a `201` with a valid `Rating` object. Re-checked nova's
notifications: still `count: 1`. The rating clearly succeeded but produced no
notification, matching the reported behavior exactly.

**How I found the root cause:** Both `/playlists/<id>/songs` (POST) and
`/songs/<id>/rate` route to functions in the same file,
`services/notification_service.py` — `add_to_playlist()` and `rate_song()`. I
read `add_to_playlist()` first since it's the *working* case: it does its
write (appending the song to the playlist), commits, and then has an explicit
block —
```python
if song.shared_by != added_by_user_id:
    create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", ...)
```
I then read `rate_song()` top to bottom, line by line, expecting to find the
equivalent block after its `db.session.commit()`. The function instead just
`return`s the rating immediately after commit — there's no call to
`create_notification` anywhere in the function, and no other code path in the
file calls it on `rate_song`'s behalf. That was the moment I was confident:
this isn't a broken condition or a typo, it's a step that was simply never
written for this function, even though the module already had every piece
(`create_notification`, `song.shared_by`, the "skip if actor is the sharer"
check) needed to add it.

**The root cause:** `rate_song()` and `add_to_playlist()` are structurally
identical write-paths — both look up the song, perform their write, commit,
and (per the module's own established pattern) should notify `song.shared_by`
unless the acting user *is* the sharer. `add_to_playlist()` implements that
last step; `rate_song()` never had it implemented at all. This is an
architectural gap, not a logic error — the notification step is simply
missing from one of the two otherwise-parallel code paths.

**Fix and side-effect check:** Added the same notify-unless-self-sharer call
used by `add_to_playlist()` to the end of `rate_song()`, right after the
`db.session.commit()` that saves the rating, using a new `"song_rated"`
notification type and a body string describing the score given. To check for
side effects I verified three related scenarios directly against
`rate_song()`: (1) a friend rating someone else's song produces exactly one
new notification for the sharer; (2) a user rating **their own** song produces
no notification (mirroring the existing self-add exemption in
`add_to_playlist`); (3) updating an already-existing rating (the `existing`
branch) still fires a notification, matching `add_to_playlist`'s behavior of
notifying on every successful call rather than only the first. I also reran
the full test suite — all previously-passing tests (search, streak) stayed
green, confirming the change is isolated to the rating path.

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:** Seeded fresh data, picked a playlist, and queried the
`playlist_entries` association table directly to get ground truth:
7 rows, positions 1–7. Then hit `GET /playlists/<playlist_id>/songs` on the
same playlist and got back `{"count": 6, ...}` — the song at position 7 (the
one most recently added) was missing from the response entirely, while
positions 1–6 were all present and correctly ordered.

**How I found the root cause:** Went straight to `playlist_service.py` since
it's the only file `get_playlist_songs` (called from `routes/playlists.py:get_songs`)
lives in. The query itself — join `Song` to `playlist_entries`, filter by
playlist, order by `position` ascending — builds the exact right list of 7
songs in the correct order; I confirmed this by printing `len(songs)` right
after the query, which was `7`. The very next line is
`return [song.to_dict() for song in songs[:-1]]`. Comparing the query's own
`len() == 7` against the returned list's `len() == 6` pinpointed the `[:-1]`
slice as the exact point where a song gets dropped — and the function's own
docstring, two lines above the query ("This function returns all songs in the
playlist"), directly contradicts what the code does, which is what made me
confident this was the root cause rather than a plausible-looking area.

**The root cause:** `get_playlist_songs()` retrieves the fully correct,
position-ordered list of songs, but its `return` statement slices it with
`songs[:-1]`, which drops the last element of any non-empty list. Since the
query already orders by position ascending, "last element" always means the
song at the highest position — i.e., the most recently added song — so every
playlist, regardless of size, loses exactly one song: whichever one is last.
For a playlist with only one song, this slice returns an empty list.

**Fix and side-effect check:** Removed the `[:-1]` slice so the function
returns `[song.to_dict() for song in songs]` — the full list the query already
produces correctly. To check the boundary on both ends: I ran the two existing
tests that assert playlists return all 5 seeded songs in the correct order
(`test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order`) —
both now pass — plus `test_empty_playlist_returns_empty_list`, which still
passes since an empty list is unaffected by the fix either way. I also
manually built a single-song playlist and called `get_playlist_songs()`
directly: before the fix this returned `[]` (the worst case for `[:-1]`, since
it drops the *only* song), and after the fix it correctly returns that one
song. Ran the full test suite afterward (13/13 passing) to confirm nothing in
search or streaks was affected, since this file has no dependency on either.
