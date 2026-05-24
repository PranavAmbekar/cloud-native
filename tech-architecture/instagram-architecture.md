# Instagram Technical Architecture

> How Instagram handles 2+ billion users, 100+ million photos/videos daily, and serves content globally with sub-second latency.

## Overview

Instagram is one of the world's largest photo and video sharing platforms. Understanding its architecture provides insights into building highly scalable, globally distributed systems.

### Scale (2024-2026 estimates)

| Metric | Scale |
|--------|-------|
| Monthly Active Users | 2+ billion |
| Daily Active Users | 500+ million |
| Photos uploaded daily | 100+ million |
| Videos uploaded daily | 50+ million |
| Stories posted daily | 500+ million |
| Likes per day | 4+ billion |
| Comments per day | 500+ million |
| Peak requests/second | 1+ million |
| Total photos stored | 100+ billion |
| Storage size | Exabytes |

## High-Level Architecture

```
                            +-------------+
                            |    User     |
                            |    Phone    |
                            +------+------+
                                   |
                                   v
+--------------------------------------------------------------------------+
|                              CDN LAYER                                    |
|                   (Facebook/Meta CDN - Global Edge PoPs)                  |
|   +--------+   +--------+   +--------+   +--------+   +--------+         |
|   |   US   |   | Europe |   |  Asia  |   | LATAM  |   | Africa |         |
|   +--------+   +--------+   +--------+   +--------+   +--------+         |
+--------------------------------------------------------------------------+
                                   |
                                   v
+--------------------------------------------------------------------------+
|                           LOAD BALANCERS                                  |
|                  (L4/L7 Load Balancing - Katran/Proxygen)                 |
+--------------------------------------------------------------------------+
                                   |
                +------------------+------------------+
                |                  |                  |
                v                  v                  v
         +------------+     +------------+     +------------+
         |  API Tier  |     |  Web Tier  |     | Media Tier |
         |  (Django)  |     |  (React)   |     |  (Upload)  |
         +------------+     +------------+     +------------+
                |                  |                  |
                +------------------+------------------+
                                   |
                                   v
+--------------------------------------------------------------------------+
|                            SERVICE MESH                                   |
|                (Thrift RPC - Internal Service Communication)              |
+--------------------------------------------------------------------------+
                |                  |                  |
                v                  v                  v
         +------------+     +------------+     +------------+
         |    Feed    |     |  Stories   |     |   Search   |
         |  Service   |     |  Service   |     |  Service   |
         +------------+     +------------+     +------------+
         +------------+     +------------+     +------------+
         | Messaging  |     |   Notif    |     |    Ads     |
         |  Service   |     |  Service   |     |  Service   |
         +------------+     +------------+     +------------+
                                   |
                                   v
+--------------------------------------------------------------------------+
|                             DATA LAYER                                    |
|                                                                           |
|    +------------+       +------------+       +------------+               |
|    | PostgreSQL |       | Cassandra  |       |    TAO     |               |
|    | (Users,    |       | (Feed,     |       | (Social    |               |
|    |  Metadata) |       |  Inbox)    |       |  Graph)    |               |
|    +------------+       +------------+       +------------+               |
|                                                                           |
|    +------------+       +------------+       +------------+               |
|    | Memcached  |       |   Redis    |       |  RocksDB   |               |
|    | (Cache)    |       | (Sessions) |       | (Local)    |               |
|    +------------+       +------------+       +------------+               |
+--------------------------------------------------------------------------+
                                   |
                                   v
+--------------------------------------------------------------------------+
|                           STORAGE LAYER                                   |
|                                                                           |
|         +--------------------+       +--------------------+               |
|         |     Haystack       |       |        F4          |               |
|         |  (Photo Storage)   |       | (Warm/Cold Storage)|               |
|         |  - Hot photos      |       | - Older photos     |               |
|         |  - Fast retrieval  |       | - Compressed       |               |
|         +--------------------+       +--------------------+               |
+--------------------------------------------------------------------------+
```

## The Photo Upload Journey

### Complete Flow: Upload to Post

```
STEP 1: User Takes/Selects Photo
+----------------------------------+
|  Instagram App (Mobile)          |
|                                  |
|  - Camera capture OR             |
|  - Gallery selection             |
|                                  |
|  Client-side processing:         |
|  - Resize to multiple sizes      |
|  - Apply filters (local)         |
|  - Generate thumbnail            |
|  - Compress (JPEG/HEIC)          |
+----------------------------------+
                |
                v
STEP 2: Upload Request
+----------------------------------+
|  HTTPS POST to Upload Server     |
|                                  |
|  Headers:                        |
|  - Auth token (JWT)              |
|  - Content-Type: multipart       |
|  - X-IG-Upload-ID                |
|                                  |
|  Body:                           |
|  - Photo binary (chunked)        |
|  - Caption text                  |
|  - Location data                 |
|  - Tagged users                  |
|  - Hashtags                      |
+----------------------------------+
                |
                v
STEP 3: CDN Edge Reception
+----------------------------------+
|  Edge PoP (Nearest to User)      |
|                                  |
|  - SSL termination               |
|  - Request validation            |
|  - Rate limiting check           |
|  - Route to origin               |
+----------------------------------+
                |
                v
STEP 4: Load Balancer
+----------------------------------+
|  L7 Load Balancer (Proxygen)     |
|                                  |
|  - Health check backends         |
|  - Sticky sessions               |
|  - Route to upload cluster       |
+----------------------------------+
                |
                v
STEP 5: Upload Server Processing
+----------------------------------+
|  Upload Service (Python/Django)  |
|                                  |
|  1. Auth verification            |
|  2. Spam/abuse detection         |
|  3. Content policy check         |
|  4. Generate media_id            |
|  5. Extract EXIF metadata        |
|  6. Queue for processing         |
+----------------------------------+
                |
       +--------+--------+
       |                 |
       v                 v
+----------------+  +----------------+
| STEP 6A:       |  | STEP 6B:       |
| Media Pipeline |  | Write Metadata |
|                |  |                |
| - Resize to    |  | PostgreSQL:    |
|   multiple     |  | - post_id      |
|   resolutions  |  | - user_id      |
| - Generate     |  | - caption      |
|   thumbnails   |  | - location_id  |
| - Face detect  |  | - created_at   |
| - Object detect|  | - media_ids[]  |
| - NSFW scan    |  |                |
| - Quality score|  |                |
+----------------+  +----------------+
       |
       v
STEP 7: Store to Haystack
+----------------------------------+
|  Haystack Photo Storage          |
|                                  |
|  Write to 3 replicas:            |
|  - Primary datacenter            |
|  - Secondary datacenter          |
|  - Tertiary datacenter           |
|                                  |
|  Returns:                        |
|  - haystack_handle               |
|  - storage_url                   |
+----------------------------------+
       |
       v
STEP 8: Update Social Graph (TAO)
+----------------------------------+
|  TAO (Social Graph Database)     |
|                                  |
|  Create edges:                   |
|  - user -> posted -> photo       |
|  - photo -> tagged -> users      |
|  - photo -> location -> loc      |
|  - photo -> hashtag -> tags      |
+----------------------------------+
       |
       v
STEP 9: Feed Fanout
+----------------------------------+
|  Feed Service (Async)            |
|                                  |
|  Push to followers feeds:        |
|  - Get follower list             |
|  - For each follower:            |
|    - Add to feed cache           |
|    - (Cassandra write)           |
|                                  |
|  Fanout strategies:              |
|  - < 10K followers: Push         |
|  - > 10K followers: Pull         |
+----------------------------------+
       |
       v
STEP 10: Notifications
+----------------------------------+
|  Notification Service            |
|                                  |
|  Trigger notifications:          |
|  - Push to tagged users          |
|  - Push to mentioned users       |
|  - Update activity feed          |
|                                  |
|  Channels:                       |
|  - APNs (iOS)                    |
|  - FCM (Android)                 |
|  - In-app notifications          |
+----------------------------------+
       |
       v
STEP 11: Response to Client
+----------------------------------+
|  API Response                    |
|                                  |
|  HTTP 200 OK                     |
|  {                               |
|    "media_id": "xxx",            |
|    "status": "published",        |
|    "url": "cdn.ig.com/...",      |
|    "created_at": "..."           |
|  }                               |
+----------------------------------+
```

