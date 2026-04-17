---
name: "sql-patterns"
description: "SQL best practices: CTEs, window functions, query optimization, indexing, safe migrations (Flyway/Liquibase). Use when writing SQL queries, optimizing slow queries, designing schemas, or managing database migrations."
---

# SQL Patterns Skill

Write efficient SQL queries and maintainable database migrations.

## Scope and Notes

- Use this skill for raw SQL, indexing, and migration strategy.
- Verify query advice with `EXPLAIN`/actual plans on the target database; optimizer behavior differs across PostgreSQL, MySQL, Oracle, and others.
- For ORM-specific issues, prefer `spring-boot-engineer` → `references/data.md`.

## When to Use
- Writing SQL queries, CTEs, or window functions
- Reviewing or designing database schemas
- Optimizing slow queries
- Writing or reviewing database migrations (Flyway/Liquibase)

---

## CTEs (Common Table Expressions)

```sql
-- Simple CTE — named subquery, improves readability
WITH active_users AS (
    SELECT id, name, email
    FROM users
    WHERE status = 'active'
)
SELECT * FROM active_users WHERE created_at > '2024-01-01';

-- Chained CTEs — each builds on the previous
WITH
    active_users AS (
        SELECT id, name FROM users WHERE status = 'active'
    ),
    user_orders AS (
        SELECT user_id, COUNT(*) AS order_count
        FROM orders GROUP BY user_id
    )
SELECT u.name, COALESCE(o.order_count, 0) AS orders
FROM active_users u
LEFT JOIN user_orders o ON u.id = o.user_id;

-- Recursive CTE — tree/hierarchy traversal
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS depth
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY depth, name;
```

---

## Window Functions

| Function | Use |
|---|---|
| `ROW_NUMBER()` | Unique sequential number per partition |
| `RANK()` | Rank with gaps on ties (1, 2, 2, 4) |
| `DENSE_RANK()` | Rank without gaps (1, 2, 2, 3) |
| `LAG(col, n)` | Value from n rows before |
| `LEAD(col, n)` | Value from n rows after |
| `SUM() OVER` | Running total |
| `AVG() OVER` | Moving average |
| `FIRST_VALUE()` / `LAST_VALUE()` | First/last in window frame |

```sql
-- Running total + day-over-day delta
SELECT
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date)          AS prev_day,
    revenue - LAG(revenue, 1) OVER (ORDER BY date) AS delta,
    SUM(revenue) OVER (ORDER BY date)              AS running_total
FROM daily_sales;

-- Top 1 per group (no subquery)
SELECT *
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) ranked
WHERE rn = 1;
```

---

## JOIN Reference

| Type | Returns |
|---|---|
| `INNER JOIN` | Only matching rows |
| `LEFT JOIN` | All left rows + matching right (NULL where no match) |
| `RIGHT JOIN` | All right rows + matching left |
| `FULL OUTER JOIN` | All rows from both sides |
| `CROSS JOIN` | Cartesian product |

```sql
-- ✅ Prefer LEFT JOIN + IS NULL for "not in other table" (handles NULLs correctly)
SELECT u.id FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;  -- users with no orders

-- ❌ NOT IN fails silently when subquery contains NULLs
SELECT id FROM users WHERE id NOT IN (SELECT user_id FROM orders);
```

---

## Flyway Migrations

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    name        VARCHAR(100) NOT NULL,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

Naming convention: `V{version}__{description}.sql`

---

## Indexing

```sql
-- ✅ Index columns used in WHERE, JOIN, ORDER BY
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);

-- ✅ Partial index for common filtered queries (exception to the low-cardinality rule below —
-- partial indexes are efficient even on low-cardinality columns because they only index the
-- matching subset, e.g. the 5% of rows with status = 'PENDING')
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'PENDING';

-- ❌ Don't create full indexes on low-cardinality columns (boolean, status with few values)
CREATE INDEX idx_users_active ON users(is_active); -- bad if 90% are active
-- Consider a partial index instead if you only query one specific value
```

---

## Query Optimization

```sql
-- ✅ Often prefer EXISTS for correlated subqueries, but verify with EXPLAIN
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id AND u.is_active = true);

-- ⚠️ IN with subquery can be slower on some databases/workloads
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE is_active = true);

-- ✅ Good — avoid SELECT *
SELECT id, email, name FROM users WHERE id = $1;

-- ✅ Keyset pagination — O(log n), stays fast on large tables
SELECT id, name FROM products
WHERE id > :lastSeenId
ORDER BY id LIMIT 20;

-- ⚠️ OFFSET pagination — degrades as offset grows (full scan to skip rows)
SELECT id, name FROM products ORDER BY created_at DESC LIMIT 20 OFFSET 40;
```

---

## Safe Migrations

```sql
-- ✅ Good — add column with default (non-blocking in Postgres 11+)
ALTER TABLE users ADD COLUMN preferences JSONB DEFAULT '{}';

-- ✅ Good — create index concurrently (no table lock)
CREATE INDEX CONCURRENTLY idx_users_name ON users(name);

-- ❌ Bad — adding NOT NULL without default locks table
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
```

---

## Liquibase Migrations

Liquibase uses XML/YAML/JSON changelogs instead of plain SQL files. Each `changeSet` is tracked in `DATABASECHANGELOG` table.

```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/changes/001-create-users-table.yaml
  - include:
      file: db/changelog/changes/002-add-users-phone.yaml
```

```yaml
# db/changelog/changes/001-create-users-table.yaml
databaseChangeLog:
  - changeSet:
      id: 001
      author: dev
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: email
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: created_at
                  type: TIMESTAMP WITH TIME ZONE
                  defaultValueComputed: NOW()
                  constraints:
                    nullable: false
```

Point your framework's datasource config to `classpath:db/changelog/db.changelog-master.yaml`.

Key differences from Flyway:
- Changelogs are **immutable** — never edit an applied `changeSet`, add a new one instead.
- Supports rollback via `<rollback>` blocks (Flyway Pro only).
- Use `liquibase:rollback` or `liquibase:rollbackCount` for controlled rollbacks.

---

## Anti-Patterns

| Mistake | Problem | Fix |
|---|---|---|
| `SELECT *` | Fetches unused columns, breaks on schema change | List columns explicitly |
| `WHERE YEAR(date) = 2024` | Prevents index use (function on column) | `WHERE date >= '2024-01-01' AND date < '2025-01-01'` |
| `NOT IN` with nullable column | Silent wrong results when NULLs present | Use `NOT EXISTS` or `LEFT JOIN ... IS NULL` |
| `OR` in WHERE across different columns | Prevents index use | Split into `UNION ALL` or redesign index |
| N+1 queries | 1 query per row instead of 1 join | Use JOIN, batch fetch, or EXISTS |
| No `LIMIT` on user-driven queries | Full table scan risk | Always paginate external-facing queries |
| Premature denormalization | Maintenance nightmare | Normalize first, denormalize only with evidence |
