# Distributed Ticket Reservation System — Master Interview Guide

> **Project Repository:** `distributed-ticketing-system`  
> **Deployment Architecture:** 10 Docker Containers (Microservices, PostgreSQL ×3, Redis, RabbitMQ, Nginx, React)  
> **Live Demo:** `http://localhost` *(run locally with Docker Compose)* | **System Lab:** `http://localhost/lab`  
> **Core Focus:** Extreme Concurrency, Distributed Locking (Redis Lua), ACID Transactions, Zero Double-Booking Guarantee, Resilient Event-Driven Messaging.

---

## Table of Contents
1. [The 90-Second Professional Elevator Pitch](#1-the-90-second-professional-elevator-pitch)
2. [High-Level Architecture & Container Topology](#2-high-level-architecture--container-topology)
3. [The Core Problem: Race Conditions & Double-Booking](#3-the-core-problem-race-conditions--double-booking)
4. [The 3-Layer Defense Mechanism (Deep Dive)](#4-the-3-layer-defense-mechanism-deep-dive)
5. [Atomic Multi-Seat Group Booking & Compensated Workflow](#5-atomic-multi-seat-group-booking--compensated-workflow)
6. [Asynchronous Messaging: RabbitMQ Retry & Dead-Letter Queues](#6-asynchronous-messaging-rabbitmq-retry--dead-letter-queues)
7. [Caching, Load Balancing & System Lab Mode](#7-caching-load-balancing--system-lab-mode)
8. [Top 20 Technical Interview Questions & Model Answers](#8-top-20-technical-interview-questions--model-answers)
9. [Trade-offs, Architectural Dilemmas & Production Evolutions](#9-trade-offs-architectural-dilemmas--production-evolutions)

---

## 1. The 90-Second Professional Elevator Pitch

### What to Say to an Interviewer:

> *"I designed and built a **Distributed Ticket Reservation System** engineered to guarantee **zero double-booking under extreme concurrent flash-crowd traffic** (such as high-demand concert or sports ticket sales).*
>
> *The central engineering challenge in ticketing systems is the **read-then-write race condition**: when 50 users click on the exact same seat at the exact same millisecond, a standard database read allows multiple users to see the seat as 'available', resulting in catastrophic double-selling.*
>
> *To solve this, I designed a **3-Layer Defense Pipeline**:*
> 1. ***Layer 1 (Contention Control in Redis):*** *We use custom atomic **Lua scripts** to acquire temporary, expiring distributed seat holds (`seat:{event_id}:{seat_no}`) with all-or-nothing semantics for multi-seat groups.*
> 2. ***Layer 2 (Concurrency Serialization in PostgreSQL):*** *During confirmation, we execute row-level pessimistic locking using `SELECT ... FOR UPDATE` ordered by seat number to prevent deadlocks while transitioning status from `held` to `confirmed`.*
> 3. ***Layer 3 (Durable Database Invariant):*** *We enforce a **PostgreSQL partial unique index** (`CREATE UNIQUE INDEX ... WHERE status = 'confirmed'`), making it mathematically and physically impossible for two confirmed rows to exist for the same event seat.*
>
> *Post-confirmation, the system offloads ticket generation and email dispatch asynchronously via **RabbitMQ** with a 3-tier retry queue and dead-letter mechanism.*
>
> *The entire system runs across **10 containerized microservices** behind an **Nginx load balancer**, featuring both a customer booking journey and an interactive **System Lab** where interviewers can race 50 concurrent threads against a single seat to watch distributed locks and conflict responses live in action."*

---

## 2. High-Level Architecture & Container Topology

The platform adheres to the **Database-per-Service** architectural pattern to avoid database coupling and bottlenecking.

```text
                                 Client Browser
                                       │
                                       ▼
                       Nginx Reverse Proxy & Load Balancer
                                   (Port 80)
                                       │
            ┌──────────────────────────┼──────────────────────────┐
            │                          │                          │
            ▼                          ▼                          ▼
       User Service              Event Service              Booking Service
       (Express/Node)         (2 Replicas: S1, S2)          (Express/Node)
            │                          │                          │
            ▼                          ▼                          ▼
        Users DB                   Events DB                 Bookings DB
      (PostgreSQL)               (PostgreSQL)               (PostgreSQL)
                                       │                          │
                                       ▼                          ▼
                                Redis Cache ◄────────────── Distributed Holds
                               (Catalog & TTL)               (Atomic Lua)
                                                                  │
                                                                  ▼
                                                               RabbitMQ
                                                           (Topic Exchange)
                                                                  │
                                                                  ▼
                                                         Notification Service
                                                        (Worker / Ticket / Email)
```

### The 10 Docker Containers:
1. `nginx`: Edge reverse proxy, SSL termination, static React frontend host, round-robin upstream for event service replicas.
2. `user-service`: JWT authentication, password hashing with bcrypt, profile state.
3. `users_db`: PostgreSQL 16 instance for user credentials.
4. `event-service-1`: Replica 1 of event catalog, theatre layout schemas, VIP pricing, seat tiers.
5. `event-service-2`: Replica 2 of event catalog (load-balanced by Nginx).
6. `events_db`: PostgreSQL 16 instance for events and venues.
7. `booking-service`: Core transactional booking orchestration, Redis distributed locking, multi-seat validation, PostgreSQL transitions.
8. `bookings_db`: PostgreSQL 16 instance storing reservations and bookings.
9. `redis`: Redis 7 in-memory store for distributed seat lock keys and read-through event caching.
10. `rabbitmq`: RabbitMQ 3 message broker for asynchronous reliable event dispatch.
11. `notification-service`: Asynchronous worker processing confirmed tickets, PDF generation, email simulation, with retry and dead-letter queues.

---

## 3. The Core Problem: Race Conditions & Double-Booking

### The Naive Flow (How Double-Booking Happens):
```text
User A (Thread 1)                      User B (Thread 2)
      │                                      │
1. SELECT status FROM bookings         1. SELECT status FROM bookings
   WHERE seat = 42;                       WHERE seat = 42;
   -> Returns: NULL (Available!)          -> Returns: NULL (Available!)
      │                                      │
2. INSERT INTO bookings (seat=42,      2. INSERT INTO bookings (seat=42,
   status='confirmed', user='UserA');     status='confirmed', user='UserB');
   -> SUCCESS!                            -> SUCCESS! (DOUBLE-BOOKED!)
```

### Why Standard Relational Databases Fail Under High Concurrency:
1. **Isolation Level Gaps:** At standard `READ COMMITTED` isolation, Thread 2's query executes before Thread 1 commits. Both read an empty result.
2. **Database Connection Exhaustion:** If 1,000 users race for 10 seats, hitting the relational database directly with write locks saturates the database connection pool, leading to connection timeouts and 500 errors.
3. **Partial Group Failures:** If a family books seats `[42, 43, 44]`, but seat `44` is already taken, a naive system might reserve seats `42` and `43`, leaving the family split up and holding unwanted seats.

---

## 4. The 3-Layer Defense Mechanism (Deep Dive)

Our architecture handles concurrency in **three distinct tiers**:

```text
┌─────────────────────────────────────────────────────────────┐
│ LAYER 1: REDIS CONTENTION CONTROL (Atomic In-Memory Hold)   │
│ Fast, non-blocking contention absorbing at the edge.        │
│ Atomic multi-key Lua scripts hold seats for 600 seconds.    │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ LAYER 2: POSTGRESQL ROW LOCKING (Serialization)             │
│ `SELECT ... FOR UPDATE` ordered by seat number.             │
│ Eliminates deadlocks while verifying held status.           │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ LAYER 3: PARTIAL UNIQUE INDEX (Mathematical Invariant)       │
│ Database hard constraint: Only ONE 'confirmed' row allowed.  │
│ Final, unbreakable wall against data corruption.            │
└─────────────────────────────────────────────────────────────┘
```

---

### Layer 1: Redis Distributed Holds with Atomic Lua Scripts
**File:** `services/booking-service/src/db/redis.ts`

To absorb concurrency before it hits the database, we use Redis keys with a 600-second (10-minute) TTL:
`seat:{event_id}:{seat_number}` $\rightarrow$ stores `hold_token` (UUID).

#### Why Lua Scripts?
In Redis, executing individual commands like `EXISTS` followed by `SET` is **not atomic**—another thread can execute between them. Redis executes Lua scripts as a single atomic operation; no other command runs until the script finishes.

#### 1. Atomic Multi-Seat Hold Script:
```lua
-- All-or-nothing multi-seat lock acquisition
for _, key in ipairs(KEYS) do
  if redis.call("exists", key) == 1 then
    return 0 -- Even if 1 seat is taken, abort immediately!
  end
end

for _, key in ipairs(KEYS) do
  redis.call("set", key, ARGV[1], "EX", ARGV[2])
end

return 1 -- All requested seats acquired atomically!
```
* **All-or-Nothing Guarantee:** If a user requests seats `[42, 43, 44]`, but seat `44` is already locked, the script writes **zero keys** and returns `0`. The user gets an immediate `409 Conflict`, and seats `42` and `43` remain available for others.

#### 2. Ownership-Safe Compare-and-Delete Release Script:
```lua
local released = 0
for _, key in ipairs(KEYS) do
  if redis.call("get", key) == ARGV[1] then
    released = released + redis.call("del", key)
  end
end
return released
```
* **Why this is critical:** If User A's hold expires at second 600, User B immediately grabs seat 42. User A's slow confirmation request then arrives at second 601. A naive `redis.del(key)` would delete **User B's brand-new lock!** By checking `if get(key) == token`, we guarantee a request only deletes locks that it still legitimately owns.

---

### Layer 2: PostgreSQL Pessimistic Row Locking (`SELECT ... FOR UPDATE`)
**File:** `services/booking-service/src/routes/bookings.ts`

When the user confirms their purchase (`POST /api/bookings/confirm`), we execute an ACID transaction in PostgreSQL:

```typescript
const confirmed = await prisma.$transaction(async (tx) => {
  const locked = await tx.$queryRaw<BookingRow[]>`
    SELECT id, reservation_id, user_id, event_id, seat_number, status
    FROM bookings
    WHERE event_id = ${eventId}
      AND seat_number IN (${Prisma.join(seatNumbers)})
      AND status IN ('held', 'confirmed')
    ORDER BY seat_number
    FOR UPDATE
  `;
  
  // Verify all seats belong to this user and are still in 'held' status
  ...
  
  // Transition all held rows to 'confirmed'
  await tx.booking.updateMany({
    where: { id: { in: bookingIds }, status: BookingStatus.held },
    data: { status: BookingStatus.confirmed },
  });
});
```

#### Why `ORDER BY seat_number FOR UPDATE`?
**Deadlock Prevention.**
* Suppose User 1 reserves seats `[10, 11]`.
* Suppose User 2 reserves seats `[11, 10]`.
* If User 1 locks row 10 and waits for row 11, while User 2 locks row 11 and waits for row 10 $\rightarrow$ **Deadlock!** The database has to abort one transaction.
* By enforcing strict sorting (`ORDER BY seat_number`), every concurrent process locks rows in the identical physical order ($10 \rightarrow 11$), making deadlocks mathematically impossible.

---

### Layer 3: The Partial Unique Index (The Unbreakable Invariant)
**File:** `services/booking-service/prisma/migrations/20260930000100_unique_confirmed_seat/migration.sql`

```sql
CREATE UNIQUE INDEX "unique_confirmed_seat"
ON "bookings" ("event_id", "seat_number")
WHERE "status" = 'confirmed';
```

#### Why a Partial Unique Index instead of a Standard Unique Constraint?
In a real ticketing system, seat 42 will have **many historical rows**:
* User X held seat 42 $\rightarrow$ expired.
* User Y held seat 42 $\rightarrow$ cancelled.
* User Z held seat 42 $\rightarrow$ held.

A standard `UNIQUE(event_id, seat_number)` constraint would break because you could never store more than one row per seat, preventing audit logs and expired history.
A **Partial Unique Index** ignores all rows where `status IN ('held', 'expired', 'cancelled')`. It enforces uniqueness **only for rows where `status = 'confirmed'`**. 

Even if Redis explodes and the application server crashes mid-flight, the PostgreSQL storage engine itself will physically reject any attempt to insert or update a second confirmed seat.

---

## 5. Atomic Multi-Seat Group Booking & Compensated Workflow

Distributed systems cannot easily use distributed ACID transactions (like Two-Phase Commit / 2PC) across Redis and PostgreSQL without catastrophic performance penalties. 

Instead, our booking service uses a **Compensated Workflow**:

```text
[Client: Reserve Seats 42, 43, 44]
                 │
                 ▼
1. Acquire Redis Holds (Atomic Lua)
   ├── FAILED ──► Return 409 Conflict (Zero keys written)
   └── SUCCESS
         │
         ▼
2. PostgreSQL Transaction
   ├── Expire any stale/abandoned historical held rows
   └── Insert 3 new 'held' rows in bookings table
         │
         ├── TRANSACTION FAILS?
         │         │
         │         ▼
         │   Rollback PostgreSQL
         │         │
         │         ▼
         │   Compensate Redis: Run releaseSeatHolds(token)
         │   (Only deletes keys owned by this token)
         │         │
         │         ▼
         │   Return 500 Error (State completely clean)
         │
         └── TRANSACTION SUCCEEDS
                   │
                   ▼
       Return 201 Created with hold_token & 600s countdown
```

---

## 6. Asynchronous Messaging: RabbitMQ Retry & Dead-Letter Queues

**File:** `services/notification-service/src/consumer.ts`

When a booking is confirmed, the user should not wait for PDF ticket rendering and email delivery. The booking service publishes a `booking.confirmed` message to RabbitMQ and immediately responds `200 OK` to the user.

```text
                        booking_events (Topic Exchange)
                                      │
                                      ▼
                      [booking.confirmed Queue]
                                      │
                               Worker consumes
                                      │
                         ┌────────────┴────────────┐
                         ▼                         ▼
                      Success                    Error
                         │                         │
                     ACK message               NACK message
                                                   │
                                                   ▼ (Dead-letters to)
                                       booking_events.retry (Direct Exchange)
                                                   │
                                                   ▼
                                       [booking.confirmed.retry Queue]
                                         ('x-message-ttl': 5000 ms)
                                                   │
                                            TTL expires (5s)
                                                   │
                                                   ▼ (Dead-letters back to)
                                       booking_events (Topic Exchange)
                                                   │
                                                   ▼
                                       [booking.confirmed Queue]
                                         (Attempts 2 and 3)
                                                   │
                                       Failed >= 3 times?
                                                   │
                                                   ▼
                                       booking_events.dead Exchange
                                                   │
                                                   ▼
                                       [booking.confirmed.dead Queue] (DLQ)
                                       (Preserved for manual inspection)
```

### Key Engineering Details:
1. **Prefetch = 1:** `channel.prefetch(1)` ensures a worker only receives one message at a time, preventing worker overload.
2. **Persistent Messages:** Messages are marked `persistent: true` and published to `durable: true` exchanges, surviving RabbitMQ restarts.
3. **Dead-Letter Header Tracking:** The worker inspects the `x-death` array header injected by RabbitMQ to know exactly how many times the message has cycled. Once `attempts >= 3`, it routes the payload to the Dead-Letter Queue (DLQ).

---

## 7. Caching, Load Balancing & System Lab Mode

### Nginx Load Balancing (`nginx/nginx.conf`)
* Nginx exposes a single external gateway on Port 80.
* Routes `/api/events` to an upstream pool containing `event-service-1:3002` and `event-service-2:3002` using round-robin distribution.
* Routes `/api/bookings` to `booking-service:3003`.
* Routes `/api/auth` to `user-service:3001`.

### Redis Read-Through Caching (`services/event-service`)
* `GET /api/events/:id`: The service checks Redis key `cache:events:{id}`.
  * **Cache Hit:** Returns cached JSON in 2 milliseconds with header `X-Cache: HIT`.
  * **Cache Miss:** Queries `events_db` in PostgreSQL, stores JSON in Redis with a 60-second TTL, and returns with header `X-Cache: MISS`.

### System Lab Mode (`http://localhost/lab`)
A built-in interactive dashboard that exposes real-time telemetry:
* **The 50-Seat Race:** Fires 50 concurrent authenticated requests at a single seat. Graphically demonstrates 1 green success badge (`201 Created`) and 49 red conflict badges (`409 Conflict`).
* **Upstream Load Distribution:** Sends 50 requests to `/api/events` and displays a live bar chart proving Nginx alternates 50/50 between `event-service-1` and `event-service-2`.
* **Cache Telemetry:** Displays response times comparing Redis Cache Hits (1-2 ms) vs. PostgreSQL Cold Cache Misses (25-40 ms).

---

## 8. Top 20 Technical Interview Questions & Model Answers

### Q1: How do you prevent two users from booking the exact same seat at the same time?
**Answer:** We prevent double-booking using a 3-layer architecture:
1. **At the edge:** Redis distributed locks via atomic Lua scripts guarantee that only one user can hold a seat key (`seat:{event}:{seat}`).
2. **During confirmation:** An ACID PostgreSQL transaction runs `SELECT ... FOR UPDATE` ordered by seat number to lock the rows and verify their status.
3. **At the storage level:** A PostgreSQL partial unique index (`WHERE status = 'confirmed'`) enforces that only one confirmed booking can ever exist per seat.

---

### Q2: Why did you write custom Lua scripts instead of using standard Redis `SET key value NX EX`?
**Answer:** Standard `SET NX EX` works fine for a single key. But in real ticketing, users book **multi-seat groups** (e.g., seats 42, 43, 44). If you execute three separate `SET NX` commands over the network, Thread A might grab 42 and 43, while Thread B grabs 44. Both groups fail, leaving orphaned partial holds. 
Our custom Lua script receives all seat keys in `KEYS` and runs an atomic loop: if *any* seat exists, it aborts immediately with zero writes. All keys are acquired together in a single in-memory transaction.

---

### Q3: What happens if a user reserves a seat and their phone battery dies? Does the seat stay locked forever?
**Answer:** No. Every Redis hold key is assigned a strict 600-second (10-minute) TTL (`EX 600`). If the user disconnects or abandons the purchase, Redis automatically evicts the keys. When another user subsequently reserves those seats, our database transaction automatically marks the abandoned historical rows in PostgreSQL as `expired` before inserting new `held` rows.

---

### Q4: Why use a Partial Unique Index instead of a normal unique constraint?
**Answer:** A standard `UNIQUE(event_id, seat_number)` constraint only permits one row per seat across the entire table's lifetime. That would prevent audit logs of expired holds or cancellations. 
A PostgreSQL partial unique index (`WHERE status = 'confirmed'`) allows unlimited historical rows with status `held`, `expired`, or `cancelled`, but strictly enforces that **no more than one row can ever have status `confirmed`**.

---

### Q5: Why did you add `ORDER BY seat_number` before `FOR UPDATE` in PostgreSQL?
**Answer:** To eliminate database deadlocks. If Transaction 1 tries to lock seats 10 and 11, and Transaction 2 tries to lock seats 11 and 10 concurrently, Transaction 1 locks 10 and waits for 11, while Transaction 2 locks 11 and waits for 10 $\rightarrow$ deadlock. By strictly sorting the seats in ascending order before locking, all concurrent transactions acquire row locks in the exact same physical order, making deadlocks impossible.

---

### Q6: What happens if Redis succeeds, but PostgreSQL crashes before inserting the held row?
**Answer:** We wrap the PostgreSQL write in a `try...catch` block. If the database transaction fails, the catch block triggers an automated **compensation call** (`releaseSeatHolds(token)`), executing a Lua script that safely deletes only the keys owned by that token. If the node crashes completely before compensation, the Redis keys simply expire naturally via their 10-minute TTL. Crucially, no duplicate confirmed seat can ever be created because PostgreSQL uniqueness is untouched.

---

### Q7: Why not use Two-Phase Commit (2PC) between Redis and PostgreSQL?
**Answer:** Two-Phase Commit introduces blocking coordinator protocols that kill throughput and create single points of failure under high-concurrency flash sales. In ticket booking, a temporary stale hold in Redis (resolved by TTL) is acceptable, whereas an unavailable or stalled checkout pipeline is disastrous. A compensated workflow with eventual consistency is significantly faster and more resilient.

---

### Q8: How does your compare-and-delete Lua script prevent deleting another user's lock?
**Answer:** Suppose User 1 holds seat 42. Their network lags, and their 600-second TTL expires. User 2 immediately acquires seat 42 with a new token. If User 1's browser finally wakes up and calls release, a naive `DEL seat:42` would wipe out User 2's lock. 
Our Lua release script checks `if redis.call("get", key) == ARGV[1] then redis.call("del", key)`. Because User 1's token doesn't match User 2's token in Redis, the script does nothing.

---

### Q9: How do your RabbitMQ retry queues work without infinite loops?
**Answer:** We use RabbitMQ's Dead-Letter Exchange (DLX) and message TTL:
1. When message processing fails, the worker `nack`s the message without requeue.
2. The main queue dead-letters the message to `booking_events.retry`.
3. The retry queue holds the message with `'x-message-ttl': 5000` (5 seconds).
4. When 5 seconds expire, RabbitMQ dead-letters it back to the main queue for another attempt.
5. The consumer checks the message's `x-death` header. Once retry attempts reach 3, it routes the message directly to `booking.confirmed.dead` (DLQ) and `ack`s it to stop the cycle.

---

### Q10: Why did you choose Database-per-Service instead of a single shared database?
**Answer:** In microservices architecture, a shared database creates tight schema coupling, shared connection pool saturation, and a single point of failure. If the Event Catalog receives 100,000 read requests, its database connection pool could exhaust all connections, preventing the Booking Service from confirming tickets. Giving each service its own PostgreSQL container ensures fault isolation and independent scaling.

---

### Q11: What is the Dual-Write Problem, and does this system have it?
**Answer:** The Dual-Write problem happens when an application must update a database and publish a message to a queue. If the database commit succeeds, but the network fails before publishing to RabbitMQ, the message is lost. In our current implementation, there is a narrow window between PostgreSQL commit and RabbitMQ publish. In a production evolution, we would implement the **Transactional Outbox Pattern**: saving the event in an `outbox` table within the same PostgreSQL transaction, with an asynchronous process (Debezium or a polling worker) publishing it to RabbitMQ.

---

### Q12: How do you verify that your concurrency defense actually works?
**Answer:** We wrote an automated integration test suite in Jest (`tests/booking-concurrency.test.ts`) that runs against real PostgreSQL and Redis containers in Docker and GitHub Actions CI. One test spawns 20 concurrent promises simultaneously racing to reserve the same seat: the test asserts that exactly one promise resolves with `201 Created` and all other 19 reject with `409 Conflict`. We also test overlapping multi-seat groups (e.g. `[42, 43, 44]` vs `[44, 45, 46]`).

---

### Q13: Why did you use Redis for the temporary hold rather than just doing `UPDATE bookings SET status='held'` in PostgreSQL?
**Answer:** Performance and connection scalability. Redis is an in-memory data store capable of handling 100,000+ operations per second with sub-millisecond latency. A relational database disk write involves WAL logging, transaction overhead, and row lock contention. Using Redis as an edge filter absorbs 98% of losing requests instantly, protecting PostgreSQL from connection exhaustion.

---

### Q14: How does Nginx load-balance traffic between your service replicas?
**Answer:** In `nginx/nginx.conf`, we define an `upstream event_servers` block listing `event-service-1:3002` and `event-service-2:3002`. Nginx distributes incoming HTTP requests using round-robin balancing by default. In our System Lab demo, we visualize this by making 50 requests and displaying headers showing alternating servicing container IDs.

---

### Q15: What is the difference between Pessimistic Locking and Optimistic Locking, and why did you choose Pessimistic here?
**Answer:** 
* **Optimistic Locking** uses version columns (`WHERE version = 1`). It assumes conflicts are rare; if a conflict happens, it rolls back and tells the client to retry.
* **Pessimistic Locking** (`SELECT ... FOR UPDATE`) acquires an exclusive lock upfront, making concurrent requests wait.
* Under extreme flash-sale contention (e.g., 50 people fighting for 1 seat), optimistic locking causes 49 transactions to fail and retry repeatedly, generating massive CPU spikes. Pessimistic locking combined with Redis edge filtering is much cleaner: Redis turns away 49 losers immediately, and the single winner completes a deterministic row lock in PostgreSQL.

---

### Q16: How do you handle idempotency during booking confirmation?
**Answer:** If a user clicks 'Confirm' twice, or their network drops and retries:
The confirmation endpoint queries the booking rows. If all rows already have `status = 'confirmed'` and belong to that user, the service immediately returns `200 OK` with `{ message: "already confirmed" }` and skips publishing duplicate notification messages to RabbitMQ.

---

### Q17: What happens if a user requests seats outside the theatre capacity or closed rows?
**Answer:** The Booking Service does not trust frontend inputs. Before touching Redis or the database, it queries the Event Service to fetch the theatre's layout schema, total seats, VIP pricing, and blocked rows. If a user tries to reserve seat 999 or a seat in a closed row, the booking service rejects it immediately with `400 Bad Request` or `409 Conflict`.

---

### Q18: What is the difference between RabbitMQ and Kafka, and why RabbitMQ here?
**Answer:** 
* **Kafka** is a distributed, append-only commit log designed for high-throughput streaming, event replay, and long retention.
* **RabbitMQ** is a traditional message broker designed for complex routing (topic/direct exchanges), per-message acknowledgments, and granular worker queue management.
* For transactional notification dispatch (sending an email or PDF receipt once a ticket is booked), RabbitMQ's native acknowledgments, delay exchanges, and dead-letter queues make it a much better fit than Kafka.

---

### Q19: How do you ensure passwords and tokens are secure?
**Answer:** User passwords are encrypted with `bcrypt` (salt rounds = 10) in `user-service`. Authentication issues signed **JSON Web Tokens (JWT)** with expiration. Protected endpoints require the `Authorization: Bearer <token>` header, verified by an `authMiddleware` that attaches the decoded user ID to the request object.

---

### Q20: How would you scale this system from 1,000 users to 1,000,000 users?
**Answer:**
1. **Redis Cluster with Redlock:** Scale Redis from a single node to a distributed Redis Cluster across availability zones.
2. **Transactional Outbox + Kafka:** Replace direct RabbitMQ publishing with Debezium CDC streaming outbox events to Kafka.
3. **Database Read Replicas:** Route catalog read traffic to PostgreSQL read replicas with connection pooling via PgBouncer.
4. **Virtual Waiting Room:** Place an edge queue (e.g., Cloudflare Waiting Room or AWS SQS token bucket) in front of the reserve endpoint to throttle incoming traffic spikes to the system's exact throughput capacity.

---

## 9. Trade-offs, Architectural Dilemmas & Production Evolutions

| Architectural Decision | Trade-off Made | Why It Was the Right Choice | Production Evolution |
| :--- | :--- | :--- | :--- |
| **Single Redis Node vs. Cluster** | Simpler multi-key Lua scripts; single point of failure if Redis dies. | Redis Cluster requires keys in a multi-key Lua script to hash to the same slot (`{hash_tag}`). | Use hash tags: `{event_id}:seat:{seat_no}` so all seats for an event map to the same Redis shard. |
| **Compensated Workflow vs. 2PC** | Potential temporary stale lock if process crashes between Redis and Postgres. | Two-Phase Commit is blocking, slow, and unsupported natively across heterogeneous stores. | Add a background reconciler worker that cleans up orphan Redis keys without matching Postgres rows. |
| **Direct RabbitMQ Publish** | Potential Dual-Write failure window between Postgres commit and message queue publish. | Simple, zero extra infrastructure components required for MVP. | Implement **Transactional Outbox Pattern** with Debezium and Kafka. |
| **Pessimistic `FOR UPDATE`** | Database threads hold locks for the duration of the confirmation transaction. | Eliminates optimistic retry storms under high contention. | Keep confirmation transaction execution time under 5 milliseconds. |

---

*This guide represents the exact architectural decisions, source code implementations, and operational realities of the `distributed-ticketing-system` codebase.*
