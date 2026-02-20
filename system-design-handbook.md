# 🚀 Senior Backend System Design Interview Handbook (India Product Bar)

This handbook is built specifically for engineers targeting:

-   MakeMyTrip
-   Uber
-   Flipkart
-   BookMyShow
-   Swiggy
-   Razorpay
-   High-scale Indian product startups

It includes:

✔ Deep HLD coverage\
✔ Deep LLD coverage\
✔ Java examples\
✔ Concurrency & distributed systems\
✔ Real interview speaking scripts\
✔ Failure analysis\
✔ Tradeoff discussions

You should be able to *speak for 45--60 minutes per major design* after
mastering this.

------------------------------------------------------------------------

# 🎯 The 7-Step Interview Framework (What Senior Interviewers Expect)

Use this structure in EVERY system design interview.

1️⃣ Clarify Requirements\
2️⃣ Define Non-Functional Requirements\
3️⃣ Estimate Scale\
4️⃣ High-Level Architecture\
5️⃣ Deep Dive into Core Components\
6️⃣ Bottlenecks & Scaling\
7️⃣ Failure Handling & Tradeoffs

------------------------------------------------------------------------

🧠 SPEAKING SCRIPT TEMPLATE:

"Before jumping into architecture, I'd like to clarify functional and
non-functional requirements."

"I'll assume we have X million DAU and Y QPS peak."

"Let me start with a high-level architecture and then deep dive into
booking concurrency."

This structured thinking is what separates Staff+ engineers from
mid-level engineers.

------------------------------------------------------------------------

# 🎟️ Design BookMyShow / Seat Booking (MakeMyTrip Level Depth)

## Functional Requirements

-   Search movies
-   Select seat
-   Prevent double booking
-   Payment integration
-   Cancellation & refund

## Non-Functional

-   99.99% availability
-   Strong consistency for booking
-   Handle flash sale traffic spikes

------------------------------------------------------------------------

## Architecture

User → API Gateway → Booking Service → DB → Payment Service → Cache →
Redis Lock

------------------------------------------------------------------------

## Concurrency Strategies

### Option 1: Pessimistic Lock

SELECT \* FROM seats WHERE id=101 FOR UPDATE;

Pros: - Safe

Cons: - Not scalable for flash sales

------------------------------------------------------------------------

### Option 2: Optimistic Lock (Preferred)

``` java
@Entity
class Seat {
    @Id
    private Long id;

    @Version
    private int version;

    private boolean booked;
}
```

If update fails → retry.

------------------------------------------------------------------------

### Option 3: Redis Distributed Lock

SET seat:101 locked NX PX 5000

Used only for short reservation window (5 minutes).

------------------------------------------------------------------------

## Idempotency (CRITICAL at Uber/MMT)

Payment must not double charge.

``` java
class PaymentRequest {
    String idempotencyKey;
}
```

Store key in DB → reject duplicates.

------------------------------------------------------------------------

🧠 INTERVIEW SPEAKING SCRIPT:

"Booking systems require strong consistency only for the final seat
allocation. Search can be eventually consistent."

"I would use optimistic locking with retries because flash sales require
high concurrency."

"If Redis crashes, DB constraint is final source of truth."

------------------------------------------------------------------------

# 🚗 Design Uber Ride Matching (Uber-Level Depth)

## Core Challenges

-   Real-time driver location updates
-   Nearest driver matching
-   Surge pricing
-   High write throughput

------------------------------------------------------------------------

## Geo Matching Strategy

Use Geohashing.

Driver location → store in Redis sorted set.

Key: geo:12abc

Search nearby hashes.

------------------------------------------------------------------------

## Matching Flow

Rider Request → Location Service → Matching Service → Driver
Notification → Trip Service

------------------------------------------------------------------------

## Surge Pricing

Compute demand/supply ratio per geo cell.

If demand \> supply: price = base \* multiplier

------------------------------------------------------------------------

## Failure Handling

-   If driver rejects → retry next driver
-   If payment fails → auto-cancel

------------------------------------------------------------------------

🧠 SPEAKING SCRIPT:

"Matching should be near real-time, so Redis in-memory structures are
preferred."

"Trip creation must be idempotent."

"I would decouple matching and trip creation using Kafka."

------------------------------------------------------------------------

# 💬 Design WhatsApp-like Chat

## Requirements

-   1:1 chat
-   Group chat
-   Ordering guarantee
-   Delivery receipt

------------------------------------------------------------------------

## Architecture

Sender → Chat Gateway → Kafka → Fanout Workers → Recipient

------------------------------------------------------------------------

## Ordering Guarantee

Use per-user partitioning in Kafka.

Key = conversationId

------------------------------------------------------------------------

## Storage

Recent → Cassandra Media → S3 + CDN Cold storage → Archival DB

------------------------------------------------------------------------

🧠 SPEAKING SCRIPT:

"I will ensure ordering by partitioning messages by conversation ID."

"If a user is offline, messages remain stored and delivered on
reconnect."

------------------------------------------------------------------------

# 🧱 LLD -- Parking Lot (Senior Level)

## Classes

``` java
class ParkingLot {
    private List<Floor> floors;
}

class Floor {
    private List<ParkingSpot> spots;
}

class ParkingSpot {
    private SpotType type;
    private Vehicle vehicle;
}
```

## Thread Safety

Use per-spot locking.

``` java
class ParkingSpot {
    private final ReentrantLock lock = new ReentrantLock();
}
```

🧠 SPEAKING SCRIPT:

"I avoid global lock because it reduces concurrency."

"I'll make spot booking atomic."

------------------------------------------------------------------------

# ⏳ Rate Limiter (Flipkart / Razorpay Favorite)

## Token Bucket

``` java
class TokenBucket {
    private int capacity;
    private int tokens;

    synchronized boolean allow() {
        if (tokens > 0) {
            tokens--;
            return true;
        }
        return false;
    }
}
```

## Distributed

Store counters in Redis with TTL.

------------------------------------------------------------------------

🧠 SPEAKING SCRIPT:

"I prefer sliding window for fairness."

"Redis replication ensures high availability."

------------------------------------------------------------------------

# ⚖️ CAP Theorem & Tradeoffs

           Consistency
              /\
             /  \
            /    \

Availability ------ Partition Tolerance

Booking → CP\
Feed → AP\
Chat → Tunable

------------------------------------------------------------------------

🧠 SPEAKING SCRIPT:

"For booking systems, I sacrifice availability slightly to ensure no
double booking."

------------------------------------------------------------------------

# 🔥 Failure Scenarios Interviewers Love

What if Redis crashes? What if DB primary goes down? What if payment
succeeds but booking fails? What if Kafka lags?

Always answer with:

-   Retry strategy
-   Idempotency
-   Dead-letter queue
-   Circuit breaker
-   Observability

------------------------------------------------------------------------

# 📈 Scaling to 100M Users

-   Horizontal scaling
-   Sharding strategy
-   Read replicas
-   Caching layers
-   CDN

Sharding Strategies:

1.  Hash-based
2.  Range-based
3.  Geo-based

Always discuss rebalancing cost.

------------------------------------------------------------------------

# 🏁 Final Interview Readiness Checklist

✅ Clarified requirements\
✅ Estimated scale\
✅ Explained tradeoffs\
✅ Handled failures\
✅ Discussed consistency\
✅ Used correct data store\
✅ Explained concurrency\
✅ Added idempotency\
✅ Considered monitoring

If you can confidently speak through 5 major designs using this depth,
you are operating at Senior/Staff level.

🔥 Now revise this multiple times and practice speaking aloud.

------------------------------------------------------------------------