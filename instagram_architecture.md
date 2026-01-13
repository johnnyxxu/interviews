# Instagram Architecture Overview: Load Balancing, Databases, Failover & Retry Strategies

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENT LAYER                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                              │
│  │ iOS App  │  │ Android  │  │   Web    │                              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                              │
└───────┼─────────────┼─────────────┼────────────────────────────────────┘
        │             │             │
        └─────────────┴─────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        EDGE / CDN LAYER                                  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │    CloudFront CDN (Static Assets: Images, Videos, JS, CSS)       │  │
│  │    - Global Edge Locations (reduces latency)                      │  │
│  │    - Caches media files close to users                            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     LOAD BALANCER LAYER (ELB)                           │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Amazon Elastic Load Balancer (ELB)                              │  │
│  │  - Health checks every 2-3 seconds                                │  │
│  │  - Removes unhealthy instances automatically                      │  │
│  │  - Strategies: Round-robin, Least connections, IP hash            │  │
│  │  - Distributes traffic across availability zones                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    APPLICATION SERVER LAYER                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │  Django  │  │  Django  │  │  Django  │  │  Django  │  (25+ servers) │
│  │  Server  │  │  Server  │  │  Server  │  │  Server  │               │
│  │  (EC2)   │  │  (EC2)   │  │  (EC2)   │  │  (EC2)   │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
│       │             │             │             │                        │
│  STATELESS: Any server can handle any request                           │
│  - No session storage on server                                         │
│  - Horizontally scalable                                                 │
└─────────────────────────────────────────────────────────────────────────┘
        │
        └──────────────┬──────────────┬──────────────┬──────────────┐
                       │              │              │              │
                       ▼              ▼              ▼              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      CACHING LAYER                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Memcached Cluster (6+ instances)                                │   │
│  │  - User data, feed data, photo metadata                          │   │
│  │  - Reduces database load by 90%+                                 │   │
│  │  - TTL-based expiration                                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Redis Cluster (Sharded)                                         │   │
│  │  - Photo ID → User ID mapping (300M+ entries in 5GB)            │   │
│  │  - User feeds (lists of Media IDs)                               │   │
│  │  - Session data                                                   │   │
│  │  - Real-time data structures                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                   CONNECTION POOLER LAYER                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Pgbouncer (Connection Pooler)                                   │   │
│  │  - Reduces memory overhead (1.3MB per connection)                │   │
│  │  - Reuses existing database connections                          │   │
│  │  - Connection pool per database shard                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┬──────────────┐
        ▼              ▼              ▼              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                   DATABASE LAYER (PostgreSQL)                            │
│                                                                           │
│  ┌───────────────────┐  ┌───────────────────┐  ┌──────────────────┐   │
│  │   Shard 1 (User)  │  │   Shard 2 (User)  │  │  Shard N (User)  │   │
│  │   ┌───────────┐   │  │   ┌───────────┐   │  │  ┌───────────┐  │   │
│  │   │  Primary  │   │  │   │  Primary  │   │  │  │  Primary  │  │   │
│  │   └─────┬─────┘   │  │   └─────┬─────┘   │  │  └─────┬─────┘  │   │
│  │         │         │  │         │         │  │        │        │   │
│  │   ┌─────▼─────┐   │  │   ┌─────▼─────┐   │  │  ┌─────▼─────┐  │   │
│  │   │ Replica 1 │   │  │   │ Replica 1 │   │  │  │ Replica 1 │  │   │
│  │   └───────────┘   │  │   └───────────┘   │  │  └───────────┘  │   │
│  │   ┌───────────┐   │  │   ┌───────────┐   │  │  ┌───────────┐  │   │
│  │   │ Replica 2 │   │  │   │ Replica 2 │   │  │   │ Replica 2 │  │   │
│  │   └───────────┘   │  │   └───────────┘   │  │  └───────────┘  │   │
│  └───────────────────┘  └───────────────────┘  └──────────────────┘   │
│                                                                           │
│  SHARDING STRATEGY:                                                      │
│  - Thousands of LOGICAL shards → Fewer PHYSICAL servers                 │
│  - Shard key: User ID (user_id % 2000 = logical_shard_id)              │
│  - Logical shards can be moved between physical servers easily          │
│  - No resharding needed when adding capacity                            │
└──────────────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      OBJECT STORAGE LAYER                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Amazon S3                                                       │   │
│  │  - Original photos and videos                                    │   │
│  │  - Multiple processed versions (thumbnails, optimized sizes)     │   │
│  │  - Highly durable (99.999999999%)                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    ASYNC PROCESSING LAYER                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Task Queue (Celery + RabbitMQ)                                  │   │
│  │  - Photo processing (resize, filters)                            │   │
│  │  - Feed generation for followers                                 │   │
│  │  - Notification delivery                                          │   │
│  │  - Search indexing (Solr)                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Database Sharding Architecture (Detail)

