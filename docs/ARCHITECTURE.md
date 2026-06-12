# ARCHITECTURE.md — BookMyShow Full System Architecture

## Overview

This document describes the complete system architecture for the BookMyShow high-scale ticketing
platform, designed to handle 500,000 RPS peak load during flash-sale events (IPL finals,
blockbuster releases). Every component and data-flow connection is labelled.

---

## Full System Diagram

```
                         USERS (500K RPS peak)
                               │
                               │ HTTPS requests
                               ▼
         ┌─────────────────────────────────────────────────────┐
         │              CloudFront CDN                          │
         │                                                       │
         │  ┌──────────────────┐    ┌────────────────────────┐  │
         │  │   CACHE HIT      │    │     CACHE MISS         │  │
         │  │                  │    │                        │  │
         │  │ Serves directly: │    │ Passes through to ALB: │  │
         │  │ • Static assets  │    │ • /api/* routes        │  │
         │  │   (JS/CSS/fonts  │    │ • Dynamic pages        │  │
         │  │    max-age=1yr)  │    │ • Auth requests        │  │
         │  │ • Event pages    │    │ • POST/PUT/DELETE      │  │
         │  │   (TTL=300s)     │    │                        │  │
         │  │ • Venue SVGs     │    │  [~15% of traffic      │  │
         │  │   (TTL=3600s)    │    │   reaches origin]      │  │
         │  │                  │    │                        │  │
         │  │ [~85% hit ratio] │    │                        │  │
         │  └──────────────────┘    └────────────┬───────────┘  │
         └────────────────────────────────────────┼─────────────┘
                                                  │
                      ← CloudFront CDN (see CACHING.md: 85% offload at 500K RPS)
                                                  │
                                   HTTPS + SSL termination
                                   ~75K RPS reaches ALB
                                                  │
                                                  ▼
         ┌─────────────────────────────────────────────────────┐
         │         Application Load Balancer (ALB)              │
         │                                                       │
         │  • SSL/TLS termination (ACM certificate)             │
         │  • Health check: GET /health every 10 seconds        │
         │    (unhealthy threshold: 3 consecutive failures)      │
         │  • Rate limit rule: 1,000 req/s per IP (WAF)         │
         │  • Round-robin to healthy API server instances        │
         │  • Sticky sessions: DISABLED (stateless API design)   │
         └──────────────────────────┬──────────────────────────┘
                                    │
                    ← ALB (SSL termination, 10s health interval, 1K RPS/IP rate limit)
                                    │
                      Routes to healthy instances only
                                    │
                ┌───────────────────┼───────────────────┐
                │                   │                   │
                ▼                   ▼                   ▼
    ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐
    │  Node.js API #1 │  │  Node.js API #2  │  │  Node.js API #N │
    │  (t3.medium)    │  │  (t3.medium)     │  │  (auto-scale)   │
    └────────┬────────┘  └────────┬─────────┘  └────────┬────────┘
             │                    │                      │
             └────────────────────┼──────────────────────┘
                                  │
          ← Node.js API servers (auto-scale group, see CONCURRENCY.md: stateless to support 500K RPS)
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
          │ Reads from Redis:     │ Writes to DB:         │ Publishes to SQS:
          │ • event:{id}:detail   │ • INSERT bookings     │ • booking message
          │   (TTL=300s)          │   (status=pending)    │   on seat lock success
          │ • event:{id}:seat_map │ • UPDATE seats        │
          │   (TTL=30s, or DEL    │   (status=held)       │
          │    on booking)        │ • Audit log inserts   │
          │ • SET seat_lock NX EX │                       │
          │   (SETNX lock, 600s)  │                       │
          │                       │                       │
          ▼                       ▼                       ▼
┌──────────────────┐   ┌────────────────────┐   ┌────────────────────────┐
│   Redis Cluster  │   │  PostgreSQL Primary │   │   SQS Payment Queue    │
│   (3 shards,     │   │  (writes only)      │   │                        │
│    r6g.large)    │   │                     │   │ ← Async queue          │
│                  │   │ ← Writes only       │   │ (see QUEUE.md: sync    │
│ USE A — CACHE:   │   │ (see SCHEMA.md:     │   │  payment at 5L users = │
│ Key: event:{id}  │   │  read/write split)  │   │  pool exhaustion at    │
│ TTL: 30–3600s    │   │                     │   │  394 RPS)              │
│                  │   │ • INSERT bookings   │   │                        │
│ USE B — LOCKS:   │   │ • UPDATE seats      │   │ Message format:        │
│ Key:             │   │ • UPDATE bookings   │   │ {bookingId (UUID),     │
│  seat_lock:      │   │ • INSERT            │   │  userId, eventId,      │
│  {evId}:{seatId} │   │   booking_events    │   │  seats[], amount,      │
│ SETNX + TTL=600s │   │                     │   │  paymentToken,         │
│                  │   │                     │   │  lockExpiry}           │
│ ← Redis SETNX    │   │           │         │   │                        │
│ lock (see        │   │           │ streams │   │ Visibility timeout:    │
│ CONCURRENCY.md:  │   │           │  to     │   │ 300 seconds            │
│ 500K RPS demand) │   │           ▼         │   │                        │
└────────┬─────────┘   │  ┌──────────────────┤   │ DLQ after 3 failures   │
         │             │  │  Read Replicas×2 │   │                        │
         │             │  │                  │   └───────────┬────────────┘
         │             │  │ ← Read replicas  │               │
         │             │  │ (see SCHEMA.md:  │               │ SQS pull (long-poll)
         │             │  │  seat avail read │               │ read message
         │             │  │  separate from   │               ▼
         │             │  │  write txns)     │   ┌────────────────────────┐
         │             │  │                  │   │  Payment Worker (ECS)  │
         │             │  │ Handles:         │   │  (4 tasks, t3.small)   │
         │             │  │ • seat map reads │   │                        │
         │             │  │ • event listings │   │ Step 1: Read SQS msg   │
         │             │  │ • booking history│   │   ↓                    │
         │             │  │ • user queries   │   │ Step 2: Check lock     │
         │             │  └──────────────────┘   │   expiry (lockExpiry   │
         │             └────────────────────────┘│   field vs NOW())      │
         │                                        │   ↓                    │
         │ DEL seat_lock on                       │ Step 3: Call payment   │
         │ confirmed booking                      │   gateway (Razorpay/   │
         │ (belt-and-suspenders                   │   Paytm) with          │
         │ after DB confirms)                     │   idempotency key =    │
         └────────────────────────────────────────│   bookingId (UUID)     │
                                                  │   ↓                    │
                                                  │ Step 4: Update DB      │
                                                  │   UPDATE bookings      │
                                                  │   SET status=confirmed │
                                                  │   + UPDATE seats       │
                                                  │   SET status=booked    │
                                                  │   (PostgreSQL Primary) │
                                                  │   ↓                    │
                                                  │ Step 5: Publish SNS    │
                                                  │   topic:               │
                                                  │   booking-confirmed    │
                                                  │   ↓                    │
                                                  │ Step 6: Delete SQS msg │
                                                  │   (only after all      │
                                                  │   above succeed)       │
                                                  │                        │
                                                  │ On failure:            │
                                                  │   msg returns to SQS   │
                                                  │   after 300s timeout   │
                                                  │   DLQ after 3 retries  │
                                                  └──────────┬─────────────┘
                                                             │
                                          SNS publish: booking-confirmed
                                          Payload: {bookingId, userId,
                                                    eventName, seats,
                                                    totalAmount, venue}
                                                             │
                                              ┌──────────────┴──────────────┐
                                              │                             │
                                              ▼                             ▼
                               ┌─────────────────────┐       ┌─────────────────────┐
                               │   SES (Email)        │       │   SNS → SMS         │
                               │                      │       │                     │
                               │ Sends to user email: │       │ Sends to user phone:│
                               │ • Booking confirmed  │       │ • "Your 2 seats for │
                               │ • Seat details       │       │   IPL Final are     │
                               │ • QR code for entry  │       │   confirmed. Ref:   │
                               │ • Payment receipt    │       │   BMS-XXXX"         │
                               │                      │       │                     │
                               │ ← SNS→SES triggered  │       │ ← Triggered by      │
                               │ by payment worker    │       │ payment worker on   │
                               │ on confirmation      │       │ confirmation        │
                               └─────────────────────┘       └─────────────────────┘
```

