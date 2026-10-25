# Progress — vsnvi-food

> Drafted from defaults — review and adjust. Tick items as they are merged.

Each module follows the same definition of done: **migration → domain → service → API → tests**.

## Phase 0 — Foundation

- [ ] Project docs (architecture, conventions, API guidelines, ADR-001)
- [ ] `application.yaml` profiles (`local`, `test`, `prod`) and Postgres config
- [ ] `shared` kernel
  - [ ] `BaseEntity` (UUID id, auditing, version)
  - [ ] `Money` value object and JPA mapping
  - [ ] `DomainException` hierarchy and `ErrorCode`
  - [ ] Global `@RestControllerAdvice` returning `ProblemDetail`
  - [ ] Pagination envelope (offset and cursor)
  - [ ] Trace ID filter (MDC)
- [ ] Security baseline (stateless JWT filter, password encoding, role setup)
- [ ] Idempotency-Key support (storage, filter or aspect)
- [ ] Outbox table and relay skeleton
- [ ] Flyway baseline migration
- [ ] Testcontainers integration test base
- [ ] CI pipeline (`./mvnw verify`)

## Phase 1 — Users, restaurants, menus

### identity
- [ ] Migration: users, roles, addresses, refresh_tokens
- [ ] Domain: `User`, `Address`, `RefreshToken`
- [ ] Service: register, login, refresh, logout, profile, addresses
- [ ] API: `/auth/*`, `/users/me`, `/users/me/addresses`
- [ ] Event: `UserRegistered`
- [ ] Tests

### restaurant
- [ ] Migration: restaurants, outlets, opening_hours
- [ ] Domain: `Restaurant`, `OpeningHours`, serviceable radius
- [ ] Service: create, update, open/close, approval status
- [ ] API: `/restaurants`, search by location, cuisine and rating
- [ ] Tests

### catalog
- [ ] Migration: menu_categories, menu_items, variants, add_ons
- [ ] Domain: `MenuItem`, `Variant`, `AddOn`, availability
- [ ] Service: menu CRUD, availability toggle
- [ ] Public facade: `CatalogApi` (pricing lookup for cart and order)
- [ ] API: `/restaurants/{id}/menu`, `/menu-items`, `/menu-categories`
- [ ] Tests

## Phase 2 — Cart, orders, payments

### cart
- [ ] Migration: carts, cart_items
- [ ] Domain: single-restaurant rule, totals calculation
- [ ] Service: add, update, remove, clear
- [ ] API: `/cart`
- [ ] Tests

### order
- [ ] Migration: orders, order_items, order_status_history
- [ ] Domain: `Order` aggregate, state machine, pricing snapshot
- [ ] Service: checkout, cancel, owner actions (accept, reject, preparing, ready)
- [ ] Events: `OrderPlaced`, `OrderAccepted`, `OrderRejected`, `OrderReadyForPickup`, `OrderCancelled`
- [ ] Listeners: payment and delivery events update status
- [ ] API: `/orders`, `/users/me/orders`, `/restaurants/{id}/orders`
- [ ] Tests (including every allowed and forbidden transition)

### payment
- [ ] Migration: payments, payment_attempts, refunds, processed_webhooks
- [ ] `PaymentGateway` interface and fake implementation
- [ ] Razorpay (or Stripe) implementation
- [ ] Payment intent creation
- [ ] Webhook: signature verification and idempotent processing
- [ ] Events: `PaymentSucceeded`, `PaymentFailed`, `RefundIssued`
- [ ] Payment timeout job (15 minutes)
- [ ] Refund on reject or cancel
- [ ] Cash on delivery flow
- [ ] API: `/payments/*`
- [ ] Tests

## Phase 3 — Delivery and live tracking

### delivery
- [ ] Migration: delivery_partners, delivery_tasks, delivery_location_history
- [ ] Domain: `DeliveryTask` lifecycle, partner availability
- [ ] Assignment: nearest available partner, offer timeout, fallback
- [ ] Events: `DeliveryAssigned`, `OrderPickedUp`, `OrderDelivered`
- [ ] API: `/delivery/availability`, `/delivery-tasks/*`
- [ ] Tests

### tracking
- [ ] Location ping endpoint, validation and rate limiting
- [ ] `LocationStore` (in-memory) for latest positions
- [ ] Sampled location history persistence
- [ ] `EtaCalculator` (haversine)
- [ ] SSE emitter registry and `GET /orders/{id}/tracking`
- [ ] Tests

## Phase 4 — Notifications and reviews

### notification
- [ ] Migration: notification_log
- [ ] Email templates (welcome, order placed, out for delivery, delivered, refund)
- [ ] Event listeners for the event catalog
- [ ] Tests (fake mail sender)

### review
- [ ] Migration: reviews
- [ ] Domain: one review per order, only after `DELIVERED`
- [ ] Restaurant rating aggregation
- [ ] API: `/orders/{id}/reviews`, `/restaurants/{id}/reviews`
- [ ] Tests

### admin
- [ ] Restaurant approval and suspension
- [ ] User blocking
- [ ] Order search

## Phase 5 — Hardening

- [ ] Outbox relay with retries and a dead-letter state
- [ ] Rate limiting (auth, location pings)
- [ ] Caching (restaurant search, menus)
- [ ] Observability: metrics, structured logging, dashboards
- [ ] Module boundary verification (Spring Modulith or ArchUnit)
- [ ] Security review (OWASP checks, secrets management)
- [ ] Load testing (checkout and tracking)
- [ ] Redis for location store and SSE fan-out (multi-instance)
- [ ] Deployment setup (Docker image, environment config)
