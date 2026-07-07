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