### Instagram's 64-bit ID Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Instagram's Sortable ID Format                        │
│                           (64 bits total)                                │
│                                                                           │
│  ┌────────────────────┬──────────────────┬────────────────────────┐    │
│  │   41 bits          │   13 bits        │   10 bits              │    │
│  │   Timestamp        │   Shard ID       │   Auto-increment       │    │
│  │   (milliseconds)   │   (0-8191)       │   (0-1023 per ms)      │    │
│  └────────────────────┴──────────────────┴────────────────────────┘    │
│                                                                           │
│  BENEFITS:                                                               │
│  ✓ Sortable by time (newest first in queries)                           │
│  ✓ No coordination needed between shards                                │
│  ✓ Can generate 1024 IDs per shard per millisecond                      │
│  ✓ 41 years of IDs from custom epoch                                    │
└───────────────────────────────────────────────────────────────────────────┘
```

### Logical vs Physical Sharding

```
LOGICAL SHARDS (Thousands)                    PHYSICAL SERVERS (Dozens)
┌──────────────────────────┐                 ┌────────────────────────┐
│ Logical Shard 0-499      │─────────────────│  Physical Server 1     │
│ (PostgreSQL Schemas)     │                 │  (12 cores, 68GB RAM)  │
└──────────────────────────┘                 └────────────────────────┘
                                              
┌──────────────────────────┐                 ┌────────────────────────┐
│ Logical Shard 500-999    │─────────────────│  Physical Server 2     │
│ (PostgreSQL Schemas)     │                 │  (12 cores, 68GB RAM)  │
└──────────────────────────┘                 └────────────────────────┘

┌──────────────────────────┐                 ┌────────────────────────┐
│ Logical Shard 1000-1499  │─────────────────│  Physical Server 3     │
│ (PostgreSQL Schemas)     │                 │  (12 cores, 68GB RAM)  │
└──────────────────────────┘                 └────────────────────────┘

CONFIG MAPPING (JSON):
{
  "logical_shards": {
    "0-499": "postgres_server_1",
    "500-999": "postgres_server_2",
    "1000-1499": "postgres_server_3"
  }
}

MIGRATION PROCESS:
When Server 2 fills up → Move logical shard 750-999 to new Server 4
→ Update config file only
→ No data resharding needed!
```

---

## Load Balancer Strategies & Failover

### Load Balancing Algorithm Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCER ROUTING STRATEGIES                      │
└─────────────────────────────────────────────────────────────────────────┘

REQUEST TYPE: API Call for User Feed
├─ Strategy: IP Hash (Sticky Sessions)
├─ Why: Same user → same app server → better cache hit rate in local memory
└─ Fallback: If server down, hash to next available

REQUEST TYPE: Photo Upload
├─ Strategy: Least Connections
├─ Why: Upload processing is CPU-intensive, balance actual load
└─ Failover: Auto-remove server if health check fails

REQUEST TYPE: Public Content Browse
├─ Strategy: Round Robin
├─ Why: Stateless requests, even distribution
└─ Failover: Remove from rotation, automatic health check recovery

┌─────────────────────────────────────────────────────────────────────────┐
│                         HEALTH CHECK MECHANISM                           │
└─────────────────────────────────────────────────────────────────────────┘

ELB Health Check Configuration:
- Protocol: HTTP/HTTPS
- Endpoint: GET /health
- Interval: 3 seconds
- Timeout: 2 seconds  
- Unhealthy threshold: 2 consecutive failures
- Healthy threshold: 3 consecutive successes

Django /health Endpoint Response:
{
  "status": "healthy",
  "database": "connected",
  "cache": "connected",
  "timestamp": 1678234567890
}

AUTOMATIC FAILOVER SEQUENCE:
1. ELB detects 2 failed health checks (6 seconds total)
2. Server marked "unhealthy" - stops receiving new requests
3. Existing connections: Drain for 60 seconds (connection draining)
4. New requests: Distributed to healthy servers only
5. Monitoring alert: PagerDuty notifies on-call engineer
```

