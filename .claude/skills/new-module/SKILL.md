---
name: new-module
description: Scaffold a new business module in vsnvi-food (Spring Modulith modular monolith) with the api / api.web / application / domain / infrastructure packages, its facade, events, Flyway migration, a sample domain test and module test, and the doc updates a new module requires. Use when asked to add, create or scaffold a module (e.g. "add a review module").
---

# Scaffold a new module

The conventions this skill applies live in `.claude/rules/java.md`, `.claude/rules/testing.md`,
`.claude/rules/persistence.md` and `docs/ARCHITECTURE.md` §2–4. If anything here disagrees with
them, they win; fix this skill.

The examples use a `review` module (listed under "Future" in `docs/PROGRESS.md`): customers rate
an order after it is delivered. Replace `review` / `Review` with the real module name.

## Step 0: check prerequisites, then stop if they're missing

The templates use Phase 0 pieces. Check that they exist before generating code:

- `common`: `BaseEntity`, `DomainException` / `ErrorCode`, `NotFoundException`, a `Clock` bean.
- Spring Modulith in `pom.xml` (`spring-modulith-starter-core`, `-starter-jpa`, `-starter-test`).
- The `ApplicationModules.verify()` test.

If any are missing, tell the user which ones and stop. Don't add dependencies to `pom.xml` without
asking (`CLAUDE.md`).

## Step 1: place the module in the dependency graph (before writing code)

This touches several modules' docs, so propose this plan to the user first.

1. **Name it:** a lowercase singular noun, a direct sub-package of `com.vsnvi.vsnvi_food`.
2. **List what it depends on**, i.e. which modules it will call or whose events it will listen to.
   For `review`: `order` (listens to `OrderDelivered`, reads order details through `OrderApi`).
3. **List what will depend on it**, i.e. who calls its facade or listens to its events. For
   `review`: `restaurant` would read ratings — **but `restaurant` sits below `order` in the graph,
   so `restaurant → review → order → restaurant` would be a cycle.** Resolve cycles now:
   restructure the flow, or have `review` own the aggregate rating that consumers read.
4. **Apply the rule:** calls go *down* the graph, events go *up*. A module never listens to events
   of a module that depends on it (`docs/ARCHITECTURE.md` §3).

Don't continue until the new edges keep the graph acyclic.

## Step 2: create the packages

```
src/main/java/com/vsnvi/vsnvi_food/review/
├── api/
│   ├── package-info.java            @NamedInterface("api")
│   ├── ReviewApi.java               facade interface (public surface)
│   ├── ReviewSummary.java           facade DTO record
│   ├── ReviewSubmitted.java         published event record
│   └── web/
│       ├── ReviewController.java
│       ├── SubmitReviewRequest.java
│       └── ReviewResponse.java
├── application/
│   ├── ReviewService.java           implements ReviewApi, @Transactional
│   └── ReviewEventListener.java     @ApplicationModuleListener
├── domain/
│   ├── Review.java                  entity, extends BaseEntity
│   └── ReviewRepository.java        repository interface
└── infrastructure/
    └── ReviewJpaRepository.java     Spring Data implementation
src/main/resources/db/migration/V{n}__review_create_reviews.sql
src/test/java/com/vsnvi/vsnvi_food/review/
├── domain/ReviewTest.java           unit test, no Spring
└── ReviewModuleTests.java           @ApplicationModuleTest
```

Create only the files the module needs. A module with no HTTP endpoints has no `api/web/`; a
module that publishes nothing has no event records.

## Step 3: write the code from these templates

Check every import against the versions in `pom.xml` when it first compiles. The Spring Modulith
and Boot 4 package names below are the expected ones, not verified.

### `api/` (the only package other modules may import)

```java
// api/package-info.java
@org.springframework.modulith.NamedInterface("api")
package com.vsnvi.vsnvi_food.review.api;
```

