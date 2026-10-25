---
paths:
  - "src/test/**"
---

# Testing rules

Details: `docs/ARCHITECTURE.md` (state machine, events, payment flow).

| Layer | Tool | Covers |
|---|---|---|
| Unit (domain, application) | JUnit 5 + AssertJ, no Spring | aggregates, state machine, pricing, value objects |
| Web | `@WebMvcTest` + MockMvc | validation, status codes, ProblemDetail body, security |
| Persistence | `@DataJpaTest` + Testcontainers | custom queries, mappings, migrations |
| Module | `@ApplicationModuleTest` + Testcontainers, `Scenario` for events | one module plus its dependencies, published and consumed events |
| Flow | `@SpringBootTest` + Testcontainers | end-to-end use cases, WebSocket |

- Name test methods `should_<expected>_when_<condition>`.
- Use Testcontainers (Postgres, Redis) through the existing `TestcontainersConfiguration`. Never use H2 or embedded Redis.
- Never call real external systems. Use `FakePaymentGateway`, a fake mail sender and a fixed `Clock`.
- Every new endpoint needs at least one happy-path test and one failure-path test, and the failure test must assert the error `code`.
- Every allowed **and** forbidden order state transition must be tested.
- Every coupon rule needs a test, and so does concurrent redemption of a coupon's last use.
- Payment webhooks need tests for a valid signature, an invalid signature, a duplicate event ID and an amount mismatch.
- Every secured endpoint needs tests for unauthenticated (401), wrong role (403) and another user's resource (404).
- Every event consumer needs a test showing it is idempotent (the same event delivered twice).
- A test must fetch `/v3/api-docs`, assert that every `operationId` is unique and every operation has a `@Tag`, and write `target/openapi.json`.
- `ApplicationModules.of(VsnviFoodApplication.class).verify()` must pass. Never weaken it to make a test green.