---

## Retry Strategy & Circuit Breaker Pattern

### Application-Level Retry Logic

```python
# Instagram's Retry Strategy (Conceptual)

import time
import random
from typing import Callable, Any
from enum import Enum

class RetryStrategy(Enum):
    EXPONENTIAL_BACKOFF = "exponential"
    CONSTANT_BACKOFF = "constant"
    JITTERED_BACKOFF = "jittered"

def retry_with_backoff(
    func: Callable,
    max_attempts: int = 3,
    base_delay: float = 0.1,
    max_delay: float = 10.0,
    strategy: RetryStrategy = RetryStrategy.EXPONENTIAL_BACKOFF,
    retriable_exceptions: tuple = (TimeoutError, ConnectionError)
):
    """
    Retry mechanism used across Instagram services
    
    Key Principles:
    - Don't retry on client errors (4xx)
    - Retry on transient server errors (5xx, timeouts, connection errors)
    - Use exponential backoff to prevent retry storms
    - Add jitter to prevent thundering herd
    """
    
    for attempt in range(max_attempts):
        try:
            return func()
        except retriable_exceptions as e:
            if attempt == max_attempts - 1:
                # Last attempt failed, re-raise
                raise
            
            # Calculate delay based on strategy
            if strategy == RetryStrategy.EXPONENTIAL_BACKOFF:
                delay = min(base_delay * (2 ** attempt), max_delay)
            elif strategy == RetryStrategy.CONSTANT_BACKOFF:
                delay = base_delay
            else:  # JITTERED_BACKOFF
                delay = min(base_delay * (2 ** attempt), max_delay)
                delay = delay * (0.5 + random.random() * 0.5)  # Add 50% jitter
            
            # Log retry attempt
            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay:.2f}s...")
            time.sleep(delay)

# CIRCUIT BREAKER PATTERN
class CircuitBreaker:
    """
    Prevents cascading failures by temporarily blocking requests to failing service
    
    States:
    - CLOSED: Normal operation, requests pass through
    - OPEN: Service failing, requests blocked immediately  
    - HALF_OPEN: Testing if service recovered
    """
    
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: float = 60.0,
        success_threshold: int = 2
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func: Callable) -> Any:
        if self.state == "OPEN":
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                # Try half-open state
                self.state = "HALF_OPEN"
                self.success_count = 0
            else:
                raise Exception("Circuit breaker is OPEN - service unavailable")
        
        try:
            result = func()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        self.failure_count = 0
        if self.state == "HALF_OPEN":
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = "CLOSED"
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"

# EXAMPLE USAGE
def fetch_user_data_from_db(user_id: int):
    """Simulate database call that might fail"""
    # ... actual database query
    pass

# Wrap with retry logic
def get_user_data(user_id: int):
    circuit_breaker = CircuitBreaker()
    
    def db_call():
        return retry_with_backoff(
            lambda: fetch_user_data_from_db(user_id),
            max_attempts=3,
            strategy=RetryStrategy.JITTERED_BACKOFF
        )
    
    return circuit_breaker.call(db_call)
```

---

## Database Failover Mechanisms

### PostgreSQL High Availability Setup

