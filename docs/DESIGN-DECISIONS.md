# DESIGN-DECISIONS.md — Justified Architectural Choices

This document records every non-obvious architectural decision in the BookMyShow system design,
with full context, options considered, and quantified justification. Each decision includes
explicit tradeoffs and the conditions under which the decision should be revisited.

---

## Decision: Concurrency Strategy — Redis SETNX vs PostgreSQL FOR UPDATE

**Context:**
During a flash-sale event (IPL final, blockbuster opening), 500,000 concurrent users attempt
to select seats within the first 60–90 seconds. Two users must never receive the same seat.
The system needs a serialisation mechanism that prevents double-booking without becoming a
throughput bottleneck.

**Options considered:**

1. **PostgreSQL `SELECT ... FOR UPDATE` (row-level lock)**
   - Why considered: native to the DB, ACID guarantees, no external dependency, familiar pattern.
   - Why not chosen: A `FOR UPDATE` lock is held for the entire payment duration (~30–45 seconds).
     At 22,000 concurrent seat holds (50,000 seats selling in 2 minutes), the system requires
     22,000 simultaneous DB connections. PostgreSQL's effective connection limit with PgBouncer
     in transaction mode is ~1,000–2,000 — but `FOR UPDATE` cannot release the connection
     mid-transaction, so transaction-mode pooling doesn't help here. Connection pool exhaustion
     occurs at ~200–400 concurrent holds. Additionally, contention on popular seats (front row,
     VIP) creates a cascade of waiting transactions that degrades the entire DB, not just the
     hot seat.

2. **Redis SETNX distributed lock** ← chosen
   - Why considered: Redis is already in the architecture for caching. SETNX provides atomic
     test-and-set with no blocking semantics — a lock attempt either succeeds or fails
     immediately, with no waiting queue on the caller.
   - Why chosen: see below.

**Why chosen:**
Redis SETNX (`SET key value NX EX 600`) is atomic and non-blocking. The DB sees only the fast
write (`UPDATE seats SET status='held'`) after the lock is acquired — no connection is pinned
during payment. At 500K RPS across 50,000 seats, each seat's lock key is hit ~10 times/second
on average, well within Redis's 1M+ operations/second per shard. Redis Cluster shards lock keys
by `{seatId}`, so lock contention for one seat never affects another. Cost: Redis cluster
(3×r6g.large) = ~$360/month vs. the RDS primary upgrade required for `FOR UPDATE` at scale
(r5.4xlarge = ~$800+/month), plus RDS cannot horizontally scale write connections.

**Tradeoffs accepted:**
- Redis is now a hard dependency — a Redis outage means seat selection is unavailable.
- The lock and the DB write are not in the same transaction. A crash between lock acquisition
  and DB write leaves a ghost lock (expires after 600s) but no DB record — this is acceptable
  as the seat simply stays unavailable for up to 10 minutes then auto-releases.
- Cannot use DB-level cascade semantics to tie seat hold to booking status atomically.

**Revision trigger:**
- Redis cluster uptime drops below 99.9% over 90 days.
- Event scale drops permanently below 10K concurrent users (at which point `FOR UPDATE` with
  connection pooling is viable and removes the Redis dependency).
- PostgreSQL advisory lock + async payment pattern matures enough to match Redis performance.

---

## Decision: Cache Invalidation — TTL-Only vs Event-Driven Invalidation

**Context:**
The seat availability map is the most-read, fastest-changing data in the system. During a sale,
seats transition from `available` → `held` → `booked` continuously. Users need a reasonably
current view of the seat map to make selections, but the Redis seat lock (not the cached seat
map) is the correctness layer. The cache is for display only.

Event detail pages (name, description, lineup) change rarely — maybe once before an event,
never during active sale.

**Options considered:**

