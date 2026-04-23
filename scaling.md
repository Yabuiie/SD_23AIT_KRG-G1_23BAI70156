# Scaling Strategy — Music Streaming Platform

---

## 1. Load Balancing

### API Gateway Layer
- **Global DNS Load Balancing** (AWS Route 53 / Cloudflare) routes users to the nearest regional cluster using latency-based or geolocation routing.
- Each region runs an **Application Load Balancer (ALB)** that distributes traffic across service pods.
- Health checks every 10 seconds — unhealthy pods are removed within 30 seconds.

### Microservice Layer
- Each service runs in **Kubernetes** with a Horizontal Pod Autoscaler (HPA) that scales pods based on CPU (target: 70%) and custom metrics (e.g., request queue depth).
- Services communicate internally via **gRPC** (low latency) or Kafka events (async).
- **Consistent hashing** is used in the Stream Service so that a given user's requests consistently hit the same pod — this improves in-memory cache utilization.

### Audio / CDN Layer
- Audio delivery is entirely offloaded to **Cloudflare or AWS CloudFront**.
- CDN has **300+ PoPs (Points of Presence)** worldwide. Audio chunks are served from the node closest to the listener, minimizing latency.
- CDN origin is S3. Cache miss triggers S3 fetch and caches the file at the edge.

---

## 2. Caching Strategy

### What We Cache

| Data | Store | TTL | Strategy |
|---|---|---|---|
| Song / Album / Artist metadata | Redis | 1 hour | Cache-aside |
| User session / plan info | Redis | 15 minutes | Write-through |
| Search autocomplete suggestions | Redis Sorted Sets | 10 minutes | Read-through |
| Pre-computed recommendations | Redis | 24 hours | Cache-aside (batch populated) |
| Trending charts | Redis | 6 hours | Scheduled refresh |
| Playlist metadata | Redis | 30 minutes | Cache-aside with invalidation on write |
| Audio files (chunks) | CDN Edge Caches | Immutable (∞) | Cache-forever (audio files never change) |

### Cache-Aside Pattern (Lazy Loading)
Used for song metadata, playlists, etc.
1. App checks Redis.
2. On cache miss → fetch from PostgreSQL, write to Redis with TTL, return result.
3. On write (update/delete) → invalidate the Redis key.

### Write-Through (for sessions)
On login, session data is written to both PostgreSQL (refresh token) and Redis simultaneously. Reads always go to Redis first for speed.

### CDN Caching for Audio
Audio chunk objects stored in S3 are immutable (a transcoded file never changes). CDN Cache-Control is set to `max-age=31536000, immutable`. This means a cached audio chunk is never re-fetched from origin — reducing S3 costs and latency to near zero for popular songs.

### Cache Invalidation
- On playlist update: delete `playlist:{id}` from Redis. Next read repopulates.
- On song metadata update: delete `song:{id}`. 
- For recommendations: nightly Spark jobs overwrite the key entirely.

---

## 3. Database Scaling

### PostgreSQL (User, Catalog, Playlist data)

**Vertical scaling** handles most traffic. Beyond that:

**Read Replicas:** PostgreSQL primary handles all writes. 3–5 read replicas serve read traffic (catalog browsing, playlist fetching). Application uses the primary for writes and round-robins reads across replicas.

**Connection Pooling:** Each service uses **PgBouncer** to pool connections, keeping actual PostgreSQL connections well below the limit even at high concurrency (thousands of simultaneous requests).

**Sharding (when needed):** For very large tables (e.g., `play_events` if stored in Postgres, or `liked_songs`), horizontal sharding by `user_id` using **Citus** extension. Each shard owns a range of user IDs.

**Partitioning:** The `play_events` table (if in Postgres) is range-partitioned by month. Old partitions are archived to cold storage after 6 months.

### Cassandra (Analytics — Play Events, Trending)

Cassandra is designed for horizontal scale from the start. Scaling strategy:

- **Add more nodes** — Cassandra auto-rebalances using consistent hashing (virtual nodes). No downtime during scale-out.
- **Replication Factor = 3** across 3 availability zones. Reads/writes use `LOCAL_QUORUM` consistency (2/3 nodes must respond) for a balance of speed and durability.
- **Compaction Strategy:** `TimeWindowCompactionStrategy` for `play_events` — since data is time-ordered, compaction is efficient and old SSTables are quickly expired.

### Redis (Cache & Queues)

- **Redis Cluster** with 6 nodes (3 primary + 3 replicas). Data is sharded across primaries by key hash slot.
- Eviction policy: `allkeys-lru` — when memory is full, least recently used keys are evicted.
- Persistence: AOF (Append-Only File) enabled for session data only. Cache data is marked volatile and not persisted (it can be repopulated from the source of truth).

### Elasticsearch (Search Index)

- **Cluster with 5 data nodes** + 3 master-eligible nodes.
- Index sharded by content type (songs, artists, albums, playlists) — each gets its own index.
- Replicas: 1 replica per shard (high availability + read throughput).
- Index updates are fed by **Kafka consumers** — every song upload/edit triggers a Kafka event, which an indexer consumes and upserts into Elasticsearch. Eventual consistency is acceptable for search.

---

## 4. Audio Upload & Ingest Scaling

- Uploads go directly to S3 via **pre-signed upload URLs** (no file upload traffic hits the application servers).
- On S3 `PutObject` completion, S3 triggers an **SQS/Kafka event** picked up by Ingest workers.
- Ingest workers (FFmpeg containers) run in **Kubernetes Jobs** — auto-scaled based on queue depth. Peak upload periods spin up dozens of parallel workers.

---

## 5. Event Streaming (Kafka)

Kafka handles all async communication between services.

**Topics and throughput:**

| Topic | Producers | Consumers | Est. Volume |
|---|---|---|---|
| `play-events` | Stream Service | Analytics, Recommendation | ~5M events/min at peak |
| `upload-events` | Upload Service | Ingest, Search Indexer | Low (thousands/day) |
| `notification-events` | Catalog, Analytics | Notification Service | Medium |

**Scaling Kafka:**
- `play-events` topic has **60 partitions** — supports 60 parallel consumer threads.
- Kafka cluster: 6 brokers, replication factor 3.
- Consumer groups allow independent scaling of Analytics consumers vs. Recommendation consumers without coordination.

---

## 6. Key Bottleneck Mitigations

| Bottleneck | Mitigation |
|---|---|
| Song stream start latency | CDN edge caching, HLS chunking (play starts after first chunk loads) |
| Play event write storms (peak hours) | Kafka buffers writes; Cassandra absorbs high ingest |
| Search latency | Elasticsearch in-memory indices; Redis autocomplete cache |
| Database read hotspots (viral songs) | Redis caches song metadata; CDN caches audio |
| New artist release spike | Pre-warm CDN on scheduled releases; Kafka fans out notifications async |
| Recommendation computation | Batch (Spark nightly); results stored in Redis — zero compute at request time |

---

## 7. Capacity Estimates

| Metric | Estimate |
|---|---|
| Daily Active Users | 50M |
| Concurrent Streams (peak) | 5M |
| Avg stream bandwidth (mixed plans) | 200kbps |
| Total streaming bandwidth | ~1 Tbps |
| Play events/day | ~2 billion |
| Audio storage (100M songs × 3 bitrates × ~10MB avg) | ~3 PB |
| New storage per month | ~10–20 TB |
| PostgreSQL rows (songs + playlists + users) | ~500M rows |
| Redis memory required | ~200 GB |
| Cassandra cluster size | ~20 nodes × 8TB SSD = 160TB |