## Storage Architecture

### Haystack: Photo Storage System

```
PROBLEM: Serving billions of photos with minimal disk I/O

Traditional Storage:           Haystack Storage:
+------------------+           +---------------------------+
| photo1.jpg       |           | Haystack Volume (100GB)   |
| photo2.jpg       |           |                           |
| photo3.jpg       |  ------>  | [photo1][photo2][photo3]  |
| ...              |           | [offset0][off1 ][off2  ]  |
| photo1000000.jpg |           |                           |
+------------------+           +---------------------------+
1M files = 1M inodes           1 file = 1 inode + index
= 1M disk seeks                = 1 disk seek + offset read


HAYSTACK COMPONENTS:

+------------------+     +------------------+     +------------------+
|    Directory     |     |      Cache       |     |      Store       |
|    (Routing)     |     |    (Hot Data)    |     |   (Persistence)  |
+------------------+     +------------------+     +------------------+
         |                        |                        |
         v                        v                        v
+------------------+     +------------------+     +------------------+
| - Maps photo_id  |     | - Memcached      |     | - Append-only    |
|   to volume      |     | - Recently       |     |   log files      |
| - Volume to      |     |   accessed       |     | - 100GB volumes  |
|   physical host  |     |   photos         |     | - In-memory      |
| - Replication    |     | - CDN miss       |     |   index          |
|   tracking       |     |   handler        |     | - 3x replication |
+------------------+     +------------------+     +------------------+


PHOTO URL STRUCTURE:

  https://cdn.instagram.com/<volume_id>/<photo_id>_<size>.jpg

  Example: https://cdn.instagram.com/v234/1847294817_1080x1080.jpg

  volume_id: Which haystack volume
  photo_id:  Unique photo identifier
  size:      Resolution (150x150, 320x320, 640x640, 1080x1080)


HAYSTACK INDEX (In-Memory):

+------------------+------------------+------------------+------------------+
| photo_id (64bit) | offset (32bit)   | size (32bit)     | flags (16bit)    |
+------------------+------------------+------------------+------------------+
| 1847294817       | 0                | 245760           | 0x0001           |
| 1847294818       | 245760           | 312400           | 0x0001           |
| 1847294819       | 558160           | 198000           | 0x0001           |
+------------------+------------------+------------------+------------------+

Index size: ~40 bytes per photo
1 billion photos = 40GB RAM (fits in memory)
```

### Replication Strategy

```
WRITE PATH (Synchronous 3-way replication):

                    Upload Server
                          |
                          v
                   +------------+
                   |  Primary   |
                   |  Haystack  |
                   | (Virginia) |
                   +------------+
                          |
        +-----------------+-----------------+
        |                 |                 |
        v                 v                 v
  +----------+      +----------+      +----------+
  | Replica  |      | Replica  |      | Replica  |
  |    1     |      |    2     |      |    3     |
  | (DC 1)   |      | (DC 2)   |      | (DC 3)   |
  | Virginia |      | Oregon   |      | Europe   |
  +----------+      +----------+      +----------+

  Write succeeds when: 2 of 3 replicas acknowledge
  Read serves from: Nearest replica with valid data


GEOGRAPHIC DISTRIBUTION:

                      +----------+
                      |  Europe  |
                      |    DC    |
                      +----+-----+
                           |
          +----------------+----------------+
          |                                 |
     +----+-----+                     +-----+----+
     |  US East |                     |  US West |
     | (Primary)|                     |          |
     +----+-----+                     +-----+----+
          |                                 |
          +----------------+----------------+
                           |
                     +-----+-----+
                     | Singapore |
                     |    DC     |
                     +-----------+

  Cross-DC replication: Asynchronous (eventual consistency)
  Typical lag: 100-500ms
  Conflict resolution: Last-write-wins with vector clocks
```

### F4: Warm/Cold Storage

```
PHOTOS LIFECYCLE:

+----------+   30 days   +----------+   1 year   +----------+
|   Hot    | ----------> |   Warm   | ---------> |   Cold   |
| Haystack |             |    F4    |            |    F4    |
+----------+             +----------+            +----------+


STORAGE CHARACTERISTICS:

+-----------------+-----------------+-----------------+
|      Hot        |      Warm       |      Cold       |
+-----------------+-----------------+-----------------+
| 3x replication  | 1.4x replication| 1.4x replication|
| SSD storage     | HDD storage     | HDD storage     |
| < 10ms latency  | < 50ms latency  | < 200ms latency |
| ~10% of data    | ~30% of data    | ~60% of data    |
+-----------------+-----------------+-----------------+

F4 uses Reed-Solomon erasure coding:
- 10 data blocks + 4 parity blocks
- Can lose any 4 blocks and recover
- 1.4x storage overhead vs 3x replication
- Saves 50%+ storage cost for older photos
```

## Database Architecture

### PostgreSQL (Relational Data)

```
Used for: Users, Posts metadata, Comments, Relationships

SHARDING STRATEGY:

                         Shard Router
                   user_id % 4096 = shard_number
                              |
    +------+------+------+------+------+------+------+
    |      |      |      |      |      |      |      |
    v      v      v      v      v      v      v      v
+------+ +------+ +------+ +------+ +------+ +------+ +------+
|Shard | |Shard | |Shard | |Shard | | .... | |Shard | |Shard |
|  0   | |  1   | |  2   | |  3   | |      | | 4094 | | 4095 |
+------+ +------+ +------+ +------+ +------+ +------+ +------+

Each shard:
- Primary + 2 read replicas
- Cross-datacenter async replication
- ~500GB per shard


SCHEMA EXAMPLES:

users table (sharded by user_id):
+---------+----------+-------+----------------+------+----------------+
| user_id | username | email | profile_pic_id | bio  | follower_count |
+---------+----------+-------+----------------+------+----------------+
| 12345   | john_doe | j@... | hay://v1/123   | ...  | 1523           |
+---------+----------+-------+----------------+------+----------------+

posts table (sharded by user_id, not post_id):
+---------+---------+-------------+---------+-------------+------------+
| post_id | user_id | media_ids[] | caption | location_id | created_at |
+---------+---------+-------------+---------+-------------+------------+
| 9876543 | 12345   | [m1,m2,m3]  | "Hello" | 555         | 2026-05-24 |
+---------+---------+-------------+---------+-------------+------------+

WHY SHARD BY USER_ID (not post_id)?
- User posts are co-located (single shard query)
- User profile + posts = single shard
- Avoid cross-shard joins
- Trade-off: Hot users create hot shards
```

### Cassandra (Feed & Inbox)

```
Used for: User feeds, Direct message inbox, Activity notifications

WHY CASSANDRA FOR FEEDS?
- Write-heavy workload (fanout)
- Time-series data (feeds ordered by time)
- No joins needed
- Eventual consistency acceptable
- Linear scalability


FEED TABLE SCHEMA:

+------------------------+------------------------+-----------+
| user_id (partition key)| post_time (clustering) | post_id   |
+------------------------+------------------------+-----------+
| 12345                  | 2026-05-24 10:30:00    | 9876543   |
| 12345                  | 2026-05-24 09:15:00    | 9876542   |
| 12345                  | 2026-05-24 08:00:00    | 9876541   |
+------------------------+------------------------+-----------+

Partition: All feed items for a user on same node
Clustering: Sorted by time (newest first)
TTL: 30 days (old feed items auto-expire)


CLUSTER TOPOLOGY:

       +--------+     +--------+     +--------+
       | Node 1 |-----| Node 2 |-----| Node 3 |
       |  DC1   |     |  DC1   |     |  DC1   |
       +---+----+     +---+----+     +---+----+
           |              |              |
           +--------------+--------------+
                          |
       +------------------+------------------+
       |                  |                  |
   +---+----+         +---+----+         +---+----+
   | Node 4 |         | Node 5 |         | Node 6 |
   |  DC2   |         |  DC2   |         |  DC2   |
   +--------+         +--------+         +--------+

Replication factor: 3 per datacenter
Consistency: LOCAL_QUORUM (2 of 3 in local DC)
```

