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