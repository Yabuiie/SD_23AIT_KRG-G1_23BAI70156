# API Design — Music Streaming Platform

All APIs follow RESTful conventions. Base URL: `https://api.musicapp.com/v1`

Authentication: `Authorization: Bearer <access_token>` (JWT) on all protected routes.

---

## Status Codes Used

| Code | Meaning |
|---|---|
| 200 | OK — successful GET / PUT / PATCH |
| 201 | Created — successful POST (resource created) |
| 204 | No Content — successful DELETE |
| 400 | Bad Request — invalid input / missing fields |
| 401 | Unauthorized — missing or invalid token |
| 403 | Forbidden — authenticated but not authorized |
| 404 | Not Found — resource doesn't exist |
| 409 | Conflict — duplicate (e.g., email already registered) |
| 422 | Unprocessable Entity — validation error |
| 429 | Too Many Requests — rate limit exceeded |
| 500 | Internal Server Error |

---

## 1. Auth Endpoints

### POST `/auth/register`
Register a new user.

**Request:**
```json
{
  "email": "user@example.com",
  "username": "john_doe",
  "password": "SecurePass123!",
  "country": "IN"
}
```

**Response `201`:**
```json
{
  "user": {
    "id": "uuid-1234",
    "email": "user@example.com",
    "username": "john_doe",
    "plan": "free"
  },
  "access_token": "eyJ...",
  "refresh_token": "eyJ..."
}
```

---

### POST `/auth/login`
**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