1. **TTL-only invalidation**
   - Why considered: zero code complexity. Every cache key has a TTL; on expiry, the next
     request repopulates from the DB. No invalidation logic, no pub/sub, no race conditions
     in invalidation paths.
   - Why not chosen (for seat maps): a 30-second TTL on seat maps means a user could see 10
     seats as "available" when all 10 are held. They select one, hit "confirm", and receive
     a 409 (already locked). This false-availability UX drives abandonment and support tickets.
   - Why chosen (for event details): event descriptions, lineups, and venue info change at most
     once during the pre-sale period. 5-minute staleness is entirely acceptable and saves
     engineering complexity in the CMS update path.

2. **Event-driven invalidation (explicit DEL on state change)** ← chosen for seat maps
   - Why considered: when a seat is booked or held, explicitly delete the seat map cache key.
     The next read repopulates with fresh data.
   - Why chosen: one `redis.del('event:{eventId}:seat_map')` call in the payment worker reduces
     the false-availability window from up to 30 seconds to near-zero. Implementation cost is
     a single line of code. The seat map is fetched at most once per 30 seconds per event
     anyway (floor TTL still applies), so cache stampede risk is low.

**Why chosen:**
Hybrid approach: TTL-only for slowly-changing data (event details, venue layouts), event-driven
DEL for high-churn data (seat maps). This minimises engineering complexity while targeting
invalidation effort at the data type where staleness actually harms UX.

**Tradeoffs accepted:**
- If the payment worker fails to publish the `DEL` (crash before invalidation step), the seat
  map stays stale until the 30s TTL expires. Acceptable — the Redis lock still prevents
  double-booking.
- Requires careful ordering in the payment worker: DB write → SNS → `DEL` cache → SQS delete.
  If `DEL` happens before DB write completes, the repopulated cache may still show the old state.
- Cannot easily track which events' caches need invalidation if a bulk operation changes many
  seats (e.g., venue layout revision) — manual cache flush required for bulk admin operations.

**Revision trigger:**
- If seat map cache inconsistency causes booking failure rate > 2%, reduce seat map TTL to 10s
  or implement WebSocket push for real-time seat state updates.
- If invalidation fan-out becomes complex (multiple workers invalidating the same key
  simultaneously), introduce a dedicated invalidation queue to serialise DEL operations.

---

## Decision: UUID vs SERIAL for booking_id

**Context:**
Each booking record needs a primary key. The key is used as: (a) the DB primary key,
(b) the SQS message identifier, (c) the payment gateway idempotency key, and (d) the
booking reference shown to users. The system uses an async payment queue where the same
SQS message may be processed more than once (at-least-once delivery).

**Options considered:**

