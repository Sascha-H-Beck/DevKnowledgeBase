# PostgreSQL Zero to Hero — Billing System

A progressive guide from absolute basics to advanced queries, using a sample **Billing System** with `accounts`, `contacts`, and `payments` tables.

---

## Table of Contents

1. [Schema Setup](#schema-setup)
2. [Basic SELECT](#basic-select)
3. [Filtering with WHERE](#filtering-with-where)
4. [Sorting and Limiting](#sorting-and-limiting)
5. [Aggregations](#aggregations)
6. [GROUP BY and HAVING](#group-by-and-having)
7. [Finding Duplicates](#finding-duplicates)
8. [JOINs](#joins)
9. [Nested Selects (Subqueries)](#nested-selects-subqueries)
10. [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
11. [Window Functions](#window-functions)
12. [Recursive Queries](#recursive-queries)
13. [Advanced Techniques](#advanced-techniques)

---

## Schema Setup

```sql
CREATE TABLE accounts (
    id            SERIAL PRIMARY KEY,
    company_name  TEXT NOT NULL,
    industry      TEXT,
    status        TEXT NOT NULL DEFAULT 'active',  -- active | suspended | closed
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE contacts (
    id          SERIAL PRIMARY KEY,
    account_id  INT NOT NULL REFERENCES accounts(id),
    first_name  TEXT NOT NULL,
    last_name   TEXT NOT NULL,
    email       TEXT NOT NULL,
    role        TEXT,  -- billing | technical | admin
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payments (
    id          SERIAL PRIMARY KEY,
    account_id  INT NOT NULL REFERENCES accounts(id),
    amount      NUMERIC(12,2) NOT NULL,
    currency    TEXT NOT NULL DEFAULT 'EUR',
    status      TEXT NOT NULL DEFAULT 'pending',  -- pending | completed | failed | refunded
    due_date    DATE NOT NULL,
    paid_at     TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_account ON payments(account_id);
CREATE INDEX idx_contacts_account ON contacts(account_id);
CREATE INDEX idx_payments_status  ON payments(status);
```

---

## Basic SELECT

Retrieve all columns from a table:

```sql
SELECT * FROM accounts;
```

Select specific columns:

```sql
SELECT id, company_name, status FROM accounts;
```

Alias columns for readability:

```sql
SELECT
    company_name AS "Company",
    status       AS "Account Status"
FROM accounts;
```

---

## Filtering with WHERE

Active accounts only:

```sql
SELECT * FROM accounts
WHERE status = 'active';
```

Payments above a threshold:

```sql
SELECT * FROM payments
WHERE amount > 1000.00;
```

Combine conditions with AND / OR:

```sql
SELECT * FROM payments
WHERE status = 'failed'
  AND amount > 500.00;
```

Pattern matching with LIKE / ILIKE:

```sql
SELECT * FROM contacts
WHERE email ILIKE '%@example.com';
```

Filter with IN for multiple values:

```sql
SELECT * FROM accounts
WHERE industry IN ('SaaS', 'Fintech', 'Healthcare');
```

Date range filtering:

```sql
SELECT * FROM payments
WHERE due_date BETWEEN '2025-01-01' AND '2025-03-31';
```

NULL checks:

```sql
SELECT * FROM payments
WHERE paid_at IS NULL;
```

---

## Sorting and Limiting

Order by amount descending:

```sql
SELECT * FROM payments
ORDER BY amount DESC;
```

Pagination with LIMIT and OFFSET:

```sql
SELECT * FROM payments
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;  -- page 3, 20 per page
```

---

## Aggregations

Count all payments:

```sql
SELECT COUNT(*) FROM payments;
```

Sum of completed payments:

```sql
SELECT SUM(amount) AS total_revenue
FROM payments
WHERE status = 'completed';
```

Average payment amount per currency:

```sql
SELECT currency, AVG(amount) AS avg_payment
FROM payments
GROUP BY currency;
```

Min/Max due dates:

```sql
SELECT
    MIN(due_date) AS earliest_due,
    MAX(due_date) AS latest_due
FROM payments;
```

---

## GROUP BY and HAVING

Revenue per account:

```sql
SELECT
    account_id,
    COUNT(*)     AS payment_count,
    SUM(amount)  AS total_paid
FROM payments
WHERE status = 'completed'
GROUP BY account_id
ORDER BY total_paid DESC;
```

Only accounts with more than 5 payments (HAVING filters after aggregation):

```sql
SELECT
    account_id,
    COUNT(*) AS payment_count
FROM payments
GROUP BY account_id
HAVING COUNT(*) > 5;
```

---

## Finding Duplicates

### Duplicate emails in contacts

```sql
SELECT email, COUNT(*) AS occurrences
FROM contacts
GROUP BY email
HAVING COUNT(*) > 1;
```

### Full duplicate rows (same account, same email, same role)

```sql
SELECT account_id, email, role, COUNT(*) AS cnt
FROM contacts
GROUP BY account_id, email, role
HAVING COUNT(*) > 1;
```

### Retrieve the actual duplicate records

```sql
SELECT c.*
FROM contacts c
JOIN (
    SELECT email
    FROM contacts
    GROUP BY email
    HAVING COUNT(*) > 1
) dupes ON dupes.email = c.email
ORDER BY c.email, c.id;
```

### Keep only the oldest record, identify removable duplicates

```sql
SELECT id
FROM (
    SELECT
        id,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at) AS rn
    FROM contacts
) sub
WHERE rn > 1;
```

---

## JOINs

### INNER JOIN — Accounts with their contacts

```sql
SELECT
    a.company_name,
    c.first_name || ' ' || c.last_name AS contact_name,
    c.email
FROM accounts a
JOIN contacts c ON c.account_id = a.id;
```

### LEFT JOIN — All accounts, even those without payments

```sql
SELECT
    a.company_name,
    COALESCE(SUM(p.amount), 0) AS total_billed
FROM accounts a
LEFT JOIN payments p ON p.account_id = a.id
GROUP BY a.company_name;
```

### Multi-table JOIN

```sql
SELECT
    a.company_name,
    c.email           AS billing_contact,
    p.amount,
    p.status          AS payment_status
FROM payments p
JOIN accounts a ON a.id = p.account_id
JOIN contacts c ON c.account_id = a.id AND c.role = 'billing';
```

---

## Nested Selects (Subqueries)

### Scalar subquery — Accounts with above-average total payments

```sql
SELECT a.company_name, a.status
FROM accounts a
WHERE (
    SELECT SUM(p.amount)
    FROM payments p
    WHERE p.account_id = a.id AND p.status = 'completed'
) > (
    SELECT AVG(total)
    FROM (
        SELECT SUM(amount) AS total
        FROM payments
        WHERE status = 'completed'
        GROUP BY account_id
    ) avg_sub
);
```

### EXISTS — Accounts that have at least one failed payment

```sql
SELECT a.*
FROM accounts a
WHERE EXISTS (
    SELECT 1
    FROM payments p
    WHERE p.account_id = a.id
      AND p.status = 'failed'
);
```

### NOT EXISTS — Contacts with no associated payments on their account

```sql
SELECT c.*
FROM contacts c
WHERE NOT EXISTS (
    SELECT 1
    FROM payments p
    WHERE p.account_id = c.account_id
);
```

### IN with subquery — Accounts that paid in January 2025

```sql
SELECT * FROM accounts
WHERE id IN (
    SELECT account_id
    FROM payments
    WHERE paid_at >= '2025-01-01'
      AND paid_at <  '2025-02-01'
      AND status = 'completed'
);
```

### Correlated subquery — Latest payment per account

```sql
SELECT p.*
FROM payments p
WHERE p.id = (
    SELECT p2.id
    FROM payments p2
    WHERE p2.account_id = p.account_id
    ORDER BY p2.created_at DESC
    LIMIT 1
);
```

---

## Common Table Expressions (CTEs)

CTEs improve readability over deeply nested subqueries.

### Account revenue summary

```sql
WITH revenue AS (
    SELECT
        account_id,
        SUM(amount)  AS total_revenue,
        COUNT(*)     AS payment_count
    FROM payments
    WHERE status = 'completed'
    GROUP BY account_id
)
SELECT
    a.company_name,
    r.total_revenue,
    r.payment_count
FROM accounts a
JOIN revenue r ON r.account_id = a.id
ORDER BY r.total_revenue DESC;
```

### Chained CTEs — Overdue accounts with billing contacts

```sql
WITH overdue AS (
    SELECT DISTINCT account_id
    FROM payments
    WHERE status = 'pending'
      AND due_date < CURRENT_DATE
),
billing_contacts AS (
    SELECT account_id, email
    FROM contacts
    WHERE role = 'billing'
)
SELECT
    a.company_name,
    bc.email AS billing_email
FROM overdue o
JOIN accounts a          ON a.id = o.account_id
LEFT JOIN billing_contacts bc ON bc.account_id = o.account_id;
```

---

## Window Functions

### Running total of payments per account

```sql
SELECT
    account_id,
    paid_at,
    amount,
    SUM(amount) OVER (
        PARTITION BY account_id
        ORDER BY paid_at
    ) AS running_total
FROM payments
WHERE status = 'completed';
```

### Rank accounts by total revenue

```sql
SELECT
    account_id,
    SUM(amount) AS total_revenue,
    RANK() OVER (ORDER BY SUM(amount) DESC) AS revenue_rank
FROM payments
WHERE status = 'completed'
GROUP BY account_id;
```

### Difference to previous payment (LAG)

```sql
SELECT
    id,
    account_id,
    amount,
    amount - LAG(amount) OVER (
        PARTITION BY account_id ORDER BY created_at
    ) AS diff_from_previous
FROM payments;
```

### Percentile / NTILE — Split accounts into quartiles by revenue

```sql
WITH revenue AS (
    SELECT account_id, SUM(amount) AS total
    FROM payments
    WHERE status = 'completed'
    GROUP BY account_id
)
SELECT
    account_id,
    total,
    NTILE(4) OVER (ORDER BY total DESC) AS quartile
FROM revenue;
```

---

## Recursive Queries

Useful when accounts have a parent-child hierarchy. Add a `parent_id` column:

```sql
ALTER TABLE accounts ADD COLUMN parent_id INT REFERENCES accounts(id);
```

### Retrieve full hierarchy tree for a given root account

```sql
WITH RECURSIVE account_tree AS (
    -- anchor: the root account
    SELECT id, company_name, parent_id, 0 AS depth
    FROM accounts
    WHERE id = 1

    UNION ALL

    -- recursive step: children
    SELECT a.id, a.company_name, a.parent_id, at.depth + 1
    FROM accounts a
    JOIN account_tree at ON a.parent_id = at.id
)
SELECT * FROM account_tree
ORDER BY depth, company_name;
```

### Aggregate revenue across entire hierarchy

```sql
WITH RECURSIVE subtree AS (
    SELECT id FROM accounts WHERE id = 1
    UNION ALL
    SELECT a.id FROM accounts a JOIN subtree s ON a.parent_id = s.id
)
SELECT SUM(p.amount) AS hierarchy_revenue
FROM payments p
WHERE p.account_id IN (SELECT id FROM subtree)
  AND p.status = 'completed';
```

---

## Advanced Techniques

### LATERAL JOIN — Top 3 payments per account

```sql
SELECT a.company_name, top3.*
FROM accounts a
CROSS JOIN LATERAL (
    SELECT amount, paid_at
    FROM payments p
    WHERE p.account_id = a.id
      AND p.status = 'completed'
    ORDER BY p.amount DESC
    LIMIT 3
) top3;
```

### FILTER clause — Conditional aggregation without CASE

```sql
SELECT
    account_id,
    COUNT(*)                              AS total_payments,
    COUNT(*) FILTER (WHERE status = 'completed') AS completed,
    COUNT(*) FILTER (WHERE status = 'failed')    AS failed,
    SUM(amount) FILTER (WHERE status = 'completed') AS revenue
FROM payments
GROUP BY account_id;
```

### GROUPING SETS — Multiple aggregation levels in one pass

```sql
SELECT
    account_id,
    currency,
    SUM(amount) AS total
FROM payments
WHERE status = 'completed'
GROUP BY GROUPING SETS (
    (account_id, currency),
    (account_id),
    (currency),
    ()
)
ORDER BY account_id NULLS LAST, currency NULLS LAST;
```

### Materialized CTE for deduplication + upsert pattern

```sql
WITH deduped AS MATERIALIZED (
    SELECT DISTINCT ON (account_id, due_date)
        account_id, amount, due_date, status
    FROM payments
    ORDER BY account_id, due_date, created_at DESC
)
SELECT * FROM deduped
WHERE status = 'pending';
```

### Generate Series — Fill gaps in monthly revenue reports

```sql
WITH months AS (
    SELECT generate_series(
        '2025-01-01'::date,
        '2025-12-01'::date,
        '1 month'::interval
    )::date AS month
),
monthly_revenue AS (
    SELECT
        date_trunc('month', paid_at)::date AS month,
        SUM(amount) AS revenue
    FROM payments
    WHERE status = 'completed'
    GROUP BY 1
)
SELECT
    m.month,
    COALESCE(mr.revenue, 0) AS revenue
FROM months m
LEFT JOIN monthly_revenue mr ON mr.month = m.month
ORDER BY m.month;
```

### Row-Level Security (RLS) — Tenant isolation

```sql
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON payments
    USING (account_id = current_setting('app.current_account_id')::int);
```

### Index-Only Scans — Covering index for payment lookups

```sql
CREATE INDEX idx_payments_covering
    ON payments (account_id, status)
    INCLUDE (amount, paid_at);
```

### EXPLAIN ANALYZE — Understanding query performance

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT a.company_name, SUM(p.amount)
FROM accounts a
JOIN payments p ON p.account_id = a.id
WHERE p.status = 'completed'
GROUP BY a.company_name;
```

Key things to look for:
- **Seq Scan** vs **Index Scan** — sequential scans on large tables indicate missing indexes
- **Nested Loop** vs **Hash Join** — nested loops are costly for large result sets
- **Buffers: shared hit** vs **shared read** — reads from disk are slow
- **Rows Removed by Filter** — high counts suggest the index isn't selective enough

---

## Quick Reference Cheat Sheet

| Task | Key Syntax |
|------|-----------|
| Filter rows | `WHERE col = value` |
| Pattern match | `ILIKE '%pattern%'` |
| Null check | `IS NULL` / `IS NOT NULL` |
| Aggregate | `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` |
| Filter aggregates | `HAVING COUNT(*) > n` |
| Find duplicates | `GROUP BY ... HAVING COUNT(*) > 1` |
| Subquery | `WHERE col IN (SELECT ...)` |
| Correlated subquery | `WHERE col = (SELECT ... WHERE outer.id = inner.fk)` |
| CTE | `WITH name AS (...) SELECT ...` |
| Window function | `OVER (PARTITION BY ... ORDER BY ...)` |
| Recursive | `WITH RECURSIVE ... UNION ALL ...` |
| Lateral join | `CROSS JOIN LATERAL (SELECT ... LIMIT n)` |
| Conditional agg | `FILTER (WHERE ...)` |
| Gap filling | `generate_series(...)` |
| Performance | `EXPLAIN (ANALYZE, BUFFERS)` |

---

*End of guide.*