```
┌─────────────────────────────────────────────────────────────────────────┐
│              PostgreSQL Primary-Replica Failover                         │
└─────────────────────────────────────────────────────────────────────────┘

NORMAL OPERATION:
┌──────────────┐         Async Replication        ┌──────────────┐
│   PRIMARY    │────────────────────────────────>│  REPLICA 1   │
│  (Write)     │         WAL Shipping             │  (Read)      │
└──────────────┘                                  └──────────────┘
       │                                                  │
       │                                                  │
       └──────────────────────────────────────────────────┘
                          │
                          ▼
                  ┌──────────────┐
                  │  REPLICA 2   │
                  │  (Read)      │
                  └──────────────┘

FAILOVER SCENARIO (Primary Fails):

Step 1: Detection (3-5 seconds)
- Health checks fail on primary
- Pgbouncer can't establish connections
- Monitoring alerts triggered

Step 2: Promotion (10-30 seconds)
- Replica 1 promoted to new primary
- WAL replay completed
- Write capabilities enabled

Step 3: Reconfiguration (30-60 seconds)
- Pgbouncer updated with new primary endpoint
- Application servers reconnect automatically
- Old primary becomes replica after recovery

TOTAL FAILOVER TIME: ~60 seconds for writes, 0 seconds for reads

┌─────────────────────────────────────────────────────────────────────────┐
│                    DATA CONSISTENCY GUARANTEES                           │
└─────────────────────────────────────────────────────────────────────────┘

Synchronous Replication (Critical Data):
- Write acknowledged only after replica confirms
- Zero data loss
- Higher latency (~5-10ms)
- Used for: Financial transactions, user authentication

Asynchronous Replication (Most Data):
- Write acknowledged immediately
- Potential data loss: Last few seconds of writes
- Lower latency (~1ms)
- Used for: Posts, likes, comments

Instagram's Choice:
- Async replication for 95% of data
- Risk mitigation: Can lose ~5 seconds of data during failover
- Acceptable for social media use case
- Faster response times more important than perfect consistency
```

---

## Request Flow with Failure Handling

### Photo Upload Flow (Write Path)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     PHOTO UPLOAD REQUEST FLOW                            │
└─────────────────────────────────────────────────────────────────────────┘

1. CLIENT: Upload photo (5MB JPEG)
   │
   ▼
2. ELB: Health check passed, route to App Server 7
   │   RETRY: If server down, try Server 8, then 9
   │   TIMEOUT: 30 seconds
   ▼
3. DJANGO SERVER: Validate request
   │   - Check auth token
   │   - Validate file size/format
   │   CLIENT ERROR (4xx): Return immediately, DON'T RETRY
   ▼
4. S3 UPLOAD: Store original image
   │   RETRY POLICY:
   │   - Max 3 attempts
   │   - Exponential backoff: 1s, 2s, 4s
   │   - Timeout per attempt: 10s
   │   FAILURES:
   │   - 503 Service Unavailable: RETRY
   │   - 500 Internal Server Error: RETRY
   │   - Network timeout: RETRY
   │   - 400 Bad Request: DON'T RETRY (client error)
   ▼
5. GENERATE MEDIA ID: Using custom 64-bit ID system
   │   - No external dependency
   │   - Never fails (purely computational)
   ▼
6. STORE METADATA: Write to PostgreSQL (sharded by user_id)
   │   DB WRITE PROCESS:
   │   ├─ Try PRIMARY on shard
   │   │  └─ TIMEOUT: 5 seconds
   │   ├─ If primary fails:
   │   │  └─ Circuit breaker opens
   │   │  └─ Wait for replica promotion (30-60s)
   │   │  └─ Retry on new primary
   │   └─ If all fails after 3 attempts:
   │      └─ Return 503 to client with retry_after header
   ▼
7. ASYNC TASKS: Enqueue background jobs
   │   - Thumbnail generation
   │   - Feed distribution to followers
   │   - Notification delivery
   │   - Search indexing
   │   
   │   QUEUE RETRY POLICY (Celery):
   │   - Max retries: 3
   │   - Retry delay: Exponential (60s, 120s, 240s)
   │   - Dead letter queue after final failure
   ▼
8. RESPONSE: Return success to client
   {
     "status": "success",
     "media_id": "2847582938475829384",
     "url": "https://cdn.instagram.com/photos/abc123.jpg"
   }

┌─────────────────────────────────────────────────────────────────────────┐
│                         ERROR HANDLING MATRIX                            │
└─────────────────────────────────────────────────────────────────────────┘

