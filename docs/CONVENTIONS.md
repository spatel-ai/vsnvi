# Coding Conventions — vsnvi-food

> Drafted from defaults — review and adjust.

## 1. Package structure per module

Every module lives under `com.vsnvi.vsnvi_food.<module>` and uses the same layout:

```
com.vsnvi.vsnvi_food
├── VsnviFoodApplication.java
├── shared/                      # shared kernel: Money, BaseEntity, error model, outbox, security utils
└── order/                       # example module
    ├── api/                     # PUBLIC: facade interface + DTO records other modules may use
    │   ├── OrderApi.java
    │   └── OrderSummaryDto.java
    ├── web/                     # REST controllers + request/response records
    │   ├── OrderController.java
    │   ├── PlaceOrderRequest.java
    │   └── OrderResponse.java
    ├── application/             # use cases / services, transactions, event publishing & listeners
    │   ├── OrderService.java
    │   └── OrderEventListener.java
    ├── domain/                  # entities, value objects, enums, domain events, repository interfaces
    │   ├── Order.java
    │   ├── OrderStatus.java
    │   ├── OrderRepository.java
    │   └── event/OrderPlaced.java
    └── infrastructure/          # JPA/Spring Data impls, external clients, config
        └── persistence/...
```

Rules:

- **Only `api/` and `domain/event/` may be referenced by other modules.** Everything else is internal.
  Make classes package-private where Java allows it.
- `web` → `application` → `domain`. `infrastructure` implements `domain` interfaces.
  `domain` depends on nothing from Spring Web, and on nothing else in the module except `shared`.
- The `api` facade is implemented in `application` (for example, `OrderService implements OrderApi`
  or a dedicated `OrderApiImpl`).
- Module list: `identity`, `restaurant`, `catalog`, `cart`, `order`, `payment`, `delivery`,
  `notification`, `review`, `shared`.

## 2. Coding rules

### General

- Java 25. Use records, sealed types, pattern matching and `var` where they make the code clearer.
- **Constructor injection only.** No field `@Autowired`. Use `@RequiredArgsConstructor` with `final` fields.
- No business logic in controllers. Controllers validate, map and delegate.
- **Never return entities from controllers.** Map them to response records.
- `@Transactional` goes on application services only. Use `readOnly = true` for queries.
- No `null` returns from public methods. Use `Optional` for "may be absent" lookups and throw
  `NotFoundException` in services.
- Use `Instant` for timestamps, in UTC. Never `LocalDateTime` for instants.
- Use `Money` (a `BigDecimal` amount plus a currency) for all monetary values. **Never `double`/`float`.**

### Entities (JPA)

- Extend `shared.BaseEntity`: `UUID id`, `Instant createdAt`, `Instant updatedAt`
  (JPA auditing), and optionally `@Version long version`.
- `@Version` is mandatory on `Order`, `Payment`, `Cart` and `DeliveryTask`.
- Lombok on entities: `@Getter`, `@NoArgsConstructor(access = PROTECTED)`, and `@Builder` when useful.
  **No `@Data`, `@Setter` or `@EqualsAndHashCode` on entities.** State changes go through intention-revealing
  methods (`order.accept()`), which enforce invariants.
- Lazy fetching by default. Avoid bidirectional relations unless needed. No cross-module relations;
  store the other aggregate's UUID instead.
- Enums are persisted with `@Enumerated(EnumType.STRING)`.

### DTOs and events

- Requests, responses, API DTOs and domain events are **records**.
- Request records use Bean Validation annotations (`@NotBlank`, `@Positive`, `@Valid`, …).
  Controllers use `@Valid @RequestBody`.
- Events: past-tense names (`OrderPlaced`), immutable, carry IDs plus minimal data, and include
  `occurredAt`.

### Errors

- Domain/business errors extend `shared.error.DomainException(ErrorCode code, String message)`.
- Standard subclasses: `NotFoundException` (404), `ConflictException` / `InvalidStateException` (409),
  `ForbiddenException` (403), `BusinessRuleException` (422).
- One global `@RestControllerAdvice` maps exceptions to `ProblemDetail`
  (format in [API_GUIDELINES.md](API_GUIDELINES.md)).
- Never swallow exceptions. Log at the boundary once, with context.

### Logging

- SLF4J via Lombok `@Slf4j`. Use parameterized messages (`log.info("Order {} placed", orderId)`).
- Never log secrets, tokens, passwords, full card or UPI data, or full addresses.

## 3. Naming

| Kind                   | Convention                                   | Example                               |
|------------------------|----------------------------------------------|---------------------------------------|
| Controller             | `<Resource>Controller`                       | `OrderController`                     |
| Application service    | `<Resource>Service`                          | `OrderService`                        |
| Public facade          | `<Module>Api`                                | `CatalogApi`                          |
| Repository             | `<Entity>Repository`                         | `OrderRepository`                     |
| Request / response     | `<Action><Resource>Request`, `<Resource>Response` | `PlaceOrderRequest`, `OrderResponse` |
| Domain event           | Past-tense noun phrase                       | `PaymentSucceeded`                    |
| Event listener         | `<Module>EventListener`                      | `DeliveryEventListener`               |
| Exception              | `<Reason>Exception`                          | `InvalidOrderStateException`          |
| Tables                 | snake_case, plural                           | `orders`, `order_items`               |
| Columns                | snake_case                                   | `restaurant_id`, `created_at`         |
| Constants              | UPPER_SNAKE_CASE                             | `MAX_CART_ITEMS`                      |

## 4. Database and migrations

- Flyway migrations live in `src/main/resources/db/migration`.
- File name: `V{n}__{module}_{description}.sql`, for example `V3__order_create_orders.sql`.
- **Never edit a migration that has been applied** to a shared environment. Add a new one.
- `spring.jpa.hibernate.ddl-auto=validate`. Hibernate never creates the schema.
- Every table has a `UUID` PK, `created_at` and `updated_at` (`timestamptz`), and FK constraints *within*
  a module only.
- Money columns are `NUMERIC(12,2)` plus `currency CHAR(3)`.
- Add indexes for every FK and every column used in frequent filters (e.g. `orders(customer_id, created_at)`).

## 5. Testing

| Layer          | Tool                                       | Scope                                          |
|----------------|--------------------------------------------|------------------------------------------------|
| Domain         | Plain JUnit 5 + AssertJ                     | Aggregates, state machine, pricing, value objects |
| Web            | `@WebMvcTest` + MockMvc                     | Validation, status codes, error format, security |
| Persistence    | `@DataJpaTest` + Testcontainers             | Custom queries, mappings                        |
| Module / flow  | `@SpringBootTest` + Testcontainers          | End-to-end use cases, events between modules    |

- Test method names follow `should_<expected>_when_<condition>`, e.g. `should_reject_transition_when_order_already_delivered`.
- Reuse `TestcontainersConfiguration`. No H2.
- External systems (payment gateway, mail) are faked via their interfaces in tests.
- Every new endpoint ships with at least one happy-path and one failure-path test.

## 6. Git workflow

- Branches: `feature/<module>-<short-desc>`, `fix/<short-desc>`, `chore/<short-desc>`.
- Commits follow Conventional Commits: `feat(order): add cancel endpoint`, `fix(payment): dedupe webhooks`.
- Small PRs, one module or concern per PR. Tests must pass (`./mvnw verify`) before merging.
- Update [PROGRESS.md](PROGRESS.md) when an item is completed. Add an ADR under `docs/adr/` for any
  significant architectural decision.
