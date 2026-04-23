# Low-Level Design — Music Streaming Platform(SUNO)

---

## Database Schema

### PostgreSQL (Relational — Users, Catalog, Playlists)

```sql
-- ─────────────────────────────
-- USERS & AUTH
-- ─────────────────────────────

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    username        VARCHAR(100) UNIQUE NOT NULL,
    password_hash   TEXT NOT NULL,
    plan            VARCHAR(20) DEFAULT 'free',  -- 'free' | 'premium'
    avatar_url      TEXT,
    country         CHAR(2),
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE refresh_tokens (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    token_hash  TEXT NOT NULL,
    expires_at  TIMESTAMP NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE user_follows (
    follower_id UUID REFERENCES users(id) ON DELETE CASCADE,
    followee_id UUID REFERENCES users(id) ON DELETE CASCADE,
    followed_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);

-- ─────────────────────────────
-- ARTISTS
-- ─────────────────────────────

CREATE TABLE artists (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id),  -- artist's user account
    name        VARCHAR(255) NOT NULL,
    bio         TEXT,
    avatar_url  TEXT,
    verified    BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE user_follows_artist (
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    artist_id   UUID REFERENCES artists(id) ON DELETE CASCADE,
    followed_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, artist_id)
);

-- ─────────────────────────────
-- MUSIC CATALOG
-- ─────────────────────────────

CREATE TABLE genres (
    id      SERIAL PRIMARY KEY,
    name    VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE albums (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    artist_id       UUID REFERENCES artists(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    cover_url       TEXT,
    release_date    DATE,
    album_type      VARCHAR(20) DEFAULT 'album',  -- 'album' | 'single' | 'ep'
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE songs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    album_id        UUID REFERENCES albums(id) ON DELETE SET NULL,
    artist_id       UUID REFERENCES artists(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    duration_secs   INT NOT NULL,
    track_number    INT,
    genre_id        INT REFERENCES genres(id),
    lyrics          TEXT,
    explicit        BOOLEAN DEFAULT FALSE,
    status          VARCHAR(20) DEFAULT 'processing', -- 'processing' | 'active' | 'removed'
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Audio file variants (different bitrates)
CREATE TABLE audio_files (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    song_id     UUID REFERENCES songs(id) ON DELETE CASCADE,
    bitrate     INT NOT NULL,           -- 96, 128, 320 (kbps)
    format      VARCHAR(10) NOT NULL,   -- 'mp3', 'ogg', 'aac'
    storage_key TEXT NOT NULL,          -- S3 object key
    file_size   BIGINT,                 -- bytes
    created_at  TIMESTAMP DEFAULT NOW()
);

-- ─────────────────────────────
-- PLAYLISTS
-- ─────────────────────────────

CREATE TABLE playlists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id        UUID REFERENCES users(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    cover_url       TEXT,
    is_public       BOOLEAN DEFAULT TRUE,
    is_collaborative BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE playlist_songs (
    playlist_id UUID REFERENCES playlists(id) ON DELETE CASCADE,
    song_id     UUID REFERENCES songs(id) ON DELETE CASCADE,
    added_by    UUID REFERENCES users(id),
    position    INT NOT NULL,
    added_at    TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (playlist_id, song_id)
);

CREATE TABLE playlist_collaborators (
    playlist_id UUID REFERENCES playlists(id) ON DELETE CASCADE,
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    added_at    TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (playlist_id, user_id)
);

-- ─────────────────────────────
-- USER LIBRARY
-- ─────────────────────────────

CREATE TABLE liked_songs (
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    song_id     UUID REFERENCES songs(id) ON DELETE CASCADE,
    liked_at    TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, song_id)
);

CREATE TABLE liked_albums (
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    album_id    UUID REFERENCES albums(id) ON DELETE CASCADE,
    saved_at    TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, album_id)
);

-- ─────────────────────────────
-- OFFLINE DOWNLOADS (Premium)
-- ─────────────────────────────

CREATE TABLE offline_downloads (
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    song_id     UUID REFERENCES songs(id) ON DELETE CASCADE,
    device_id   TEXT NOT NULL,
    downloaded_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, song_id, device_id)
);

-- ─────────────────────────────
-- PODCASTS
-- ─────────────────────────────

CREATE TABLE podcasts (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title       VARCHAR(255) NOT NULL,
    author      VARCHAR(255),
    description TEXT,
    cover_url   TEXT,
    rss_url     TEXT,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE podcast_episodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    podcast_id      UUID REFERENCES podcasts(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    duration_secs   INT,
    audio_url       TEXT,
    published_at    TIMESTAMP
);

CREATE TABLE podcast_progress (
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    episode_id  UUID REFERENCES podcast_episodes(id) ON DELETE CASCADE,
    position_secs INT DEFAULT 0,
    completed   BOOLEAN DEFAULT FALSE,
    updated_at  TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, episode_id)
);
```

---

### Cassandra (Analytics — High-write, Time-series)

```sql
-- Play events: one row per play action
CREATE TABLE play_events (
    song_id     UUID,
    played_at   TIMESTAMP,
    user_id     UUID,
    duration_played_secs INT,
    skipped     BOOLEAN,
    source      TEXT,    -- 'search' | 'playlist' | 'radio' | 'album'
    PRIMARY KEY (song_id, played_at)
) WITH CLUSTERING ORDER BY (played_at DESC);

-- Trending songs per country (updated by batch job)
CREATE TABLE trending_songs (
    country     TEXT,
    period      TEXT,     -- 'daily' | 'weekly'
    rank        INT,
    song_id     UUID,
    play_count  BIGINT,
    PRIMARY KEY ((country, period), rank)
);

-- Per-user listening history (for recommendations)
CREATE TABLE user_listening_history (
    user_id     UUID,
    played_at   TIMESTAMP,
    song_id     UUID,
    PRIMARY KEY (user_id, played_at)
) WITH CLUSTERING ORDER BY (played_at DESC);
```

