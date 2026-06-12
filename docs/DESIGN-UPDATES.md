# DESIGN-UPDATES.md — Post-Roast Architecture Updates

Based on the panel Q&A session, two gaps were identified that change the architecture.
Both are substantive: they add new components or mechanisms, not just label changes.

---

## Update 1: Per-User Seat Hold Counter in Redis

**Triggered by:** Panel Question 3 — "What stops one user from holding 200 seats?"

**What changed:**

Added a Redis counter per user per event to enforce a maximum of 8 concurrent held seats:

```
Key:   holds:{userId}:{eventId}:count
Type:  Redis integer (INCR / DECR)
TTL:   600 seconds (matches seat lock TTL — auto-expires with the locks)
```

**New lock acquisition flow in API server:**

```javascript
// Atomic Lua script — runs as a single Redis transaction
const luaScript = `
  local count = redis.call('GET', KEYS[1])
  if count and tonumber(count) >= tonumber(ARGV[1]) then
    return 0  -- Reject: user at seat hold limit
  end
  local lockAcquired = redis.call('SET', KEYS[2], ARGV[2], 'NX', 'EX', ARGV[3])
  if not lockAcquired then
    return 0  -- Reject: seat already locked
  end
  redis.call('INCR', KEYS[1])
  redis.call('EXPIRE', KEYS[1], ARGV[3])
  return 1  -- Success
`;
// KEYS[1] = holds:{userId}:{eventId}:count
// KEYS[2] = seat_lock:{eventId}:{seatId}
// ARGV[1] = 8 (max holds per user)
// ARGV[2] = {userId}:{bookingId}
// ARGV[3] = 600 (TTL in seconds)
```

**Counter decrement on seat release:**
- On booking confirmation: `DECR holds:{userId}:{eventId}:count`
- On booking expiry/abandonment: counter auto-decrements via TTL (key expires with the lock)
- On payment failure: `DECR holds:{userId}:{eventId}:count` before releasing lock

**Why this is necessary:**
The original design allowed a user to make unlimited separate API requests, each booking up
to 8 seats. A malicious user or a faulty client could hold hundreds of seats simultaneously,
preventing legitimate users from buying. During a 50,000-seat event where the first 30 seconds
determine who gets seats, this is a practical attack vector. The hold counter closes this gap.

The per-event scope (`{userId}:{eventId}`) means the limit applies within a single event —
a user can hold 8 seats at Event A and 8 seats at Event B simultaneously, which is legitimate.

**What it costs:**
- 2 additional Redis commands per seat selection (GET count check + INCR) — mitigated by the
  Lua script running atomically in a single round-trip (~0.5ms).
- Counter key storage: 50,000 active users × 1 key per event = 50K keys × ~64 bytes = ~3MB
  per event. Negligible.
- No additional infrastructure cost.

**What it still doesn't solve:**
- A coordinated attack where 8 different user accounts each hold 8 seats (64 total). This
  requires account-level abuse detection (velocity checks on account creation, payment method
  fingerprinting) — beyond the scope of the seat locking system.
- A user can still hold 8 seats and not pay, releasing them only when the 10-minute TTL expires.
  Reducing seat hold TTL during high-demand events (see Update 2 in DESIGN-UPDATES v2 if
  implemented) partially mitigates this.

---

## Update 2: CloudWatch Alarm on SQS Queue Depth + ECS Worker Cap

**Triggered by:** Panel Question 4 — "Your bill spiked to $3,200. What's your plan?"

**What changed:**

Two new operational controls added:

### 2a. CloudWatch Alarm: SQS Queue Depth > 10,000

```yaml
# CloudFormation / Terraform resource
Alarm: BookingQueueDepthHigh
MetricName: ApproximateNumberOfMessagesVisible
Namespace: AWS/SQS
QueueName: booking-payment-queue
Threshold: 10000
ComparisonOperator: GreaterThanThreshold
EvaluationPeriods: 2
Period: 60  # seconds
TreatMissingData: notBreaching
Actions:
  - SNS topic → PagerDuty → on-call engineer
  - SNS topic → SES → engineering team email
```

**Why this alarm was missing:** The original design had no visibility into SQS queue buildup.
Queue depth > 10,000 indicates one of: (a) payment gateway is slow/down and workers are
accumulating messages, (b) ECS workers have crashed and aren't pulling, (c) a traffic spike
has outpaced the worker fleet. Without this alarm, the first signal of any of these conditions
is a customer complaint or a billing shock.

### 2b. ECS Worker Max Concurrency Cap: 20 tasks

```yaml
# ECS Service auto-scaling policy
MinCapacity: 4
MaxCapacity: 20   # Previously: unlimited (DEFAULT = 100 in ECS)
ScalingPolicy:
  TargetTrackingScaling:
    TargetValue: 70  # SQS messages per worker
    ScaleOutCooldown: 60s
    ScaleInCooldown: 300s
```

**Why necessary:** ECS auto-scaling without a cap allowed the worker fleet to scale to 100+
tasks during a queue buildup — even if the queue depth was caused by a gateway outage (scaling
workers doesn't help if the gateway is down). 20 tasks at 3 seconds/payment = 6.67 payments/sec
per task × 20 tasks = 133 payments/second throughput. At 50,000 seats, this completes a full
event in 375 seconds (~6 minutes) — acceptable for async confirmation.

### 2c. Circuit Breaker: Gateway failure → reduced-throughput fallback

If the payment gateway returns > 50 consecutive failures within a 60-second window, the
Payment Worker switches to circuit-open state:

```
CLOSED (normal) → OPEN (gateway failing) → HALF-OPEN (testing recovery) → CLOSED
```

In OPEN state:
- Workers stop pulling from SQS (no new payment attempts)
- Existing in-flight messages remain visible for retry
- CloudWatch alarm triggers on-call
- After 60 seconds, one worker enters HALF-OPEN and attempts a single payment
- If it succeeds: all workers resume (CLOSED)
- If it fails: 60-second OPEN window resets

This prevents runaway ECS scaling during a gateway outage (the pattern that caused the $3,200
bill scenario).

**Why this is necessary:**
Without a circuit breaker, a gateway outage causes: messages stay in queue → workers keep
pulling and failing → ECS auto-scales up to try more workers → workers all fail → more scaling
→ runaway cost with zero throughput gain. The circuit breaker is the missing control that
decouples "worker count" from "gateway health."

**What it costs:**
- CloudWatch alarm: ~$0.10/alarm/month. Negligible.
- ECS task cap at 20: reduces maximum throughput from ~667 payments/sec (100 tasks) to
  133 payments/sec. For a 50,000-seat event this means ~6 minutes to process all payments
  instead of ~75 seconds. This is an acceptable tradeoff for cost predictability.
- Circuit breaker adds ~50ms to the worker decision loop (health check evaluation). Negligible.
- Operational complexity: on-call must understand circuit breaker states. Runbook required.

**What it still doesn't solve:**
- If the ECS cap of 20 is genuinely insufficient (multiple simultaneous events), the queue
  will back up. Monitoring the queue depth alarm (2a) is the signal to manually raise the cap.
- The circuit breaker protects against gateway-level failures, not against SQS service outages.
  An SQS outage would require the API to fall back to synchronous payment — not implemented
  in the current design.