### TAO: Social Graph Database

```
TAO = "The Associations and Objects" - Facebook/Meta graph store

OBJECTS (Nodes):              ASSOCIATIONS (Edges):
+------------------+          +---------------------------+
| User             |          | FOLLOWS (user -> user)    |
| Photo            |          | LIKES (user -> photo)     |
| Comment          |          | COMMENTS (user -> photo)  |
| Location         |          | TAGGED (photo -> user)    |
| Hashtag          |          | POSTED (user -> photo)    |
+------------------+          +---------------------------+


GRAPH EXAMPLE:

                   +-------+
                   | User  |
                   | Alice |
                   +---+---+
                       |
         +-------------+-------------+
         | FOLLOWS                   | POSTED
         v                           v
     +-------+                   +-------+
     | User  |                   | Photo |
     |  Bob  |                   |  P123 |
     +---+---+                   +---+---+
         |                           |
         | LIKES                     | TAGGED
         v                           v
     +-------+                   +-------+
     | Photo |                   | User  |
     |  P123 |                   | Carol |
     +-------+                   +-------+


TAO ARCHITECTURE:

+-------------------------------------------------------+
|                     TAO Leaders                        |
|        (Handle writes, maintain consistency)           |
+------------------------+------------------------------+
                         |
          +--------------+--------------+
          |                             |
     +----+-----+                  +----+-----+
     |TAO Cache |                  |TAO Cache |
     | Region 1 |                  | Region 2 |
     +----+-----+                  +----+-----+
          |                             |
     +----+-----+                  +----+-----+
     |  MySQL   |                  |  MySQL   |
     | (Storage)|                  | (Storage)|
     +----------+                  +----------+
```

## Caching Architecture

```
LAYER 1: CDN Cache (Edge)
+---------------------------------------------------------+
|  - Photos, videos, static assets                         |
|  - Cache hit ratio: ~95%                                 |
|  - TTL: Days to weeks                                    |
|  - Global edge locations                                 |
+---------------------------------------------------------+
                              |
                              v
LAYER 2: Memcached (Application Cache)
+---------------------------------------------------------+
|  - User sessions                                         |
|  - User profiles                                         |
|  - Feed data                                             |
|  - Follower/following counts                             |
|                                                          |
|  Cluster: 1000+ servers, ~100TB total memory             |
|  Hit ratio: ~99%                                         |
+---------------------------------------------------------+
                              |
                              v
LAYER 3: TAO Cache (Graph Cache)
+---------------------------------------------------------+
|  - Social graph edges                                    |
|  - Object metadata                                       |
|  - Association lists                                     |
|                                                          |
|  Hit ratio: ~99.9% for reads                             |
+---------------------------------------------------------+
                              |
                              v
LAYER 4: Database (Source of Truth)
+---------------------------------------------------------+
|  - PostgreSQL (relational data)                          |
|  - Cassandra (time-series/feeds)                         |
|  - MySQL (TAO storage)                                   |
+---------------------------------------------------------+


CACHE INVALIDATION STRATEGY:
- Write-through: Update cache on write
- Lease-based: Prevent thundering herd
- McRouter: Consistent hashing for distribution
```

## Feed Generation (Fanout)

```
TWO STRATEGIES:

1. PUSH (Fanout on Write) - For regular users
+----------------------------------------------------------+
|                                                           |
|  User posts photo                                         |
|        |                                                  |
|        v                                                  |
|  +-----------+                                            |
|  |  Fanout   |  "Push to all followers feeds"             |
|  |  Service  |                                            |
|  +-----------+                                            |
|        |                                                  |
|        +-----> Follower 1 feed (Cassandra)                |
|        +-----> Follower 2 feed (Cassandra)                |
|        +-----> Follower 3 feed (Cassandra)                |
|        +-----> ... (up to 10K followers)                  |
|                                                           |
|  Pros: Fast reads, simple read path                       |
|  Cons: Slow writes for popular users                      |
+----------------------------------------------------------+


2. PULL (Fanout on Read) - For celebrities (>10K followers)
+----------------------------------------------------------+
|                                                           |
|  Celebrity posts photo                                    |
|        |                                                  |
|        v                                                  |
|  +-----------+                                            |
|  | Write to  |  "Only store in celebrity own feed"        |
|  | Own Feed  |                                            |
|  +-----------+                                            |
|                                                           |
|  When follower opens app:                                 |
|        |                                                  |
|        v                                                  |
|  +-----------+                                            |
|  |   Merge   |  "Fetch from celebrities + my feed"        |
|  |  Service  |                                            |
|  +-----------+                                            |
|        |                                                  |
|        +-----> My pre-built feed                          |
|        +-----> Celebrity A recent posts                   |
|        +-----> Celebrity B recent posts                   |
|        +-----> Merge & Rank                               |
|                                                           |
|  Pros: Fast writes for celebrities                        |
|  Cons: Slower reads, more complex                         |
+----------------------------------------------------------+


RANKING ALGORITHM (Simplified):

score = (
  affinity_score(user, author) * 0.3 +
  recency_score(post_time) * 0.3 +
  engagement_score(likes, comments) * 0.2 +
  content_type_score(photo, video, reel) * 0.1 +
  diversity_score() * 0.1
)

ML models predict: P(like), P(comment), P(share), P(save)
```

## Message Queue Architecture

```
QUEUE SYSTEM: Apache Kafka + Custom Queues

                    +-------------------------------+
                    |        Kafka Cluster          |
                    |  +----+  +----+  +----+  +----+
                    |  | B1 |  | B2 |  | B3 |  | B4 |
                    |  +----+  +----+  +----+  +----+
                    +-------------------------------+
                                  |
            +---------------------+---------------------+
            |                     |                     |
            v                     v                     v
      +-----------+         +-----------+         +-----------+
      |  media-   |         |   feed-   |         |  notif-   |
      | processing|         |  fanout   |         |  events   |
      |   topic   |         |   topic   |         |   topic   |
      +-----------+         +-----------+         +-----------+
            |                     |                     |
            v                     v                     v
      +-----------+         +-----------+         +-----------+
      |   Media   |         |   Feed    |         |   Push    |
      |  Workers  |         |  Workers  |         |  Workers  |
      |   (100s)  |         |  (1000s)  |         |   (100s)  |
      +-----------+         +-----------+         +-----------+


TASKS PROCESSED ASYNC:
- Image resizing (multiple resolutions)
- Video transcoding (multiple qualities)
- Feed fanout to followers
- Push notifications (APNs, FCM)
- Search indexing (Elasticsearch)
- Analytics events
- ML model inference (recommendations)
- Content moderation (NSFW detection)
- Spam detection
```

## High Availability Design

```
MULTI-DATACENTER DEPLOYMENT:

     US-EAST              US-WEST              EUROPE
    (Primary)           (Secondary)          (Secondary)
  +------------+       +------------+       +------------+
  | Full Stack |       | Full Stack |       | Full Stack |
  | - API      |       | - API      |       | - API      |
  | - Web      |       | - Web      |       | - Web      |
  | - Workers  |       | - Workers  |       | - Workers  |
  | - Database |       | - Database |       | - Database |
  | - Cache    |       | - Cache    |       | - Cache    |
  | - Storage  |       | - Storage  |       | - Storage  |
  +-----+------+       +-----+------+       +-----+------+
        |                    |                    |
        +--------------------+--------------------+
                             |
                   Cross-DC Replication


FAILOVER STRATEGY:

1. DNS-based failover (Route53/similar)
   - Health checks every 10 seconds
   - Automatic traffic shift on failure

2. Database failover
   - PostgreSQL: Promote replica to primary
   - Cassandra: Automatic (masterless)
   - TAO: Automatic leader election

3. Storage failover
   - Haystack: Read from any replica
   - 3-way replication ensures availability


SLA TARGETS:

+------------------+--------------+-----------------+
| Service          | Availability | Latency (p99)   |
+------------------+--------------+-----------------+
| Feed Load        | 99.99%       | < 200ms         |
| Photo Upload     | 99.95%       | < 2s            |
| Photo View       | 99.99%       | < 100ms         |
| Stories          | 99.99%       | < 150ms         |
| Direct Messages  | 99.95%       | < 300ms         |
+------------------+--------------+-----------------+
```

