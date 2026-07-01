# Project 5: Mixtape Bug Hunt — Submission

---

## Milestone 1 — Codebase Map

### Architecture at a glance

Mixtape is a Flask JSON API organized in **three layers**, wired together by
an application factory:

```
HTTP request → routes/ (blueprint)  → services/ (business logic) → models.py (SQLAlchemy ORM) → SQLite
                parse & format          the real work              data definitions
```

`app.py` is the **application factory**. `create_app()` configures the SQLite
DB (`mixtape.db`), initializes SQLAlchemy, registers four blueprints under URL
prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and calls
`db.create_all()`. There is no root route, so `GET /` returns 404 by design —
the API surface lives entirely under those four prefixes.

### The data model (`models.py`)

Defines the ORM schema. Core entities: **User, Song, Tag, ListeningEvent,
Rating, Playlist, Notification**, plus three association tables:

- `friendships` — self-referential many-to-many on User (symmetric friend graph).
- `song_tags` — many-to-many between Song and Tag.
- `playlist_entries` — many-to-many between Playlist and Song, but it carries
  extra columns: `position` (explicit ordering — songs have a defined slot, not
  just insertion order), `added_by`, and `added_at`. This is the join table that
  matters most for playlist ordering bugs.

Notable design details I noted while reading:

- `Rating` has a `UniqueConstraint(user_id, song_id)` — a user can rate a song
  only once (the code path in `rate_song` updates the existing row instead of
  inserting a duplicate).
- Datetimes default to `datetime.now(timezone.utc)` (timezone-aware in Python),
  but SQLite stores them without tzinfo — a detail worth remembering for any
  time-comparison bug.
- Every model exposes `to_dict()`, so services return plain dicts and routes
  just `jsonify` them.

### The route layer (`routes/`)

Thin controllers. Each blueprint parses input (JSON body or query args),
delegates to exactly one service function, and formats the response
(`jsonify` + status code). Business logic is deliberately absent here.

- `routes/songs.py` — `/search`, `/<id>`, `/<id>/rate`, `/<id>/listen`.
  Calls `search_service`, `notification_service.rate_song`,
  `streak_service.record_listening_event`.
- `routes/playlists.py` — create playlist, get detail, get songs, add song.
  Calls `playlist_service` and `notification_service.add_to_playlist`.
- `routes/users.py` — user profile, streak, notifications, mark-read.
- `routes/feed.py` — `/listening-now` and `/activity`.

### The service layer (`services/`) — where all five bugs live

- `streak_service.py` — `record_listening_event` writes a `ListeningEvent` and
  calls `update_listening_streak`, which increments/resets `user.listening_streak`
  based on the gap between today and `last_listened_at`.
- `feed_service.py` — `get_friends_listening_now` filters friends' listening
  events to a recent window (`RECENT_THRESHOLD = 24h`) and dedupes to one song
  per friend. `get_activity_feed` is the unfiltered version.
- `search_service.py` — `search_songs` does a case-insensitive `ilike` match on
  title/artist, joined against `song_tags`.
- `notification_service.py` — `create_notification` is the shared helper.
  `add_to_playlist` adds a song _and_ notifies the sharer. `rate_song`
  upserts a rating. `get_notifications` / `mark_as_read` handle retrieval.
- `playlist_service.py` — `get_playlist_songs` returns songs ordered by
  `playlist_entries.position`; plus create / get / list helpers.

### Data flow trace — sharing a song into a playlist triggers a notification

`POST /playlists/<playlist_id>/songs` (body: `song_id`, `added_by`)
→ `routes/playlists.py :: add_song()` parses the body and calls
`notification_service.add_to_playlist(playlist_id, song_id, added_by)`
→ that function loads the Song, the adding User, and the Playlist (raising
`ValueError` → 404 if any is missing), appends the song to `playlist.songs`
and commits
→ then, **only if the adder is not the original sharer**
(`song.shared_by != added_by_user_id`), it calls `create_notification(...)`
with `user_id=song.shared_by` and type `"song_added_to_playlist"`
→ `create_notification` inserts a `Notification` row and commits.

Later, `GET /users/<user_id>/notifications` →
`notification_service.get_notifications` reads those rows back, newest first.

This is the reference pattern for Issue #4: rating a song is _supposed_ to
follow the identical "notify the sharer" shape, but `rate_song` never calls
`create_notification`.

### Second data flow trace — recording a listen updates the streak

