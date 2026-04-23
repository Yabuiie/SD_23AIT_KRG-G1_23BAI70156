# Requirements — Music Streaming Platform (SUNO)

---

## Functional Requirements

### User Management
- Users can register, log in, and manage their profile (name, email, password, profile picture).
- Supports two account tiers: **Free** (ad-supported) and **Premium** (unlimited, offline).
- Users can follow/unfollow artists and other users.

### Music Catalog
- Artists can upload songs with metadata: title, genre, duration, release date, lyrics.
- Songs are organized into **albums** and **playlists**.
- Platform supports a global music catalog browsable by genre, artist, album, or mood.

### Search & Discovery
- Full-text search across songs, albums, artists, and playlists.
- Personalized recommendations (Daily Mix, Discover Weekly) based on listening history.
- Browse curated playlists by mood, genre, or trending charts.

### Playback
- Stream songs in real-time with adaptive bitrate (128kbps free / 320kbps premium).
- Support **offline playback** (Premium only) — download tracks for local use.
- Queue management: next song, shuffle, repeat (one / all).
- Gapless playback and crossfade between tracks.

### Playlists
- Users can create, edit, delete, and share playlists.
- Collaborative playlists — multiple users can add/remove songs.
- Users can like/save songs to their **Liked Songs** library.

### Social Features
- Activity feed showing what friends are listening to.
- Share songs/playlists via link or to social media.

### Podcast Support
- Browse and subscribe to podcasts.
- Track listen progress per episode.

### Notifications
- Notify users when a followed artist releases new music.
- Weekly listening stats (Spotify Wrapped-like summary).

### Ads (Free Tier)
- Play audio/banner ads after every ~2–3 songs.
- Skip ads option available to Premium users only.

---

## Non-Functional Requirements

| Property | Target |
|---|---|
| **Availability** | 99.99% uptime (< 52 min downtime/year) |
| **Scalability** | Support 50M+ daily active users, 100M+ songs |
| **Latency** | Song playback starts within **200ms** |
| **Throughput** | Handle 5M+ concurrent streams |
| **Durability** | Zero data loss for user data and uploaded audio |
| **Consistency** | Eventual consistency acceptable for recommendations and play counts; strong consistency for user accounts and subscriptions |
| **Security** | OAuth 2.0, encrypted audio delivery (HTTPS/DRM), GDPR compliance |
| **Scalable Storage** | ~100 TB of audio storage growing at ~10 TB/month |
| **CDN** | Audio delivered via CDN nodes close to user location |
| **Observability** | Full logging, metrics, and distributed tracing |
