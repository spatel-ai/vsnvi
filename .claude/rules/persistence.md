---
paths:
  - "src/main/resources/db/**"
---

# Persistence rules

Details: `docs/DATABASE-DESIGN.md`.

- Migrations live in `src/main/resources/db/migration` and are named `V{n}__{module}_{description}.sql`, e.g. `V3__order_create_orders.sql`.
- Never edit a migration that has already been applied. Add a new one.
- A migration may change only the tables of its own module. No foreign keys to another module's tables; store the UUID.
- Every table has a `UUID` primary key and `created_at` / `updated_at` as `timestamptz`.
- Only `orders`, `payments`, `carts` and `delivery_tasks` have a `version` column.
- Money is `NUMERIC(12,2)` plus `currency CHAR(3)`. Enums are `VARCHAR` holding the enum name.
- snake_case names, plural tables. Index every FK and every column used in frequent filters or sorts.
- The `event_publication` migration must copy the PostgreSQL schema shipped in the exact Spring Modulith version in `pom.xml` (its `schema-postgresql.sql`). Re-check it when upgrading Modulith.
- `ddl-auto` stays `validate`, so every entity field must match a column created by a migration.