## Technology Stack

### Backend

| Component | Technology |
|-----------|------------|
| Primary Language | Python (Django) |
| API Framework | Django REST Framework |
| RPC | Thrift |
| Web Server | uWSGI + Nginx |
| Load Balancer | Proxygen (L7), Katran (L4) |
| Service Mesh | Custom (Thrift-based) |

### Databases

| Use Case | Technology |
|----------|------------|
| User Data | PostgreSQL (sharded) |
| Feeds/Inbox | Cassandra |
| Social Graph | TAO (MySQL backend) |
| Sessions | Redis |
| Search | Elasticsearch |
| Analytics | Hive, Presto, Spark |

### Storage

| Use Case | Technology |
|----------|------------|
| Hot Photos | Haystack |
| Warm/Cold Photos | F4 (erasure coded) |
| Videos | Custom blob storage |
| CDN | Meta CDN (custom) |

### Caching

| Layer | Technology |
|-------|------------|
| Application Cache | Memcached (McRouter) |
| Graph Cache | TAO Cache |
| CDN Cache | Edge PoPs |
| Local Cache | RocksDB |

### Infrastructure

| Component | Technology |
|-----------|------------|
| Containers | Custom (Tupperware) |
| Orchestration | Custom (similar to K8s) |
| Config Management | Configerator |
| Monitoring | ODS, Scuba |
| Logging | Scribe |

## Key Design Decisions

### Why Django/Python?

```
INSTAGRAM PYTHON/DJANGO CHOICE

Started as: 2 engineers, rapid prototyping
Scaled to: 2+ billion users on same stack

Optimizations made:
1. Cython compilation for hot paths
2. Custom memory allocator
3. Disabled garbage collection (manual memory management)
4. Multi-process deployment (bypass GIL)
5. C extensions for critical code
6. Aggressive caching (99%+ hit rates)

Result: Python serves 500M+ daily users efficiently
```

### Sharding Strategy

```
SHARDING EVOLUTION

Phase 1: Single PostgreSQL
- Worked until ~10M users

Phase 2: Read replicas
- Worked until ~50M users

Phase 3: Vertical sharding
- Separate DBs for users, posts, comments
- Worked until ~200M users

Phase 4: Horizontal sharding
- 4096 logical shards
- Consistent hashing by user_id
- Scales to billions

SHARD REBALANCING:
- Logical shards (4096) map to physical hosts
- Move logical shards between hosts as needed
- Online migration with dual-write
- Zero downtime rebalancing
```

## Lessons Learned

```
KEY LESSONS FROM INSTAGRAM

1. SIMPLICITY SCALES
   - Started with Django/PostgreSQL
   - Added complexity only when needed
   - Monolith -> Services migration was gradual

2. CACHE EVERYTHING
   - 99%+ cache hit rates
   - Multiple cache layers
   - Design for cache invalidation

3. ASYNC PROCESSING
   - Upload returns immediately
   - Heavy work done in background
   - User sees "processing" indicator

4. PUSH VS PULL TRADE-OFF
   - Push for regular users (fast reads)
   - Pull for celebrities (fast writes)
   - Hybrid approach for optimal performance

5. CUSTOM SOLUTIONS AT SCALE
   - Haystack > General file systems
   - TAO > General graph databases
   - Build custom when scale demands it

6. MEASURE EVERYTHING
   - Real-time metrics on all operations
   - A/B test everything
   - Data-driven decisions
```

## Summary

```
INSTAGRAM ARCHITECTURE SUMMARY

PHOTO UPLOAD FLOW:
Mobile App -> CDN -> Load Balancer -> Upload Service -> Kafka ->
[Media Processing | DB Write | Haystack Storage] -> Feed Fanout ->
Notifications -> Done

PHOTO VIEW FLOW:
Mobile App -> CDN (95% hit) -> Haystack (5% miss) -> Response

KEY TECHNOLOGIES:
- Django/Python (API)
- PostgreSQL (users, posts)
- Cassandra (feeds)
- TAO (social graph)
- Haystack/F4 (photo storage)
- Memcached (caching)
- Kafka (async processing)

REPLICATION:
- Photos: 3-way sync replication (Haystack)
- Database: Async cross-DC replication
- Cache: Regional cache clusters

SCALE TECHNIQUES:
- Horizontal sharding (4096 shards)
- Multi-layer caching (99%+ hit rate)
- Async processing (Kafka queues)
- Push/Pull hybrid feed generation
- CDN for static content
```

---

# Deep Technical Internals

## Haystack: Binary Format & Internals

```
HAYSTACK VOLUME FILE FORMAT (On-Disk):

+------------------+------------------+------------------+------------------+
|    Superblock    |     Needle 1     |     Needle 2     |     Needle N     |
|    (8 bytes)     |   (Variable)     |   (Variable)     |   (Variable)     |
+------------------+------------------+------------------+------------------+

SUPERBLOCK FORMAT:
+--------+--------+--------+--------+--------+--------+--------+--------+
| Magic Number (4 bytes)            | Version (2 bytes) | Flags (2 bytes) |
| 0x48 0x41 0x59 0x53 ("HAYS")      | 0x0002            | 0x0001          |
+--------+--------+--------+--------+--------+--------+--------+--------+


NEEDLE FORMAT (Each Photo):
+--------+--------+--------+--------+--------+--------+--------+--------+
| Header                                                                 |
+--------+--------+--------+--------+--------+--------+--------+--------+
| Magic  | Cookie          | Key (photo_id)  | Alternate Key   | Flags  |
| 4 byte | 4 bytes         | 8 bytes         | 4 bytes         | 1 byte |
+--------+--------+--------+--------+--------+--------+--------+--------+
| Size           | Data (JPEG binary)                          | CRC    |
| 4 bytes        | Variable length                             | 4 bytes|
+--------+--------+--------+--------+--------+--------+--------+--------+

Total header overhead: ~29 bytes per photo
Typical photo: 100KB = 0.03% overhead


IN-MEMORY INDEX STRUCTURE (C++ pseudocode):

struct NeedleIndex {
    uint64_t key;           // photo_id
    uint32_t alternate_key; // size variant (150, 320, 640, 1080)
    uint32_t offset;        // byte offset in volume file
    uint32_t size;          // needle size in bytes
    uint16_t flags;         // deletion flag, etc.
} __attribute__((packed));  // 22 bytes per entry

// Hash map for O(1) lookup
class VolumeIndex {
    std::unordered_map<uint64_t, std::vector<NeedleIndex>> index;
    // Key: photo_id
    // Value: vector of size variants (150, 320, 640, 1080)

    NeedleIndex* lookup(uint64_t photo_id, uint32_t size) {
        auto it = index.find(photo_id);
        if (it == index.end()) return nullptr;
        for (auto& needle : it->second) {
            if (needle.alternate_key == size) return &needle;
        }
        return nullptr;
    }
};


READ PATH (Syscall Level):

1. Hash photo_id -> find volume
2. In-memory index lookup -> get offset, size
3. Single pread() syscall:

   ssize_t bytes = pread(volume_fd, buffer, needle_size, offset);

4. Verify CRC32, strip header, return JPEG bytes

// Actual read latency breakdown:
// - Index lookup: ~100ns (in-memory hash)
// - SSD read:     ~100μs (single sequential read)
// - Network:      ~1-5ms (to edge CDN)
// Total: <10ms p99 for cache miss


WRITE PATH (Append-Only):

int write_needle(Volume* vol, uint64_t photo_id, const char* data, size_t len) {
    NeedleHeader header;
    header.magic = NEEDLE_MAGIC;
    header.cookie = generate_cookie();
    header.key = photo_id;
    header.size = len;

    // Atomic append with mutex
    pthread_mutex_lock(&vol->write_mutex);

    off_t offset = vol->current_offset;

    // Write header + data + footer atomically
    struct iovec iov[3];
    iov[0] = {&header, sizeof(header)};
    iov[1] = {(void*)data, len};
    iov[2] = {&crc, sizeof(crc)};

    writev(vol->fd, iov, 3);  // Single syscall
    fdatasync(vol->fd);        // Ensure durability

    vol->current_offset += sizeof(header) + len + sizeof(crc);

    // Update in-memory index
    vol->index.insert(photo_id, offset, len);

    pthread_mutex_unlock(&vol->write_mutex);

    return 0;
}


COMPACTION (Reclaim Deleted Space):

// Deleted photos: flag set, but space not reclaimed
// Compaction: copy live needles to new volume

void compact_volume(Volume* old_vol, Volume* new_vol) {
    for (auto& entry : old_vol->index) {
        if (entry.flags & DELETED) continue;

        // Read from old, write to new
        char* data = read_needle(old_vol, entry.key);
        write_needle(new_vol, entry.key, data, entry.size);
    }

    // Atomic swap
    rename(new_vol->path, old_vol->path);
}

// Compaction triggered when:
// - Deleted space > 20% of volume
// - Scheduled during low-traffic hours (2-6 AM)
```