ERROR TYPE              | RETRY? | BACKOFF   | MAX ATTEMPTS | CLIENT ACTION
─────────────────────────────────────────────────────────────────────────────
Network Timeout         | YES    | Exp (1s)  | 3            | Auto retry
503 Service Unavailable | YES    | Exp (1s)  | 3            | Auto retry
500 Internal Error      | YES    | Exp (1s)  | 3            | Show error UI
502 Bad Gateway         | YES    | Exp (1s)  | 3            | Auto retry
429 Rate Limited        | YES    | Linear    | 1            | Wait & retry
400 Bad Request         | NO     | N/A       | 0            | Fix client
401 Unauthorized        | NO     | N/A       | 0            | Re-auth
404 Not Found           | NO     | N/A       | 0            | Handle gracefully
413 Payload Too Large   | NO     | N/A       | 0            | Compress/resize
```

### Feed Retrieval Flow (Read Path)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        FEED RETRIEVAL FLOW                               │
└─────────────────────────────────────────────────────────────────────────┘

1. CLIENT: Request feed (GET /api/v1/feed)
   │
   ▼
2. CDN CHECK: CloudFront edge cache
   │   CACHE HIT: Return immediately (< 50ms)
   │   CACHE MISS: Forward to origin
   ▼
3. ELB: Route to app server
   │   Strategy: IP hash (sticky sessions for cache locality)
   ▼
4. APP SERVER: Check Memcached
   │   Key: "feed:user:{user_id}:page:{page}"
   │   TTL: 120 seconds
   │   
   │   CACHE HIT (80% of requests):
   │   └─ Return cached feed data
   │   
   │   CACHE MISS (20% of requests):
   │   └─ Proceed to fetch from database
   ▼
5. REDIS: Fetch Media IDs from user's feed list
   │   Key: "user_feed:{user_id}"
   │   Data structure: Sorted List (by timestamp)
   │   
   │   REDIS FAILOVER:
   │   ├─ Primary Redis down?
   │   │  └─ Automatic failover to replica (< 5 seconds)
   │   ├─ Entire Redis cluster down?
   │   │  └─ Fallback to PostgreSQL (slower but works)
   │   
   │   RESULT: List of 50 Media IDs
   ▼
6. MEMCACHED: Batch fetch photo metadata
   │   Keys: ["photo:{media_id_1}", "photo:{media_id_2}", ...]
   │   
   │   PARTIAL HIT:
   │   ├─ Cache hits: 45 photos (90%)
   │   └─ Cache misses: 5 photos (10%)
   ▼
7. POSTGRESQL: Fetch missing photo metadata
   │   Query: SELECT * FROM photos WHERE id IN (...)
   │   
   │   SHARD ROUTING:
   │   ├─ Photo ID contains shard information (13 bits)
   │   ├─ Query appropriate shard via Pgbouncer
   │   
   │   READ REPLICA STRATEGY:
   │   ├─ 90% of reads: Random replica
   │   ├─ 10% of reads: Primary (if replica lag detected)
   │   
   │   REPLICA FAILURE:
   │   ├─ Try next replica in pool
   │   ├─ If all replicas down, query primary
   │   ├─ Circuit breaker prevents cascade
   ▼
8. AGGREGATE & CACHE: Combine data
   │   - Merge cached + fetched metadata
   │   - Store in Memcached for next request
   │   - Update CDN if publicly cacheable
   ▼
9. RESPONSE: Return feed to client
   {
     "items": [...],
     "next_page_token": "...",
     "cache_timestamp": 1678234567890
   }

┌─────────────────────────────────────────────────────────────────────────┐
│                    DEGRADED MODE STRATEGIES                              │
└─────────────────────────────────────────────────────────────────────────┘

SCENARIO                      | DEGRADATION STRATEGY
─────────────────────────────────────────────────────────────────────────────
Database overload             | Serve shorter feeds (20 items instead of 50)
Memcached cluster down        | Direct DB queries, aggressive rate limiting
Redis cluster down            | Fallback to SQL joins (slower but functional)
S3 slow/unavailable           | Serve low-res cached thumbnails only
Multiple services degraded    | Static error page with retry button
Complete regional outage      | Failover to secondary region (manual process)
```

---

## Monitoring & Alerting Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       MONITORING ARCHITECTURE                            │
└─────────────────────────────────────────────────────────────────────────┘

