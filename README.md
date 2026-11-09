# Distributed Ticket Reservation System

[![CI](https://github.com/Karan-Brar-01/distributed-ticketing-system/actions/workflows/ci.yml/badge.svg)](https://github.com/Karan-Brar-01/distributed-ticketing-system/actions/workflows/ci.yml)
[![Docker](https://img.shields.io/badge/Docker_Compose-10_Containers-blue?style=for-the-badge&logo=docker)](docker-compose.yml)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?style=for-the-badge&logo=postgresql)](docker-compose.yml)
[![Redis](https://img.shields.io/badge/Redis-7-red?style=for-the-badge&logo=redis)](docker-compose.yml)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3-orange?style=for-the-badge&logo=rabbitmq)](docker-compose.yml)

> 🐳 **LOCAL DOCKER DEPLOYMENT:**  
> Run locally with `docker-compose up --build` across 10 Docker containers behind Nginx reverse proxy.  
> * **Customer Booking Journey:** `http://localhost`  
> * **Engineering System Lab:** `http://localhost/lab` *(50-concurrency seat races & Nginx load balancing)*  
> * **Seeded Demo Accounts:** `user@demo.com` / `user123` | **Admin:** `admin@demo.com` / `admin123`

---

A containerized ticket-booking platform designed to prevent double booking under extreme concurrent seat reservations. It features two complementary interfaces: a full customer-facing booking journey and an engineering System Lab for observing distributed locks, cache telemetry, and replica load balancing underneath it.

## Quick Links

| Resource | Link |
| :--- | :--- |
| 🌐 **Web Application** | `http://localhost` *(run locally with Docker Compose)* |
| 🔬 **System Engineering Lab** | `http://localhost/lab` |
| 🧪 **Concurrency Integration Tests** | [`services/booking-service/tests/booking-concurrency.test.ts`](services/booking-service/tests/booking-concurrency.test.ts) |
| 🐳 **Docker Compose Topology** | [`docker-compose.yml`](docker-compose.yml) |
| 🔀 **Nginx Load Balancer Config** | [`nginx/nginx.conf`](nginx/nginx.conf) |

---

## Demo modes

### Product Mode

Product Mode is the default customer experience at `http://localhost`:

1. Browse the event catalog and open an event.
2. Inspect the live seat map and select an available seat.
3. Sign in with the seeded demo account if needed (`user@demo.com` / `user123`).
4. Reserve the seat through `POST /api/bookings/reserve`.
5. Confirm before the visible hold countdown expires through `POST /api/bookings/confirm`.
6. See explicit confirmed, conflict, expired-hold, and already-booked states.

Events are priced in Indian rupees. Each event uses one of two theatre layouts with labelled rows, visible staircase aisles, premium VIP rows at twice the standard price, and one closed row. The booking service obtains this seating configuration from the event service and enforces the closed row itself; hiding or altering frontend controls cannot make those seats reservable.

The browser never invents a successful reservation. Seat availability comes from the booking service, a hold is shown only after the Redis-backed reserve endpoint succeeds, and success is shown only after the PostgreSQL-backed confirmation endpoint responds successfully.

### System Lab Mode

Open **System Lab** in the main navigation or visit [http://localhost/lab](http://localhost/lab). This preserves the original system-design controls in a dedicated engineering view:

- Send 50 concurrent catalog requests and visualize Nginx replica distribution.
- Probe Redis read-through cache headers, upstream identity, and response time.
- Race 50 authenticated reservation requests for one seat and verify one winner with conflict responses for the rest.
- Inspect recent API request latency, route, status, and serving replica.

The project focuses on four engineering problems:

- Temporary distributed seat holds with Redis
- Transactional booking confirmation in PostgreSQL
- Asynchronous notifications with RabbitMQ
- Horizontal event-service scaling behind Nginx

## Architecture

```text
                         Nginx
                           │
          ┌────────────────┼─────────────────┐
          │                │                 │
       User API       Event API ×2      Booking API
          │                │                 │
      Users DB          Events DB        Bookings DB
                           │                 │
                         Redis ←──────── Seat holds
                                             │
                                             ▼
                                         RabbitMQ
                                             │
                                             ▼
                                     Notification Worker
```

Each API owns its PostgreSQL database. Nginx exposes one entry point and round-robins event requests across two service replicas. Redis is shared by the event cache and the booking service's expiring seat locks.

The booking service reads an event's capacity, theatre layout, row restrictions, and INR pricing from the existing event service before exposing its seat map or accepting a reservation. Redis locking and PostgreSQL confirmation remain unchanged after this validation step.

## The problem

Two users can select the same seat at almost the same instant. A normal read-then-write flow allows both requests to observe that the seat is free and can create two successful bookings. This system closes that race at three layers:

1. **Redis contention control:** one-seat holds use the same Lua path as grouped holds; the script checks every `seat:{event}:{seat}` key and creates all requested locks with one ownership token and TTL only when every key is free.
2. **PostgreSQL transaction:** confirmation locks the relevant booking rows with `SELECT ... FOR UPDATE` before changing `held` to `confirmed`.
3. **Database invariant:** a partial unique index permits historical held, expired, and cancelled rows, but guarantees at most one confirmed booking for an event seat.

```sql
CREATE UNIQUE INDEX unique_confirmed_seat
ON bookings (event_id, seat_number)
WHERE status = 'confirmed';
```

Redis handles temporary contention, the transaction handles confirmation concurrency, and the database constraint is the final invariant.

## Booking flow

```text
User selects seat 42 (or seats 42, 43, and 44)
        │
        ▼
POST /api/bookings/reserve
        │
        ├─ validate capacity, blocked rows, pricing, and confirmed seats
        ├─ atomically acquire every Redis hold with one Lua script
        └─ create every HELD booking row in one PostgreSQL transaction
        │
        ▼
POST /api/bookings/confirm
        │
        ├─ validate user and ownership of every hold key
        ├─ lock all matching rows in one PostgreSQL transaction
        ├─ transition the complete group HELD → CONFIRMED
        ├─ release only Redis locks still owned by the request token
        └─ publish one booking.confirmed event per seat
        │
        ▼
RabbitMQ → notification worker → simulated ticket and email
```

Confirmation is idempotent for the owner: a repeated request returns the existing confirmed booking without publishing a second event.

## Atomic multi-seat API

The reserve endpoint remains backwards compatible. Existing clients can continue sending `seat_number`; grouped reservations send `seat_numbers`:

```json
{
  "event_id": "event-id",
  "seat_numbers": [42, 43, 44]
}
```

A successful grouped response contains one reservation ID, one private hold token shared by the group, the durable booking row for each seat, and backend-calculated seat metadata and pricing:

```json
{
  "reservation": {
    "id": "reservation-uuid",
    "event_id": "event-id",
    "status": "held",
    "booking_ids": ["booking-1", "booking-2", "booking-3"],
    "seat_numbers": [42, 43, 44],
    "seats": [
      { "seat_number": 42, "seat_type": "standard", "price": 500, "currency": "INR" }
    ],
    "total_price": 1500,
    "currency": "INR"
  },
  "hold_token": "private-ownership-token",
  "hold_ttl_seconds": 600
}
```

Confirm the group together:

```json
{
  "reservation_id": "reservation-uuid",
  "hold_token": "private-ownership-token"
}
```

The legacy confirmation body, `{ "booking_id": "...", "hold_token": "..." }`, remains valid. If that booking belongs to a group, the complete group is confirmed. A repeated confirmation returns `already confirmed` and does not republish notifications.

### Why acquisition is all-or-nothing

The Redis Lua script receives every seat key in `KEYS`. Redis executes the script atomically: it first checks the complete key set, returns failure without writing if any key exists, and only then writes every key with the same token and expiry. Concurrent groups that overlap on even one seat therefore cannot both succeed, and the losing group acquires none of its other seats.

After Redis succeeds, Prisma expires abandoned historical hold rows and inserts every new `held` row inside one PostgreSQL transaction. A failed insert rolls the entire database transaction back. The booking service then runs a multi-key compare-and-delete Lua script; it removes only keys whose value still equals this request's ownership token, so it cannot delete a lock acquired later by another request.

Redis and PostgreSQL are still separate systems, so this is a compensated workflow rather than a distributed transaction. A process crash after Redis acquisition but before database commit/compensation can leave temporary locks until their TTL expires. PostgreSQL's partial unique index remains the durable final guard against more than one confirmed booking per event seat.

### Manual curl test

Start the stack and get a JWT using the seeded demo account:

```bash
curl -s http://localhost/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"user@demo.com","password":"user123"}'
```

Copy the returned `token`, choose an event ID from `GET /api/events`, then reserve a group:

```bash
curl -i http://localhost/api/bookings/reserve \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{"event_id":"<event-id>","seat_numbers":[42,43,44]}'
```

Copy `reservation.id` and `hold_token` from the `201` response and confirm all seats together:

```bash
curl -i http://localhost/api/bookings/confirm \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{"reservation_id":"<reservation-id>","hold_token":"<hold-token>"}'
```

Send the same confirmation again to verify the idempotent `already confirmed` response. To observe contention, send two reserve requests concurrently with overlapping groups such as `[42,43,44]` and `[44,45,46]`; exactly one returns `201` and the other returns `409` without holding its non-overlapping seats.

## Concurrency tests

The booking integration suite uses real PostgreSQL and Redis. It covers:

- Atomic three-seat reservation and backend-owned group pricing
- Duplicate, blocked, out-of-range, already-confirmed, and already-held group rejection
- No partial Redis locks or PostgreSQL rows after a failed group reservation
- Overlapping concurrent groups with exactly one winner
- Database rollback plus ownership-safe Redis compensation
- Atomic group confirmation and repeated-confirmation idempotency
- 20 users racing for one seat: one hold succeeds and 19 conflict
- Backend rejection of every seat in the configured closed theatre row
- Backend-provided row labels, VIP tiers, and INR prices
- Concurrent reservations for different seats
- The partial unique index under competing confirmations
- Repeated confirmation by the same user
- Expired holds and incorrect hold tokens
- Cross-user confirmation attempts
- Reservations of already-confirmed seats
- Compare-and-delete Redis lock ownership

## Asynchronous notifications

`booking.confirmed` messages are published persistently to a durable topic exchange. The notification consumer acknowledges only after ticket and email simulation succeeds.

Failed deliveries follow a bounded path:

```text
notifications queue → retry queue (5-second TTL) → notifications queue
        │                                              │
        └──────── after 3 failed retries ──────────────┘
                              │
                              ▼
                       dead-letter queue
```

Retry delay, limit, exchange, and queue names are configurable through environment variables.

## Tech stack

| Layer | Technology |
|---|---|
| Edge | Nginx |
| Frontend | React, TypeScript, Vite, Tailwind CSS |
| APIs | Node.js, Express, TypeScript |
| Persistence | PostgreSQL, Prisma |
| Distributed locks/cache | Redis |
| Messaging | RabbitMQ |
| Runtime | Docker Compose |
| CI | GitHub Actions |

## Run locally

Requirements: Docker Desktop and Docker Compose.

```bash
cp .env.example .env
docker compose up --build
```

Open [http://localhost](http://localhost) for Product Mode or [http://localhost/lab](http://localhost/lab) for System Lab Mode. RabbitMQ management is available at [http://localhost:15672](http://localhost:15672).

This release replaces Prisma `db push` with versioned migrations. If you previously ran the old schema locally, recreate the disposable development volumes once:

```bash
docker compose down -v
docker compose up --build
```

## Run the booking tests

Start only the required infrastructure:

```bash
docker compose up -d bookings_db redis
```

Then run the migration and tests:

```bash
cd services/booking-service
npm ci
DATABASE_URL=postgresql://ticketmaster:ticketmaster@127.0.0.1:5434/bookings_db npm run prisma:migrate:deploy
TEST_DATABASE_URL=postgresql://ticketmaster:ticketmaster@127.0.0.1:5434/bookings_db \
TEST_REDIS_URL=redis://127.0.0.1:6379 \
npm test
```

GitHub Actions builds the frontend and all four services on every push and pull request, then runs the booking concurrency suite with PostgreSQL and Redis service containers.

## Limitations and production follow-ups

- Payment and email/PDF delivery are simulations.
- Authentication uses a shared JWT secret rather than a dedicated identity platform.
- The database commit and RabbitMQ publish are separate operations, leaving a dual-write failure window. A production version should use a **transactional outbox**, committing the booking and outbox event atomically before a worker publishes it.
- Redis acquisition and the PostgreSQL transaction are coordinated with compensation, not a distributed commit. A process crash in that narrow window can leave temporary locks until their TTL expires; it cannot create duplicate confirmed seats because PostgreSQL retains the final uniqueness invariant.
- Retry and dead-letter queues preserve failed notification events, but there is no operator replay UI.
- The deployment is intentionally Docker Compose-based; Kubernetes, Kafka, service meshes, and event sourcing are outside this project's scope.
