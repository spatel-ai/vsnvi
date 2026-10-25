# ADR-003: Spring Modulith for module boundaries and event delivery

- **Status:** Accepted
- **Date:** 2026-10-25
- **Deciders:** vsnvi-food backend team

## Context

ADR-001 splits the monolith into modules that may use each other only through a public `api`
package and domain events. Two things are needed to make that real:

1. **Boundary enforcement.** Each module is split into `api`, `application`, `domain` and
   `infrastructure` sub-packages, so services must be `public`. Java visibility therefore can't
   stop another module from importing internal classes.
2. **Reliable cross-module events.** Events must survive a crash between commit and the
   listener running (at-least-once delivery). The first draft planned a hand-written
   transactional outbox.

## Decision

We use **Spring Modulith 2.x** for both.

- **Boundaries:**
  - Each direct sub-package of `com.vsnvi.vsnvi_food` is an application module.
  - `api/package-info.java` declares `@NamedInterface("api")`. `common` is an open module.
  - A test runs `ApplicationModules.of(VsnviFoodApplication.class).verify()`. It fails the build
    on access to another module's internal types and on any **dependency cycle**.
- **Calls go down, events go up.** Modulith counts listening to an event as a dependency, the same
  as calling a facade. So the module graph must be acyclic, and:
  - a module calls the `<Module>Api` of modules it depends on to make them act;
  - it publishes events for the modules that depend on it;
  - it never listens to events of a module that depends on it.

  For example, `order` calls `PaymentApi.cancelPayment` instead of `payment` listening to
  `OrderCancelled`, and `delivery` calls `OrderApi.markPickedUp` after which `order` publishes
  `OrderPickedUp`. The full graph is in [ARCHITECTURE.md §2](../ARCHITECTURE.md#2-modules).
- **Events:** the **Event Publication Registry** (JPA) replaces the hand-written outbox. It
  records one entry per listener in `event_publication` in the publishing transaction and
  resubmits incomplete entries. Listeners use `@ApplicationModuleListener` (after commit, async,
  own transaction) and must be idempotent.

## Alternatives considered

### ArchUnit rules + hand-written outbox
- **Pros:** ArchUnit can express "only `<Module>Api`, its DTOs and events" precisely; no extra
  runtime dependency; full control over the outbox table and relay.
- **Cons:** two separate mechanisms to build and maintain; the outbox relay, retries and cleanup
  are custom code that Modulith already provides; no module-level test support.
- **Rejected:** Modulith covers boundaries, events, module tests (`@ApplicationModuleTest`,
  `Scenario`) and later broker externalization in one tool.

### Conventions only
- **Rejected:** boundaries erode without an automated check.

## Consequences

- **Positive:**
  - The build fails on boundary violations and cycles.
  - Event delivery is reliable without custom outbox code.
  - Events can later be sent to Kafka or RabbitMQ with `@Externalized`.
- **Negative:**
  - Some flows that would naturally be events must be facade calls to keep the graph acyclic.
  - We must create the `event_publication` table with our own Flyway migration (copied from the
    Modulith version's schema) and re-check it on every Modulith upgrade.
  - When the verify test is first added, confirm that `api.web` isn't exposed as part of the
    `api` named interface.