METRICS COLLECTION (Munin / StatsD):
├─ System Metrics (per server)
│  ├─ CPU usage (alert if > 80% for 5 min)
│  ├─ Memory usage (alert if > 90%)
│  ├─ Disk I/O (alert if saturated)
│  └─ Network throughput
│
├─ Application Metrics
│  ├─ Requests per second
│  ├─ Response time (p50, p95, p99)
│  ├─ Error rate (alert if > 1%)
│  ├─ Photos uploaded per second
│  └─ Active users online
│
├─ Database Metrics
│  ├─ Query latency (alert if p95 > 100ms)
│  ├─ Connection pool usage (alert if > 80%)
│  ├─ Replication lag (alert if > 5 seconds)
│  ├─ Disk space (alert if < 20% free)
│  └─ Lock contention
│
└─ Cache Metrics
   ├─ Hit rate (alert if < 70%)
   ├─ Memory usage (alert if > 90%)
   ├─ Eviction rate
   └─ Connection errors

EXTERNAL MONITORING (Pingdom):
├─ Endpoint availability checks
├─ Global latency testing
└─ SSL certificate expiration

ALERTING (PagerDuty):
├─ Critical: Page on-call immediately
│  - Database primary down
│  - Error rate > 5%
│  - Site completely down
│
├─ Warning: Email/Slack notification
│  - Cache hit rate degraded
│  - Disk space < 30%
│  - Replica lag > 10s
│
└─ Info: Logged only
   - Autoscaling triggered
   - Deployment completed
   - Cache invalidation
```

---

## Key Takeaways: Instagram's Reliability Principles

### 1. **Stateless Application Servers**
   - Any server can handle any request
   - Enables horizontal scaling without coordination
   - Easy to add/remove servers during failures

### 2. **Multi-Layer Caching**
   - CDN for static content (images, videos)
   - Memcached for database query results
   - Redis for structured real-time data
   - Dramatically reduces database load (10x reduction)

### 3. **Database Sharding with Flexibility**
   - Logical shards mapped to physical servers
   - Easy rebalancing without resharding data
   - Custom ID system enables distribution

### 4. **Intelligent Retry Strategies**
   - Exponential backoff with jitter
   - Circuit breakers prevent cascade failures
   - Don't retry client errors (4xx)
   - Retry transient failures (5xx, timeouts)

### 5. **Graceful Degradation**
   - Serve shorter feeds under load
   - Lower-quality images when S3 slow
   - Read-only mode if write path fails
   - Better to be slow than down

### 6. **Automated Failover**
   - Load balancer health checks (3s interval)
   - Database replica promotion (30-60s)
   - Redis automatic failover (< 5s)
   - Connection draining for zero dropped requests

### 7. **Defense in Depth**
   - Multiple replicas per database shard
   - Multiple caching layers
   - Multiple availability zones
   - Async queues buffer spikes

### 8. **Observability First**
   - Comprehensive metrics collection
   - Real-time dashboards
   - Automated alerting with escalation
   - Performance profiling to find bottlenecks

---

## Scaling Evolution Timeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                INSTAGRAM'S INFRASTRUCTURE EVOLUTION                      │
└─────────────────────────────────────────────────────────────────────────┘

LAUNCH (Oct 2010) - 25K users on Day 1:
├─ 1 application server
├─ 1 PostgreSQL database  
├─ 1 Redis instance
└─ Immediate migration to AWS (server "melted")

3 MONTHS (Jan 2011) - 1M users:
├─ 3 application servers behind ELB
├─ 1 PostgreSQL database (separate machine)
├─ 3 Redis instances (sharded)
└─ Added Memcached (3 instances)

1 YEAR (Dec 2011) - 14M users, 3 engineers:
├─ 25 Django application servers
├─ 12 PostgreSQL servers (sharded by user ID)
├─ 6 Memcached instances
├─ Multiple Redis clusters
└─ Pgbouncer connection pooling deployed

2 YEARS (2012) - Acquired by Facebook for $1B:
├─ 100+ EC2 instances
├─ Thousands of logical shards
├─ Dozens of physical database servers
├─ Heavy use of CDN for media
└─ Still only ~6 backend engineers!

5 YEARS (2015) - 400M users:
├─ Migrated from AWS to Facebook data centers
├─ Introduction of Cassandra for some workloads
├─ TAO (Facebook's social graph cache)
└─ Microservices architecture begins

10 YEARS (2020) - 1B+ users:
├─ Multiple geographic regions
├─ Advanced ML-powered feed ranking
├─ Live video infrastructure
├─ Sophisticated content delivery network
└─ Hundreds of engineers

KEY INSIGHT:
Small team + Simple architecture + Proven technology + AWS
= Ability to scale 100x with minimal engineering overhead
```