## Cassandra: Deep Tuning for Instagram Scale

```
CASSANDRA CLUSTER SPECS:

- Nodes per cluster: 100-500
- Replication factor: 3 (LOCAL_QUORUM)
- Data per node: 2-4 TB
- Heap size: 8-16 GB (CMS GC, not G1)
- Off-heap memtables: Enabled
- Compaction: LeveledCompactionStrategy


FEED TABLE DDL:

CREATE TABLE instagram.user_feed (
    user_id bigint,
    post_time timestamp,
    post_id bigint,
    author_id bigint,
    media_type tinyint,
    PRIMARY KEY (user_id, post_time)
) WITH CLUSTERING ORDER BY (post_time DESC)
  AND bloom_filter_fp_chance = 0.01
  AND caching = {'keys': 'ALL', 'rows_per_partition': '1000'}
  AND compaction = {
    'class': 'LeveledCompactionStrategy',
    'sstable_size_in_mb': '160'
  }
  AND compression = {
    'sstable_compression': 'LZ4Compressor',
    'chunk_length_kb': '16'
  }
  AND default_time_to_live = 2592000  -- 30 days TTL
  AND gc_grace_seconds = 86400
  AND memtable_flush_period_in_ms = 0
  AND speculative_retry = '99.0PERCENTILE';


WHY LEVELED COMPACTION?

Size-Tiered (default):        Leveled (Instagram choice):
+--------+                    +--------+
|  L0    | <- Many files      |  L0    | <- Memtable flushes
+--------+                    +--------+
|  L1    | <- Fewer, larger   |  L1    | <- 10 files x 160MB
+--------+                    +--------+
|  L2    | <- Even larger     |  L2    | <- 100 files x 160MB
+--------+                    +--------+
                              |  L3    | <- 1000 files x 160MB
                              +--------+

Leveled benefits:
- Predictable read latency (max 1 file per level)
- Better space amplification (1.1x vs 2-3x)
- Consistent 99th percentile reads


BLOOM FILTER TUNING:

// 1% false positive rate = ~10 bits per key
// 1 billion keys = 1.25 GB bloom filter per node

// Read path with bloom filter:
1. Check bloom filter (in memory) - O(k) hash lookups
2. If negative: guaranteed not in SSTable, skip
3. If positive: read SSTable index, may be false positive

// False positive cost: 1 extra disk read
// 1% FP rate = 1 extra read per 100 queries


PARTITION HOTSPOT MITIGATION:

Problem: Celebrity user_id = hot partition
Solution: Bucket partitioning

CREATE TABLE instagram.user_feed_bucketed (
    user_id bigint,
    bucket int,           -- 0-15 (random or time-based)
    post_time timestamp,
    post_id bigint,
    PRIMARY KEY ((user_id, bucket), post_time)
);

// Write: random bucket assignment
int bucket = random() % 16;
INSERT INTO user_feed_bucketed (user_id, bucket, ...) VALUES (?, ?, ...);

// Read: parallel fetch from all buckets
List<Future<ResultSet>> futures = new ArrayList<>();
for (int bucket = 0; bucket < 16; bucket++) {
    futures.add(session.executeAsync(
        "SELECT * FROM user_feed_bucketed WHERE user_id = ? AND bucket = ? LIMIT 100",
        userId, bucket));
}
// Merge results client-side


WRITE AMPLIFICATION ANALYSIS:

// Single feed write (fanout to 1000 followers):
// - 1 Cassandra write = 1 memtable entry
// - Memtable flush = 1 SSTable write
// - L0 -> L1 compaction = 1 rewrite
// - L1 -> L2 compaction = 1 rewrite
// - L2 -> L3 compaction = 1 rewrite

// Total write amplification: ~4-5x
// Mitigated by: TTL (30 days), fewer compaction levels


REPAIR AND CONSISTENCY:

// Anti-entropy repair (weekly):
nodetool repair -pr instagram  // Primary range only

// Read repair probability: 10%
read_repair_chance: 0.1  // Deprecated in newer versions

// Hinted handoff:
hinted_handoff_enabled: true
max_hint_window_in_ms: 10800000  // 3 hours
```

## Connection Management at Scale

```
MILLION CONCURRENT CONNECTIONS:

                         +------------------------+
                         |    Global DNS (Route53)|
                         +-----------+------------+
                                     |
              +----------------------+----------------------+
              |                      |                      |
     +--------v--------+   +--------v--------+   +--------v--------+
     |   Edge PoP 1    |   |   Edge PoP 2    |   |   Edge PoP N    |
     | (100K conns)    |   | (100K conns)    |   | (100K conns)    |
     +--------+--------+   +--------+--------+   +--------+--------+
              |                      |                      |
              +----------------------+----------------------+
                                     |
                         +-----------v------------+
                         |   L4 Load Balancer     |
                         |   (Katran - eBPF)      |
                         +-----------+------------+
                                     |
              +----------------------+----------------------+
              |                      |                      |
     +--------v--------+   +--------v--------+   +--------v--------+
     |  L7 Proxy Pool  |   |  L7 Proxy Pool  |   |  L7 Proxy Pool  |
     |  (Proxygen)     |   |  (Proxygen)     |   |  (Proxygen)     |
     |  10K conns each |   |  10K conns each |   |  10K conns each |
     +-----------------+   +-----------------+   +-----------------+


KATRAN (eBPF L4 Load Balancer):

// eBPF program loaded into kernel
// No userspace context switch for packet routing

SEC("xdp")
int katran_lb(struct xdp_md *ctx) {
    // Parse packet headers
    struct ethhdr *eth = data;
    struct iphdr *ip = data + sizeof(*eth);
    struct tcphdr *tcp = data + sizeof(*eth) + sizeof(*ip);

    // Consistent hash on 5-tuple
    uint32_t hash = jhash_3words(
        ip->saddr, ip->daddr,
        (tcp->source << 16) | tcp->dest,
        seed);

    // Select backend from hash ring
    struct backend *be = lookup_backend(hash);

    // Rewrite destination IP
    ip->daddr = be->ip;

    // Recalculate checksum (incremental)
    ip->check = csum_diff(old_ip, new_ip, ip->check);

    return XDP_TX;  // Transmit modified packet
}

// Performance: 10M+ packets/sec per core
// Latency: ~1μs added per packet


PROXYGEN (L7 HTTP/2 Proxy):

// Event-driven, non-blocking I/O
// Based on Facebook's Folly library

class InstagramHandler : public proxygen::RequestHandler {
    void onRequest(std::unique_ptr<HTTPMessage> headers) override {
        // Extract auth token
        auto token = headers->getHeaders().getSingleOrEmpty("Authorization");

        // Async auth check (non-blocking)
        authService_->verify(token)
            .thenValue([this](bool valid) {
                if (!valid) {
                    sendError(401, "Unauthorized");
                    return;
                }
                // Route to backend
                routeToBackend();
            });
    }

    void onBody(std::unique_ptr<folly::IOBuf> body) override {
        // Stream body chunks to backend (zero-copy)
        backendTxn_->sendBody(std::move(body));
    }
};


CONNECTION POOLING (Backend):

// HTTP/2 multiplexing: 100+ requests per connection
// Pool size: 10 connections per backend per worker

class ConnectionPool {
    std::unordered_map<std::string, std::queue<Connection*>> pools_;
    std::mutex mutex_;

    Connection* getConnection(const std::string& backend) {
        std::lock_guard<std::mutex> lock(mutex_);

        auto& pool = pools_[backend];
        if (pool.empty()) {
            return createNewConnection(backend);
        }

        Connection* conn = pool.front();
        pool.pop();

        // Validate connection still alive
        if (!conn->isValid()) {
            delete conn;
            return createNewConnection(backend);
        }

        return conn;
    }

    void returnConnection(const std::string& backend, Connection* conn) {
        std::lock_guard<std::mutex> lock(mutex_);

        if (pools_[backend].size() < MAX_POOL_SIZE) {
            pools_[backend].push(conn);
        } else {
            delete conn;  // Pool full
        }
    }
};


TCP TUNING (sysctl):

# Increase connection backlog
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# Faster TIME_WAIT recycling
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15

# Increase port range
net.ipv4.ip_local_port_range = 1024 65535

# Buffer sizes for high throughput
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 87380 16777216

# Enable TCP BBR congestion control
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

## Real-Time Messaging (Instagram Direct)

```
MQTT-BASED MESSAGING ARCHITECTURE:

+------------------+                    +------------------+
|  iOS App         |                    |  Android App     |
|  (MQTT Client)   |                    |  (MQTT Client)   |
+--------+---------+                    +--------+---------+
         |                                       |
         | TLS 1.3                               | TLS 1.3
         | MQTT 3.1.1                            | MQTT 3.1.1
         |                                       |
         +-------------------+-------------------+
                             |
                    +--------v--------+
                    |  MQTT Broker    |
                    |  Cluster        |
                    |  (Custom)       |
                    +--------+--------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v----+  +------v------+  +----v--------+
     | Presence    |  | Message     |  | Delivery    |
     | Service     |  | Router      |  | Tracker     |
     +-------------+  +-------------+  +-------------+


MQTT PROTOCOL DETAILS:

// Connection: persistent, bidirectional
CONNECT {
    client_id: "ig_<user_id>_<device_id>",
    username: "<user_id>",
    password: "<mqtt_token>",
    clean_session: false,      // Persistent session
    keep_alive: 60,            // Ping every 60s
    will_topic: "presence/<user_id>",
    will_message: "offline"
}

// Subscribe to personal inbox
SUBSCRIBE {
    topic: "inbox/<user_id>",
    qos: 1                     // At least once delivery
}

// Send message
PUBLISH {
    topic: "inbox/<recipient_id>",
    qos: 1,
    payload: {
        msg_id: "<uuid>",
        sender_id: "<user_id>",
        type: "text",
        content: "Hello!",
        timestamp: 1716566400000
    }
}


MESSAGE DELIVERY GUARANTEES:

1. QoS 1 (At Least Once):

   Sender                  Broker                 Receiver
     |                        |                        |
     |------- PUBLISH ------->|                        |
     |                        |------- PUBLISH ------->|
     |                        |<------ PUBACK ---------|
     |<------ PUBACK ---------|                        |
     |                        |                        |

   // Duplicate detection via msg_id

2. Offline Message Queue:

   // Messages stored in Cassandra when recipient offline
   CREATE TABLE direct_messages (
       recipient_id bigint,
       msg_id timeuuid,
       sender_id bigint,
       content blob,
       delivered boolean,
       PRIMARY KEY (recipient_id, msg_id)
   ) WITH CLUSTERING ORDER BY (msg_id DESC)
     AND default_time_to_live = 604800;  // 7 days

   // On reconnect: deliver queued messages
   SELECT * FROM direct_messages
   WHERE recipient_id = ? AND delivered = false;


PRESENCE SYSTEM:

// User online/offline status
// Challenge: 500M+ users, real-time updates

class PresenceService {
    // In-memory state (Redis cluster)
    // Key: user_id, Value: {timestamp, device_id, status}
    Redis redis;

    void setOnline(uint64_t userId, const std::string& deviceId) {
        // HSET with TTL
        redis.hset("presence:" + std::to_string(userId), {
            {"status", "online"},
            {"device", deviceId},
            {"ts", std::to_string(now())}
        });
        redis.expire("presence:" + std::to_string(userId), 90);

        // Publish to subscribers
        redis.publish("presence_updates",
            json({{"user_id", userId}, {"status", "online"}}));
    }

    bool isOnline(uint64_t userId) {
        auto status = redis.hget("presence:" + std::to_string(userId), "status");
        return status == "online";
    }

    // Heartbeat to maintain presence (every 60s)
    void heartbeat(uint64_t userId) {
        redis.expire("presence:" + std::to_string(userId), 90);
    }
};


TYPING INDICATORS:

// Ephemeral, not persisted
// Pub/Sub via Redis

PUBLISH {
    topic: "typing/<thread_id>",
    payload: {
        user_id: 12345,
        action: "start"  // or "stop"
    }
}

// Client subscribes to typing events for active threads
// Events discarded if recipient not connected (no queueing)


READ RECEIPTS:

// Persisted for offline delivery
// Batched to reduce write load

class ReadReceiptBatcher {
    std::unordered_map<uint64_t, std::vector<uint64_t>> pending;
    std::mutex mutex_;

    void markRead(uint64_t threadId, uint64_t msgId) {
        std::lock_guard<std::mutex> lock(mutex_);
        pending[threadId].push_back(msgId);
    }

    // Flush every 5 seconds
    void flush() {
        std::lock_guard<std::mutex> lock(mutex_);
        for (auto& [threadId, msgIds] : pending) {
            cassandra.execute(
                "UPDATE direct_threads SET last_read = ? WHERE thread_id = ?",
                msgIds.back(), threadId);
        }
        pending.clear();
    }
};
```

## ML Pipeline for Feed Ranking

```
RANKING SYSTEM ARCHITECTURE:

+------------------+     +------------------+     +------------------+
|  Feature Store   |     |  Model Serving   |     |  Ranking Engine  |
|  (Online/Offline)|     |  (TorchServe)    |     |  (C++ Scorer)    |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
         v                        v                        v
+--------------------------------------------------------------------------+
|                           Feature Pipeline                                |
|                                                                          |
|  User Features:              Post Features:          Context Features:   |
|  - Follow graph embedding    - Image embedding       - Time of day       |
|  - Interest vector           - Caption embedding     - Device type       |
|  - Engagement history        - Hashtag features      - Network type      |
|  - Session context           - Author features       - Location          |
+--------------------------------------------------------------------------+


FEATURE COMPUTATION (Real-time):

class FeatureExtractor {
    // Pre-computed embeddings from offline pipeline
    EmbeddingStore userEmbeddings;   // 256-dim float32
    EmbeddingStore postEmbeddings;   // 512-dim float32

    Features extractFeatures(uint64_t userId, uint64_t postId, Context ctx) {
        Features f;

        // User features (cached, updated daily)
        auto userEmb = userEmbeddings.get(userId);
        f.set("user_embedding", userEmb);
        f.set("follower_count", userStats.get(userId).followers);
        f.set("avg_session_duration", userStats.get(userId).avgSession);

        // Post features (computed on upload)
        auto postEmb = postEmbeddings.get(postId);
        f.set("post_embedding", postEmb);
        f.set("post_age_hours", (now() - postTimestamp) / 3600);
        f.set("like_count", postStats.get(postId).likes);
        f.set("comment_count", postStats.get(postId).comments);

        // Interaction features (computed real-time)
        f.set("author_affinity", computeAffinity(userId, post.authorId));
        f.set("hashtag_affinity", computeHashtagAffinity(userId, post.hashtags));

        // Context features
        f.set("hour_of_day", ctx.localHour);
        f.set("day_of_week", ctx.dayOfWeek);
        f.set("is_wifi", ctx.networkType == WIFI);

        return f;
    }

