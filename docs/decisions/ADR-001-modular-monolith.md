# ADR-001: Modular monolith architecture

- **Status:** Accepted
- **Date:** 2026-10-25
- **Deciders:** vsnvi-food backend team

## Context

We are building a food-delivery backend that covers customers, restaurants, delivery partners,
Razorpay payments and live tracking. The product is early-stage and the team is small. We need to:

- ship features quickly and change domain models often while we learn;
- keep business areas (orders, payments, delivery and so on) cleanly separated, so the codebase
  doesn't turn into a "big ball of mud";
- keep operations simple: one deployable, one database, and easy local development and testing;
- keep the option to split out services later, if scale or team size demands it.

## Decision

We will build a **modular monolith** with Spring Boot:

- **Deployment:** a single deployable application, a single PostgreSQL database, and Redis for
  geo indexing and real-time fan-out.
- **Modules:** `auth`, `user`, `restaurant`, `menu`, `cart`, `order`, `payment`, `delivery`,
  `promotion` and `notification`, plus a small `common` kernel.
- **Ownership:** each module owns its tables. It exposes only its public surface: the
  `<Module>Api` facade interface, the DTOs that facade uses, and its events.
  No module accesses another module's entities, repositories or tables.
- **Communication:**
  - Modules depend on each other in one direction only (an acyclic graph).
  - A module calls the `<Module>Api` facades of the modules it depends on.
  - It notifies the modules that depend on it through **domain events**, which are consumed
    after commit.
- **Reliability:** Spring Modulith's **Event Publication Registry** records every event in the
  same transaction, for at-least-once delivery. Event externalization gives us a migration path
  to a message broker.
- **Boundary enforcement:** **Spring Modulith**. `ApplicationModules.verify()` runs in the test
  suite and rejects internal access and dependency cycles. The public surface is declared with
  `@NamedInterface`. See [ADR-003](ADR-003-spring-modulith.md).

## Alternatives considered

### Microservices from day one
- **Pros:** independent deployment and scaling, freedom to choose technology per service, and
  fault isolation.
- **Cons:**
  - High operational overhead: discovery, distributed tracing, many pipelines.
  - Checkout and payment would need distributed transactions.
  - Network failure modes and slower iteration.
  - Domain boundaries aren't stable yet, so getting the cuts wrong would be expensive.
- **Rejected:** too early for our team size and stage.

### Traditional layered monolith (controller / service / repository across the whole app)
- **Pros:** the simplest way to start, and familiar to everyone.
- **Cons:** there are no business boundaries, so coupling grows quickly. The result is hard to
  reason about, test or split later.
- **Rejected:** it gives up the boundaries we will need as the product grows.

## Consequences

### Positive
- Fast development, simple deployment and debugging, and local transactions where we need them.
- Each module has clear ownership and boundaries, and can be understood and tested on its own.
- Event-driven integration keeps modules loosely coupled and ready to be extracted.
- A single database keeps consistency and reporting straightforward.

### Negative
- **Everything scales together.** A hot spot such as location pings scales the whole application.
  Redis GEO and STOMP fan-out reduce the load on Postgres and the JVM, but don't remove this.
- **Boundary rules add friction:** some flows that would naturally be events must be facade calls
  instead, to keep the graph acyclic.
- **The database is a shared failure domain**, and schema ownership has to be policed.
- **In-process events share the JVM**, so a slow listener affects the whole app. Asynchronous
  `@ApplicationModuleListener`s and the Event Publication Registry mitigate this.

### When to extract a module into a service
Consider extracting a module when **one or more** of these are true:

1. It needs to scale independently. **delivery/tracking** is the likely first candidate.
2. A separate team owns it and is blocked by the shared release cadence.
3. It has different availability, security or compliance requirements, e.g. PCI scope for
   **payment**.
4. Its load or failures measurably degrade other modules.

Prerequisites for extraction:
- The module communicates only through its `<Module>Api` facade and its events.
- Its tables have no foreign keys to other modules.
- Its events are externalized to a real broker, and STOMP fan-out goes through a broker relay or
  Redis pub/sub.
