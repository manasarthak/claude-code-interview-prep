# SQL Deep Dive — The Most-Tested DE Interview Topic

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Data Modeling](./02-data-modeling.md) · [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md) · [Partitioning & Performance](./13-partitioning-performance.md)

---

## Why This File Exists

Every DE interview tests SQL. Every one. Even at companies that "use Spark for everything," you'll be asked to solve problems in SQL because it's the lingua franca and because how you write SQL reveals how you think about [data modeling](./02-data-modeling.md), [partitioning](./13-partitioning-performance.md), and [query optimization](#query-optimization-advanced).

If your SQL is weak, no other skill saves you in the interview.

---

## ⚠️ Online Assessment Engine Warning — Read This Before Drilling

Most timed online SQL assessments run **MySQL, MS SQL Server, Oracle, or DB2** — not Snowflake / BigQuery / Redshift. Many "killer features" of warehouse SQL **do not exist** on these engines:

| Feature | Snowflake / BigQuery / Spark | MySQL / MS SQL / Oracle / DB2 |
|---|---|---|
| `QUALIFY` | ✅ | ❌ — use CTE + `ROW_NUMBER()` + `WHERE rn = 1` |
| `DATE_TRUNC('month', x)` | ✅ | ❌ — use `DATE_FORMAT()`, `MONTH()`, `TRUNC()`, dialect-specific |
| `DATEDIFF(day, a, b)` | ✅ (3-arg) | MySQL is 2-arg `DATEDIFF(a, b)`; Oracle subtracts dates directly |
| `LISTAGG` / `STRING_AGG` / `GROUP_CONCAT` | All differ | MySQL: `GROUP_CONCAT`; MS SQL: `STRING_AGG`; Oracle: `LISTAGG` |
| Boolean type | ✅ | MySQL: TINYINT; SQL Server: BIT; Oracle: no native boolean |
| `LIMIT n` | ✅ | MySQL/Postgres: `LIMIT`; MS SQL: `TOP n` / `OFFSET..FETCH`; Oracle 12+: `FETCH FIRST n ROWS ONLY` |

**Rule of thumb for timed rounds:** before you write a single character, identify the engine (it's shown on the question), and bias toward the *portable* pattern (CTE + window function + standard `CASE`) over any dialect-specific sugar. Full dialect cheat sheet in the [Dialect Reference](#dialect-reference) section below.

---

## BEGINNER — The Core Mechanics

### The Six Logical Clauses (and Their Real Execution Order)

Most people learn SQL in the order it's written:
```
SELECT cols
FROM table
WHERE condition
GROUP BY ...
HAVING ...
ORDER BY ...
LIMIT n
```

But the database evaluates it in a different order — knowing this prevents 80% of "why doesn't my query work?" confusion:

| Step | Clause | What it does |
|---|---|---|
| 1 | FROM / JOIN | Build the working set of rows |
| 2 | WHERE | Filter rows (BEFORE grouping — can't reference aggregates) |
| 3 | GROUP BY | Bucket rows into groups |
| 4 | HAVING | Filter groups (AFTER aggregation — CAN reference aggregates) |
| 5 | SELECT | Compute output columns |
| 6 | DISTINCT | Deduplicate |
| 7 | ORDER BY | Sort |
| 8 | LIMIT / OFFSET | Trim rows |

**Why interviewers ask this:** It explains why `WHERE COUNT(*) > 5` fails but `HAVING COUNT(*) > 5` works, and why an alias defined in SELECT can't be used in WHERE.

### Joins You Must Know Cold

| Join | What you get |
|---|---|
| INNER JOIN | Only rows with matches in both tables |
| LEFT JOIN | All left rows + matching right rows (NULL where no match) |
| RIGHT JOIN | Mirror of LEFT (rare in practice) |
| FULL OUTER JOIN | All rows from both, NULLs where no match |
| CROSS JOIN | Cartesian product (every left × every right) |
| SELF JOIN | Join a table to itself (e.g., employee → manager hierarchy) |
| ANTI JOIN | "Rows in A with no match in B" — done via LEFT JOIN + IS NULL or NOT EXISTS |
| SEMI JOIN | "Rows in A that have a match in B" — done via EXISTS or IN |

**Pitfall to memorize:** Filtering a LEFT JOIN's right-side column in WHERE silently turns it into an INNER JOIN. Put right-side filters in the ON clause if you want to preserve outer-join semantics:

```sql
-- Wrong (becomes inner join): WHERE r.status = 'active'
-- Right (preserves outer): ON l.id = r.left_id AND r.status = 'active'
```

### NULL Behavior That Trips People Up

- `NULL = NULL` → NULL (not true). Use `IS NULL`.
- `NULL <> 'x'` → NULL, not true. Filter `WHERE col <> 'x'` excludes NULL rows; you need `OR col IS NULL` to include them.
- `COUNT(*)` counts all rows; `COUNT(col)` skips NULLs; `COUNT(DISTINCT col)` skips NULLs and dedupes.
- `SUM`, `AVG`, `MIN`, `MAX` all skip NULLs.
- `NULL` in `IN (...)` causes weird behavior — `WHERE x IN (1, 2, NULL)` works but `WHERE x NOT IN (1, 2, NULL)` returns nothing.

### GROUP BY Rules

- Every non-aggregated column in SELECT must appear in GROUP BY (in standard SQL).
- Some engines (MySQL with `ONLY_FULL_GROUP_BY` off, BigQuery with ANY_VALUE) relax this. Don't rely on it.
- `GROUP BY 1, 2` references SELECT columns by ordinal — fine for ad-hoc, error-prone in production code.

### NULL Handling Functions (constant in interviews)

NULL behavior was covered above — these are the **functions** you use to handle it. They appear on nearly every timed SQL question.

| Function | What it does | Available in |
|---|---|---|
| `COALESCE(a, b, c, ...)` | Returns the first non-NULL argument | Everywhere — standard SQL |
| `NULLIF(a, b)` | Returns NULL if a = b; else a. Most-used as a **divide-by-zero guard** | Everywhere — standard SQL |
| `IFNULL(a, b)` | MySQL-specific 2-arg version of COALESCE | MySQL, SQLite |
| `ISNULL(a, b)` | MS SQL Server's 2-arg replace-NULL | MS SQL only — **does not work** in MySQL/Postgres |
| `NVL(a, b)` | Oracle's 2-arg COALESCE | Oracle |
| `NVL2(a, if_not_null, if_null)` | Oracle 3-way variant | Oracle |
| `IS NULL` / `IS NOT NULL` | Predicate (not a function) — the only correct NULL comparison | Everywhere |

```sql
-- Replace NULLs in output (e.g., "show 0 instead of NULL for users with no orders")
SELECT u.user_id,
       COALESCE(SUM(o.amount), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON o.user_id = u.user_id
GROUP BY u.user_id;

-- NULLIF as divide-by-zero guard (returns NULL instead of erroring,
-- then COALESCE the NULL out)
SELECT product_id,
       COALESCE(SUM(revenue) / NULLIF(SUM(units), 0), 0) AS revenue_per_unit
FROM sales
GROUP BY product_id;

-- NULLIF to "blank out" sentinel values for downstream COALESCE
SELECT COALESCE(NULLIF(phone, ''), 'unknown') AS phone
FROM users;
```

**Portable rule:** prefer `COALESCE` over `IFNULL`/`ISNULL`/`NVL` — it's the only one that works in every engine.

### The Integer-Division & Numeric-Cast Trap (the #1 hidden-test killer)

The single most common "passes examples, fails hidden tests" bug in timed SQL rounds. Postgres, SQL Server, Oracle, Spark SQL all use **integer division** when both operands are integers:

```sql
-- WRONG: returns 0 for any ratio < 1, then ROUND can't save you
SELECT ROUND(shipped / total * 100, 2) FROM order_stats;
-- shipped=3, total=10 → 3/10 = 0 (integer!) → 0 * 100 = 0 → ROUND(0,2) = 0.00
```

Five idiomatic fixes — pick one and use it every time:

```sql
-- 1. Multiply by 1.0 (or 100.0) to promote to floating point — most portable
SELECT ROUND(100.0 * shipped / NULLIF(total, 0), 2) AS pct_shipped FROM order_stats;

-- 2. CAST one operand to DECIMAL/NUMERIC explicitly (preferred for exact decimals)
SELECT ROUND(CAST(shipped AS DECIMAL(10,4)) / NULLIF(total, 0) * 100, 2) FROM order_stats;

-- 3. CAST AS FLOAT / DOUBLE (when you don't care about decimal precision)
SELECT CAST(shipped AS FLOAT) / NULLIF(total, 0) FROM order_stats;

-- 4. MySQL-specific: division of integers already promotes to DECIMAL —
--    BUT this is engine-specific and unreliable to rely on.

-- 5. Postgres-specific shorthand: ::numeric
SELECT ROUND(shipped::numeric / NULLIF(total, 0) * 100, 2) FROM order_stats;
```

**Always pair the cast with `NULLIF(denominator, 0)`** to avoid the divide-by-zero error on edge rows.

`ROUND(x, n)` quirks to know:
- `ROUND(2.5, 0)` → some engines round half-up (3), some banker's-round (2). MySQL rounds half-away-from-zero (3); Postgres uses banker's rounding (2) for numeric, half-up for float. **Always verify on your target engine** if the question grades exact decimals.
- `ROUND` returns NULL if input is NULL — chain with `COALESCE` if your spec says "0 for NULL."

### String Functions — Constant in Timed Rounds, Often Underdrilled

You'll need at least one string function on roughly half of timed SQL questions. The standard set:

| Function | What it does | Notes |
|---|---|---|
| `LENGTH(s)` / `CHAR_LENGTH(s)` | Character count | MySQL: `LENGTH` returns *bytes* (multibyte!); use `CHAR_LENGTH` for char count |
| `SUBSTRING(s FROM start FOR len)` / `SUBSTR(s, start, len)` | Substring (1-indexed!) | Both forms; standard syntax uses FROM/FOR |
| `LEFT(s, n)` / `RIGHT(s, n)` | First/last n chars | MySQL/MS SQL/Postgres; not Oracle (use SUBSTR) |
| `TRIM(s)` / `LTRIM` / `RTRIM` | Strip whitespace (or chars: `TRIM(BOTH 'x' FROM s)`) | Standard |
| `UPPER(s)` / `LOWER(s)` | Case conversion | Standard |
| `REPLACE(s, from, to)` | Replace all occurrences | Standard |
| `CONCAT(a, b, c)` or `a \|\| b` | Concatenation | MySQL accepts `\|\|` only with `PIPES_AS_CONCAT`; Oracle/Postgres always use `\|\|`; MS SQL uses `+` |
| `POSITION(sub IN s)` / `STRPOS(s, sub)` / `INSTR(s, sub)` / `CHARINDEX(sub, s)` | Find position; 0 if absent | Naming differs by engine |
| `SPLIT_PART(s, delim, n)` | nth token after split (Postgres) | MySQL: `SUBSTRING_INDEX(s, delim, n)` |
| `REGEXP_REPLACE(s, pattern, repl)` | Regex substitution | Available in Postgres/MySQL 8/Oracle/MS SQL 2017+ |
| `LIKE 'foo%'` / `ILIKE 'foo%'` | Pattern match; `%` = any chars, `_` = one char; ILIKE = case-insensitive (Postgres only) | MySQL `LIKE` is case-insensitive by default for non-binary strings |
| `LPAD(s, n, c)` / `RPAD(s, n, c)` | Pad to length n with char c | Standard |
| `REVERSE(s)` | Reverse the string | MySQL / MS SQL / Postgres |
| `FORMAT(num, n)` / `TO_CHAR(num, fmt)` | Format number as string | MySQL: `FORMAT`; Oracle/Postgres: `TO_CHAR`; MS SQL: `FORMAT` |

```sql
-- "Find emails whose domain is gmail.com" — classic pattern
SELECT email
FROM users
WHERE LOWER(SUBSTRING(email FROM POSITION('@' IN email) + 1)) = 'gmail.com';

-- "Capitalize first letter of each name" — common ETL question
SELECT CONCAT(UPPER(LEFT(name, 1)), LOWER(SUBSTRING(name, 2))) AS proper_name
FROM users;

-- "Strip non-digits from phone" — Postgres/MySQL 8 regex
SELECT REGEXP_REPLACE(phone, '[^0-9]', '', 'g') AS phone_digits FROM users;

-- "Extract domain from URL" — SPLIT_PART idiom
SELECT SPLIT_PART(SPLIT_PART(url, '://', 2), '/', 1) AS domain FROM page_views;

-- MySQL equivalent using SUBSTRING_INDEX
SELECT SUBSTRING_INDEX(SUBSTRING_INDEX(url, '://', -1), '/', 1) AS domain FROM page_views;
```

**Common string traps:**
- **1-indexed, not 0-indexed.** `SUBSTRING('hello', 1, 3)` → `'hel'`, not `'ell'`.
- **LIKE escapes:** to match a literal `%`, use `LIKE '50\%' ESCAPE '\\'` (or double the escape per dialect).
- **NULL propagation in CONCAT:** in MySQL, `CONCAT('a', NULL)` returns NULL; in Postgres `||`, same. Use `CONCAT_WS` or wrap each arg in `COALESCE`.
- **Collation / case:** comparison case sensitivity depends on column collation — never assume.

### CASE Expressions — Plain and Aggregated

`CASE` is the universal conditional. You've seen `SUM(CASE WHEN ...)` for pivoting; the **plain CASE in the SELECT list** is equally common and often forgotten.

```sql
-- Plain CASE in SELECT — bucket a numeric column into categories
SELECT order_id,
       amount,
       CASE
         WHEN amount < 50            THEN 'small'
         WHEN amount < 500           THEN 'medium'
         WHEN amount < 5000          THEN 'large'
         ELSE                             'enterprise'
       END AS size_bucket
FROM orders;

-- CASE in ORDER BY — custom sort order
SELECT *
FROM tickets
ORDER BY CASE priority
           WHEN 'critical' THEN 1
           WHEN 'high'     THEN 2
           WHEN 'medium'   THEN 3
           ELSE                 4
         END;

-- CASE in WHERE — conditional filter
SELECT *
FROM orders
WHERE CASE WHEN region = 'US' THEN amount > 100
           ELSE                    amount > 50
      END;

-- Simple CASE syntax (compares one expression to multiple values)
SELECT CASE status
         WHEN 'A' THEN 'Active'
         WHEN 'I' THEN 'Inactive'
         ELSE          'Unknown'
       END AS status_label
FROM users;

-- CASE inside aggregates — the "pivot" pattern
SELECT user_id,
       SUM(CASE WHEN status = 'shipped'  THEN 1 ELSE 0 END) AS shipped_count,
       SUM(CASE WHEN status = 'returned' THEN amount ELSE 0 END) AS returned_revenue
FROM orders
GROUP BY user_id;

-- Postgres-only cleaner equivalent using FILTER
SELECT user_id,
       COUNT(*) FILTER (WHERE status = 'shipped')  AS shipped_count,
       SUM(amount) FILTER (WHERE status = 'returned') AS returned_revenue
FROM orders
GROUP BY user_id;
```

**Gotchas:**
- `CASE` returns NULL if no branch matches and no `ELSE`. Always include `ELSE`.
- All branches must return a **type-compatible** value — mixing INT and STRING errors out on most engines.
- Short-circuit evaluation: branches are evaluated in order; the first match wins.

### UNION vs UNION ALL — Quietly Critical

| | Behavior | Cost |
|---|---|---|
| `UNION` | Combines results and **dedupes** (implicit DISTINCT + sort) | Expensive — full sort/hash |
| `UNION ALL` | Combines results, keeps **all** rows including duplicates | Cheap — just concatenation |

```sql
-- WRONG: silently drops duplicate (user_id, order_date) rows
SELECT user_id, order_date FROM us_orders
UNION
SELECT user_id, order_date FROM eu_orders;

-- RIGHT: keeps every row
SELECT user_id, order_date FROM us_orders
UNION ALL
SELECT user_id, order_date FROM eu_orders;
```

**Default to `UNION ALL` unless you specifically want dedup.** Three reasons:
1. **Correctness** — if the same value can legitimately appear in both halves and you want both rows, `UNION` silently drops one.
2. **Performance** — the dedup is a full sort/hash on the combined result; on large datasets it dominates the cost.
3. **Clarity** — `UNION ALL` reads as "stack these two results"; `UNION` reads as "stack and dedupe" which is rarely what you want.

**Both halves must have the same number of columns and compatible types**, in the same order. Column names come from the first SELECT.

```sql
-- Tagging the source — extremely common DE pattern
SELECT 'US' AS region, user_id, amount FROM us_orders
UNION ALL
SELECT 'EU' AS region, user_id, amount FROM eu_orders;
```

For "rows in A but not in B" use `EXCEPT` (Postgres / MS SQL / Snowflake / Oracle 21+) or `MINUS` (Oracle, MySQL via `LEFT JOIN ... IS NULL`). For "rows in both" use `INTERSECT`. Both dedupe by default; `EXCEPT ALL` / `INTERSECT ALL` preserve multiplicity (Postgres / Spark).

---

## INTERMEDIATE — Where Mid-Level Interviews Live

### Window Functions — The #1 Most-Tested Intermediate Topic

Window functions compute a value across a "window" of related rows without collapsing the row count. If you remember one thing: `GROUP BY` aggregates and reduces rows, window functions aggregate and keep rows.

```sql
SELECT
  user_id,
  order_date,
  amount,
  -- running total per user
  SUM(amount) OVER (PARTITION BY user_id ORDER BY order_date) AS running_total,
  -- rank of this order's amount within the user's orders
  RANK()       OVER (PARTITION BY user_id ORDER BY amount DESC) AS amount_rank,
  -- previous order amount for the same user
  LAG(amount)  OVER (PARTITION BY user_id ORDER BY order_date) AS prev_amount
FROM orders;
```

**The four flavors of window functions:**

| Type | Examples | Use for |
|---|---|---|
| Aggregate | `SUM`, `AVG`, `COUNT`, `MAX`, `MIN` | Running totals, moving averages |
| Ranking | `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` | "Top N per group", percentile bucketing |
| Offset | `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE` | Compare row to prior/next, churn analysis |
| Distribution | `PERCENT_RANK`, `CUME_DIST` | Statistical questions |

**The classic interview question — "Top N per group":**

```sql
-- "For each user, return their 3 most recent orders"
WITH ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) AS rn
  FROM orders
)
SELECT * FROM ranked WHERE rn <= 3;
```

**`ROW_NUMBER` vs `RANK` vs `DENSE_RANK`:**
- `ROW_NUMBER`: 1, 2, 3, 4 (always unique, breaks ties arbitrarily)
- `RANK`: 1, 2, 2, 4 (ties share rank, gaps after)
- `DENSE_RANK`: 1, 2, 2, 3 (ties share rank, no gaps)

**Frame clauses** — what counts as "the window":

```sql
SUM(amount) OVER (
  PARTITION BY user_id
  ORDER BY order_date
  ROWS BETWEEN 6 PRECEDING AND CURRENT ROW   -- 7-day moving sum
)
```

| Frame | Meaning |
|---|---|
| `ROWS BETWEEN N PRECEDING AND CURRENT ROW` | Row-count window |
| `RANGE BETWEEN INTERVAL '7' DAY PRECEDING AND CURRENT ROW` | Time-based window |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | Running total (default for cumulative aggregates with ORDER BY) |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | All rows in partition |

### CTEs and Recursive CTEs

CTEs (`WITH`) make complex queries readable and reusable within a single query.

```sql
WITH active_users AS (
  SELECT user_id FROM users WHERE status = 'active'
), recent_orders AS (
  SELECT * FROM orders WHERE order_date >= CURRENT_DATE - INTERVAL '30' DAY
)
SELECT u.user_id, COUNT(*) AS order_count
FROM active_users u
JOIN recent_orders r ON u.user_id = r.user_id
GROUP BY u.user_id;
```

**CTEs vs subqueries:**
- CTEs are not always faster — many engines inline them. Some (older Postgres) materialize them, which can be slow.
- Use CTEs for *readability*, not as a perf optimization.

**Recursive CTEs** for hierarchies (org charts, category trees, BOM explosions):

```sql
WITH RECURSIVE org AS (
  -- anchor: top-level employees
  SELECT id, name, manager_id, 1 AS depth
  FROM employees WHERE manager_id IS NULL

  UNION ALL

  -- recursive: employees reporting to someone already in the CTE
  SELECT e.id, e.name, e.manager_id, o.depth + 1
  FROM employees e
  JOIN org o ON e.manager_id = o.id
)
SELECT * FROM org ORDER BY depth, name;
```

### Subquery Patterns

| Pattern | Example | Use case |
|---|---|---|
| Scalar subquery | `WHERE x > (SELECT AVG(x) FROM t)` | Compare against an aggregate |
| Correlated | `WHERE EXISTS (SELECT 1 FROM b WHERE b.id = a.id)` | Per-row check against another table |
| `IN` / `NOT IN` | `WHERE id IN (SELECT id FROM ...)` | Set membership |
| `EXISTS` | `WHERE EXISTS (...)` | "Does any matching row exist?" |
| `LATERAL` join | `FROM a, LATERAL (SELECT ... WHERE x = a.x)` | Per-row subquery that can reference outer row |

**`EXISTS` vs `IN` vs `JOIN`:**
- All three can express "rows in A with matching rows in B."
- `EXISTS` short-circuits — stops at first match per row. Best when matches are rare-but-deep.
- `IN` materializes the set first. Fine for small result sets.
- `JOIN` works when you need columns from B too.
- Modern optimizers usually pick the same plan for all three. Write for clarity.

### Pivot / Unpivot

Pivoting (rows → columns) and unpivoting (columns → rows) is interview catnip and dbt-adjacent work.

```sql
-- Pivot via conditional aggregation (works everywhere)
SELECT
  user_id,
  SUM(CASE WHEN status = 'shipped'  THEN 1 ELSE 0 END) AS shipped_count,
  SUM(CASE WHEN status = 'returned' THEN 1 ELSE 0 END) AS returned_count
FROM orders
GROUP BY user_id;
```

Some engines have a `PIVOT` operator (Snowflake, BigQuery, SQL Server, Spark SQL). The conditional-aggregation form is universal.

### Date / Time Patterns

```sql
DATE_TRUNC('month', ts)              -- bucket to month start
EXTRACT(DOW FROM ts)                  -- day of week
DATEDIFF(day, start, end)             -- days between (Snowflake/Redshift)
ts AT TIME ZONE 'UTC'                 -- timezone conversion
GENERATE_SERIES(...)                  -- Postgres date series
```

**Gotcha:** Time-zone handling. Always store timestamps in UTC; convert at the SELECT layer for display. Lots of bugs come from naive `TIMESTAMP` columns silently in different zones.

---

## ADVANCED — Performance, Query Plans, and the Hard Questions {#query-optimization-advanced}

### Reading a Query Plan

Every interviewer worth their salt will say "explain the query plan." `EXPLAIN` (or `EXPLAIN ANALYZE` in Postgres, `EXPLAIN PLAN` in Redshift, `EXPLAIN FORMATTED` in Spark) shows what the optimizer chose.

The things to look for:

| Operator | What it means | When to worry |
|---|---|---|
| Seq Scan / Full Table Scan | Reads every row | Large tables without selective filter — wants an index or partition prune |
| Index Scan / Index Seek | Uses an index | Usually good; bad if it's followed by huge "key lookup" |
| Hash Join | Build hash table from smaller side, probe with larger | Good for big × small; bad if "smaller" side doesn't fit memory |
| Nested Loop Join | For each left row, scan/seek right | Great for tiny × indexed-large; deadly for big × big |
| Merge Join | Both sides sorted, merge in tandem | Great when inputs already sorted |
| Sort | Materializes sorted result | Watch for large sorts; can spill to disk |
| Broadcast (Spark) | Replicate small table to every executor | Great for ≤ 100MB; OOM if too big |
| Shuffle / Exchange (Spark) | Redistribute by key | Expensive — minimize them |

**Worked example — interview classic:** "Why is my query slow?" walk-through:
1. Pull the plan.
2. Find the operator with the highest cost / longest time.
3. Look at row estimates vs actuals — large mismatches = bad statistics.
4. Check for sequential scans on large tables; check filter selectivity.
5. Check joins: hash join with build side that doesn't fit? Nested loop on big × big?
6. Check for unnecessary sorts.
7. Then propose: index, partition prune, rewrite, broadcast hint, statistics refresh.

### Indexes (OLTP) vs Sort Keys / Clustering (OLAP)

OLTP databases (Postgres, MySQL) use B-tree indexes. OLAP warehouses (Redshift, Snowflake, BigQuery) use sort keys / clustering keys / partition pruning instead — see [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md) and [Partitioning & Performance](./13-partitioning-performance.md).

| System | Speed-up mechanism |
|---|---|
| Postgres / MySQL | B-tree, hash, GIN indexes |
| Redshift | DISTKEY (data distribution), SORTKEY (zone maps) |
| Snowflake | Micro-partitions + clustering keys |
| BigQuery | Partitioning + clustering |
| Spark / Iceberg / Delta | Partitioning + Z-ordering / liquid clustering + bloom filters |

**Why it matters in interviews:** indexes shouldn't be thrown around as the answer to slow analytical queries. Knowing which mechanism applies to which engine signals real experience.

### Query Optimization Patterns

| Symptom | Likely cause | Fix |
|---|---|---|
| Slow GROUP BY | Hash table doesn't fit memory | Pre-aggregate, partition prune, increase memory |
| Slow JOIN on large tables | Shuffle dominates | Repartition by join key, broadcast small side |
| Slow ORDER BY at end | Full sort of large result | Use `LIMIT` with index/sort key on ordered column; pre-sort upstream |
| `SELECT *` is slow | Reads/transports all columns | List only needed columns (huge win on columnar engines) |
| `OR` in WHERE is slow | Optimizer can't use indexes | Rewrite as `UNION ALL` of index-friendly conditions |
| `LIKE '%foo%'` is slow | No prefix → can't use index | Full-text index, trigram, or `LIKE 'foo%'` if business allows |
| `NOT IN` with NULL | Returns nothing | Use `NOT EXISTS` or filter NULLs out |

### Idempotent and Incremental Queries

DE-specific SQL skill. See [ETL vs ELT & Pipeline Patterns](./04-etl-vs-elt.md) for the broader story.

**Idempotent insert pattern (no dups on re-run):**
```sql
-- Postgres
INSERT INTO target (id, ...)
SELECT id, ... FROM source
ON CONFLICT (id) DO UPDATE SET ...

-- Snowflake / BigQuery
MERGE INTO target USING source
ON target.id = source.id
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...
```

**Incremental window pattern (process only new data):**
```sql
INSERT INTO daily_agg
SELECT date_trunc('day', ts) AS day, COUNT(*) AS events
FROM events
WHERE ts >= (SELECT COALESCE(MAX(day), '2020-01-01') FROM daily_agg)
  AND ts <  date_trunc('day', current_timestamp)  -- exclude in-flight day
GROUP BY 1;
```

### Common Interview Question Patterns to Drill

These questions show up *constantly*. Memorize the patterns, not the specific SQL:

1. **Top N per group** — `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` then filter.
2. **Nth highest salary** — `DENSE_RANK()` then filter; or correlated subquery.
3. **Consecutive event detection** (e.g., 3+ days in a row of activity) — `LAG`/`LEAD` to compare adjacent rows, or "gaps and islands" with `ROW_NUMBER` differences.
4. **Sessionization** — group events into sessions where gaps > N minutes. `LAG` for previous timestamp, `SUM(case when gap > N)` for session id.
5. **Funnel / cohort analysis** — `LEAD` to find next event in funnel, count drop-off; `DATE_TRUNC` to bucket cohorts.
6. **Median / percentile** — `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY x)` or `NTILE(100)`.
7. **Running totals + reset on group change** — partitioned window functions.
8. **Find duplicates** — `GROUP BY ... HAVING COUNT(*) > 1` or `ROW_NUMBER() ... rn > 1`.
9. **Anti-joins** ("users who never ordered") — `LEFT JOIN ... WHERE r.id IS NULL` or `NOT EXISTS`.
10. **Pivot dynamic columns** — conditional aggregation; for true dynamic, build query string in dbt/Jinja.

If you can't write each of these in under 5 minutes from a blank screen, drill them on LeetCode/StrataScratch/DataLemur until you can.

### Engine-Specific Differences That Matter

| Feature | Postgres | Snowflake | BigQuery | Redshift | Spark SQL |
|---|---|---|---|---|---|
| `QUALIFY` clause for filtering window output | ❌ | ✅ | ✅ | ❌ | ✅ |
| `MERGE` | ✅ (15+) | ✅ | ✅ | ✅ | ✅ (Delta/Iceberg) |
| Array / struct types | ✅ JSONB | ✅ VARIANT | ✅ STRUCT/ARRAY | Limited | ✅ |
| Time travel | Limited (logical) | ✅ | ✅ (limited) | ❌ | ✅ via Iceberg/Delta |
| Auto-clustering | ❌ | ✅ | ✅ | Manual | Liquid clustering (Delta) |

`QUALIFY` is a quality-of-life killer feature — lets you filter on window functions without a CTE:

```sql
-- Snowflake/BigQuery/Spark ONLY — does NOT work on MySQL/MS SQL/Oracle/Postgres
SELECT *
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) = 1;
```

**⚠️ DO NOT use QUALIFY on most online SQL assessments.** Their engines (MySQL, MS SQL, Oracle, DB2, sometimes Postgres) don't support it. Always default to the portable pattern:

```sql
-- Portable equivalent — works on every engine
WITH ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) AS rn
  FROM orders
)
SELECT * FROM ranked WHERE rn = 1;
```

A Postgres-only shorthand for "one row per group" is `DISTINCT ON`:

```sql
-- Postgres only — first row per user_id by date desc
SELECT DISTINCT ON (user_id) *
FROM orders
ORDER BY user_id, order_date DESC;
```

### NULLS ordering and FILTER (small wins worth knowing)

```sql
-- NULL placement in ORDER BY — Postgres, Oracle, SQL Server 2022+
SELECT * FROM users ORDER BY last_login DESC NULLS LAST;
SELECT * FROM users ORDER BY last_login ASC  NULLS FIRST;

-- MySQL has no NULLS LAST/FIRST — emulate with a boolean sort key
SELECT * FROM users ORDER BY last_login IS NULL, last_login DESC;
-- (IS NULL returns 0 for non-NULL, 1 for NULL → non-NULL sorts first)

-- FILTER clause — Postgres / Spark / DuckDB / SQLite 3.30+
-- Same effect as SUM(CASE WHEN ... THEN x ELSE 0 END) but cleaner:
SELECT
  user_id,
  COUNT(*)       FILTER (WHERE status = 'shipped')  AS shipped_count,
  SUM(amount)    FILTER (WHERE status = 'returned') AS returned_amount,
  AVG(rating)    FILTER (WHERE rating IS NOT NULL)  AS avg_rating
FROM orders
GROUP BY user_id;
```

If your target engine doesn't support `FILTER`, fall back to `SUM(CASE WHEN ... THEN x ELSE 0 END)` — universally supported, slightly noisier.

---

## Dialect Reference {#dialect-reference}

Most online assessments pin you to one engine. **Before writing anything, confirm the engine** (the question header tells you), then use this table to translate your mental model:

### Dates and Times

| Need | MySQL | MS SQL Server | Oracle | Postgres / Snowflake / BQ / Spark |
|---|---|---|---|---|
| Current timestamp | `NOW()` / `CURRENT_TIMESTAMP` | `GETDATE()` / `SYSDATETIME()` | `SYSDATE` / `CURRENT_TIMESTAMP` | `NOW()` / `CURRENT_TIMESTAMP` |
| Today's date | `CURDATE()` | `CAST(GETDATE() AS DATE)` | `TRUNC(SYSDATE)` | `CURRENT_DATE` |
| Year/Month/Day extract | `YEAR(d)` / `MONTH(d)` / `DAY(d)` | Same | `EXTRACT(YEAR FROM d)` | `EXTRACT` or `DATE_PART` |
| Truncate to month | `DATE_FORMAT(d, '%Y-%m-01')` | `DATEFROMPARTS(YEAR(d), MONTH(d), 1)` | `TRUNC(d, 'MM')` | `DATE_TRUNC('month', d)` |
| Days between two dates | `DATEDIFF(d2, d1)` (2-arg) | `DATEDIFF(day, d1, d2)` (3-arg) | `d2 - d1` (returns days) | `(d2 - d1)` or `DATE_DIFF(d2, d1, DAY)` |
| Add 7 days | `DATE_ADD(d, INTERVAL 7 DAY)` | `DATEADD(day, 7, d)` | `d + 7` or `d + INTERVAL '7' DAY` | `d + INTERVAL '7 days'` |
| Parse string → date | `STR_TO_DATE(s, '%Y-%m-%d')` | `CONVERT(DATE, s, 23)` or `TRY_CAST` | `TO_DATE(s, 'YYYY-MM-DD')` | `TO_DATE(s, 'YYYY-MM-DD')` or `s::date` |
| Format date → string | `DATE_FORMAT(d, '%Y-%m-%d')` | `FORMAT(d, 'yyyy-MM-dd')` | `TO_CHAR(d, 'YYYY-MM-DD')` | `TO_CHAR(d, 'YYYY-MM-DD')` |
| Day-of-week (1–7) | `DAYOFWEEK(d)` (Sun=1) | `DATEPART(weekday, d)` | `TO_CHAR(d,'D')` | `EXTRACT(DOW FROM d)` (Sun=0) |
| Day name | `DAYNAME(d)` | `DATENAME(weekday, d)` | `TO_CHAR(d,'DAY')` | `TO_CHAR(d,'Day')` |

### Strings

| Need | MySQL | MS SQL | Oracle | Postgres |
|---|---|---|---|---|
| Concat | `CONCAT(a,b)` (NULLs → NULL) or `CONCAT_WS(sep,...)` | `a + b` or `CONCAT` | `a \|\| b` or `CONCAT(a,b)` (2-arg) | `a \|\| b` or `CONCAT` |
| Substring | `SUBSTRING(s, start, len)` | `SUBSTRING(s, start, len)` | `SUBSTR(s, start, len)` | `SUBSTRING(s FROM start FOR len)` |
| Length (chars) | `CHAR_LENGTH(s)` (LENGTH = bytes) | `LEN(s)` | `LENGTH(s)` | `CHAR_LENGTH(s)` or `LENGTH(s)` |
| Position of substring | `LOCATE(sub, s)` or `INSTR` | `CHARINDEX(sub, s)` | `INSTR(s, sub)` | `POSITION(sub IN s)` or `STRPOS(s, sub)` |
| Replace | `REPLACE(s, from, to)` | Same | Same | Same |
| Pad | `LPAD(s, n, c)` / `RPAD` | `RIGHT(REPLICATE('0',n)+s, n)` (no LPAD) | `LPAD(s, n, c)` | `LPAD(s, n, c)` |
| Split nth token | `SUBSTRING_INDEX(s, delim, n)` | `STRING_SPLIT(s, delim)` (table) | `REGEXP_SUBSTR(s, '[^x]+', 1, n)` | `SPLIT_PART(s, delim, n)` |
| Regex match | `s REGEXP pat` | `LIKE` only natively; CLR for regex | `REGEXP_LIKE(s, pat)` | `s ~ pat` (case-sens) / `s ~* pat` (case-insens) |
| Regex replace | `REGEXP_REPLACE(s, pat, repl)` (MySQL 8+) | `STRING_AGG` workarounds | `REGEXP_REPLACE` | `REGEXP_REPLACE(s, pat, repl, 'g')` |
| Group concat | `GROUP_CONCAT(col SEPARATOR ',')` | `STRING_AGG(col, ',')` (2017+) | `LISTAGG(col, ',') WITHIN GROUP (ORDER BY ...)` | `STRING_AGG(col, ',')` |

### Top-N / Pagination

| Need | MySQL | MS SQL | Oracle | Postgres / Snowflake |
|---|---|---|---|---|
| Top N rows | `LIMIT n` | `SELECT TOP n ...` or `OFFSET 0 ROWS FETCH NEXT n ROWS ONLY` | `FETCH FIRST n ROWS ONLY` (12c+) or `WHERE ROWNUM <= n` | `LIMIT n` |
| Skip & take | `LIMIT offset, n` | `OFFSET k ROWS FETCH NEXT n ROWS ONLY` | `OFFSET k ROWS FETCH NEXT n ROWS ONLY` | `LIMIT n OFFSET k` |
| Filter on window function | CTE + `WHERE rn = 1` | Same | Same | Same; Snowflake/BQ have `QUALIFY` |

### Numeric / Type Casting

| Need | Portable form |
|---|---|
| Promote to decimal | `CAST(x AS DECIMAL(10,4))` |
| Promote to float | `CAST(x AS FLOAT)` or `x * 1.0` |
| Truncate to int | `CAST(x AS UNSIGNED)` (MySQL) / `CAST(x AS INT)` (most) / `TRUNC(x)` (Oracle) |
| Postgres shorthand | `x::numeric` / `x::int` / `x::text` |

### Booleans

- MySQL: no real BOOLEAN — `TRUE`/`FALSE` are aliases for `1`/`0`; column type is `TINYINT(1)`.
- MS SQL: use `BIT` (0/1); no `TRUE`/`FALSE` keywords in expressions.
- Oracle: no boolean type; use `CHAR(1) CHECK (c IN ('Y','N'))` or `NUMBER(1)`.
- Postgres / Snowflake / BigQuery: real `BOOLEAN` type.

When in doubt, return `1`/`0` integers — every engine accepts that.

### The "Before You Hit Run" Checklist

1. **Confirm the engine.** Date/string syntax depends on it.
2. **Trace integer division.** Every `/` between two integer columns → cast or multiply by 1.0.
3. **Wrap every denominator in `NULLIF(d, 0)`.**
4. **`COALESCE` every aggregate that could return NULL** (LEFT JOIN sums, etc.) if the spec says "0 for missing."
5. **Choose `UNION ALL` unless you specifically want dedup.**
6. **Round to the spec's decimals — `ROUND(x, 2)` is almost always the rounding the grader expects.**
7. **`ORDER BY` matches the example output exactly** — sort columns, direction, tiebreaker.
8. **Hidden tests usually break on:** division-by-zero, NULL handling, ties, empty input, leap years, time-zone edge cases.

---

## Interview-Ready Cheat Sheet

**"Walk me through this slow query."** Pull the plan → find the dominant operator → check row estimates vs actuals → identify scan/join/sort issue → propose index, partition, broadcast, or rewrite.

**"What's the difference between WHERE and HAVING?"** WHERE filters rows before aggregation; HAVING filters groups after aggregation. WHERE can't reference aggregates; HAVING can.

**"Find each user's 3 most recent orders."** `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC)` ≤ 3.

**"What's the difference between RANK and DENSE_RANK?"** RANK leaves gaps after ties (1, 2, 2, 4); DENSE_RANK doesn't (1, 2, 2, 3).

**"How would you write an idempotent insert?"** `INSERT ... ON CONFLICT DO UPDATE` (Postgres) or `MERGE` (Snowflake/BQ/Redshift). Always include a unique business key.

**"`COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)`?"** All rows / non-NULL rows / unique non-NULL rows.

**"How do you detect gaps and islands (consecutive runs)?"** `ROW_NUMBER() OVER (PARTITION BY user ORDER BY date) - DENSE_RANK() based on date sequence` to assign a group id to consecutive runs, then aggregate.

**"What's `COALESCE` and `NULLIF` good for?"** `COALESCE(a,b,c)` returns the first non-NULL — used to default missing values. `NULLIF(a,b)` returns NULL if `a=b` — most commonly `x / NULLIF(y, 0)` to avoid divide-by-zero (the error becomes a NULL you can then `COALESCE`).

**"Why does my percentage column return 0?"** Integer division. `shipped / total` between two ints rounds to 0 for any ratio < 1. Fix with `100.0 * shipped / NULLIF(total, 0)` or `CAST(shipped AS DECIMAL) / NULLIF(total, 0)`.

**"`UNION` vs `UNION ALL`?"** `UNION` dedupes (full sort/hash, expensive, can silently drop rows you wanted); `UNION ALL` keeps everything (cheap concat). Default to `UNION ALL`.

**"How do you write `LIKE` patterns?"** `%` = any chars (incl. empty), `_` = exactly one char. `LIKE 'foo%'` uses an index; `LIKE '%foo%'` cannot. Use `ILIKE` (Postgres) or `LOWER(col) LIKE LOWER(pat)` for case-insensitive matching.

**Quick trade-off pairs:**
- Subquery vs JOIN vs EXISTS: usually equivalent plans; write for clarity.
- CTE vs subquery: CTE for readability; not always faster.
- `COUNT(*)` vs `COUNT(1)`: identical, no perf difference.
- `UNION` vs `UNION ALL`: dedupe + sort vs cheap concat; default to ALL.
- `IFNULL` / `ISNULL` / `NVL` vs `COALESCE`: dialect-specific 2-arg vs portable n-arg. Always prefer COALESCE.
- `QUALIFY` (Snowflake/BQ/Spark) vs `CTE + WHERE rn = 1`: shorthand vs portable. Default to portable in timed rounds.
- Indexes (OLTP) vs sort keys / clustering (OLAP): different mental models for different engines.

---

## Resources & Links

### Reference docs (bookmark, by dialect)
- [PostgreSQL docs — Functions and Operators](https://www.postgresql.org/docs/current/functions.html)
- [MySQL 8 — Functions and Operators](https://dev.mysql.com/doc/refman/8.0/en/functions.html)
- [MS SQL Server — Functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/functions)
- [Oracle SQL — Functions](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Functions.html)
- [Snowflake — SQL Function Reference](https://docs.snowflake.com/en/sql-reference-functions)
- [BigQuery — Function Reference](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators)

### Practice
- [PostgreSQL Window Functions Tutorial](https://www.postgresql.org/docs/current/tutorial-window.html) — clearest explanation
- [Use The Index, Luke!](https://use-the-index-luke.com/) — index/query plan deep dive
- [Mode Analytics SQL Tutorial](https://mode.com/sql-tutorial/) — interview-style SQL drills
- [DataLemur](https://datalemur.com/) — DE-specific SQL practice
- [StrataScratch](https://www.stratascratch.com/) — real interview SQL questions
- [SQLZoo](https://sqlzoo.net/) — engine-switchable drills (MySQL, MS SQL, Oracle, Postgres)
- [LeetCode SQL track](https://leetcode.com/problemset/database/) — timed-assessment-style problems

### Companion files
- [Pandas Data Manipulation](./16-pandas-data-manipulation.md) — the pandas equivalent for coding-round data wrangling
- [Data Modeling](./02-data-modeling.md)
- [Partitioning & Performance](./13-partitioning-performance.md)

*Next: drill the 10 patterns above weekly. Add notes here as you encounter engine-specific quirks.*