---

## Component Annotations Index

Every component below maps to a Part A design decision:

| Component | Annotation | Source Document |
|---|---|---|
| Redis SETNX seat lock | 500K RPS makes DB row locking infeasible (connection pool exhaustion) | CONCURRENCY.md |
| Redis cache (event detail, seat map) | 85% CDN+cache offload needed to handle 500K RPS without 250 DB instances | CACHING.md |
| SQS async queue | Sync payment at 5L users = pool exhaustion at 394 RPS throughput wall | QUEUE.md |
| SQS visibility timeout 300s | Worker crash recovery window: 30s gateway timeout + 90s ECS restart + 60s backoff | QUEUE.md |
| PostgreSQL read replicas ×2 | Seat availability reads must not compete with booking write transactions | SCHEMA.md |
| UUID booking_id | Idempotency key for SQS at-least-once delivery; no double-charge on reprocess | SCHEMA.md |
| CloudFront TTL=300s event pages | Stale-window acceptable for non-seat content; reduces origin load 6.7× | CACHING.md |
| Event-driven seat_map invalidation | Seat map DEL on booking reduces false-availability window vs pure TTL | CACHING.md |
| Payment Worker DLQ after 3 retries | Failed payments need human inspection; infinite retry risks double-charges | QUEUE.md |
| ALB health check 10s interval | Balance between fast failure detection and flapping on transient issues | (operational decision) |

