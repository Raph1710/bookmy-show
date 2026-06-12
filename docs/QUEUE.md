# QUEUE.md — Async Payment Queue Design

## Context

At peak load, BookMyShow processes payments for a 50,000-seat event that sells out in under
2 minutes. Users complete seat selection and must reach a payment gateway within a short window.

### The Synchronous Payment Problem

If the API server calls the payment gateway **synchronously** (inline with the HTTP request):

```
User → API Server → Payment Gateway → DB → Response
```

The API server thread is blocked for the entire payment round-trip (~2–8 seconds per gateway call).

#### Capacity math

- Peak seat selections in 2-minute sellout: 50,000 bookings
- Average successful payment time: ~3 seconds (gateway round-trip)
- Concurrent payment threads needed: 50,000 / 120 seconds × 3 seconds = **1,250 concurrent threads**
- Node.js default thread pool: 4 threads (libuv)
- Even with async I/O, 1,250 simultaneous open gateway connections per Node instance = memory
  exhaustion at ~150–200 connections in practice

At 500,000 RPS (5L users), the numbers are catastrophic:
- 500,000 / 120 × 3 = **12,500 concurrent gateway connections**
- This would require ~63 Node.js API servers at 200 connections/server
- Cost: ~$6,200/month just for API layer
- Single payment gateway outage takes down entire booking flow

**The result: synchronous payment at 5L users causes pool exhaustion at 394 sustained RPS.**

394 RPS = 50,000 seats / 127 seconds (time to exhaust 1,250 threads at 3s avg) — this is the
throughput wall where the system collapses under synchronous load.

---

## Option 1: Synchronous Payment (Inline)

User waits on HTTP connection while API calls payment gateway.

**Why considered:** Simple implementation, immediate confirmation to user, no queue infrastructure.

**Why not chosen:**
- Thread/connection exhaustion at scale (quantified above)
- Single gateway timeout cascades to all users
- No retry mechanism — gateway failures = lost bookings
- Horizontal scaling of API servers costs more than queue infrastructure
- P99 latency explodes: gateway hiccup → all users wait

---

## Option 2: SQS-Backed Async Payment Queue ✅ CHOSEN

```
API Server → SQS Queue → Payment Worker (ECS) → Payment Gateway → DB → SNS → User Email/SMS
```

The API server does three fast operations and returns immediately:
1. Acquire Redis lock on seat (< 1ms)
2. Write `PENDING` booking record to DB (< 5ms)
3. Publish message to SQS (< 10ms)
4. Return `202 Accepted` with `bookingId` to user

The user polls `/bookings/{bookingId}/status` or receives a push notification when done.

**Why chosen:**

### 1. Decouples booking acceptance from payment processing

The API layer handles only fast operations. At 500K RPS, 16 Node.js servers (t3.medium, ~30K
RPS each) can handle queue ingestion at < $480/month. Payment workers scale independently.

### 2. Handles gateway variability

Payment gateways have variable latency (2–15 seconds). Workers pull from SQS at their own pace.
A slow gateway day doesn't affect booking acceptance rate.

### 3. Built-in retry with dead-letter queue

SQS visibility timeout = **300 seconds** (5 minutes). If a worker crashes mid-payment, the
message becomes visible again after 5 minutes and a new worker retries. After 3 failed attempts,
the message moves to the **Dead Letter Queue (DLQ)** for manual inspection.

Why 300 seconds?
- Average payment processing: 3–8 seconds
- Worst-case gateway timeout: 30 seconds
- Worker instance replacement time: ~90 seconds (ECS task restart)
- Buffer for network retry backoff: 60 seconds
- Total: 180 seconds minimum → 300 seconds gives comfortable margin without leaving users waiting
  too long for a failed booking to re-attempt

### 4. Payment idempotency

Each SQS message carries a `bookingId` (UUID). The Payment Worker uses this as an idempotency
key with the payment gateway. If the same message is processed twice (SQS at-least-once delivery),
the second gateway call returns "already processed" and the worker updates DB idempotently:

```sql
UPDATE bookings SET status = 'confirmed', payment_ref = $2
WHERE booking_id = $1 AND status = 'pending';
-- Only updates if still pending; no double-charge
```

### 5. Cost

- SQS: $0.40 per million messages. At 50,000 messages/event × 100 events/month = 5M messages
  = **$2/month**
- ECS Payment Workers: 4 tasks × t3.small = ~$120/month
- Total queue infrastructure: ~$122/month vs. $6,200/month for synchronous API scaling

---

## SQS Message Format

```json
{
  "bookingId": "uuid-v4",
  "userId": "uuid-v4",
  "eventId": "uuid-v4",
  "seats": [
    { "seatId": "uuid-v4", "sectionId": "A1", "price": 1200 }
  ],
  "totalAmount": 1200,
  "currency": "INR",
  "paymentMethodToken": "tok_xxxx",
  "lockExpiry": "2026-06-12T11:45:00Z",
  "enqueuedAt": "2026-06-12T11:35:00Z",
  "retryCount": 0
}
```

- `lockExpiry` lets the worker abandon processing if the Redis seat lock has already expired
  (user waited too long → release the seat, no charge)
- `retryCount` is incremented by the worker on requeue to differentiate intentional retries
  from SQS redeliveries

---

## Tradeoffs Accepted

| What we gain | What we give up |
|---|---|
| Handles 500K RPS booking acceptance | User doesn't get instant payment confirmation |
| Gateway failures don't block bookings | UX requires polling or push notification |
| Auto-retry on worker failure | More complex failure states to communicate to user |
| $122/month vs $6,200/month | Eventual consistency between booking acceptance and confirmation |
| Seat lock TTL protects inventory | User's seat held for up to 10 min before confirmation |

---

## Revision Trigger

Reconsider this design if:
- Payment gateway SLA improves to < 500ms consistently (sync becomes viable again)
- User research shows polling/push UX significantly hurts conversion rate
- Event scale permanently drops below 5,000 concurrent bookings (sync is fine there)
- SQS is replaced by a self-hosted queue (Kafka) for cost reasons at very high volume
