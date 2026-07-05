# Mixtape Bug Hunt — Submission

---

## Codebase Map

### Main Files

**`app.py`** — The Flask app factory. Creates and configures the app,
connects the database, and registers the 4 route blueprints (songs,
playlists, users, feed). Always started with
`FLASK_APP=app:create_app flask run`.

**`models.py`** — Defines all 6 SQLAlchemy database models:
- `User` — has a username, listening streak counter, and last_listened_at
  timestamp. Friends are a many-to-many self-join.
- `Song` — belongs to a user (shared_by), has tags via a many-to-many
  join table.
- `ListeningEvent` — records when a user listened to a song, with a
  timestamp.
- `Rating` — a user's 1–5 score on a song. One rating per user/song pair
  enforced by a unique constraint.
- `Playlist` — has songs via playlist_entries, which adds a position
  column for explicit ordering.
- `Notification` — a message for a user with a type, body, and read flag.

**`routes/`** — Four blueprint files (songs, playlists, users, feed).
Each route does input parsing and response formatting only — all business
logic is delegated immediately to a service function.

**`services/`** — Five service files, one per feature area. All bugs live
here.

### Data Flow Example — User Rates a Song

1. Client sends `POST /songs/<song_id>/rate` with a score
2. `routes/songs.py` parses the user ID and score from the request
3. It calls `notification_service.rate_song(user_id, song_id, score)`
4. `rate_song()` validates the score, looks up the song and user,
   saves or updates a Rating record, then (after the fix) calls
   `create_notification()` to notify the song's original sharer
5. The route returns the rating as JSON

### Patterns I Noticed
- Every route delegates immediately to a service — routes never touch
  the database directly.
- The `playlist_entries` association table has a `position` column,
  meaning songs in a playlist have an explicit order, not just insertion
  order.
- Notifications are always created the same way: check if the actor is
  different from the owner, then call `create_notification()`.

---
## Screenshot
<img width="582" height="172" alt="image" src="https://github.com/user-attachments/assets/25a1393e-5158-4da3-b9ab-f8b6d620343e" />


## Bug #1 — Listening streak keeps resetting

**How I reproduced it:**
Triggered `record_listening_event()` for a user on a Saturday, then again
on Sunday. Expected the streak to increment, but it reset to 1 instead.
Any listen on a Sunday after a Saturday would break the streak.

**How I found the root cause:**
Followed the call chain: route → `streak_service.record_listening_event()`
→ `update_listening_streak()`. Inside that function, the elif branch that
increments the streak had an extra condition that looked suspicious:
`today.weekday() != 6`

**Root cause:**
Python's `datetime.weekday()` returns 0 for Monday through 6 for Sunday.
The condition `today.weekday() != 6` means "today is not Sunday."
So even when a user listened on consecutive days (Saturday → Sunday),
the increment branch was skipped because Sunday matched `weekday() == 6`,
and the streak fell through to the else branch and reset to 1.
Sunday was incorrectly treated as a streak-breaking day.

**Fix and side-effect check:**
Removed the `and today.weekday() != 6` condition entirely, so the branch
now reads `elif days_since_last == 1`. The other two cases (listened today,
or skipped a day) were correct and untouched. Verified that listening on
Monday still correctly continues a Sunday streak.

## Bug #2 — No notification when a friend rates my song

**How I reproduced it:**
Had one user rate a song shared by a different user, then checked the
sharer's notifications via `GET /users/<id>/notifications`. The notification
for the rating never appeared, even though rating a song is supposed to
notify the original sharer.

**How I found the root cause:**
The README traced the call chain: `POST /songs/<id>/rate` →
`notification_service.rate_song()`. I read that function and compared it
line by line to `add_to_playlist()`, which handles a similar action.
`add_to_playlist()` calls `create_notification()` after doing its work.
`rate_song()` never did — that call was simply absent.

**Root cause:**
The `rate_song()` function correctly saves the rating to the database but
never calls `create_notification()`. The notification infrastructure
(the `create_notification()` helper, the Notification model, the
notification_type field) all existed and worked correctly — the call
was just missing entirely from `rate_song()`.

**Fix and side-effect check:**
Added a `create_notification()` call at the end of `rate_song()`, before
the return statement, guarded by `if song.shared_by != user_id` so users
don't get notified when they rate their own songs. This mirrors the exact
same pattern used in `add_to_playlist()`. Verified that
`get_notifications()` and `mark_as_read()` were unaffected.


## Bug #3 — The last song in a playlist never shows up

**How I reproduced it:**
Called `GET /playlists/<id>/songs` on any playlist with multiple songs.
The last song in the playlist was always missing from the response,
regardless of which playlist or how many songs it had.

**How I found the root cause:**
Followed the call chain: route → `playlist_service.get_playlist_songs()`.
The function correctly queries songs ordered by position, but the return
statement immediately caught my eye.

**Root cause:**
In `get_playlist_songs()`, the return statement was:
`return [song.to_dict() for song in songs[:-1]]`

The `[:-1]` slice in Python means "everything except the last element."
So no matter how many songs were in the playlist, the final one was always
dropped before the list was returned.

**Fix and side-effect check:**
Changed `songs[:-1]` to `songs` so all results are returned.
Checked `get_playlist()` and `get_user_playlists()` — neither touches
this slice, so they were unaffected. The query and ordering logic was
correct all along; only the return statement was wrong.

## AI Usage
I used Claude (Anthropic) as an assistant during this project.
Claude helped me understand what suspicious lines of code actually did
(e.g. what Python's `weekday()` returns for each day of the week) and
explained the difference between working and broken code paths.
For each bug, I read and verified the root cause myself before making
any changes. All fixes and RCA entries reflect my own understanding.
