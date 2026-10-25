# ADR-001: Modular monolith architecture

- **Status:** Accepted
- **Date:** 2026-10-25
- **Deciders:** vsnvi-food backend team

> Drafted from defaults — review and adjust.

## Context

We are building a food-delivery backend (customers, restaurants, delivery partners, payments,
live tracking). The product is early-stage and the team is small. We need to:

- ship features quickly and change domain models often while we learn;
- keep business areas (orders, payments, delivery…) cleanly separated so the codebase doesn't
  become a "big ball of mud";
- keep operations simple: one deployable, one database, simple local development and testing;
- keep the option to split out services later, when scale or team size demands it.

## Decision

We will build a **modular monolith** with Spring Boot:

- A single deployable application and a single PostgreSQL database.
- The code is split into business modules: `identity`, `restaurant`, `catalog`, `cart`, `order`,
  `payment`, `delivery`, `notification`, `review`, plus a small `shared` kernel.
- Each module owns its tables and exposes only a public `api` facade and its domain events.
  There is no cross-module access to entities, repositories or tables.
- Synchronous cross-module calls go through `api` facades. Side effects go through **domain events**
  (Spring application events, consumed after commit).
- Events are recorded in a **transactional outbox** for at-least-once delivery, which also gives
  us a migration path to a message broker.
- Boundaries are enforced by package conventions now, and by automated verification
  (Spring Modulith or ArchUnit) later.

## Alternatives considered

### Microservices from day one
- **Pros:** independent deployment and scaling, technology freedom, fault isolation.
- **Cons:** large operational overhead (service discovery, distributed tracing, many pipelines),
  distributed transactions for checkout and payment, network failure modes, slower iteration.
  Domain boundaries aren't stable yet, so wrong cuts would be expensive to fix.
- **Rejected** as premature for the current team size and stage.

### Traditional layered monolith (controller / service / repository across the whole app)
- **Pros:** simplest to start, familiar.
- **Cons:** no business boundaries, so coupling grows quickly (any service calls any repository),
  and it becomes hard to reason about, test or extract later.
- **Rejected** because it gives up the boundaries we will need as the product grows.

## Consequences

### Positive
- Fast development, simple deployment, simple debugging, and local transactions where needed.
- Clear ownership and boundaries per module, so each module is easy to understand and test on its own.
- Event-driven integration keeps modules loosely coupled and ready for extraction.
- One database makes consistency and reporting straightforward.

### Negative
- Everything scales together. A hot spot (e.g. location pings) scales the whole app.
- Boundaries depend on discipline until automated verification is in place.
- A single database is a shared failure domain, and schema ownership must be policed.
- In-process events mean listeners share the JVM. A slow listener affects the app, so we mitigate with
  async execution and the outbox.

### When to extract a module into a service
Consider extracting a module when **one or more** of these hold:

1. It needs to scale independently (likely first: **delivery/tracking**).
2. A separate team owns it and is blocked by the shared release cadence.
3. It has different availability, security or compliance requirements (e.g. **payment** PCI scope).
4. Its load or failures measurably degrade other modules.

Prerequisites for extraction:
- The module communicates only through its `api` and events.
- Its tables have no foreign keys to other modules.
- The outbox relay publishes to a real broker.
