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