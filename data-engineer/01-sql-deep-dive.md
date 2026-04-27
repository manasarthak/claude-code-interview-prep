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
-- Snowflake/BigQuery/Spark
SELECT *
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) = 1;
```

---

## Interview-Ready Cheat Sheet

**"Walk me through this slow query."** Pull the plan → find the dominant operator → check row estimates vs actuals → identify scan/join/sort issue → propose index, partition, broadcast, or rewrite.

**"What's the difference between WHERE and HAVING?"** WHERE filters rows before aggregation; HAVING filters groups after aggregation. WHERE can't reference aggregates; HAVING can.

**"Find each user's 3 most recent orders."** `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC)` ≤ 3.

**"What's the difference between RANK and DENSE_RANK?"** RANK leaves gaps after ties (1, 2, 2, 4); DENSE_RANK doesn't (1, 2, 2, 3).

**"How would you write an idempotent insert?"** `INSERT ... ON CONFLICT DO UPDATE` (Postgres) or `MERGE` (Snowflake/BQ/Redshift). Always include a unique business key.

**"`COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)`?"** All rows / non-NULL rows / unique non-NULL rows.

**"How do you detect gaps and islands (consecutive runs)?"** `ROW_NUMBER() OVER (PARTITION BY user ORDER BY date) - DENSE_RANK() based on date sequence` to assign a group id to consecutive runs, then aggregate.

**Quick trade-off pairs:**
- Subquery vs JOIN vs EXISTS: usually equivalent plans; write for clarity.
- CTE vs subquery: CTE for readability; not always faster.
- `COUNT(*)` vs `COUNT(1)`: identical, no perf difference.
- Indexes (OLTP) vs sort keys / clustering (OLAP): different mental models for different engines.

---

## Resources & Links

- [PostgreSQL Window Functions Tutorial](https://www.postgresql.org/docs/current/tutorial-window.html) — clearest explanation
- [Use The Index, Luke!](https://use-the-index-luke.com/) — index/query plan deep dive
- [Mode Analytics SQL Tutorial](https://mode.com/sql-tutorial/) — interview-style SQL drills
- [DataLemur](https://datalemur.com/) — DE-specific SQL practice
- [StrataScratch](https://www.stratascratch.com/) — real interview SQL questions

*Next: drill the 10 patterns above weekly. Add notes here as you encounter engine-specific quirks.*
