# ADR-004: Real-time tracking with Redis GEO and STOMP, pushed by `delivery`

- **Status:** Accepted
- **Date:** 2026-10-25
- **Deciders:** vsnvi-food backend team

## Context

Customers need to see their delivery partner move on a map, and the system needs to find the
nearest available partner when food is ready. Partners send a location ping about every 5
seconds. The first draft used Server-Sent Events and an in-memory location store, assigned
partners on `OrderAccepted`, and left open which module pushes order-status updates.

## Decision

- **Transport: WebSocket with STOMP** at `/ws`. Customers subscribe to
  `/topic/orders/{orderId}/tracking`, restaurants to `/topic/restaurants/{id}/orders`, and partners
  receive offers on `/user/queue/offers`. A single instance uses the simple broker; multiple
  instances add Redis pub/sub or a broker relay.
- **Location: Redis GEO.** Online, approved, available partners are in the `delivery:partners:geo`
  set, latest positions in a hash with a TTL, and a sweeper removes expired members. Assignment
  uses `GEOSEARCH`. Postgres only keeps sampled history.
- **Assignment starts on `OrderReadyForPickup`.** The event carries the restaurant, customer and
  drop-off coordinates, so `delivery` needs no extra calls. If no partner accepts within the
  configured time, `delivery` publishes `DeliveryAssignmentStalled` and admins decide.
- **Only `delivery` publishes to the tracking topic, starting at `READY_FOR_PICKUP`.** It sends
  `status` (READY_FOR_PICKUP, PARTNER_ASSIGNED, PICKED_UP, DELIVERED, CANCELLED), `location` and
  `eta` messages. Earlier statuses (PLACED, ACCEPTED, PREPARING) are not pushed; the customer app
  polls `GET /orders/{id}`. `notification` pushes only the restaurant dashboard and in-app
  notifications.
- Details: [ARCHITECTURE.md §6](../ARCHITECTURE.md#6-real-time-tracking-redis-geo--websocket-stomp).

## Alternatives considered

### Server-Sent Events + in-memory store
- **Pros:** simpler; plain HTTP.
- **Cons:** one-way only; an in-memory store doesn't work across instances and has no geo search.
- **Rejected:** we need partner offers over the same channel and nearest-partner search.

### Assign on `OrderAccepted`
- **Pros:** the partner can arrive while the food is being prepared.
- **Cons:** partners wait at the restaurant, and an assignment is wasted if the order is cancelled
  during preparation.
- **Rejected for v1.** Earlier assignment based on `prepTimeMinutes` can be added later without
  changing the module graph.

### `notification` pushes every status change
- **Rejected:** it would split one customer screen across two publishers; v1 keeps live
  tracking in the module that owns the delivery.

## Consequences

- **Positive:** one module owns everything the customer sees live; Postgres isn't hit by every
  ping; nearest-partner search is a single Redis command.
- **Negative:**
  - The food may wait for the partner after it's ready.
  - Customers see earlier status changes only by polling.
  - Redis is required in every environment, and scaling out needs a cross-instance relay and
    single-run scheduled jobs (both Phase 5).
