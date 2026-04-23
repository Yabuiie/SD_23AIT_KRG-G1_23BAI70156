# High-Level Design — Music Streaming Platform(SUNO)

---

## System Architecture Diagram

```
                        ┌─────────────────────────────────────┐
                        │           CLIENT LAYER               │
                        │  Web App │ iOS App │ Android App     │
                        └──────────────┬──────────────────────┘
                                       │ HTTPS
                        ┌──────────────▼──────────────────────┐
                        │           API GATEWAY                │
                        │  Rate Limiting │ Auth │ Routing      │
                        └──┬──────┬──────┬──────┬─────────────┘
                           │      │      │      │
          ┌────────────────┘      │      │      └─────────────────┐
          │                       │      │                         │
┌─────────▼──────┐   ┌────────────▼──┐  │  ┌─────────────────┐  │
│  User Service  │   │ Search Service│  │  │  Stream Service  │  │
│  (Auth/Profile)│   │ (Elasticsearch│  │  │  (Audio Delivery)│  │
└────────┬───────┘   └───────────────┘  │  └────────┬────────┘  │
         │                              │           │            │
┌────────▼───────┐   ┌─────────────────▼──┐  ┌─────▼─────────┐ │
│ Playlist Service│  │ Recommendation Svc  │  │  CDN Layer    │ │
│ (CRUD playlists)│  │ (ML-based Discovery)│  │  (Audio Files)│ │
└────────┬───────┘   └─────────────────────┘  └───────────────┘ │
         │                                                        │
┌────────▼────────────────────────────────────────────────────┐  │
│                    MESSAGE BROKER (Kafka)                    │  │
│   play-events │ upload-events │ notification-events         │  │
└──┬─────────────────┬──────────────────────┬─────────────────┘  │
   │                 │                      │                     │
┌──▼────────┐  ┌────▼────────┐  ┌──────────▼──────┐  ┌─────────▼──┐
│ Analytics │  │ Notification│  │  Upload/Ingest  │  │ Ad Service │
│  Service  │  │   Service   │  │    Service      │  │            │
└──┬────────┘  └─────────────┘  └─────────────────┘  └────────────┘
   │
┌──▼──────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                  │
│                                                                  │
│  PostgreSQL        Redis              Cassandra      S3 / GCS    │
│  (User, Albums,    (Sessions,         (Play counts,  (Audio,     │
│   Playlists)        Cache, Queues)     Trending)      Images)    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. API Gateway
- Single entry point for all client requests.
- Handles: JWT authentication, rate limiting (per user/IP), request routing to microservices, SSL termination.
- Technology: **Kong** or **AWS API Gateway**.

### 2. User Service
- Manages registration, login, profiles, subscriptions, and follow relationships.
- Issues and validates JWT access tokens (short-lived) + refresh tokens (long-lived).
- Database: **PostgreSQL** (relational — users, subscriptions, followers).

### 3. Music Catalog Service
- Manages songs, albums, artists, and genres.
- Stores metadata in **PostgreSQL**; audio files in **Object Storage (S3/GCS)**.
- On upload: triggers the Ingest Service to transcode audio to multiple bitrates.

### 4. Ingest / Upload Service
- Receives raw audio uploads from artists.
- Transcodes to multiple formats: 96kbps (mobile/free), 128kbps (standard), 320kbps (premium).
- Pushes transcoded files to S3 and triggers CDN invalidation.
- Technology: **FFmpeg** workers behind a job queue (Kafka or SQS).

### 5. Stream Service
- Generates pre-signed, time-limited CDN URLs for audio chunks.
- Enforces tier access: free users get 128kbps URLs, premium gets 320kbps.
- Implements **chunked streaming** (HLS/DASH) so playback starts in < 200ms.

### 6. Search Service
- Full-text search powered by **Elasticsearch**.
- Indexes: songs, albums, artists, playlists, podcasts.
- Updated in near real-time via Kafka events on catalog changes.

### 7. Playlist Service
- CRUD operations for playlists.
- Manages collaborative playlists, Liked Songs, and user library.
- Database: **PostgreSQL** with a join table for playlist–song relationships.

### 8. Recommendation Service
- Generates personalized recommendations (Daily Mix, Discover Weekly, Radio).
- Uses collaborative filtering + content-based features (genre, tempo, mood).
- Batch jobs run nightly using **Apache Spark**; results stored in Redis.

### 9. Analytics Service
- Consumes `play-events` from Kafka.
- Tracks: play counts, skip rates, listening duration, trending songs.
- Storage: **Apache Cassandra** (high write throughput, time-series data).

### 10. Notification Service
- Consumes events from Kafka (new release, weekly recap).
- Sends push notifications (FCM/APNs), emails (SendGrid), and in-app alerts.

### 11. Ad Service
- Selects and serves ads to free-tier users.
- Tracks ad impressions and completions.
- Integrates with third-party ad networks (e.g., Google Ad Manager).

### 12. CDN Layer
- Distributes audio chunks globally for low-latency delivery.
- Technology: **Cloudflare** or **AWS CloudFront**.
- Cache-Control headers set per audio file (long TTL — audio doesn't change).

---

## Key Technology Choices

| Component | Technology | Reason |
|---|---|---|
| API Gateway | Kong / AWS API GW | Auth, rate limit, routing |
| Primary DB | PostgreSQL | Relational, ACID for user/playlist data |
| Cache | Redis | Low-latency session, recommendation results |
| Search | Elasticsearch | Full-text, fuzzy search at scale |
| Message Broker | Apache Kafka | High-throughput event streaming |
| Time-series / Analytics | Apache Cassandra | High write throughput, horizontal scale |
| Audio Storage | AWS S3 / GCS | Durable, cheap, globally replicated object storage |
| CDN | Cloudflare / CloudFront | Low-latency audio delivery worldwide |
| Streaming Protocol | HLS / DASH | Adaptive bitrate, chunked delivery |
| Batch Processing | Apache Spark | ML feature computation for recommendations |
| Container Orchestration | Kubernetes (EKS/GKE) | Auto-scaling microservices |