```java
// api/ReviewApi.java
public interface ReviewApi {
    Optional<ReviewSummary> findByOrder(UUID orderId);
}

// api/ReviewSummary.java  (facade DTO: a record, never an entity)
public record ReviewSummary(UUID reviewId, UUID orderId, UUID restaurantId, int rating) {}

// api/ReviewSubmitted.java  (event: past tense, IDs + minimal data, occurredAt)
public record ReviewSubmitted(UUID reviewId, UUID orderId, UUID restaurantId, int rating,
                              Instant occurredAt) {}
```

- After creating the first `verify()`-tested module, confirm that `api.web` is **not** listed in
  the `api` named interface (`docs/decisions/ADR-003-spring-modulith.md`).

### `api/web/` (controllers: validate, map, delegate)

```java
@RestController
@RequestMapping("/api/v1/orders/{orderId}/reviews")
@RequiredArgsConstructor
@Tag(name = "review")
class ReviewController {

    private final ReviewService reviewService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @PreAuthorize("hasRole('CUSTOMER')")
    @Operation(summary = "Rate a delivered order", operationId = "submitReview")
    ReviewResponse submit(@PathVariable UUID orderId, @Valid @RequestBody SubmitReviewRequest request) {
        return ReviewResponse.from(reviewService.submit(orderId, request.rating(), request.comment()));
    }
}

record SubmitReviewRequest(@Min(1) @Max(5) int rating, @Size(max = 1000) String comment) {}
```

- `operationId` must be unique across the whole API.
- Ownership is checked in the service against the authenticated user, never against a request
  field. Another user's order returns `404`.
- `POST` that creates returns `201`; follow `docs/API-DESIGN.md` for paths, errors and pagination.

### `domain/` (no Spring Web, no other modules except `common`)

```java
@Entity
@Table(name = "reviews")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Review extends BaseEntity {

    private UUID orderId;        // other modules' aggregates by ID only, no JPA relation
    private UUID customerId;
    private UUID restaurantId;
    private int rating;
    private String comment;

    public static Review submit(UUID orderId, UUID customerId, UUID restaurantId,
                                int rating, String comment) {
        if (rating < 1 || rating > 5) {
            throw new IllegalArgumentException("rating must be 1-5");
        }
        var review = new Review();
        review.orderId = orderId;
        review.customerId = customerId;
        review.restaurantId = restaurantId;
        review.rating = rating;
        review.comment = comment;
        return review;
    }
}
```

```java
public interface ReviewRepository {
    Review save(Review review);
    Optional<Review> findByOrderId(UUID orderId);
}
```

- No `@Data`, `@Setter` or `@EqualsAndHashCode`. State changes only through intention-revealing
  methods.
- Add `@Version` only if the aggregate is updated concurrently, and record it in
  `.claude/rules/java.md` and `.claude/rules/persistence.md`.
- Business-rule failures throw a `DomainException` subclass with a new `ErrorCode`; add the code to
  the catalog in `docs/API-DESIGN.md` §4.

### `application/` (transactions, facade implementation, events)

```java
@Service
@RequiredArgsConstructor
public class ReviewService implements ReviewApi {   // public: the controller is in another package

    private final ReviewRepository reviews;
    private final OrderApi orderApi;                       // module below us in the graph
    private final ApplicationEventPublisher events;
    private final Clock clock;

    @Transactional
    public Review submit(UUID orderId, int rating, String comment) {
        var customerId = CurrentUser.id();                  // from common; never from the request
        var order = orderApi.findForCustomer(orderId, customerId)
                .orElseThrow(() -> new NotFoundException("Order " + orderId));
        var review = reviews.save(Review.submit(orderId, customerId, order.restaurantId(), rating, comment));
        events.publishEvent(new ReviewSubmitted(review.getId(), orderId, order.restaurantId(),
                rating, Instant.now(clock)));
        return review;
    }

    @Override
    @Transactional(readOnly = true)
    public Optional<ReviewSummary> findByOrder(UUID orderId) {
        return reviews.findByOrderId(orderId)
                .map(r -> new ReviewSummary(r.getId(), r.getOrderId(), r.getRestaurantId(), r.getRating()));
    }
}
```

