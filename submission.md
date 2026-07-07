# Mixtape — Codebase Map

Mixtape is a Flask + SQLAlchemy JSON API for a social music app. It includes services such as share songs, rate songs, build collaborative playlists, track listening streaks, and see what friends are playing. The format is layered as **routes → services → models**.

## Main files

- **`app.py`** — App factory. Defines the shared `db` instance, reads config from env, registers the four blueprints (`/songs`, `/playlists`, `/users`, `/feed`), and runs `db.create_all()`.
- **`models.py`** — 6 models (User, Song, Tag, ListeningEvent, Rating, Notification) + 3 association tables. Ratings are a separate model (unique per user+song), not a column on Song. `playlist_entries` is a join table carrying extra columns (`position`, `added_by`, `added_at`) so playlist songs have explicit order.
- **`routes/`** — Thin HTTP layer, only has one blueprint per file. Each route parses input, validates presence, calls one service function, and turns `ValueError` into a 400/404. `songs.py` (search/get/rate/listen), `playlists.py` (create/get/list songs/add song), `users.py` (profile/streak/notifications), `feed.py` (listening-now/activity).
- **`services/`** — All business logic:
  - `streak_service.py` — records listening events, increments/resets streaks by calendar day.
  - `feed_service.py` — "friends listening now" (24h window, one song per friend) and the activity feed.
  - `search_service.py` — case-insensitive title/artist search.
  - `notification_service.py` — creates notifications, rates songs, adds songs to playlists.
  - `playlist_service.py` — playlist create/read, ordered song retrieval.
- **`seed_data.py`** — populates the DB with test data. **`tests/`** — pytest suites for streaks, search, playlists.

## Data flow — adding a song to a playlist notifies the sharer

1. `POST /playlists/<id>/songs` with `{song_id, added_by}`.
2. `routes/playlists.py::add_song` validates the body and calls `notification_service.add_to_playlist()`.
3. That service loads the song, adder, and playlist; appends the song to the playlist; then **if `song.shared_by != added_by`**, calls `create_notification()` targeting the song's original sharer.
4. `create_notification()` writes a Notification row for that sharer.
5. The sharer later reads it via `GET /users/<id>/notifications`.

The recipient is derived from `Song.shared_by` — the person who first shared the song, not the playlist owner.

## Patterns

- **Strict route→service→model layering.** Routes only parse/validate/format; all logic is in `services/`. 
- **`ValueError` as the universal error signal.** Services stay framework-agnostic and raise `ValueError`; routes decide 400 vs 404.
- **Event-sourced reads.** Feeds and streaks derive from the `ListeningEvent` log; only `listening_streak` is cached on User.
- **Docstrings describe intended behavior**, which makes the planted service bugs easy to spot by comparing them against the code.


## Bug Fix choices

- #1: My listening streak keeps resetting
- #4: I got notified when a friend added my song to a playlist but not when they rated it
- #5: The last song in a playlist never shows up

## How to Reproduce each Bug

### #1 — My listening streak keeps resetting

1. Pick a user and set their `last_listened_at` to a Saturday and `listening_streak` to some value (e.g. 5).
2. On the following **Sunday**, call `POST /songs/<song_id>/listen` with `{"user_id": "<user_id>"}`.
3. Check the streak via `GET /users/<user_id>/streak`.
4. **Expected:** streak increments to 6 (listened on consecutive days). **Actual:** streak resets to 1, because the Sunday (`weekday == 6`) guard blocks the increment.

### #4 — Notified when a song is added to a playlist but not when it's rated

1. As user A, share a song (so `song.shared_by == A`).
2. As user B, rate that song: `POST /songs/<song_id>/rate` with `{"user_id": "<B>", "score": 5}`.
3. As user A, fetch notifications: `GET /users/<A>/notifications`.
4. **Expected:** a `song_rated` notification for A. **Actual:** no notification exists, because `rate_song()` never calls `create_notification()` (unlike `add_to_playlist()`).

### #5 — The last song in a playlist never shows up

1. Create a playlist and add several songs to it via `POST /playlists/<playlist_id>/songs`.
2. Fetch the songs: `GET /playlists/<playlist_id>/songs`.
3. **Expected:** all added songs returned in order. **Actual:** the last (highest-position) song is missing and `count` is one short, because `get_playlist_songs()` slices the last result.

## Root Cause Analysis

### #1 — My listening streak keeps resetting

**How I found the root cause:** I traced the data flow from the symptom. `POST /songs/<id>/listen` in [routes/songs.py:50](routes/songs.py#L50) calls `record_listening_event()`, which delegates to `update_listening_streak()` in [services/streak_service.py:42](services/streak_service.py#L42). Reading that function's branch logic against its own docstring, the consecutive-day branch on line 73 had an extra `and today.weekday() != 6` clause the docstring never mentions, which made the code suspicious. 

**The root cause:** The consecutive-day increment condition was `days_since_last == 1 and today.weekday() != 6`. When the current day is a Sunday (`weekday() == 6`), a legitimate consecutive-day listen fails this condition and falls through to the `else` branch, which resets `listening_streak` to 1. So any streak that would have advanced onto a Sunday got wiped instead.

**My fix and side-effect check:** I removed the `and today.weekday() != 6` clause so the branch is just `days_since_last == 1`, matching the documented rule "if the user listened yesterday, streak increments by 1." Then I verified with the full streak suite (5 passed): the Sunday case now increments, the same-day case still doesn't double-count, and the skipped-day case still resets to 1 — confirming both sides of the day-gap boundary behave correctly.