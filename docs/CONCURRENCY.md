# CONCURRENCY.md — Seat Locking Strategy

## Context

BookMyShow handles flash-sale events (IPL finals, blockbuster releases) where hundreds of thousands
of users attempt to book the same seats within seconds of ticket opening. The core problem:
**two users must never be assigned the same seat**.

### Capacity numbers driving this decision

- Peak concurrent users: **500,000 RPS** (observed during IPL ticket launches)
- Typical event: 50,000-seat stadium, selling out in < 2 minutes
- Average booking flow: seat selection → payment → confirmation = ~45 seconds total
- This means at peak: ~22,000 seats in-flight simultaneously across the payment window

---

## The Problem: Double-Booking

A seat goes through three states:

```
AVAILABLE → HELD (user selecting/paying) → BOOKED (payment confirmed)
```

Without a lock, two users can both read `AVAILABLE`, both proceed to payment, and both get
confirmed — resulting in a double-booking. This is a **correctness** problem, not a performance
problem. The fix must be correct first, fast second.

---

## Option 1: PostgreSQL `SELECT ... FOR UPDATE` (row-level locking)

```sql
BEGIN;
SELECT id FROM seats WHERE id = $1 AND status = 'available' FOR UPDATE;
-- If row returned, proceed to payment
UPDATE seats SET status = 'held', user_id = $2, held_until = NOW() + INTERVAL '10 min' WHERE id = $1;
COMMIT;
```

**Why considered:** Native to PostgreSQL — no extra infrastructure. Guarantees ACID correctness.
Familiar to every backend developer.

**Why not chosen:**

1. **Connection pinning**: A `FOR UPDATE` transaction holds a DB connection open for the entire
   duration the user is on the payment page (~30–45 seconds). At 22,000 concurrent holds,
   you need 22,000 simultaneous DB connections. PostgreSQL default max_connections = 200.
   Even with PgBouncer at transaction-mode pooling, a long-running `FOR UPDATE` transaction
   cannot release the connection mid-transaction.

2. **Lock contention cascade**: At 500K RPS, popular seats (front-row, centre blocks) get
   hammered. Every failed `FOR UPDATE` attempt waits, consuming a connection slot and CPU
   on the DB primary. A single hot seat can queue thousands of waiting transactions.

3. **DB becomes the single chokepoint**: This approach makes the PostgreSQL primary the
   serialisation point for all seat selection globally. The primary handles writes only — we
   can't read-replica this away because the lock lives on the primary.

**Quantified bottleneck:** At 394 RPS sustained payment throughput (derived from QUEUE.md),
even at 10% of peak (50K RPS), the connection pool exhausts in seconds.

---

## Option 2: Redis SETNX Distributed Lock ✅ CHOSEN

```
SETNX seat_lock:{seatId} {userId}:{bookingId}:{timestamp}
EXPIRE seat_lock:{seatId} 600   -- 10-minute TTL
```

Or atomically with `SET ... NX EX`:

```
SET seat_lock:{seatId} "{userId}:{bookingId}" NX EX 600
```

**Why chosen:**

1. **Non-blocking at DB layer**: The lock lives in Redis, not PostgreSQL. The database sees only
   the eventual `UPDATE seats SET status = 'booked'` — a fast, uncontended write. No connection
   pinning.

2. **Sub-millisecond lock acquisition**: Redis single-threaded command processing means `SET NX`
   is atomic without coordination overhead. A lock attempt either succeeds instantly or fails
   instantly — no waiting queue.

3. **Horizontal isolation**: Redis Cluster shards lock keys by `{seatId}`. Lock contention for
   seat A never affects seat B. At 500K RPS across 50,000 seats, each seat's lock is hit an
   average of 10 times/second — well within Redis's 1M+ ops/sec capacity per shard.

4. **Automatic TTL expiry**: If the user abandons mid-payment, the lock expires automatically
   after 10 minutes. No background cleanup job needed. DB row returns to `available` after
   a separate expiry handler (see SCHEMA.md).

5. **Budget**: A Redis cluster (3 shards × r6g.large) costs ~$360/month vs. the RDS instance
   upgrade that `FOR UPDATE` at scale would require (~$800+/month for r5.4xlarge primary).

**Lock key format:**
```
seat_lock:{eventId}:{sectionId}:{seatId}
Value: {userId}:{bookingId}:{iso8601_timestamp}
TTL: 600 seconds (10 minutes)
```

The composite key prevents cross-event collisions for recurring venues.

---

## Tradeoffs Accepted

| What we gain | What we give up |
|---|---|
| No DB connection pinning | Redis is now a critical dependency |
| Sub-ms lock check | Lock loss on Redis node failure (see below) |
| Auto-expiry on abandon | Slightly more complex lock lifecycle code |
| Scales to 500K RPS | Can't use DB transactions to wrap lock + update atomically |

### Failure Mode: Redis Node Crashes Mid-Lock

If the Redis primary holding a seat lock crashes before the TTL is persisted to the replica,
the lock is lost. The seat could be double-booked.

**Mitigation:**
- Redis Cluster with `min-replicas-to-write 1` — write is only acknowledged when at least one
  replica confirms. This halves the failure window.
- Payment worker does a final idempotency check: `INSERT INTO bookings ... ON CONFLICT DO NOTHING`.
  Even if two payment flows proceed, only one DB write succeeds. The second gets a conflict error
  and refunds automatically.
- Monitoring: CloudWatch alarm on Redis `KeysExpired` spike (indicates TTL flood, possible lock loss).

---

## Revision Trigger

Reconsider this design if:
- Redis Cluster uptime drops below 99.9% over a 90-day window
- The Redis infrastructure cost exceeds the RDS upgrade cost at steady load
- PostgreSQL advisory locks with connection pooling reach maturity for long-hold scenarios
- Event scale drops permanently below 10K concurrent users (then `FOR UPDATE` becomes viable)