```java
@Component
@RequiredArgsConstructor
class ReviewEventListener {

    private final ReviewRepository reviews;

    @ApplicationModuleListener                               // after commit, async, own transaction
    void on(OrderDelivered event) {
        // Must be idempotent: the same event can arrive twice. Check state before acting.
    }
}
```

- `OrderApi.findForCustomer` and `CurrentUser` are illustrative. Use whatever the target facade
  and `common` actually expose; if the facade lacks a method, add it there.
- Get time only from the injected `Clock`.
- No calls to external systems inside a transaction that holds a row lock.

### `infrastructure/`

```java
interface ReviewJpaRepository extends JpaRepository<Review, UUID>, ReviewRepository {}
```

### Migration (`src/main/resources/db/migration/V{n}__review_create_reviews.sql`)

```sql
CREATE TABLE reviews (
    id            UUID PRIMARY KEY,
    order_id      UUID        NOT NULL,   -- cross-module reference: no FK
    customer_id   UUID        NOT NULL,
    restaurant_id UUID        NOT NULL,
    rating        SMALLINT    NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment       VARCHAR(1000),
    created_at    TIMESTAMPTZ NOT NULL,
    updated_at    TIMESTAMPTZ NOT NULL,
    CONSTRAINT uq_reviews_order UNIQUE (order_id)
);
CREATE INDEX idx_reviews_restaurant_created ON reviews (restaurant_id, created_at);
```

- `{n}` is the next free version number. Never edit an applied migration.
- Only this module's tables. Every entity field needs a column (`ddl-auto=validate`).

## Step 4: write the sample tests

Name tests `should_<expected>_when_<condition>`. Use the existing `TestcontainersConfiguration`;
never H2.

```java
// src/test/java/com/vsnvi/vsnvi_food/review/domain/ReviewTest.java  (unit, no Spring)
class ReviewTest {

    @Test
    void should_reject_review_when_rating_is_out_of_range() {
        assertThatThrownBy(() -> Review.submit(randomUUID(), randomUUID(), randomUUID(), 6, null))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

```java
// src/test/java/com/vsnvi/vsnvi_food/review/ReviewModuleTests.java  (module test)
@ApplicationModuleTest
@Import(TestcontainersConfiguration.class)
class ReviewModuleTests {

    @Autowired ReviewApi reviewApi;

    @Test
    void should_publish_ReviewSubmitted_when_customer_reviews_a_delivered_order(Scenario scenario) {
        scenario.stimulate(() -> /* call the use case through the service or controller */ null)
                .andWaitForEventOfType(ReviewSubmitted.class)
                .toArriveAndVerify(event -> assertThat(event.rating()).isBetween(1, 5));
    }

    @Test
    void should_ignore_duplicate_when_OrderDelivered_arrives_twice(Scenario scenario) {
        // Publish the same upstream event twice with scenario.publish(...) and assert that state
        // changed only once (idempotent listener).
    }
}
```

Then add the tests `.claude/rules/testing.md` requires for what the module actually contains:

- every endpoint: happy path, failure path asserting the error `code`, and 401 / 403 / 404;
- every listener: idempotency;
- every state transition, allowed and forbidden.

## Step 5: update the docs in the same change

- `docs/ARCHITECTURE.md`:
  - §2: modules table, dependency graph (mermaid) and the dependency-use table;
  - §4: event catalog, for every event published or consumed.
- `docs/DATABASE-DESIGN.md` §3 and §4: entities, tables and relationships.
- `docs/API-DESIGN.md`:
  - §9: endpoints;
  - §11: OpenAPI group paths;
  - §4: new error codes.
- `docs/SECURITY.md` §2: role table and public allowlist, if they change.
- `docs/PROGRESS.md`: a section for the module, in build order (after every module it depends on).
- Module lists in `CLAUDE.md`, `.claude/rules/java.md` and
  `docs/decisions/ADR-001-modular-monolith.md`.
- A new ADR in `docs/decisions/` if the module brings a significant design decision.

## Step 6: verify

```bash
./mvnw verify
```

It must pass, including `ApplicationModules.verify()`; never weaken that test. Docker must be
running for Testcontainers.