---

## Request Flow Walkthrough

### Flow A: User views event page (cache hit)
```
User → CloudFront edge (TTL=300s hit) → 200 OK, ~10ms, no origin request
```

### Flow B: User views event page (cache miss)
```
User → CloudFront (miss) → ALB → API server
     → Redis GET event:{id}:detail (hit) → 200 OK, ~15ms
     → Redis GET event:{id}:detail (miss) → PostgreSQL Read Replica
     → Redis SET event:{id}:detail EX 300 → 200 OK, ~35ms
```

### Flow C: User selects a seat
```
User → CloudFront (pass-through, /api/) → ALB → API server
     → Redis SET seat_lock:{evId}:{seatId} {userId}:{bookingId} NX EX 600
       → NX fails (seat taken): 409 Conflict returned immediately
       → NX succeeds: continue
     → PostgreSQL Primary: UPDATE seats SET status='held'
     → PostgreSQL Primary: INSERT bookings (status='pending')
     → SQS: publish booking message
     → 202 Accepted returned to user with bookingId
```

### Flow D: Payment processing (async)
```
SQS → Payment Worker (ECS)
     → Check lockExpiry: if expired, release seat, 200 (no charge)
     → Call payment gateway (idempotency key = bookingId UUID)
       → Gateway failure: message visibility expires, SQS redelivers after 300s
       → Gateway success: continue
     → PostgreSQL Primary: UPDATE bookings SET status='confirmed'
     → PostgreSQL Primary: UPDATE seats SET status='booked'
     → Redis: DEL seat_lock:{evId}:{seatId}
     → SNS publish → SES email + SMS to user
     → SQS: delete message
```

### Flow E: User abandons mid-booking (seat lock expiry)
```
Redis TTL expires after 600s (no action from worker needed)
pg_cron job (every 60s): UPDATE seats SET status='available' WHERE held_until < NOW()
Booking record: status set to 'expired' by same job
User sees seat available again on next seat map refresh
```

---

## Infrastructure Summary

| Component | Type | Count | Est. Monthly Cost |
|---|---|---|---|
| CloudFront | CDN | Global PoPs | ~$120 |
| ALB | Load balancer | 1 | ~$25 |
| Node.js API | EC2 t3.medium (auto-scale) | 4–16 | ~$120–$480 |
| Redis Cluster | ElastiCache r6g.large | 3 shards | ~$360 |
| PostgreSQL Primary | RDS r5.large | 1 | ~$180 |
| PostgreSQL Replicas | RDS r5.large | 2 | ~$360 |
| SQS | Managed queue | 1 + 1 DLQ | ~$2 |
| ECS Payment Workers | Fargate t3.small | 4 tasks | ~$120 |
| SNS + SES | Managed messaging | — | ~$5 |
| **Total (base)** | | | **~$1,372/month** |
| **Total (peak, 16 API nodes)** | | | **~$1,732/month** |
