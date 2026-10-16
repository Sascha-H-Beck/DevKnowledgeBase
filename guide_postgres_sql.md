# PostgreSQL SQL Language Guide — Zero to Hero

A comprehensive reference for the SQL language as implemented in PostgreSQL, covering syntax, data types, operators, built-in functions, and advanced language features.

---

## Table of Contents

1. [Data Types](#data-types)
2. [Literals and Constants](#literals-and-constants)
3. [Operators](#operators)
4. [Type Casting](#type-casting)
5. [DDL — Data Definition Language](#ddl--data-definition-language)
6. [DML — Data Manipulation Language](#dml--data-manipulation-language)
7. [String Functions](#string-functions)
8. [Numeric Functions](#numeric-functions)
9. [Date/Time Functions](#datetime-functions)
10. [Conditional Expressions](#conditional-expressions)
11. [Array Functions](#array-functions)
12. [JSON/JSONB Functions](#jsonjsonb-functions)
13. [Aggregate Functions](#aggregate-functions)
14. [Window Functions](#window-functions)
15. [Set Operations](#set-operations)
16. [Subqueries and Lateral](#subqueries-and-lateral)
17. [Common Table Expressions](#common-table-expressions)
18. [Transactions and Locking](#transactions-and-locking)
19. [Views and Materialized Views](#views-and-materialized-views)
20. [Indexes](#indexes)
21. [Constraints](#constraints)
22. [Sequences and Identity](#sequences-and-identity)
23. [PL/pgSQL Functions](#plpgsql-functions)
24. [Triggers](#triggers)
25. [Full-Text Search](#full-text-search)
26. [Regular Expressions](#regular-expressions)
27. [Advisory Locks](#advisory-locks)
28. [System Catalog Queries](#system-catalog-queries)

---

## Data Types

### Numeric

| Type | Storage | Range |
|------|---------|-------|
| `SMALLINT` | 2 bytes | -32,768 to 32,767 |
| `INTEGER` | 4 bytes | -2,147,483,648 to 2,147,483,647 |
| `BIGINT` | 8 bytes | -9.2×10¹⁸ to 9.2×10¹⁸ |
| `NUMERIC(p,s)` | variable | up to 131,072 digits before decimal, 16,383 after |
| `REAL` | 4 bytes | 6 decimal digits precision |
| `DOUBLE PRECISION` | 8 bytes | 15 decimal digits precision |
| `SERIAL` | 4 bytes | auto-incrementing integer |
| `BIGSERIAL` | 8 bytes | auto-incrementing bigint |

### Character

| Type | Description |
|------|-------------|
| `TEXT` | unlimited variable-length |
| `VARCHAR(n)` | variable-length with limit |
| `CHAR(n)` | fixed-length, blank-padded |

### Boolean

`BOOLEAN` — `TRUE`, `FALSE`, or `NULL`

### Date/Time

| Type | Description |
|------|-------------|
| `DATE` | calendar date (no time) |
| `TIME` | time of day (no date) |
| `TIMESTAMP` | date + time without timezone |
| `TIMESTAMPTZ` | date + time with timezone |
| `INTERVAL` | time span |

### Binary

| Type | Description |
|------|-------------|
| `BYTEA` | variable-length binary string |

### UUID

`UUID` — 128-bit universally unique identifier

### Network

| Type | Description |
|------|-------------|
| `INET` | IPv4/IPv6 host address |
| `CIDR` | IPv4/IPv6 network |
| `MACADDR` | MAC address |

### JSON

| Type | Description |
|------|-------------|
| `JSON` | text representation, preserves whitespace |
| `JSONB` | binary representation, indexable, no duplicate keys |

### Arrays

Any data type can be an array:

```sql
INTEGER[]
TEXT[]
JSONB[]
```

### Range Types

| Type | Description |
|------|-------------|
| `INT4RANGE` | range of integer |
| `NUMRANGE` | range of numeric |
| `TSRANGE` | range of timestamp |
| `TSTZRANGE` | range of timestamptz |
| `DATERANGE` | range of date |

### Composite Types

```sql
CREATE TYPE address AS (
    street TEXT,
    city   TEXT,
    zip    TEXT
);
```

### Enum Types

```sql
CREATE TYPE status AS ENUM ('active', 'inactive', 'deleted');
```

---

## Literals and Constants

```sql
-- String literal
'Hello, world'

-- Escape string (C-style escapes)
E'Line one\nLine two'

-- Dollar-quoted string (useful in functions)
$$This doesn't need escaping for apostrophes$$
$tag$Custom delimiters$tag$

-- Numeric literals
42
3.14
1.5e3

-- Boolean
TRUE
FALSE

-- NULL
NULL

-- Array literal
ARRAY[1, 2, 3]
'{1,2,3}'::int[]

-- Interval
INTERVAL '3 days 4 hours'
```

---

## Operators

### Comparison

| Operator | Meaning |
|----------|---------|
| `=` | equal |
| `<>` or `!=` | not equal |
| `<` | less than |
| `>` | greater than |
| `<=` | less or equal |
| `>=` | greater or equal |
| `BETWEEN x AND y` | range inclusive |
| `IS NULL` | null test |
| `IS NOT NULL` | not-null test |
| `IS DISTINCT FROM` | null-safe inequality |
| `IS NOT DISTINCT FROM` | null-safe equality |

### Logical

| Operator | Meaning |
|----------|---------|
| `AND` | both true |
| `OR` | either true |
| `NOT` | negation |

### Pattern Matching

| Operator | Meaning |
|----------|---------|
| `LIKE` | case-sensitive pattern (`%` = any chars, `_` = one char) |
| `ILIKE` | case-insensitive pattern |
| `SIMILAR TO` | SQL regex (POSIX-like) |
| `~` | POSIX regex match (case-sensitive) |
| `~*` | POSIX regex match (case-insensitive) |
| `!~` | not match |
| `!~*` | not match (case-insensitive) |

### Array Operators

| Operator | Meaning |
|----------|---------|
| `@>` | contains |
| `<@` | is contained by |
| `&&` | overlap (have elements in common) |
| `||` | concatenation |
| `ANY(array)` | compare to any element |
| `ALL(array)` | compare to all elements |

### JSON/JSONB Operators

| Operator | Meaning |
|----------|---------|
| `->` | get JSON object field (as JSON) |
| `->>` | get JSON object field (as text) |
| `#>` | get JSON object at path (as JSON) |
| `#>>` | get JSON object at path (as text) |
| `@>` | JSONB contains |
| `<@` | JSONB is contained |
| `?` | key exists |
| `?|` | any key exists |
| `?&` | all keys exist |
| `||` | concatenate JSONB |
| `-` | delete key |
| `#-` | delete at path |

---

## Type Casting

```sql
-- CAST syntax
CAST('42' AS INTEGER)

-- PostgreSQL shorthand
'42'::INTEGER
'2025-01-15'::DATE
'{"a":1}'::JSONB

-- Function-style
INTEGER '42'
```

---

## DDL — Data Definition Language

### CREATE TABLE

```sql
CREATE TABLE invoices (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    account_id  INT NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    total       NUMERIC(12,2) NOT NULL CHECK (total >= 0),
    issued_at   DATE NOT NULL DEFAULT CURRENT_DATE,
    notes       TEXT
);
```

### ALTER TABLE

```sql
ALTER TABLE invoices ADD COLUMN currency TEXT DEFAULT 'EUR';
ALTER TABLE invoices DROP COLUMN notes;
ALTER TABLE invoices ALTER COLUMN total SET NOT NULL;
ALTER TABLE invoices RENAME COLUMN total TO amount;
ALTER TABLE invoices ADD CONSTRAINT positive_amount CHECK (amount > 0);
```

### DROP TABLE

```sql
DROP TABLE IF EXISTS invoices CASCADE;
```

### TRUNCATE

```sql
TRUNCATE invoices RESTART IDENTITY CASCADE;
```

### CREATE SCHEMA

```sql
CREATE SCHEMA billing;
CREATE TABLE billing.invoices (...);
```

### Temporary Tables

```sql
CREATE TEMPORARY TABLE tmp_import (
    raw_data JSONB
) ON COMMIT DROP;
```

---

## DML — Data Manipulation Language

### INSERT

```sql
INSERT INTO accounts (company_name, industry, status)
VALUES ('Acme Corp', 'SaaS', 'active');

-- Multiple rows
INSERT INTO accounts (company_name, industry, status) VALUES
    ('Beta Inc', 'Fintech', 'active'),
    ('Gamma Ltd', 'Healthcare', 'active');

-- Insert from SELECT
INSERT INTO archive_payments
SELECT * FROM payments WHERE paid_at < '2020-01-01';

-- RETURNING clause
INSERT INTO accounts (company_name, status)
VALUES ('Delta GmbH', 'active')
RETURNING id, created_at;
```

### UPDATE

```sql
UPDATE accounts
SET status = 'suspended'
WHERE id = 5;

-- Update from another table
UPDATE payments p
SET status = 'cancelled'
FROM accounts a
WHERE a.id = p.account_id
  AND a.status = 'closed';

-- RETURNING
UPDATE payments SET status = 'completed', paid_at = now()
WHERE id = 42
RETURNING *;
```

### DELETE

```sql
DELETE FROM payments WHERE status = 'failed' AND created_at < '2023-01-01';

-- DELETE with RETURNING
DELETE FROM contacts WHERE id = 99 RETURNING *;
```

### UPSERT (INSERT ... ON CONFLICT)

```sql
INSERT INTO contacts (account_id, email, first_name, last_name, role)
VALUES (1, 'billing@acme.com', 'Jane', 'Doe', 'billing')
ON CONFLICT (email)
DO UPDATE SET
    first_name = EXCLUDED.first_name,
    last_name  = EXCLUDED.last_name,
    role       = EXCLUDED.role;

-- Do nothing on conflict
INSERT INTO accounts (company_name, status)
VALUES ('Acme Corp', 'active')
ON CONFLICT (company_name) DO NOTHING;
```

### MERGE (PostgreSQL 15+)

```sql
MERGE INTO accounts target
USING staging_accounts source
ON target.company_name = source.company_name
WHEN MATCHED THEN
    UPDATE SET industry = source.industry
WHEN NOT MATCHED THEN
    INSERT (company_name, industry, status)
    VALUES (source.company_name, source.industry, 'active');
```

---

## String Functions

| Function | Description | Example |
|----------|-------------|---------|
| `length(s)` | character count | `length('hello')` → 5 |
| `char_length(s)` | same as length | |
| `octet_length(s)` | byte count | |
| `lower(s)` | lowercase | `lower('ABC')` → `'abc'` |
| `upper(s)` | uppercase | |
| `initcap(s)` | capitalize words | `initcap('hello world')` → `'Hello World'` |
| `trim(s)` | remove leading/trailing spaces | |
| `ltrim(s, chars)` | trim left | |
| `rtrim(s, chars)` | trim right | |
| `btrim(s, chars)` | trim both sides | |
| `lpad(s, len, fill)` | pad left | `lpad('42', 5, '0')` → `'00042'` |
| `rpad(s, len, fill)` | pad right | |
| `substring(s FROM n FOR m)` | extract substring | |
| `left(s, n)` | first n characters | |
| `right(s, n)` | last n characters | |
| `position(sub IN s)` | find substring position | |
| `strpos(s, sub)` | same as position | |
| `replace(s, from, to)` | replace all occurrences | |
| `translate(s, from, to)` | character-by-character replace | |
| `repeat(s, n)` | repeat string n times | |
| `reverse(s)` | reverse string | |
| `split_part(s, delim, n)` | split and get nth part | `split_part('a.b.c', '.', 2)` → `'b'` |
| `string_to_array(s, delim)` | split into array | |
| `array_to_string(arr, delim)` | join array into string | |
| `concat(a, b, ...)` | concatenate (NULL-safe) | |
| `concat_ws(sep, a, b, ...)` | concatenate with separator | |
| `format(fmt, ...)` | printf-style formatting | `format('Hello %s, id=%s', name, id)` |
| `encode(data, type)` | encode binary to text | `encode(digest('x','sha256'),'hex')` |
| `md5(s)` | MD5 hash as hex | |
| `regexp_replace(s, pat, rep)` | regex replace | |
| `regexp_matches(s, pat)` | regex match (returns arrays) | |
| `regexp_split_to_table(s, pat)` | split by regex into rows | |
| `regexp_split_to_array(s, pat)` | split by regex into array | |

### String concatenation operator

```sql
'Hello' || ' ' || 'World'   -- 'Hello World'
```

---

## Numeric Functions

| Function | Description |
|----------|-------------|
| `abs(x)` | absolute value |
| `ceil(x)` / `ceiling(x)` | round up |
| `floor(x)` | round down |
| `round(x, d)` | round to d decimal places |
| `trunc(x, d)` | truncate to d decimal places |
| `mod(x, y)` | modulo |
| `power(x, y)` | x raised to y |
| `sqrt(x)` | square root |
| `cbrt(x)` | cube root |
| `log(x)` | base-10 logarithm |
| `ln(x)` | natural logarithm |
| `exp(x)` | e^x |
| `sign(x)` | -1, 0, or 1 |
| `random()` | random value 0.0–1.0 |
| `setseed(x)` | set random seed |
| `greatest(a,b,...)` | largest value |
| `least(a,b,...)` | smallest value |
| `pi()` | π constant |
| `degrees(rad)` | radians to degrees |
| `radians(deg)` | degrees to radians |
| `div(x, y)` | integer quotient |
| `gcd(a, b)` | greatest common divisor (PG 13+) |
| `lcm(a, b)` | least common multiple (PG 13+) |
| `width_bucket(val, lo, hi, n)` | assign to histogram bucket |

---

## Date/Time Functions

### Current date/time

```sql
CURRENT_DATE              -- date only
CURRENT_TIME              -- time with tz
CURRENT_TIMESTAMP         -- timestamptz (same as now())
now()                     -- timestamptz
clock_timestamp()         -- actual wall clock (changes within statement)
statement_timestamp()     -- start of current statement
transaction_timestamp()   -- start of current transaction (= now())
```

### Extraction

```sql
EXTRACT(YEAR FROM timestamp)
EXTRACT(MONTH FROM timestamp)
EXTRACT(DOW FROM timestamp)        -- 0=Sunday
EXTRACT(EPOCH FROM timestamp)      -- Unix timestamp
date_part('hour', timestamp)       -- same as EXTRACT
```

### Truncation

```sql
date_trunc('month', timestamp)     -- first moment of that month
date_trunc('week', timestamp)
date_trunc('day', timestamp)
```

### Arithmetic

```sql
now() + INTERVAL '7 days'
now() - INTERVAL '3 hours'
'2025-12-31'::date - '2025-01-01'::date   -- integer days between
age('2025-06-01'::date, '1990-03-15'::date)  -- interval
```

### Formatting

```sql
to_char(now(), 'YYYY-MM-DD HH24:MI:SS')
to_char(amount, 'FM999,999.00')
to_date('15/03/2025', 'DD/MM/YYYY')
to_timestamp('2025-03-15 14:30', 'YYYY-MM-DD HH24:MI')
```

### Construction

```sql
make_date(2025, 3, 15)
make_timestamp(2025, 3, 15, 14, 30, 0)
make_timestamptz(2025, 3, 15, 14, 30, 0, 'Europe/Berlin')
make_interval(days => 7, hours => 3)
```

### Timezone conversion

```sql
timestamp AT TIME ZONE 'Europe/Berlin'
timestamp AT TIME ZONE 'UTC'
```

### Series generation

```sql
generate_series('2025-01-01'::date, '2025-12-01'::date, '1 month'::interval)
generate_series(1, 100)
generate_series(1, 100, 5)   -- step of 5
```

---

## Conditional Expressions

### CASE

```sql
SELECT
    company_name,
    CASE status
        WHEN 'active'    THEN 'Active'
        WHEN 'suspended' THEN 'On Hold'
        WHEN 'closed'    THEN 'Terminated'
        ELSE 'Unknown'
    END AS display_status
FROM accounts;

-- Searched CASE
SELECT
    amount,
    CASE
        WHEN amount > 10000 THEN 'large'
        WHEN amount > 1000  THEN 'medium'
        ELSE 'small'
    END AS size_category
FROM payments;
```

### COALESCE

Returns first non-NULL argument:

```sql
COALESCE(preferred_name, first_name, 'Anonymous')
```

### NULLIF

Returns NULL if both arguments are equal:

```sql
NULLIF(denominator, 0)  -- prevents division by zero
```

### GREATEST / LEAST

```sql
GREATEST(a, b, c)  -- largest non-NULL
LEAST(a, b, c)     -- smallest non-NULL
```

---

## Array Functions

```sql
-- Construction
ARRAY[1, 2, 3]
array_agg(column_name)
string_to_array('a,b,c', ',')

-- Access
arr[1]                      -- 1-indexed
arr[2:4]                    -- slice

-- Functions
array_length(arr, 1)        -- length of first dimension
array_dims(arr)             -- dimensions as text
array_upper(arr, 1)         -- upper bound
array_lower(arr, 1)         -- lower bound
array_append(arr, elem)     -- add to end
array_prepend(elem, arr)    -- add to start
array_cat(arr1, arr2)       -- concatenate
array_remove(arr, elem)     -- remove all occurrences
array_replace(arr, old, new)
array_position(arr, elem)   -- index of first occurrence
array_positions(arr, elem)  -- all indexes
unnest(arr)                 -- expand to rows
array_to_string(arr, ',')   -- join elements

-- Operators
arr @> ARRAY[2]             -- contains
arr <@ ARRAY[1,2,3]         -- is contained by
arr && ARRAY[2,5]           -- overlap
1 = ANY(arr)                -- element exists
1 = ALL(arr)                -- all elements equal
```

---

## JSON/JSONB Functions

### Construction

```sql
to_json(record)
to_jsonb(record)
json_build_object('key1', val1, 'key2', val2)
jsonb_build_object('name', 'test', 'active', true)
json_build_array(1, 2, 3)
row_to_json(row)
jsonb_agg(expression)
jsonb_object_agg(key, value)
```

### Extraction

```sql
data -> 'key'               -- JSON value
data ->> 'key'              -- text value
data -> 0                   -- array element
data #> '{a,b}'             -- nested path (JSON)
data #>> '{a,b}'            -- nested path (text)
jsonb_extract_path(data, 'a', 'b')
jsonb_extract_path_text(data, 'a', 'b')
```

### Modification (JSONB)

```sql
jsonb_set(data, '{key}', '"new_value"')
jsonb_set(data, '{nested,key}', '42', true)   -- create_if_missing
jsonb_insert(data, '{arr,0}', '"first"')
data || '{"new_key": "value"}'::jsonb         -- merge
data - 'key_to_remove'                        -- delete key
data #- '{nested,key}'                        -- delete at path
jsonb_strip_nulls(data)                       -- remove null values
```

### Iteration

```sql
jsonb_each(data)                 -- key-value pairs (value as jsonb)
jsonb_each_text(data)            -- key-value pairs (value as text)
jsonb_object_keys(data)          -- keys only
jsonb_array_elements(data)       -- array elements as jsonb rows
jsonb_array_elements_text(data)  -- array elements as text rows
jsonb_array_length(data)         -- array length
jsonb_typeof(data)               -- type name
```

### Querying JSONB

```sql
-- Containment
WHERE data @> '{"status": "active"}'

-- Key existence
WHERE data ? 'email'

-- Path exists (PG 12+)
WHERE jsonb_path_exists(data, '$.items[*] ? (@.price > 100)')

-- Path query (PG 12+)
jsonb_path_query(data, '$.items[*].name')
jsonb_path_query_first(data, '$.items[0].name')
jsonb_path_match(data, '$.active == true')
```

---

## Aggregate Functions

| Function | Description |
|----------|-------------|
| `count(*)` | row count |
| `count(col)` | non-NULL count |
| `count(DISTINCT col)` | distinct non-NULL count |
| `sum(col)` | total |
| `avg(col)` | average |
| `min(col)` | minimum |
| `max(col)` | maximum |
| `bool_and(col)` | true if all true |
| `bool_or(col)` | true if any true |
| `string_agg(col, delim)` | concatenate strings |
| `array_agg(col)` | collect into array |
| `jsonb_agg(col)` | collect into JSON array |
| `jsonb_object_agg(k, v)` | collect into JSON object |
| `percentile_cont(f) WITHIN GROUP (ORDER BY col)` | continuous percentile |
| `percentile_disc(f) WITHIN GROUP (ORDER BY col)` | discrete percentile |
| `mode() WITHIN GROUP (ORDER BY col)` | most frequent value |
| `corr(y, x)` | correlation coefficient |
| `regr_slope(y, x)` | linear regression slope |
| `stddev(col)` | standard deviation (sample) |
| `stddev_pop(col)` | standard deviation (population) |
| `variance(col)` | variance (sample) |
| `var_pop(col)` | variance (population) |

### FILTER clause

```sql
SELECT
    count(*) FILTER (WHERE status = 'active') AS active_count,
    count(*) FILTER (WHERE status = 'closed') AS closed_count
FROM accounts;
```

### WITHIN GROUP (ordered-set aggregates)

```sql
SELECT
    percentile_cont(0.5) WITHIN GROUP (ORDER BY amount) AS median,
    percentile_cont(0.95) WITHIN GROUP (ORDER BY amount) AS p95
FROM payments;
```

---

## Window Functions

### Syntax

```sql
function_name(...) OVER (
    [PARTITION BY expr, ...]
    [ORDER BY expr [ASC|DESC], ...]
    [frame_clause]
)
```

### Frame clause options

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW      -- default with ORDER BY
ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING               -- sliding window
RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING
```

### Ranking functions

| Function | Description |
|----------|-------------|
| `row_number()` | unique sequential number |
| `rank()` | rank with gaps |
| `dense_rank()` | rank without gaps |
| `ntile(n)` | divide into n groups |
| `percent_rank()` | relative rank (0–1) |
| `cume_dist()` | cumulative distribution |

### Value functions

| Function | Description |
|----------|-------------|
| `lag(col, n, default)` | value n rows before |
| `lead(col, n, default)` | value n rows after |
| `first_value(col)` | first value in frame |
| `last_value(col)` | last value in frame |
| `nth_value(col, n)` | nth value in frame |

### Aggregate as window

```sql
SELECT
    id,
    amount,
    SUM(amount) OVER (ORDER BY id) AS running_sum,
    AVG(amount) OVER (ORDER BY id ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) AS moving_avg_5
FROM payments;
```

### Named windows

```sql
SELECT
    id,
    amount,
    SUM(amount)   OVER w AS running_sum,
    COUNT(*)      OVER w AS running_count
FROM payments
WINDOW w AS (PARTITION BY account_id ORDER BY created_at);
```

---

## Set Operations

```sql
-- UNION (deduplicated)
SELECT email FROM contacts
UNION
SELECT email FROM archived_contacts;

-- UNION ALL (keep duplicates)
SELECT email FROM contacts
UNION ALL
SELECT email FROM archived_contacts;

-- INTERSECT
SELECT account_id FROM payments WHERE status = 'completed'
INTERSECT
SELECT account_id FROM payments WHERE amount > 5000;

-- EXCEPT (in first but not in second)
SELECT account_id FROM accounts
EXCEPT
SELECT DISTINCT account_id FROM payments;
```

---

## Subqueries and Lateral

### Scalar subquery

```sql
SELECT *, (SELECT MAX(amount) FROM payments p WHERE p.account_id = a.id) AS max_payment
FROM accounts a;
```

### Table subquery (derived table)

```sql
SELECT sub.account_id, sub.total
FROM (
    SELECT account_id, SUM(amount) AS total
    FROM payments GROUP BY account_id
) sub
WHERE sub.total > 10000;
```

### EXISTS / NOT EXISTS

```sql
SELECT * FROM accounts a
WHERE EXISTS (SELECT 1 FROM payments p WHERE p.account_id = a.id AND p.status = 'failed');
```

### IN / NOT IN

```sql
SELECT * FROM accounts WHERE id IN (SELECT account_id FROM payments WHERE amount > 10000);
```

### ANY / ALL

```sql
SELECT * FROM payments WHERE amount > ALL (SELECT amount FROM payments WHERE account_id = 5);
SELECT * FROM payments WHERE amount > ANY (SELECT amount FROM payments WHERE account_id = 5);
```

### LATERAL

Allows the subquery to reference columns from preceding tables:

```sql
SELECT a.company_name, latest.*
FROM accounts a
CROSS JOIN LATERAL (
    SELECT amount, paid_at
    FROM payments p
    WHERE p.account_id = a.id
    ORDER BY p.paid_at DESC
    LIMIT 1
) latest;
```

---

## Common Table Expressions

### Basic CTE

```sql
WITH active_accounts AS (
    SELECT * FROM accounts WHERE status = 'active'
)
SELECT * FROM active_accounts WHERE industry = 'SaaS';
```

### Multiple CTEs

```sql
WITH
    high_value AS (
        SELECT account_id, SUM(amount) AS total
        FROM payments WHERE status = 'completed'
        GROUP BY account_id
        HAVING SUM(amount) > 50000
    ),
    contacts_for_hv AS (
        SELECT c.* FROM contacts c JOIN high_value hv ON hv.account_id = c.account_id
    )
SELECT * FROM contacts_for_hv WHERE role = 'billing';
```

### Recursive CTE

```sql
WITH RECURSIVE hierarchy AS (
    SELECT id, company_name, parent_id, 1 AS level
    FROM accounts WHERE parent_id IS NULL

    UNION ALL

    SELECT a.id, a.company_name, a.parent_id, h.level + 1
    FROM accounts a JOIN hierarchy h ON a.parent_id = h.id
)
SELECT * FROM hierarchy ORDER BY level, company_name;
```

### Writable CTEs (data-modifying)

```sql
WITH moved AS (
    DELETE FROM payments
    WHERE status = 'completed' AND paid_at < '2020-01-01'
    RETURNING *
)
INSERT INTO archived_payments SELECT * FROM moved;
```

---

## Transactions and Locking

### Transaction control

```sql
BEGIN;
-- or: START TRANSACTION;

SAVEPOINT before_risky_op;

-- do work...

ROLLBACK TO SAVEPOINT before_risky_op;  -- undo partial
-- or:
COMMIT;
```

### Isolation levels

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;       -- default
BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;
```

### Row-level locking

```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;              -- exclusive lock
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE;       -- weaker exclusive
SELECT * FROM accounts WHERE id = 1 FOR SHARE;               -- shared lock
SELECT * FROM accounts WHERE id = 1 FOR UPDATE SKIP LOCKED;  -- skip locked rows
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;       -- error if locked
```

---

## Views and Materialized Views

### Regular view

```sql
CREATE VIEW active_billing_contacts AS
SELECT c.*, a.company_name
FROM contacts c
JOIN accounts a ON a.id = c.account_id
WHERE a.status = 'active' AND c.role = 'billing';
```

### Updatable view

```sql
CREATE VIEW pending_payments AS
SELECT * FROM payments WHERE status = 'pending'
WITH CHECK OPTION;  -- prevents inserting rows that don't match the view
```

### Materialized view

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    date_trunc('month', paid_at)::date AS month,
    SUM(amount) AS revenue
FROM payments
WHERE status = 'completed'
GROUP BY 1
WITH DATA;

-- Refresh
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

Requires a unique index for CONCURRENTLY:

```sql
CREATE UNIQUE INDEX idx_monthly_revenue_month ON monthly_revenue (month);
```

---

## Indexes

### B-tree (default)

```sql
CREATE INDEX idx_accounts_status ON accounts (status);
CREATE INDEX idx_payments_multi ON payments (account_id, status, paid_at DESC);
```

### Partial index

```sql
CREATE INDEX idx_pending_payments ON payments (due_date)
WHERE status = 'pending';
```

### Covering index (INCLUDE)

```sql
CREATE INDEX idx_payments_lookup ON payments (account_id, status)
INCLUDE (amount, paid_at);
```

### Expression index

```sql
CREATE INDEX idx_contacts_email_lower ON contacts (lower(email));
```

### Hash index

```sql
CREATE INDEX idx_accounts_name_hash ON accounts USING hash (company_name);
```

### GIN (for arrays, JSONB, full-text)

```sql
CREATE INDEX idx_data_gin ON documents USING gin (data jsonb_path_ops);
CREATE INDEX idx_tags_gin ON posts USING gin (tags);
```

### GiST (for ranges, geometry, full-text)

```sql
CREATE INDEX idx_validity_gist ON subscriptions USING gist (valid_during);
```

### BRIN (for naturally ordered large tables)

```sql
CREATE INDEX idx_payments_created_brin ON payments USING brin (created_at);
```

### Reindex

```sql
REINDEX INDEX idx_accounts_status;
REINDEX TABLE accounts;
```

---

## Constraints

```sql
-- Primary key
PRIMARY KEY

-- Unique
UNIQUE (email)
UNIQUE (account_id, email)  -- composite unique

-- Not null
NOT NULL

-- Check
CHECK (amount >= 0)
CHECK (status IN ('active', 'suspended', 'closed'))

-- Foreign key
REFERENCES accounts(id) ON DELETE CASCADE ON UPDATE CASCADE
REFERENCES accounts(id) ON DELETE SET NULL
REFERENCES accounts(id) ON DELETE RESTRICT

-- Exclusion constraint (no overlapping ranges)
CREATE TABLE reservations (
    room_id INT,
    during  TSTZRANGE,
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
);

-- Deferrable constraints
ALTER TABLE payments
ADD CONSTRAINT fk_account
    FOREIGN KEY (account_id) REFERENCES accounts(id)
    DEFERRABLE INITIALLY DEFERRED;
```

---

## Sequences and Identity

### Identity columns (preferred)

```sql
id BIGINT GENERATED ALWAYS AS IDENTITY
id BIGINT GENERATED BY DEFAULT AS IDENTITY
```

### Manual sequences

```sql
CREATE SEQUENCE invoice_number_seq START 1000 INCREMENT 1;
SELECT nextval('invoice_number_seq');
SELECT currval('invoice_number_seq');
SELECT setval('invoice_number_seq', 5000);
```

---

## PL/pgSQL Functions

### Basic function

```sql
CREATE OR REPLACE FUNCTION get_account_balance(p_account_id INT)
RETURNS NUMERIC
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_balance NUMERIC;
BEGIN
    SELECT COALESCE(SUM(amount), 0)
    INTO v_balance
    FROM payments
    WHERE account_id = p_account_id
      AND status = 'completed';

    RETURN v_balance;
END;
$$;
```

### Function returning a table

```sql
CREATE OR REPLACE FUNCTION overdue_accounts()
RETURNS TABLE(account_id INT, company_name TEXT, overdue_amount NUMERIC)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT p.account_id, a.company_name, SUM(p.amount)
    FROM payments p
    JOIN accounts a ON a.id = p.account_id
    WHERE p.status = 'pending' AND p.due_date < CURRENT_DATE
    GROUP BY p.account_id, a.company_name;
END;
$$;

-- Usage
SELECT * FROM overdue_accounts();
```

### Procedure (no return value, can manage transactions)

```sql
CREATE OR REPLACE PROCEDURE close_account(p_account_id INT)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts SET status = 'closed' WHERE id = p_account_id;
    UPDATE payments SET status = 'cancelled'
    WHERE account_id = p_account_id AND status = 'pending';

    COMMIT;
END;
$$;

-- Usage
CALL close_account(42);
```

### Error handling

```sql
CREATE OR REPLACE FUNCTION safe_divide(a NUMERIC, b NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    IF b = 0 THEN
        RAISE EXCEPTION 'Division by zero: % / %', a, b
            USING ERRCODE = 'division_by_zero';
    END IF;
    RETURN a / b;
EXCEPTION
    WHEN division_by_zero THEN
        RETURN NULL;
END;
$$;
```

---

## Triggers

### BEFORE trigger — auto-update timestamp

```sql
CREATE OR REPLACE FUNCTION update_modified_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.modified_at = now();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_accounts_modified
    BEFORE UPDATE ON accounts
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_at();
```

### AFTER trigger — audit log

```sql
CREATE OR REPLACE FUNCTION log_payment_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO payment_audit (payment_id, old_status, new_status, changed_at)
    VALUES (NEW.id, OLD.status, NEW.status, now());
    RETURN NULL;  -- AFTER triggers can return NULL
END;
$$;

CREATE TRIGGER trg_payment_audit
    AFTER UPDATE OF status ON payments
    FOR EACH ROW
    WHEN (OLD.status IS DISTINCT FROM NEW.status)
    EXECUTE FUNCTION log_payment_change();
```

### Statement-level trigger

```sql
CREATE TRIGGER trg_notify_bulk_insert
    AFTER INSERT ON payments
    FOR EACH STATEMENT
    EXECUTE FUNCTION pg_notify_new_payments();
```

---

## Full-Text Search

### tsvector and tsquery

```sql
SELECT to_tsvector('english', 'The quick brown fox jumps over the lazy dog');
SELECT to_tsquery('english', 'quick & fox');
```

### Search with ranking

```sql
SELECT
    id,
    company_name,
    ts_rank(to_tsvector('english', company_name), query) AS rank
FROM accounts, to_tsquery('english', 'acme | beta') query
WHERE to_tsvector('english', company_name) @@ query
ORDER BY rank DESC;
```

### Stored tsvector column with index

```sql
ALTER TABLE accounts ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (to_tsvector('english', coalesce(company_name,'') || ' ' || coalesce(industry,''))) STORED;

CREATE INDEX idx_accounts_fts ON accounts USING gin (search_vector);

SELECT * FROM accounts WHERE search_vector @@ to_tsquery('english', 'healthcare & active');
```

---

## Regular Expressions

```sql
-- Match
'hello123' ~ '\d+'              -- true
'Hello' ~* 'hello'              -- true (case-insensitive)

-- Extract
substring('Invoice #12345' FROM '#(\d+)')   -- '12345'

-- Replace
regexp_replace('foo  bar   baz', '\s+', ' ', 'g')  -- 'foo bar baz'

-- Split to rows
SELECT regexp_split_to_table('one,two,,four', ',');

-- All matches
SELECT regexp_matches('a1b2c3', '\d+', 'g');   -- returns {1}, {2}, {3} as rows
```

---

## Advisory Locks

Application-level cooperative locks (not tied to tables):

```sql
-- Session-level (held until explicit unlock or disconnect)
SELECT pg_advisory_lock(12345);
SELECT pg_advisory_unlock(12345);

-- Transaction-level (released at end of transaction)
SELECT pg_advisory_xact_lock(12345);

-- Try without blocking
SELECT pg_try_advisory_lock(12345);  -- returns true/false

-- Two-key variant (useful for entity locking)
SELECT pg_advisory_lock(hashtext('accounts'), 42);
```

---

## System Catalog Queries

### List all tables in current schema

```sql
SELECT tablename FROM pg_tables WHERE schemaname = 'public';
```

### Table sizes

```sql
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
    pg_size_pretty(pg_table_size(oid)) AS data_size,
    pg_size_pretty(pg_indexes_size(oid)) AS index_size
FROM pg_class
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace
ORDER BY pg_total_relation_size(oid) DESC;
```

### Index usage statistics

```sql
SELECT
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

### Currently running queries

```sql
SELECT pid, state, query_start, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

### Kill a query

```sql
SELECT pg_cancel_backend(pid);     -- gentle (cancel query)
SELECT pg_terminate_backend(pid);  -- forceful (kill connection)
```

### Table bloat estimate

```sql
SELECT
    schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS size,
    n_dead_tup,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### Lock monitoring

```sql
SELECT
    l.pid,
    l.locktype,
    l.mode,
    l.granted,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE NOT l.granted;
```

---

*End of SQL language guide.*
