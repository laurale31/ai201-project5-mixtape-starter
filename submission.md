## Bug #1 — Listening streak keeps resetting

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

## Bug #4 — No notification when a friend rates my song

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


## Bug #5 — The last song in a playlist never shows up

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