    float computeAffinity(uint64_t userId, uint64_t authorId) {
        // Based on: profile visits, likes, comments, DMs
        auto interactions = interactionStore.get(userId, authorId);
        return (
            interactions.profileVisits * 0.1 +
            interactions.likes * 0.3 +
            interactions.comments * 0.4 +
            interactions.dms * 0.5
        ) / interactions.totalPosts;
    }
};


MODEL ARCHITECTURE (Simplified):

class FeedRankingModel(nn.Module):
    def __init__(self):
        # User tower
        self.user_encoder = nn.Sequential(
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, 64)
        )

        # Post tower
        self.post_encoder = nn.Sequential(
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.Linear(256, 64)
        )

        # Cross features
        self.cross_net = CrossNet(input_dim=128, num_layers=3)

        # Final prediction heads
        self.like_head = nn.Linear(64, 1)
        self.comment_head = nn.Linear(64, 1)
        self.share_head = nn.Linear(64, 1)
        self.dwell_head = nn.Linear(64, 1)

    def forward(self, user_features, post_features, context):
        user_emb = self.user_encoder(user_features)
        post_emb = self.post_encoder(post_features)

        # Dot product attention
        interaction = user_emb * post_emb

        # Multi-task predictions
        p_like = torch.sigmoid(self.like_head(interaction))
        p_comment = torch.sigmoid(self.comment_head(interaction))
        p_share = torch.sigmoid(self.share_head(interaction))
        dwell_time = torch.relu(self.dwell_head(interaction))

        return p_like, p_comment, p_share, dwell_time


RANKING SCORE COMPUTATION:

// Final score combines multiple objectives
float computeRankingScore(Predictions pred, const Post& post) {
    float score = 0.0;

    // Engagement predictions (from ML model)
    score += pred.p_like * LIKE_WEIGHT;           // 0.25
    score += pred.p_comment * COMMENT_WEIGHT;     // 0.20
    score += pred.p_share * SHARE_WEIGHT;         // 0.15
    score += pred.p_save * SAVE_WEIGHT;           // 0.10
    score += pred.dwell_time * DWELL_WEIGHT;      // 0.15

    // Recency decay (exponential)
    float age_hours = (now() - post.timestamp) / 3600.0;
    float recency = exp(-age_hours / 24.0);       // Half-life: 24 hours
    score *= recency;

    // Diversity boost (avoid same author consecutively)
    if (recentAuthors.contains(post.authorId)) {
        score *= 0.7;  // 30% penalty
    }

    // Content type boost (Reels > Video > Photo)
    score *= CONTENT_TYPE_BOOST[post.contentType];

    // Quality signals
    score *= post.qualityScore;  // 0.0 - 1.0

    return score;
}


MODEL SERVING LATENCY BUDGET:

// Total latency budget: 200ms
// Breakdown:
// - Feature extraction:    20ms
// - Model inference:       50ms (batched, GPU)
// - Ranking/sorting:       10ms
// - Hydration:            100ms (fetch post details)
// - Network overhead:      20ms

// Batch inference for efficiency
// 500 candidate posts -> 1 batch -> 50ms GPU inference
// Without batching: 500 * 2ms = 1000ms (too slow)
```

## Rate Limiting & Abuse Prevention

```
MULTI-LAYER RATE LIMITING:

Layer 1: Edge (CDN)          Layer 2: L7 Proxy         Layer 3: Application
+-------------------+        +-------------------+      +-------------------+
| IP-based limiting |        | Token bucket per  |      | User-level        |
| 1000 req/min/IP   |        | auth token        |      | action limits     |
| DDoS protection   |        | 100 req/min       |      | 100 likes/hour    |
+-------------------+        +-------------------+      +-------------------+


TOKEN BUCKET ALGORITHM:

class TokenBucket {
    int64_t tokens;
    int64_t capacity;
    int64_t refillRate;  // tokens per second
    int64_t lastRefill;
    std::mutex mutex_;

public:
    TokenBucket(int64_t capacity, int64_t refillRate)
        : tokens(capacity), capacity(capacity),
          refillRate(refillRate), lastRefill(now()) {}

    bool tryConsume(int64_t numTokens = 1) {
        std::lock_guard<std::mutex> lock(mutex_);

        // Refill tokens based on elapsed time
        int64_t elapsed = now() - lastRefill;
        int64_t newTokens = elapsed * refillRate / 1000;
        tokens = std::min(capacity, tokens + newTokens);
        lastRefill = now();

        // Try to consume
        if (tokens >= numTokens) {
            tokens -= numTokens;
            return true;
        }
        return false;
    }
};

// Redis-based distributed rate limiter
class DistributedRateLimiter {
    Redis redis;

    bool isAllowed(const std::string& key, int limit, int windowSec) {
        auto now = currentTimeSeconds();
        auto windowKey = key + ":" + std::to_string(now / windowSec);

        // INCR and EXPIRE atomically via Lua script
        auto count = redis.eval(R"(
            local current = redis.call('INCR', KEYS[1])
            if current == 1 then
                redis.call('EXPIRE', KEYS[1], ARGV[1])
            end
            return current
        )", {windowKey}, {std::to_string(windowSec)});

        return count <= limit;
    }
};


ACTION-SPECIFIC LIMITS:

Action          | Limit              | Window  | Response
----------------|--------------------|---------|-----------------
Follow          | 200/day            | 24h     | 429 + Retry-After
Unfollow        | 200/day            | 24h     | 429 + Retry-After
Like            | 350/hour           | 1h      | 429 + temp block
Comment         | 180/hour           | 1h      | 429 + temp block
DM              | 80/hour            | 1h      | 429 + temp block
Post            | 25/day             | 24h     | 429 + Retry-After
Story           | 100/day            | 24h     | 429 + Retry-After
API calls       | 200/hour           | 1h      | 429 + rate limit header


ABUSE DETECTION SIGNALS:

class AbuseDetector {
    MLModel spamModel;

    float computeAbuseScore(const Action& action, const User& user) {
        Features f;

        // Velocity features
        f.set("actions_last_minute", user.recentActions(60));
        f.set("actions_last_hour", user.recentActions(3600));
        f.set("unique_targets_last_hour", user.uniqueTargets(3600));

        // Pattern features
        f.set("action_interval_variance", computeVariance(user.actionIntervals()));
        f.set("similar_content_ratio", user.similarContentRatio());

        // Account features
        f.set("account_age_days", user.accountAgeDays());
        f.set("follower_following_ratio", user.followers / user.following);
        f.set("profile_completeness", user.profileCompleteness());
        f.set("phone_verified", user.phoneVerified ? 1.0 : 0.0);

        // Network features
        f.set("ip_reputation", ipReputationService.getScore(action.ip));
        f.set("device_reputation", deviceReputationService.getScore(action.deviceId));

        return spamModel.predict(f);
    }

    ActionResult checkAction(const Action& action, const User& user) {
        float abuseScore = computeAbuseScore(action, user);

        if (abuseScore > 0.95) {
            return ActionResult::BLOCK;
        } else if (abuseScore > 0.8) {
            return ActionResult::CAPTCHA;
        } else if (abuseScore > 0.6) {
            return ActionResult::SHADOWBAN;  // Actions succeed but not visible
        } else {
            return ActionResult::ALLOW;
        }
    }
};
```

## Python Memory Management (Instagram's GC Hack)

```
THE PROBLEM WITH PYTHON GC:

// Python reference counting + cyclic GC
// GC pause times: 10-100ms (unpredictable)
// At Instagram scale: 10% of requests hit GC pause