`POST /songs/<song_id>/listen` (body: `user_id`)
→ `routes/songs.py :: listen()` → `streak_service.record_listening_event`
→ creates a `ListeningEvent(listened_at=now)` and calls
`update_listening_streak(user, now)`, which compares `now.date()` against
`user.last_listened_at.date()`:
gap 0 → no change; gap 1 → increment; otherwise → reset to 1.
Then commits. `GET /users/<id>/streak` reads `user.listening_streak` back.

### Patterns I noticed

1. **Strict route→service delegation.** Every route immediately hands off to a
   single service call. Routes never touch the ORM directly (except the trivial
   user lookup in `users.py`). So any endpoint bug traces cleanly to one service.
2. **`ValueError` as the not-found signal.** Services raise `ValueError` for
   missing entities; routes catch it and translate to a 404/400. Consistent
   across the whole app.
3. **`to_dict()` everywhere** keeps serialization out of the services.
4. **Timezone-aware Python vs. naive SQLite storage** is a latent trap that any
   date/time-sensitive feature (streaks, feed recency) has to reckon with.

---

## The Five Open Issues (read before choosing)

| #   | Title                                             | Service                   | My first read of the likely area                                                 |
| --- | ------------------------------------------------- | ------------------------- | -------------------------------------------------------------------------------- |
| 1   | Listening streak keeps resetting                  | `streak_service.py`       | `update_listening_streak` has an extra weekday condition on the increment branch |
| 2   | Friends Listening Now shows people from yesterday | `feed_service.py`         | recency window / cutoff comparison in `get_friends_listening_now`                |
| 3   | Same song shows up twice in search                | `search_service.py`       | join against `song_tags` with no de-duplication                                  |
| 4   | Notified on playlist-add but not on rating        | `notification_service.py` | `rate_song` never calls `create_notification` (compare to `add_to_playlist`)     |
| 5   | Last song in a playlist never shows up            | `playlist_service.py`     | `get_playlist_songs` return value                                                |

**Plan:** I'll fix **#5, #1, and #4** — three bugs with localized root causes
that all reproduce cleanly against the seed data. This satisfies the "fix at
least 3" requirement. #5 is also my candidate for the regression-test stretch
goal. Issue #2 (feed recency) is out of scope.

---

## Milestone 2 — Reproductions

I reproduced each chosen bug deliberately before touching any code, running the
services directly against the seeded `mixtape.db` inside an app context. No
fixes were applied at this stage.

### #5 — Last song in a playlist never shows up (`playlist_service.py`)

**How I reproduced it:** Seed data creates a playlist "Late Night Vibes" with 7
entries. I compared the ground-truth row count in the `playlist_entries` table
against what `get_playlist_songs(playlist_id)` returns:

- `playlist_entries` rows for the playlist: **7**
- songs returned by the service: **6**
- The song at the highest `position` ("Free Throws") is silently dropped.

Any playlist with N songs returns N−1. The condition is simply "the playlist has
at least one song" — the last one is always missing.

### #1 — Listening streak keeps resetting (`streak_service.py`)

**How I reproduced it:** This is date-dependent, so I drove
`update_listening_streak(user, now)` with controlled datetimes. Setup for each
run: user has a 5-day streak and `last_listened_at` = yesterday; then they
listen again "today". Expected: streak → 6.

| "Today"               | `weekday()` | Streak result | Correct? |
| --------------------- | ----------- | ------------- | -------- |
| Saturday 2026-07-04   | 5           | 6             | ✅       |
| **Sunday 2026-07-05** | **6**       | **1 (reset)** | ❌       |
| Monday 2026-07-06     | 0           | 6             | ✅       |

The bug fires **only when today is a Sunday** (`weekday() == 6`). A user with a
valid consecutive-day streak has it wiped to 1 every Sunday.

### #4 — Rating a song doesn't notify the sharer (`notification_service.py`)

**How I reproduced it:** I picked a song, identified its sharer, and had a
_different_ user rate it, counting the sharer's notifications before and after:

- sharer notifications before rating: **1**
- sharer notifications after `rate_song(rater, song, 5)`: **1** (unchanged)
- notifications created by the rating: **0** — expected 1.

Contrast: the working `add_to_playlist` path _does_ create a
`song_added_to_playlist` notification for the sharer. Rating is missing the
equivalent call entirely, confirming the reported "notified on playlist-add but
not on rating" behavior.