1. **SERIAL / BIGSERIAL (auto-increment integer)**
   - Why considered: smaller storage (8 bytes vs 16 bytes), faster B-tree index on sequential
     inserts, trivially human-readable (booking #1000042).
   - Why not chosen:
     - A SERIAL requires a DB round-trip to obtain the next sequence value before the insert.
       In the async flow, the API server needs the `bookingId` *before* the DB insert to put
       it in the SQS message. A SERIAL forces a `SELECT nextval()` before the message is queued
       — adding a DB call in the hot path.
     - SERIAL values leak business intelligence: `booking_id = 1000042` tells a competitor
       or adversary exactly how many bookings have been made. For a publicly-visible booking
       reference, this is a security and competitive disclosure risk.
     - SQS deduplication and payment gateway idempotency keys work best with globally-unique,
       unpredictable identifiers. A SERIAL is predictable; an attacker could probe adjacent
       booking IDs.

2. **UUID v4** ← chosen
   - Why considered: UUIDs are globally unique, generated without DB coordination, unpredictable,
     and universally supported as idempotency keys by payment gateways.
   - Why chosen: see below.

**Why chosen:**
UUID v4 is generated in the API server in memory — no DB round-trip needed before the SQS
message is published. The same UUID serves simultaneously as: DB primary key, SQS message ID
(for deduplication), and payment gateway idempotency key. This unification prevents a class
of bugs where different IDs for the same booking cause deduplication to fail. The `ON CONFLICT
DO NOTHING` pattern on `idempotency_key` (same as `booking_id`) provides a hard guarantee:
if two payment worker tasks process the same SQS message, only one DB write succeeds. Index
size overhead: 16 bytes vs 8 bytes per row — at 50,000 bookings/month, this is ~400KB/month
of additional index space, negligible.

**Tradeoffs accepted:**
- UUID v4 is random, so B-tree index insert performance degrades slightly vs sequential SERIAL
  (random page splits). Mitigation: use ULIDv7 (time-ordered UUID) if index bloat becomes
  measurable at > 100M rows. At current scale (< 1M rows/year), this is not an issue.
- Human-facing booking reference (shown in emails/SMS) is a truncated display ID derived from
  the UUID (e.g., `BMS-A3F9C1`) — the full UUID is internal only.

**Revision trigger:**
- If PostgreSQL index bloat on `bookings.booking_id` exceeds 20% of table size — migrate
  to ULID (time-ordered, 128-bit, lexicographically sortable).
- If payment gateway idempotency key format restricts to numeric-only — generate a separate
  numeric idempotency sequence alongside the UUID.

---

## Decision: SQS Visibility Timeout Value — 300 Seconds

**Context:**
SQS visibility timeout controls how long a message is hidden from other consumers after it
is received by a worker. If the worker crashes or fails to delete the message within this
window, SQS makes the message visible again for another worker to retry.

The timeout must be:
- Long enough that a slow-but-successful payment processing run completes before the message
  reappears (false retry → double-charge risk).
- Short enough that a genuinely failed payment (worker crashed, gateway down) retries promptly
  so the user's seat hold doesn't expire before they hear back.

**Options considered:**

1. **30 seconds**
   - Why considered: fast retry on failure; user hears back quickly.
   - Why not chosen: gateway worst-case latency is up to 30 seconds. ECS task restart after
     a crash takes ~60–90 seconds. A 30-second timeout almost guarantees the message reappears
     while a replacement worker is still starting up — every payment would retry once under
     normal conditions, creating unnecessary double-processing attempts.

2. **3600 seconds (1 hour)**
   - Why considered: maximum safety margin against double-charge.
   - Why not chosen: if a worker crashes mid-payment, the user waits up to 1 hour to find out
     their booking failed. Their Redis seat lock expires after 10 minutes (600s), the seat
     is released, someone else books it, then the delayed retry re-attempts payment for a seat
     that is now booked by another user — causing a logic error.

3. **300 seconds (5 minutes)** ← chosen
   - Breakdown of the decision:
     - Average gateway round-trip: 3–8 seconds
     - Worst-case gateway timeout: 30 seconds
     - ECS task restart time after crash: ~90 seconds
     - Retry backoff buffer (exponential backoff on transient errors): 60 seconds
     - Safety margin: 120 seconds
     - Total: 300 seconds (5 minutes)
   - 300 seconds is safely inside the 600-second Redis lock TTL, so if a worker crashes at
     second 1 of processing, the retry at second 301 still has 299 seconds of lock remaining.

**Why chosen:**
300 seconds is the minimum timeout that guarantees a replacement worker can start and complete
payment processing before SQS redelivers. It is comfortably inside the 600-second seat lock
window. The gap between the seat lock TTL (600s) and the visibility timeout (300s) gives exactly
one full retry window before the lock expires.

**Tradeoffs accepted:**
- A user whose payment worker crashes at second 1 waits up to 5 minutes before retry begins,
  plus payment processing time. Total wait: up to ~308 seconds before confirmation or failure
  notification. This is communicated in the UI ("Your booking is processing — check back in
  a few minutes").
- After 3 failures, the message moves to the DLQ and requires manual intervention. A booking
  stuck in DLQ = seat held until Redis TTL expires (10 min), then seat released and booking
  expires. User's card is never charged (no gateway confirmation stored in DB).

**Revision trigger:**
- If payment gateway P99 latency exceeds 180 seconds (meaning 300s timeout is no longer safe) —
  extend to 600 seconds and reduce seat lock TTL to 15 minutes correspondingly.
- If ECS task cold-start time exceeds 120 seconds (new instance types, larger images) —
  recalculate the timeout floor accordingly.
- If users report excessive wait times correlating with worker restarts — instrument visibility
  timeout utilisation in CloudWatch and tune based on empirical P99 processing time.