---

### Redis (Cache & Ephemeral State)

```
# Session / Auth
session:{user_id}           → { access_token, plan, expires_at }    TTL: 15m
refresh_token:{token_hash}  → { user_id }                           TTL: 30d

# Catalog cache
song:{song_id}              → JSON song metadata                     TTL: 1h
album:{album_id}            → JSON album metadata                    TTL: 1h
artist:{artist_id}          → JSON artist metadata                   TTL: 1h

# Search autocomplete
autocomplete:{prefix}       → Sorted Set of suggestions             TTL: 10m

# Recommendations (pre-computed batch)
recommendations:{user_id}   → JSON list of song IDs                 TTL: 24h
trending:{country}:daily    → JSON list of song IDs                 TTL: 6h

# Playback queue
queue:{user_id}:{session}   → List of song IDs                      TTL: 3h

# Rate limiting
ratelimit:{user_id}:{route} → counter                               TTL: 1m
```

---

## Class Design / Core Entities

```
┌──────────────────────────────────────────────────────────────────┐
│                           User                                   │
├──────────────────────────────────────────────────────────────────┤
│ - id: UUID                                                       │
│ - email: String                                                  │
│ - username: String                                               │
│ - passwordHash: String                                           │
│ - plan: Enum {FREE, PREMIUM}                                     │
│ - country: String                                                │
│ - avatarUrl: String                                              │
├──────────────────────────────────────────────────────────────────┤
│ + register(email, password): User                                │
│ + login(email, password): TokenPair                              │
│ + upgradeToP Premium(): void                                     │
│ + follow(userId): void                                           │
│ + getLibrary(): Library                                          │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                           Song                                   │
├──────────────────────────────────────────────────────────────────┤
│ - id: UUID                                                       │
│ - title: String                                                  │
│ - artistId: UUID                                                 │
│ - albumId: UUID                                                  │
│ - durationSecs: Int                                              │
│ - genre: String                                                  │
│ - explicit: Boolean                                              │
│ - status: Enum {PROCESSING, ACTIVE, REMOVED}                     │
│ - audioFiles: List<AudioFile>                                    │
├──────────────────────────────────────────────────────────────────┤
│ + getStreamUrl(userPlan: Plan): String                           │
│ + getMetadata(): SongMetadata                                    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                        AudioFile                                 │
├──────────────────────────────────────────────────────────────────┤
│ - id: UUID                                                       │
│ - songId: UUID                                                   │
│ - bitrate: Int      (96 / 128 / 320)                            │
│ - format: String    (mp3 / ogg / aac)                           │
│ - storageKey: String                                             │
│ - fileSize: Long                                                 │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                        Playlist                                  │
├──────────────────────────────────────────────────────────────────┤
│ - id: UUID                                                       │
│ - ownerId: UUID                                                  │
│ - title: String                                                  │
│ - isPublic: Boolean                                              │
│ - isCollaborative: Boolean                                       │
│ - songs: List<PlaylistSong>                                      │
│ - collaborators: List<UUID>                                      │
├──────────────────────────────────────────────────────────────────┤
│ + addSong(songId, userId): void                                  │
│ + removeSong(songId, userId): void                               │
│ + reorder(songId, newPosition): void                             │
│ + shareLink(): String                                            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                      StreamSession                               │
├──────────────────────────────────────────────────────────────────┤
│ - sessionId: UUID                                                │
│ - userId: UUID                                                   │
│ - currentSong: Song                                              │
│ - queue: List<Song>                                              │
│ - position: Int   (seconds elapsed)                              │
│ - shuffle: Boolean                                               │
│ - repeatMode: Enum {OFF, ONE, ALL}                               │
├──────────────────────────────────────────────────────────────────┤
│ + play(songId): StreamUrl                                        │
│ + pause(): void                                                  │
│ + next(): Song                                                   │
│ + previous(): Song                                               │
│ + seek(positionSecs): void                                       │
│ + toggleShuffle(): void                                          │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                   RecommendationEngine                           │
├──────────────────────────────────────────────────────────────────┤
│ - userId: UUID                                                   │
├──────────────────────────────────────────────────────────────────┤
│ + getDailyMix(): List<Song>                                      │
│ + getDiscoverWeekly(): List<Song>                                │
│ + getRadio(seedSongId): List<Song>                               │
│ + getTrending(country): List<Song>                               │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                      IngestPipeline                              │
├──────────────────────────────────────────────────────────────────┤
│ - jobId: UUID                                                    │
│ - songId: UUID                                                   │
│ - status: Enum {PENDING, TRANSCODING, DONE, FAILED}              │
├──────────────────────────────────────────────────────────────────┤
│ + ingest(rawFilePath): JobId                                     │
│ + transcode(bitrates: List<Int>): List<AudioFile>                │
│ + uploadToCDN(audioFile): String                                 │
│ + updateSearchIndex(song): void                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Design Decisions

**Why Cassandra for analytics?** Play events are write-heavy (millions/min), append-only, and queried by song or user with time ranges — a perfect fit for Cassandra's partition model.

**Why PostgreSQL for catalog?** Song/album/artist/playlist data is relational, requires ACID (e.g., consistent playlist ordering), and has moderate write volume.

**Why pre-signed URLs for streaming?** Audio files never pass through application servers. The Stream Service issues a time-limited signed URL directly to the client, which fetches audio chunks straight from the CDN. This eliminates a massive bottleneck.

**Why chunked HLS/DASH instead of full-file download?** Allows playback to start in under 200ms, supports adaptive bitrate switching on poor connections, and reduces wasted bandwidth on skipped songs.
