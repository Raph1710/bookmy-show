# CACHING.md — Caching Strategy

## Context

BookMyShow serves two very different read patterns that require different caching approaches:

1. **Event/venue pages** — semi-static content (event name, description, lineup, venue map)
   that changes rarely but is read millions of times. Cache-friendly.

2. **Seat availability** — changes every time a seat is held or booked. Must be reasonably
   fresh but doesn't need to be real-time (Redis lock is the correctness layer; the cache
   is the display layer).

Peak read load: **500,000 RPS** across all event pages during a flash sale announcement.
Without caching, this would require ~250 PostgreSQL read instances (assuming 2,000 RPS/instance).

---

## What Gets Cached

### Layer 1: CloudFront CDN

**Cached:** Static assets (JS, CSS, images, fonts), rendered event landing pages, venue seating
chart SVGs.

**Cache-Control headers:**
```
Static assets:   Cache-Control: public, max-age=31536000, immutable  (1 year, content-hashed filenames)
Event pages:     Cache-Control: public, max-age=300, stale-while-revalidate=60  (5-minute TTL)
Seat map SVG:    Cache-Control: public, max-age=3600  (1 hour — venue layout rarely changes)
```

**What CloudFront does NOT cache:**
- `/api/*` routes (dynamic, user-specific or seat-state-dependent)
- `/bookings/*` (user-specific)
- Anything with `Authorization` header (CloudFront default: don't cache authenticated requests)

### Layer 2: Redis Application Cache

**Cached objects and TTLs:**

| Cache Key Pattern | Content | TTL | Invalidation |
|---|---|---|---|
| `event:{eventId}:detail` | Event name, dates, venue, description | 300s | On event update |
| `event:{eventId}:seat_map` | Full seat availability grid | 30s | Scheduled + on booking |
| `venue:{venueId}:layout` | Venue seating chart data | 3600s | On venue edit (rare) |
| `user:{userId}:bookings` | User's booking list | 60s | On new booking |

**What Redis does NOT cache:**
- Individual seat status (too fine-grained; Redis locks already manage this state)
- Payment state (must always hit DB for correctness)

---

## Cache Invalidation Approach: TTL-Only vs Event-Driven

### Option 1: TTL-Only Invalidation

Every cache entry has a TTL. On expiry, the next request fetches from DB and re-populates.

**Why considered:** Zero complexity — no invalidation logic, no pub/sub, no cache-busting code.
Cache eventually converges to fresh data.

**Why partially chosen (for most data):** For event details and venue layouts, the data changes
rarely. A 5-minute stale window is acceptable — if an event description is updated, users will
see the change within 5 minutes with zero engineering effort.

**Limitation:** For seat availability, pure TTL means up to 30 seconds of stale seat counts
displayed. During a hot sale, a seat can go from available to held to released to held again
within 30 seconds. The displayed count may be misleading but the **Redis lock is always
checked on actual selection** — the cache is only for display, never for booking correctness.

### Option 2: Event-Driven Invalidation (Write-Through / Pub-Sub) ✅ CHOSEN for seat maps

When a seat is booked or held, explicitly invalidate `event:{eventId}:seat_map` in Redis.

```javascript
// In the payment worker, after confirming a booking:
await redis.del(`event:${eventId}:seat_map`);
// Next request will re-fetch from DB read replica and re-cache
```

**Why chosen for seat maps specifically:**
- Seat maps are the highest-value display for users during a sale — showing 10 seats as
  "available" when they're all held causes user frustration (they select, hit "already taken")
- Event-driven invalidation reduces this false-availability window from 30s → near-zero
- Implementation cost is low: one `DEL` call in the payment worker
- The seat map is fetched at most once per 30s per event anyway (even without TTL floor)

**Why NOT used for event details:**
- Event details rarely change during active sales
- Invalidation logic adds complexity to the admin CMS update flow
- 5-minute staleness is acceptable for text content

---

## Cache-Aside Pattern (Application Layer)

All Redis application cache follows cache-aside:

```javascript
async function getEventDetail(eventId) {
  const cached = await redis.get(`event:${eventId}:detail`);
  if (cached) return JSON.parse(cached);          // Cache hit

  const data = await db.readReplica.query(        // Cache miss → read replica
    'SELECT * FROM events WHERE event_id = $1',
    [eventId]
  );
  await redis.setex(                               // Populate cache
    `event:${eventId}:detail`,
    300,                                           // 300s TTL
    JSON.stringify(data)
  );
  return data;
}
```

On a cache miss:
1. Query goes to the PostgreSQL read replica (not the primary)
2. Result is stored in Redis with TTL
3. Concurrent cache misses (thundering herd): mitigated by a short mutex lock in Redis
   (`SET event:{eventId}:fetching NX EX 2`) so only one request fetches from DB

---

## Thundering Herd Protection

When an event goes on sale, thousands of users hit the event page simultaneously. If the cache
is cold (first hit after TTL expiry), all requests miss simultaneously and hammer the DB.

**Solution: Fetch lock + stale-while-revalidate**

```javascript
async function getWithStaleProtection(key, ttl, fetcher) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  // Try to acquire fetch lock
  const lockAcquired = await redis.set(`${key}:lock`, '1', 'NX', 'EX', 2);
  if (!lockAcquired) {
    // Another request is fetching — wait briefly and retry from cache
    await sleep(100);
    return getWithStaleProtection(key, ttl, fetcher);
  }

  const fresh = await fetcher();
  await redis.setex(key, ttl, JSON.stringify(fresh));
  await redis.del(`${key}:lock`);
  return fresh;
}
```

This ensures a single DB query per cache miss, regardless of concurrent requests.

---

## CloudFront Cache Behaviour

```
Request flow:
User → CloudFront Edge (cache hit: 200 from edge, ~10ms)
                      ↓ (cache miss)
                      ALB → API Server → Redis (hit: ~5ms)
                                       ↓ (Redis miss)
                                       PostgreSQL Read Replica (~20ms)
```

CloudFront origins:
- S3 bucket for static assets (hashed filenames, 1-year TTL, immutable)
- ALB for API and dynamic content (short TTL or no-cache)

Cache-hit ratio target: **> 85%** for event pages during peak sale. This reduces origin load
by 6.7× at 500K RPS → ~75K RPS reaching the ALB.

---

## Tradeoffs Accepted

| What we gain | What we give up |
|---|---|
| 85%+ CDN offload reduces origin cost | Stale event details for up to 5 minutes |
| Seat map invalidated on booking | Invalidation bug = stale seat display |
| Thundering herd protection | Extra Redis call per request (fetch lock check) |
| Sub-10ms response from CDN edge | Cache warming needed before flash sale |

---

## Revision Trigger

- If seat map staleness causes measurable booking failure rate > 2% — reduce seat map TTL to 10s
  or implement WebSocket push for real-time seat state
- If Redis invalidation fan-out becomes complex (multiple workers, race conditions) — adopt a
  dedicated cache invalidation service (e.g., Momento or a custom invalidation queue)
- If CDN cache-hit ratio drops below 70% — investigate Cache-Control header misconfiguration or
  add more CloudFront behaviours
