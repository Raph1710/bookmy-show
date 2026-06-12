# SCHEMA.md — Database Schema Design

## Context

The schema must support:
- **High-read, write-isolated seat availability queries** — users constantly checking which
  seats are available during a sale
- **Idempotent booking writes** — payment confirmation must be safe to retry
- **Audit trail** — every state change on a booking must be traceable
- **Horizontal read scaling** — read replicas must handle seat queries without blocking writes

---

## Core Tables

### events

```sql
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    venue_id        UUID NOT NULL REFERENCES venues(venue_id),
    name            VARCHAR(255) NOT NULL,
    event_datetime  TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'upcoming',  -- upcoming, on_sale, sold_out, cancelled
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### seats

```sql
CREATE TABLE seats (
    seat_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(event_id),
    section_id      VARCHAR(10) NOT NULL,   -- e.g. 'A1', 'GA', 'VIP'
    row_number      VARCHAR(5),
    seat_number     VARCHAR(5),
    price_tier      VARCHAR(20) NOT NULL,
    base_price      NUMERIC(10,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    held_until      TIMESTAMPTZ,            -- populated when status = 'held'
    version         INTEGER NOT NULL DEFAULT 0,  -- optimistic lock version
    CONSTRAINT seats_status_check CHECK (status IN ('available', 'held', 'booked', 'blocked'))
);

CREATE INDEX idx_seats_event_status ON seats(event_id, status);
CREATE INDEX idx_seats_held_until ON seats(held_until) WHERE status = 'held';
```

**Why `held_until` on the row?**
When a Redis seat lock expires (user abandoned), a background job (pg_cron, runs every 60s)
runs:
```sql
UPDATE seats SET status = 'available', held_until = NULL
WHERE status = 'held' AND held_until < NOW();
```
This ensures DB state converges with Redis state without relying on Redis as the source of truth
for inventory.

### bookings

```sql
CREATE TABLE bookings (
    booking_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id),
    event_id        UUID NOT NULL REFERENCES events(event_id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount    NUMERIC(10,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'INR',
    payment_ref     VARCHAR(100),           -- gateway transaction ID
    payment_method  VARCHAR(50),
    idempotency_key UUID NOT NULL UNIQUE,   -- same as booking_id, enforces no double-process
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    confirmed_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ NOT NULL,   -- booking_id held for max 10 min
    CONSTRAINT bookings_status_check CHECK (
        status IN ('pending', 'confirmed', 'failed', 'refunded', 'expired')
    )
);

CREATE INDEX idx_bookings_user_status ON bookings(user_id, status);
CREATE INDEX idx_bookings_expires ON bookings(expires_at) WHERE status = 'pending';
```

### booking_seats (join table)

```sql
CREATE TABLE booking_seats (
    booking_id  UUID NOT NULL REFERENCES bookings(booking_id),
    seat_id     UUID NOT NULL REFERENCES seats(seat_id),
    price_paid  NUMERIC(10,2) NOT NULL,
    PRIMARY KEY (booking_id, seat_id)
);
```

### booking_events (audit log)

```sql
CREATE TABLE booking_events (
    event_log_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES bookings(booking_id),
    event_type      VARCHAR(50) NOT NULL,   -- 'created', 'payment_attempted', 'confirmed', 'failed', 'refunded'
    actor           VARCHAR(50) NOT NULL,   -- 'user', 'payment_worker', 'system'
    payload         JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_booking_events_booking ON booking_events(booking_id, occurred_at);
```

This append-only log never updates — every state transition appends a row. This supports
debugging, dispute resolution, and regulatory audit requirements.

---

## UUID vs SERIAL for booking_id

**Decision: UUID ✅**

See DESIGN-DECISIONS.md for full analysis. Summary:
- UUIDs are safe to generate client-side or in the API layer before DB insert — no round-trip
  to get a sequence value
- UUIDs serve as SQS message IDs and payment gateway idempotency keys simultaneously
- No information leakage: a SERIAL `booking_id = 1000042` tells competitors your booking volume
- The idempotency guarantee (`UNIQUE` constraint on `idempotency_key`) works naturally with UUID
  — two workers processing the same SQS message will try to insert the same UUID and one will
  get a unique violation, preventing double-charge

**Cost of UUID:** Slightly larger index size (16 bytes vs 8 bytes for BIGINT). At 50,000
bookings/month, this is negligible (~800KB additional index space/month).

---

## Read vs Write Routing

### Writes → PostgreSQL Primary only

- `INSERT INTO bookings` (booking creation)
- `UPDATE seats SET status = 'held'` (seat hold on lock acquire)
- `UPDATE bookings SET status = 'confirmed'` (payment confirmation)
- `UPDATE seats SET status = 'booked'` (post-confirmation)
- Audit log inserts

### Reads → PostgreSQL Read Replicas

- `SELECT seats WHERE event_id = ? AND status = 'available'` — seat map rendering
- `SELECT bookings WHERE user_id = ?` — user booking history
- `SELECT events WHERE event_datetime > NOW()` — event listings
- Event page detail reads

**Why this split matters:** During a sale, seat availability reads can be 1,000x more frequent
than writes. Routing all reads to replicas keeps the primary's I/O budget reserved for write
transactions. The primary never competes with read load.

**Replication lag tolerance:** For seat availability, ~100ms replica lag is acceptable — the
Redis lock is the true source of "is this seat takeable right now", not the DB read. The DB read
is for display purposes; the Redis `SET NX` is for correctness.

---

## Seat Hold Expiry Flow

```
1. User selects seat
2. API: SET seat_lock:{eventId}:{seatId} {userId}:{bookingId} NX EX 600
   → If NX fails: seat is locked by another user → return 409
   → If NX succeeds: proceed
3. API: UPDATE seats SET status='held', held_until=NOW()+600s WHERE seat_id=?
4. API: INSERT INTO bookings (status='pending', expires_at=NOW()+600s)
5. API: SQS publish → return 202

On Redis TTL expiry (600s):
6. pg_cron job: UPDATE seats SET status='available' WHERE held_until < NOW() AND status='held'
7. Booking status updated to 'expired' by same job

On successful payment (worker):
8. UPDATE bookings SET status='confirmed', payment_ref=?
9. UPDATE seats SET status='booked'
10. Redis lock is explicitly deleted (belt-and-suspenders)
```

---

## Tradeoffs Accepted

| What we gain | What we give up |
|---|---|
| Read/write separation scales horizontally | Replica lag means seat map may show stale data |
| UUID idempotency prevents double-booking | Larger index footprint than SERIAL |
| Append-only audit log for disputes | More storage growth than in-place updates |
| Automatic seat expiry via pg_cron | pg_cron is an operational dependency |

---

## Revision Trigger

- If UUID index bloat becomes measurable (> 5% query slowdown) — consider ULID (time-ordered UUID)
- If pg_cron misses expiry windows at scale — replace with a dedicated Redis keyspace notification consumer
- If read replica lag exceeds 500ms under load — add a third replica or upgrade instance class