**Response `200`:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "expires_in": 900
}
```

---

### POST `/auth/refresh`
Exchange a refresh token for a new access token.

**Request:**
```json
{ "refresh_token": "eyJ..." }
```

**Response `200`:**
```json
{
  "access_token": "eyJ...",
  "expires_in": 900
}
```

---

### POST `/auth/logout`
Invalidates the refresh token. `401` if not authenticated.

**Response `204`** (no body)

---

## 2. User Endpoints

### GET `/users/me`
Get the authenticated user's profile.

**Response `200`:**
```json
{
  "id": "uuid-1234",
  "username": "john_doe",
  "email": "user@example.com",
  "plan": "premium",
  "avatar_url": "https://cdn.musicapp.com/avatars/uuid-1234.jpg",
  "country": "IN",
  "created_at": "2024-01-15T10:00:00Z"
}
```

---

### PATCH `/users/me`
Update profile fields.

**Request:**
```json
{
  "username": "john_updated",
  "avatar_url": "https://..."
}
```

**Response `200`:** Updated user object.

---

### POST `/users/me/follow/{userId}`
Follow another user.

**Response `204`** | `404` if user not found | `409` if already following.

---

### DELETE `/users/me/follow/{userId}`
Unfollow a user. **Response `204`**.

---

### POST `/users/me/follow/artists/{artistId}`
Follow an artist. **Response `204`**.

---

## 3. Songs & Catalog Endpoints

### GET `/songs/{songId}`
Get song metadata.

**Response `200`:**
```json
{
  "id": "uuid-song-1",
  "title": "Blinding Lights",
  "artist": {
    "id": "uuid-artist-1",
    "name": "The Weeknd"
  },
  "album": {
    "id": "uuid-album-1",
    "title": "After Hours",
    "cover_url": "https://cdn.musicapp.com/covers/uuid-album-1.jpg"
  },
  "duration_secs": 200,
  "genre": "Pop",
  "explicit": false
}
```

---

### GET `/songs/{songId}/stream`
Get a pre-signed CDN URL to stream the song. Bitrate is determined by the user's plan.

**Response `200`:**
```json
{
  "stream_url": "https://cdn.musicapp.com/audio/uuid-song-1_320kbps.m3u8?token=xxx&expires=1720000000",
  "bitrate": 320,
  "format": "hls",
  "expires_at": "2024-07-03T12:00:00Z"
}
```
`403` if the song is unavailable in the user's country.

---

### GET `/albums/{albumId}`
Get album with tracklist.

**Response `200`:**
```json
{
  "id": "uuid-album-1",
  "title": "After Hours",
  "artist": { "id": "uuid-artist-1", "name": "The Weeknd" },
  "cover_url": "https://...",
  "release_date": "2020-03-20",
  "album_type": "album",
  "songs": [
    {
      "id": "uuid-song-1",
      "title": "Blinding Lights",
      "track_number": 1,
      "duration_secs": 200
    }
  ]
}
```

---

### GET `/artists/{artistId}`
Get artist profile, top songs, and albums.

**Response `200`:**
```json
{
  "id": "uuid-artist-1",
  "name": "The Weeknd",
  "bio": "...",
  "avatar_url": "https://...",
  "follower_count": 45000000,
  "top_songs": [ /* array of song objects */ ],
  "albums": [ /* array of album objects */ ]
}
```

---

### POST `/songs` _(Artist only)_
Upload a new song. Uses `multipart/form-data`.

**Request fields:**
- `file` (audio file, required)
- `title` (string, required)
- `album_id` (UUID, optional)
- `genre` (string)
- `explicit` (boolean)
- `track_number` (int)

**Response `202`** (accepted for processing):
```json
{
  "song_id": "uuid-song-new",
  "status": "processing",
  "message": "Your song is being transcoded. It will be available within a few minutes."
}
```

---

## 4. Search Endpoints

### GET `/search?q={query}&type={types}&limit={n}&offset={n}`

**Query params:**
- `q` — search query (required)
- `type` — comma-separated: `songs,albums,artists,playlists` (default: all)
- `limit` — max results per type (default: 10, max: 50)
- `offset` — pagination offset

**Response `200`:**
```json
{
  "songs": {
    "total": 120,
    "items": [
      { "id": "uuid-song-1", "title": "Blinding Lights", "artist": "The Weeknd", "duration_secs": 200 }
    ]
  },
  "artists": {
    "total": 3,
    "items": [
      { "id": "uuid-artist-1", "name": "The Weeknd", "avatar_url": "..." }
    ]
  },
  "albums": { "total": 5, "items": [ /* ... */ ] },
  "playlists": { "total": 2, "items": [ /* ... */ ] }
}
```

---

## 5. Playlist Endpoints

### GET `/playlists/{playlistId}`
**Response `200`:**
```json
{
  "id": "uuid-pl-1",
  "title": "Chill Vibes",
  "owner": { "id": "uuid-user-1", "username": "john_doe" },
  "is_public": true,
  "is_collaborative": false,
  "song_count": 25,
  "duration_secs": 5400,
  "cover_url": "https://...",
  "songs": [
    {
      "position": 1,
      "song": { "id": "uuid-song-1", "title": "Blinding Lights", "artist": "The Weeknd" },
      "added_by": "john_doe",
      "added_at": "2024-06-01T10:00:00Z"
    }
  ]
}
```

---

### POST `/playlists`
Create a playlist.

**Request:**
```json
{
  "title": "Morning Run",
  "description": "High energy tracks",
  "is_public": true,
  "is_collaborative": false
}
```

**Response `201`:** Playlist object.

---

### PATCH `/playlists/{playlistId}`
Update playlist metadata. Only owner can do this. **Response `200`:** Updated playlist object.

---

### DELETE `/playlists/{playlistId}`
Delete playlist. Only owner. **Response `204`**.

---

### POST `/playlists/{playlistId}/songs`
Add a song.

**Request:**
```json
{ "song_id": "uuid-song-1" }
```

**Response `201`:**
```json
{ "position": 26, "added_at": "2024-07-01T12:00:00Z" }
```

---

### DELETE `/playlists/{playlistId}/songs/{songId}`
Remove a song. **Response `204`**.

---

### PUT `/playlists/{playlistId}/songs/reorder`
Reorder songs.

**Request:**
```json
{
  "song_id": "uuid-song-1",
  "new_position": 3
}
```

**Response `200`:** Updated playlist object.

---

## 6. User Library Endpoints

### GET `/me/liked-songs?limit={n}&offset={n}`
Get paginated liked songs. **Response `200`:** `{ "total": 342, "items": [ /* song objects */ ] }`

### POST `/me/liked-songs`
**Request:** `{ "song_id": "uuid-song-1" }` — **Response `201`**.

### DELETE `/me/liked-songs/{songId}`
**Response `204`**.

### GET `/me/playlists`
Get all of the authenticated user's playlists. **Response `200`:** `{ "items": [ /* playlist objects */ ] }`

---

## 7. Recommendation Endpoints

### GET `/recommendations/daily-mix`
Get the user's Daily Mix playlists (up to 6). **Response `200`:** `{ "mixes": [ { "title": "Daily Mix 1", "songs": [ /* ... */ ] } ] }`

### GET `/recommendations/discover-weekly`
**Response `200`:** `{ "songs": [ /* 30 song objects */ ] }`

### GET `/recommendations/radio/{songId}`
Get a radio station seeded from a song. **Response `200`:** `{ "songs": [ /* 50 song objects */ ] }`

### GET `/charts/trending?country={code}`
**Response `200`:** `{ "country": "IN", "period": "daily", "songs": [ { "rank": 1, "song": { /* ... */ }, "play_count": 2500000 } ] }`

---

## 8. Playback / Queue Endpoints

### POST `/me/player/queue`
Add a song to the play queue.

**Request:** `{ "song_id": "uuid-song-1", "position": "next" }` — `position`: `"next"` or `"last"`

**Response `204`**.

### GET `/me/player/queue`
**Response `200`:** `{ "current": { /* song */ }, "queue": [ /* song objects */ ] }`

### DELETE `/me/player/queue`
Clear queue. **Response `204`**.

---

## Common Error Response Format

All errors use this shape:

```json
{
  "error": {
    "code": "SONG_NOT_FOUND",
    "message": "The requested song does not exist or has been removed.",
    "status": 404
  }
}
```
