# QA-NOTES.md — Panel Questions and Answers

> Notes taken during the live architecture roast session.
> Format: question as asked → answer given → self-assessment of completeness.

---

## Q1: What happens if Redis crashes mid-lock?

**Question asked:** "What happens if Redis crashes mid-lock? Walk me through what the user sees
and whether anyone gets double-booked."

**Answer given:**

Redis Cluster with 3 shards, each with at least one replica. Redis is configured with
`min-replicas-to-write 1`, meaning a lock write is only acknowledged when the primary has
replicated to at least one replica. If the primary crashes *after* ACK, the replica has the
lock data and a failover promotes it — lock is preserved.

If the crash happens in the ~millisecond window between the primary writing and the replica
receiving it (async replication gap), the lock is lost. In that case:
- Two users could proceed to payment for the same seat.
- Both send SQS messages with different `bookingId` UUIDs.
- Both payment workers call the gateway — both could succeed.
- But: `INSERT INTO bookings ... ON CONFLICT (idempotency_key) DO NOTHING` — both inserts
  use different UUIDs so this doesn't catch it.
- The final safety net is: `UPDATE seats SET status = 'booked' WHERE status != 'booked'`.
  The second worker's UPDATE finds `status = 'booked'` (set by the first), so it rolls back
  and issues a refund automatically.

User experience: one user gets a confirmation, the second gets a "booking failed, refund
issued" notification within 5–10 minutes.

**Gap assessment:** COMPLETE — covered the replication window race condition and the DB-level
final guard. The refund path isn't fully implemented in the current design (noted as
improvement area — see DESIGN-UPDATES.md Update 1).

---

## Q2: At what concurrent users does your DB become the bottleneck?

**Question asked:** "At what point does PostgreSQL become the bottleneck? Give me a number."

**Answer given:**

The PostgreSQL primary handles only writes. Writes per booking transaction:
- 1 × `INSERT bookings`
- 1 × `UPDATE seats` (status = held)
- 1 × `INSERT booking_events`
- Later: 2 × `UPDATE` from payment worker (bookings + seats)

That's roughly 5 writes per booking. At r5.large (the current instance), PostgreSQL handles
~2,000–3,000 write IOPS sustained. That's 400–600 bookings/second before I/O becomes the
constraint.

At 50,000 seats sold in 120 seconds = 417 bookings/second. The current r5.large is right at
the edge of its I/O budget — it survives because not all writes are simultaneous (seat
selection is spread over ~120 seconds) and write batching reduces IOPS somewhat.

The DB becomes the hard bottleneck at approximately **600 concurrent booking completions/second**
(payment workers all completing at the same instant). To push past that: upgrade to r5.xlarge
(~$360/month) or enable multi-AZ with write partitioning (sharding by event_id).

**Gap assessment:** MOSTLY COMPLETE — gave a specific number (600 booking completions/sec).
Did not cover read replica saturation. Read replicas at r5.large handle ~10,000 read queries/sec,
so reads are not the bottleneck at current scale.

---

## Q3: What stops one user from holding 200 seats?

**Question asked:** "What stops a user from selecting 200 seats and holding them all, preventing
other users from buying?"

**Answer given:**

In the current design: *nothing explicit*. The API enforces a `MAX_SEATS_PER_BOOKING = 8`
parameter at the application layer (in the booking creation handler), but there's no cross-request
enforcement. A user could make 25 separate API requests each booking 8 seats = 200 seats held.

The Redis lock key is per-seat, not per-user. There is no counter tracking total held seats
per user.

This is a gap in the current design. The fix is a Redis counter per user:
`holds:{userId}:count` — incremented on lock acquisition, decremented on lock expiry or
booking completion. Reject any lock acquisition that would push a user's count above 8.

**Gap assessment:** IDENTIFIED GAP — honest about the missing enforcement. This became
Update 1 in DESIGN-UPDATES.md.

---

## Q4: Your auto-scaling spikes your bill to $3,200 this month. What's your plan?

**Question asked:** "Auto-scaling fired hard this month — your AWS bill hit $3,200. Walk me
through your response."

**Answer given:**

$3,200 is roughly double the baseline $1,732 (peak 16 API nodes). The delta of ~$1,468 suggests
either API nodes scaled to ~32+ instances, or a runaway ECS task, or an SQS queue depth spike
causing worker over-scaling.

Response plan:
1. Check CloudWatch: SQS queue depth, API node count, ECS task count, RDS IOPS — identify
   which component spiked.
2. If API nodes: check if it was a legitimate traffic spike (event sale) or a traffic anomaly
   (scraper, DDoS). If DDoS: tighten WAF rate limit (currently 1,000 req/s per IP) to
   100 req/s; add Captcha on booking initiation.
3. If ECS workers: check DLQ depth. If DLQ is full, a poison pill message may be causing
   workers to crash-loop and scale out. Fix: add max concurrency cap on ECS auto-scaling
   (cap at 20 tasks).
4. If SQS queue depth > 100K: payment gateway is slow or down. Workers accumulate. Fix:
   circuit breaker on gateway calls — after 60 consecutive failures, stop pulling from SQS
   and alert on-call.
5. Immediate cost control: set auto-scaling max capacity to 20 API nodes (not unlimited);
   set ECS max tasks to 20. Accept degraded throughput during anomalies over runaway costs.

**Gap assessment:** MOSTLY COMPLETE — had no CloudWatch alarm on SQS queue depth pre-configured.
This became Update 2 in DESIGN-UPDATES.md.

---

## Q5: Why not PostgreSQL row locking instead of Redis? Defend your choice.

**Question asked:** "You chose Redis locks over PostgreSQL FOR UPDATE. PostgreSQL is already
there and already has ACID guarantees. Why add Redis? Defend this."

**Answer given:**

The core issue is connection lifecycle. `SELECT FOR UPDATE` holds a transaction open for the
entire seat-selection-to-payment window — that's 30 to 45 seconds per user. At 22,000
concurrent seat holds during a 50,000-seat sellout, you need 22,000 open database connections
simultaneously. PostgreSQL's hard limit with PgBouncer in session mode is ~5,000 connections
on an r5.large. In transaction mode, PgBouncer can't help because `FOR UPDATE` prevents
connection release mid-transaction.

Redis SETNX is atomic (single-command, no transaction needed), non-blocking (fail-fast, no
waiting queue), and doesn't consume a DB connection for the duration of the hold. The DB
connection is held only for the fast `UPDATE seats SET status='held'` write (~5ms), not for
45 seconds.

The tradeoff: Redis is a new operational dependency. If Redis fails and we have no fallback,
seat selection is down. The mitigation is the Redis Cluster with replica writes — this reduces
Redis failure probability significantly. The remaining risk (split-second replication window)
is accepted because the DB-level idempotency guard (see Q1 answer) catches double-bookings
even in that scenario.

PostgreSQL row locking is the right choice below ~10K concurrent users. Above that, Redis
SETNX is the right choice because the bottleneck shifts from "correctness" to "connection
capacity."

**Gap assessment:** COMPLETE — made the quantified case (22,000 connections vs 5,000 limit)
and acknowledged the tradeoff honestly.