// Standard Python memory layout:
Object -> refcount (8 bytes) -> type pointer -> data
// Every object manipulation: refcount increment/decrement
// Every reference cycle: full GC collection


INSTAGRAM'S SOLUTION: DISABLE GC

// In uwsgi.ini
[uwsgi]
; Disable GC in worker processes
lazy-apps = true
; Fork after loading app code

// In application startup
import gc
gc.disable()  # Disable automatic GC

// Why this works:
// 1. Web requests are short-lived (< 200ms)
// 2. Most objects freed by refcounting (no cycles)
// 3. Cyclic garbage is rare in request handling
// 4. Worker process recycled every N requests


WORKER RECYCLING STRATEGY:

# uwsgi configuration
max-requests = 10000        # Recycle after 10K requests
max-worker-lifetime = 3600  # Or after 1 hour
reload-on-rss = 2048        # Or if RSS exceeds 2GB

# Process lifecycle:
1. Fork new worker from master
2. Handle 10K requests (GC disabled)
3. Memory grows slowly (leaked cycles)
4. After 10K requests: exit gracefully
5. Master forks new worker
6. Old worker's memory returned to OS


MEMORY ARENA OPTIMIZATION:

// Problem: malloc fragmentation with many small allocations
// Solution: Custom memory arenas

// Using jemalloc instead of glibc malloc
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2

// jemalloc config (via environment)
MALLOC_CONF="background_thread:true,metadata_thp:auto,dirty_decay_ms:5000"

// Results:
// - 20-30% memory reduction
// - More predictable allocation times
// - Better multi-threaded performance


CYTHON FOR HOT PATHS:

# Original Python (hot path)
def compute_feed_score(user_features, post_features):
    score = 0.0
    for i in range(len(user_features)):
        score += user_features[i] * post_features[i]
    return score

# Cython version (10-100x faster)
# feed_scorer.pyx
cimport cython
from libc.math cimport exp

@cython.boundscheck(False)
@cython.wraparound(False)
cpdef double compute_feed_score(double[:] user_features, double[:] post_features):
    cdef double score = 0.0
    cdef Py_ssize_t i
    cdef Py_ssize_t n = user_features.shape[0]

    for i in range(n):
        score += user_features[i] * post_features[i]

    return score

# Compile: cython -3 feed_scorer.pyx && gcc -shared -o feed_scorer.so ...


MEMORY PROFILING:

// Using memory_profiler for debugging
@profile
def process_request(request):
    user = load_user(request.user_id)  # +50KB
    feed = generate_feed(user)          # +200KB
    response = serialize(feed)          # +100KB
    return response                      # Total: ~350KB per request

// Using tracemalloc in production (sampling)
import tracemalloc
tracemalloc.start(25)  # 25 frames deep

# Periodically dump top allocations
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')
for stat in top_stats[:10]:
    print(stat)
```

## Observability & Debugging at Scale

```
METRICS PIPELINE:

Application          ODS (Collection)         Scuba (Analysis)
+--------+          +---------------+         +---------------+
| Counter|--------->| Time-series   |-------->| OLAP queries  |
| Gauge  |  UDP     | Database      |  ETL    | Real-time     |
| Timer  |          | (1-sec gran)  |         | dashboards    |
+--------+          +---------------+         +---------------+


KEY METRICS (Examples):

// Request latency histogram
timer.record("api.request.latency", latencyMs, {
    {"endpoint", "/api/v1/feed"},
    {"status", "200"},
    {"region", "us-east"}
});

// Throughput counter
counter.increment("api.requests.total", 1, {
    {"endpoint", "/api/v1/feed"},
    {"method", "GET"}
});

// Gauge for active connections
gauge.set("connections.active", activeConnections, {
    {"pool", "mysql"},
    {"shard", "0"}
});


DISTRIBUTED TRACING:

// Custom trace context propagation
class TraceContext:
    trace_id: str      # 128-bit UUID
    span_id: str       # 64-bit
    parent_span_id: str
    sampled: bool
    baggage: dict      # App-specific metadata

// Propagated via HTTP headers
X-Trace-ID: abc123def456
X-Span-ID: span789
X-Parent-Span-ID: parentspan456
X-Sampled: 1
X-Baggage-User-ID: 12345

// Trace structure
Trace: abc123def456
├── Span: api_handler (50ms)
│   ├── Span: auth_check (5ms)
│   ├── Span: load_user (10ms)
│   │   └── Span: postgres_query (8ms)
│   ├── Span: generate_feed (30ms)
│   │   ├── Span: cassandra_read (15ms)
│   │   └── Span: ml_ranking (10ms)
│   └── Span: serialize_response (3ms)


ERROR TRACKING:

class ErrorTracker {
    void trackException(const std::exception& e, const Context& ctx) {
        ErrorEvent event;
        event.exception_type = typeid(e).name();
        event.message = e.what();
        event.stack_trace = captureStackTrace();

        // Context for debugging
        event.user_id = ctx.userId;
        event.request_id = ctx.requestId;
        event.endpoint = ctx.endpoint;
        event.params = ctx.sanitizedParams();  // Redact PII

        // Aggregation key (for grouping similar errors)
        event.fingerprint = computeFingerprint(e, event.stack_trace);

        // Sample rate (1% of errors logged in detail)
        if (shouldSample(event.fingerprint)) {
            errorQueue.push(event);
        }

        // Always increment counter
        counter.increment("errors.total", 1, {
            {"type", event.exception_type},
            {"endpoint", ctx.endpoint}
        });
    }
};


ALERTING RULES (Examples):

# P99 latency alert
- alert: HighP99Latency
  expr: histogram_quantile(0.99, rate(api_request_duration_seconds_bucket[5m])) > 0.5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "P99 latency above 500ms"

# Error rate alert
- alert: HighErrorRate
  expr: rate(api_requests_total{status=~"5.."}[5m]) / rate(api_requests_total[5m]) > 0.01
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Error rate above 1%"

# Cascading failure detection
- alert: CascadingFailure
  expr: |
    (rate(api_requests_total{status="503"}[1m]) > 100)
    and
    (rate(downstream_requests_total{status="503"}[1m]) > 50)
  for: 1m
  labels:
    severity: critical


DEBUGGING PRODUCTION ISSUES:

// 1. Check real-time dashboards
//    - Request rate, latency, error rate
//    - Resource utilization (CPU, memory, network)

// 2. Query distributed traces
SELECT * FROM traces
WHERE service = 'feed-service'
  AND duration_ms > 1000
  AND timestamp > now() - interval '5 minutes'
ORDER BY duration_ms DESC
LIMIT 100;

// 3. Drill into specific trace
SELECT * FROM spans
WHERE trace_id = 'abc123def456'
ORDER BY start_time;

// 4. Check error aggregations
SELECT fingerprint, count(*), any(stack_trace)
FROM errors
WHERE timestamp > now() - interval '1 hour'
GROUP BY fingerprint
ORDER BY count(*) DESC
LIMIT 20;

// 5. Compare to baseline
SELECT
    date_trunc('minute', timestamp) as minute,
    percentile_cont(0.99) WITHIN GROUP (ORDER BY latency_ms) as p99
FROM requests
WHERE timestamp > now() - interval '1 day'
GROUP BY minute;
```

## References

- [Instagram Engineering Blog](https://instagram-engineering.com/)
- [Scaling Instagram Infrastructure](https://www.youtube.com/watch?v=hnpzNAPiC0E)
- [Instagram at Scale with Python](https://www.youtube.com/watch?v=66XoCk79kjM)
- [Haystack: Facebook's Photo Storage](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf)
- [TAO: Facebook's Distributed Data Store](https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf)
- [Cassandra at Instagram](https://www.datastax.com/blog/instagram-cassandra)
- [Dismissing Python Garbage Collection at Instagram](https://instagram-engineering.com/dismissing-python-garbage-collection-at-instagram-4dca40b29172)
- [Katran: Facebook's L4 Load Balancer](https://engineering.fb.com/2018/05/22/open-source/open-sourcing-katran-a-scalable-network-load-balancer/)
