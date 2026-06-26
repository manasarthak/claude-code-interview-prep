# Pandas Data Manipulation — Complete Guide (tuned for QuantumBlack-style DE OAs)

**Phase:** 1 (Foundations) → 2 (Production)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [SQL Deep Dive](./01-sql-deep-dive.md) · [Spark Fundamentals](./07-spark-fundamentals.md) · [Python Internals](../python/02-python-internals.md) · [Arrays & Buffers](../python/03-arrays-and-buffers.md)

---

## Why This File Exists

DE online assessments — QuantumBlack, McKinsey, Stripe, Two Sigma, and most consulting/quant DE shops — hit pandas hard. The patterns are repeatable: fill-NA-with-group-mean, string extraction/cleaning, nested-JSON ingestion, joins/anti-joins, dedup/data-quality, feature engineering, and the McKinsey API-pagination staple. This file is a re-learning + reference guide that covers the full pandas surface an OA hits, with runnable code, the **SQL→pandas bridge** (your fastest path since your [SQL](./01-sql-deep-dive.md) is solid), and a gotchas list at the end (the OA punishes edge cases, not concepts).

Assumes **pandas 2.x** — some old idioms like `df.append` and `fillna(method=...)` are removed/deprecated; the current ones are used throughout.

```python
import pandas as pd
import numpy as np
```

---

## 0. Mental Model (read once, then re-read)

- **Series** = one labeled column (a 1-D array + an index).
- **DataFrame** = dict of Series sharing one index.
- **`axis=0`** = down the rows (operate per column). **`axis=1`** = across columns (operate per row). Default is `axis=0`.
- **Index** is not "just row numbers" — it's a label set that survives filters/joins. After a `groupby`, your group keys become the index; `reset_index()` turns them back into columns.
- **Vectorize, don't loop.** `df['a'] + df['b']` is fast; `df.apply(..., axis=1)` and Python `for` loops are slow and a common reason OA solutions time out. The reason is [in the Python internals file](../python/02-python-internals.md): pure-Python loops box every value and pay GIL/dispatch overhead, while vectorized ops run in compiled C/SIMD on contiguous NumPy buffers.

---

## 1. Create & Inspect

```python
df = pd.DataFrame({
    'id':    [1, 2, 3, 4],
    'dept':  ['A', 'A', 'B', 'B'],
    'salary':[100, np.nan, 200, 250],
})

df.head()                  # first 5 rows
df.shape                   # (rows, cols)
df.info()                  # dtypes + non-null counts  <-- read this first on any unknown df
df.describe()              # numeric summary
df.describe(include='all') # include object/datetime columns
df.dtypes
df.memory_usage(deep=True) # real memory cost (object dtype is HUGE)

df['dept'].unique()
df['dept'].value_counts(dropna=False)  # counts per category; pass dropna=False to see NaN counts
df['dept'].nunique()                   # distinct count (skips NaN)
df.isna().sum()                        # nulls per column  <-- do this early, always
df.isna().mean() * 100                 # % null per column
```

**`info()` and `isna().sum()` are your first two moves on any mystery DataFrame in the OA.**

---

## 2. Select & Filter

```python
# Columns
df['salary']             # Series
df[['id', 'salary']]     # DataFrame (note double brackets)

# Rows by boolean mask
df[df['salary'] > 150]
df[(df['dept'] == 'B') & (df['salary'] > 150)]   # use & | ~, wrap each condition in ()
df[df['dept'].isin(['A', 'B'])]
df[df['salary'].between(100, 200)]               # inclusive both ends
df[df['salary'].isna()]                          # NEVER df['salary'] == np.nan (always False)
df[~df['dept'].isin(['A'])]                      # ~ for NOT

# Label vs position
df.loc[df['dept'] == 'A', 'salary']   # label-based: rows by mask, col by name
df.loc[0:2, 'id':'salary']            # label slice is INCLUSIVE on both ends
df.iloc[0:2, 1:3]                     # position-based: integer slices, exclusive end

# query() — readable, good for OA
df.query('dept == "A" and salary > 50')
df.query('dept in @dept_list')         # @ references Python variable
```

**Gotcha — assignment:** to set values on a filtered slice, do it in **one** `.loc`, never chained:

```python
df.loc[df['dept'] == 'A', 'salary'] = 0      # correct
# df[df['dept']=='A']['salary'] = 0          # WRONG: SettingWithCopyWarning, may not write
```

The "chained indexing" rule: any time you see `df[...][...] = ...`, you're playing dice with whether the assignment lands on a copy or the original. Always collapse to one `.loc[mask, col]`.

---

## 3. Missing Data (high-frequency on the OA)

```python
df['salary'].isna()
df.dropna(subset=['salary'])          # drop rows where salary is null
df.dropna(how='all')                  # drop only fully-empty rows
df['salary'].fillna(0)
df['salary'].fillna(df['salary'].mean())
df['salary'].ffill()                  # forward fill (was fillna(method='ffill'))
df['salary'].bfill()                  # back fill
df['salary'].interpolate()            # linear interpolation between non-null values
```

### The Named OA Question — Fill NA with the Group Mean

```python
# Fill missing salary with the mean salary of that row's dept
df['salary'] = df['salary'].fillna(
    df.groupby('dept')['salary'].transform('mean')
)
```

`transform('mean')` returns a Series **aligned to the original rows** (same length), so it slots straight into `fillna`. This is *the* idiom — memorize it. Same shape works for `median`, `min`, `max`, or any aggregator. (Compare to the SQL version: a window function `AVG(salary) OVER (PARTITION BY dept)` — see [SQL Deep Dive](./01-sql-deep-dive.md).)

---

## 4. Transform Columns (the CASE WHEN of pandas)

```python
# Vectorized arithmetic
df['bonus'] = df['salary'] * 0.1

# Two-way condition  (SQL: CASE WHEN x THEN a ELSE b END)
df['band'] = np.where(df['salary'] >= 200, 'high', 'low')

# Multi-way condition  (SQL: CASE WHEN ... WHEN ... ELSE ... END)
df['band'] = np.select(
    [df['salary'] >= 200, df['salary'] >= 100],   # conditions, in order
    ['high', 'mid'],                               # results
    default='low'
)

# Map a Series through a dict (lookup table)
df['dept_full'] = df['dept'].map({'A': 'Alpha', 'B': 'Bravo'})

# Element-wise custom logic (slower — use only when not vectorizable)
df['flag'] = df['salary'].apply(lambda x: 'ok' if x and x > 100 else 'check')

# Add several columns at once
df = df.assign(
    tax   = lambda d: d['salary'] * 0.2,
    net   = lambda d: d['salary'] - d['salary'] * 0.2,
)

# Numeric helpers
df['salary'].clip(lower=0, upper=1000)      # clamp values to range
df['salary'].round(2)
df['salary'].abs()
```

**Reach for `np.where` / `np.select` before `apply`.** They're vectorized and far faster — usually 10–100× on a million-row DataFrame. `apply(axis=1)` is the canonical reason an OA solution times out.

---

## 5. String Operations (named OA question: "get the character values only")

The `.str` accessor vectorizes Python string ops over a column.

```python
s = pd.Series(['aQx 12', 'aub 6 5', 'NYC-200'])

s.str.lower()
s.str.strip()
s.str.replace(r'\d+', '', regex=True)        # remove digits
s.str.extract(r'([A-Za-z]+)')                # first letter-run -> DataFrame
s.str.extract(r'(\d+)').astype('float')      # first number-run, as numeric
s.str.findall(r'\d+')                        # ALL number-runs -> list per row
s.str.split(' ', expand=True)                # split into columns
s.str.contains('NYC')                        # boolean mask
s.str.startswith('NYC') / s.str.endswith('200')
s.str.len()
s.str.cat(sep=',')                           # join a column into one string
s.str.pad(width=5, side='left', fillchar='0')
s.str.zfill(5)                               # zero-pad to width 5
s.str.slice(0, 3)                            # substring [0:3)
s.str.get(0)                                 # first character of each value
```

```python
# "keep only the letters" / "keep only the digits" — classic OA prompts
df['letters'] = df['raw'].str.replace(r'[^A-Za-z]', '', regex=True)
df['digits']  = df['raw'].str.replace(r'\D', '', regex=True)

# Multi-group extract
df[['first', 'last']] = df['name'].str.extract(r'(\w+)\s+(\w+)')
```

**Gotchas:**
- `.str.extract` returns the *first* match by default and gives `NaN` (not error) when nothing matches — check for nulls after.
- `.str` on a non-string column errors. Cast first: `df['x'].astype(str).str.lower()`.
- Backslashes in regex strings — always use raw strings: `r'\d+'`, not `'\d+'`.

---

## 6. GroupBy & Aggregation (the workhorse)

```python
# Single aggregate
df.groupby('dept')['salary'].sum()
df.groupby('dept')['salary'].mean()
df.groupby('dept').size()              # rows per group (counts NaN)
df.groupby('dept')['salary'].count()   # non-null count per group

# Multiple aggregates, clean column names  (named agg — preferred)
out = df.groupby('dept').agg(
    total_salary = ('salary', 'sum'),
    avg_salary   = ('salary', 'mean'),
    headcount    = ('id', 'count'),
).reset_index()                        # <-- turn group keys back into columns

# Group by several keys
df.groupby(['dept', 'band'])['salary'].sum().reset_index()

# Custom aggregate
df.groupby('dept')['salary'].agg(lambda s: s.max() - s.min())

# Aggregate then filter (HAVING clause equivalent)
big_depts = df.groupby('dept').filter(lambda g: g['salary'].sum() > 500)
```

### `agg` vs `transform` — The Distinction That Trips People

| | returns | length | use for |
|---|---|---|---|
| `.agg('mean')` | one value per group | shrinks to #groups | summary tables |
| `.transform('mean')` | same value broadcast to every row | same as input | adding a group-stat column / fillna |

```python
# share of dept total, per row  (needs transform, not agg)
df['pct_of_dept'] = df['salary'] / df.groupby('dept')['salary'].transform('sum')
```

**Gotcha:** by default `groupby` **drops rows whose key is NaN** (`dropna=True`). If a question's counts look low, pass `dropna=False`.

---

## 7. Window Functions (you know these from SQL — here's the map)

| SQL | pandas |
|---|---|
| `ROW_NUMBER() OVER (PARTITION BY g ORDER BY x)` | `df.sort_values('x').groupby('g').cumcount() + 1` |
| `RANK() / DENSE_RANK()` | `df.groupby('g')['x'].rank(method='min' / 'dense')` |
| `SUM(x) OVER (PARTITION BY g)` | `df.groupby('g')['x'].transform('sum')` |
| running total | `df.groupby('g')['x'].cumsum()` |
| running max | `df.groupby('g')['x'].cummax()` |
| `LAG(x) / LEAD(x)` | `df.groupby('g')['x'].shift(1)` / `shift(-1)` |
| top-N per group | sort, then `.groupby('g').head(N)` |
| rolling 7-day sum | `df.set_index('date').groupby('g')['x'].rolling('7D').sum()` |

```python
# Top 3 earners per dept
(df.sort_values(['dept', 'salary'], ascending=[True, False])
   .groupby('dept')
   .head(3))

# Previous row's value per group (for diffs / "change since last")
df['prev_salary'] = df.groupby('dept')['salary'].shift(1)
df['delta'] = df['salary'] - df['prev_salary']

# Rank within group
df['rank_in_dept'] = df.groupby('dept')['salary'].rank(method='dense', ascending=False)
```

---

## 8. Sort & Rank

```python
df.sort_values('salary', ascending=False)
df.sort_values(['dept', 'salary'], ascending=[True, False])
df.sort_values('salary', na_position='last')         # NaN placement

df['salary'].rank(method='dense', ascending=False)   # methods: min, max, first, dense, average

df.nlargest(3, 'salary')      # faster than sort+head for top-N overall
df.nsmallest(3, 'salary')
df.nlargest(3, ['salary', 'id'])   # tiebreaker
```

---

## 9. Merge / Join (and the anti-join OA question)

```python
a.merge(b, on='id', how='inner')                 # default how='inner'
a.merge(b, on='id', how='left')
a.merge(b, left_on='aid', right_on='bid', how='outer')
a.merge(b, on=['id', 'date'], how='left')        # multi-key

# Stacking
pd.concat([a, b], axis=0, ignore_index=True)     # rows (union-all style)
pd.concat([a, b], axis=1)                         # columns (align on index!)
```

### Named OA Question — "Customers / Products Without a Sale" (anti-join)

```python
merged = customers.merge(sales, on='cust_id', how='left', indicator=True)
no_sales = merged[merged['_merge'] == 'left_only']     # in customers, not in sales
```

`indicator=True` adds a `_merge` column with `left_only` / `right_only` / `both` — the cleanest way to express SQL anti-joins and full-outer diffs.

### Gotcha — Row Explosion

A many-to-many merge multiplies rows silently. Guard it:

```python
a.merge(b, on='id', how='left', validate='one_to_many')   # raises if assumption breaks
# values: one_to_one, one_to_many, many_to_one, many_to_many
```

Also: merging on a key that contains duplicates or differing dtypes (`int` vs `str` `id`) is the **#1 source of wrong counts**. Check `a['id'].dtype == b['id'].dtype` first; cast if needed.

### `merge` vs `join` vs `concat`

| Operation | Use when |
|---|---|
| `a.merge(b, on='k')` | Joining on a column (most common; SQL-style) |
| `a.join(b)` | Joining on the **index** (faster but stricter) |
| `pd.concat([a, b])` | Stacking — same columns (axis=0) or same index (axis=1) |

---

## 10. Reshape: pivot / melt

```python
# long -> wide  (rows of region x month into a matrix)
wide = df.pivot_table(
    index='region', columns='month', values='sales',
    aggfunc='sum', fill_value=0
)

# wide -> long  (un-pivot month columns back into rows)
long = wide.reset_index().melt(
    id_vars='region', var_name='month', value_name='sales'
)

# crosstab — like pivot but for frequencies
pd.crosstab(df['region'], df['status'])
pd.crosstab(df['region'], df['status'], normalize='index')  # row-wise %
```

Use `pivot_table` (not `pivot`) when there can be duplicate index/column pairs — it aggregates; bare `pivot` raises on duplicates.

### `stack` / `unstack` — the index-aware versions

```python
df.set_index(['region', 'month']).unstack('month')   # move 'month' level to columns
df.stack()                                           # move column level into index
```

These are the answer when your data has a MultiIndex and you need to reshape *between* index and column levels.

---

## 11. Dates & Times

```python
df['ts'] = pd.to_datetime(df['ts'])          # parse strings -> datetime
df['ts'] = pd.to_datetime(df['ts'], errors='coerce')   # bad values -> NaT instead of error

df['ts'].dt.year                             # .month .day .hour .dayofweek .date
df['ts'].dt.strftime('%Y-%m')                # format to string
df['ts'].dt.normalize()                      # drop the time part
df['ts'].dt.to_period('M')                   # to month-period

df['gap_days'] = (df['end'] - df['start']).dt.days   # timedelta -> integer days

# Filter a date range
df[df['ts'].between('2024-01-01', '2024-03-31')]

# Per-month aggregation
df.groupby(df['ts'].dt.to_period('M'))['sales'].sum()

# Resample (time-series specific GROUP BY)
df.set_index('ts')['sales'].resample('D').sum()    # daily
df.set_index('ts')['sales'].resample('W').mean()   # weekly mean

# Date offsets
df['ts'] + pd.Timedelta(days=7)
df['ts'] + pd.DateOffset(months=1)           # calendar-aware (handles month-ends)

# Timezone awareness
df['ts'].dt.tz_localize('UTC')               # naive -> UTC-aware
df['ts'].dt.tz_convert('America/New_York')   # convert (must already be tz-aware)
```

**Gotcha:** parsing fails silently to `NaT` (datetime's NaN) when `errors='coerce'`. Add it and then check `.isna().sum()` to see how many rows failed parsing.

---

## 12. Reading Data (incl. the "nasty JSON" question)

```python
# CSV with real-world mess
df = pd.read_csv(
    'data.csv',
    sep=';',
    na_values=['', 'NA', 'null', '-'],   # treat these as NaN
    dtype={'zip': str},                  # stop pandas eating leading zeros
    parse_dates=['date'],
    thousands=',',
    encoding='utf-8',
    on_bad_lines='warn',                 # tolerate malformed rows
)

# Flat JSON
df = pd.read_json('data.json')

# NESTED JSON  (the named "import a nasty json with pandas" task)
records = [
    {'id': 1, 'name': 'A', 'orders': [{'sku': 'x', 'qty': 2}, {'sku': 'y', 'qty': 1}]},
    {'id': 2, 'name': 'B', 'orders': [{'sku': 'x', 'qty': 5}]},
]
flat = pd.json_normalize(
    records,
    record_path='orders',     # the list to explode into rows
    meta=['id', 'name'],      # parent fields to carry down
)
# -> columns: sku, qty, id, name

# Deeper nesting — record_path can be a list
pd.json_normalize(data, record_path=['users', 'orders'], meta=[['users', 'id']])

# Parquet (fast, columnar — what production DE actually uses)
df = pd.read_parquet('data.parquet')         # requires pyarrow or fastparquet
df.to_parquet('out.parquet', compression='snappy')

# Excel
df = pd.read_excel('data.xlsx', sheet_name='Sheet1')

# SQL
df = pd.read_sql('SELECT * FROM users WHERE active', conn)
```

`pd.json_normalize` is the answer whenever a JSON has nested objects/arrays. For a column that already holds list-of-dicts, use `df.explode('orders')` then `pd.json_normalize(df['orders'])`.

---

## 13. Dedup & Data Quality

```python
df.duplicated(subset=['id']).sum()                  # how many dup rows
df.drop_duplicates(subset=['id'], keep='last')      # keep first/last/False(drop all)

# Standardize before deduping (whitespace/case cause "fake" uniques)
df['email'] = df['email'].str.strip().str.lower()

# Range / validity checks
bad = df[(df['age'] < 0) | (df['age'] > 120)]

# Column hygiene
df.rename(columns={'old': 'new'})
df.columns = df.columns.str.strip().str.lower().str.replace(' ', '_')   # clean headers
df.astype({'id': 'int64', 'price': 'float64'})

# Drop columns / rows
df.drop(columns=['tmp', 'debug'])
df.drop(index=[0, 1])
```

---

## 14. SQL → Pandas Cheat Sheet (your fastest bridge)

| SQL | pandas |
|---|---|
| `SELECT a, b FROM t` | `df[['a', 'b']]` |
| `WHERE a > 5 AND b = 'x'` | `df[(df.a > 5) & (df.b == 'x')]` |
| `WHERE a IN (1,2)` | `df[df.a.isin([1, 2])]` |
| `WHERE a IS NULL` | `df[df.a.isna()]` |
| `WHERE a BETWEEN 1 AND 10` | `df[df.a.between(1, 10)]` |
| `WHERE a LIKE 'foo%'` | `df[df.a.str.startswith('foo')]` or `df[df.a.str.contains('^foo')]` |
| `ORDER BY a DESC` | `df.sort_values('a', ascending=False)` |
| `LIMIT 5` | `df.head(5)` |
| `DISTINCT` | `df.drop_duplicates()` |
| `COUNT(*) GROUP BY g` | `df.groupby('g').size()` |
| `SUM(x) GROUP BY g` | `df.groupby('g')['x'].sum()` |
| `HAVING sum(x) > 10` | `df.groupby('g').filter(lambda g: g.x.sum() > 10)` |
| `JOIN ... ON id` | `a.merge(b, on='id')` |
| `LEFT JOIN` | `a.merge(b, on='id', how='left')` |
| anti-join (`NOT IN`) | `merge(how='left', indicator=True)` → `_merge == 'left_only'` |
| `UNION ALL` | `pd.concat([a, b])` |
| `UNION` (dedupe) | `pd.concat([a, b]).drop_duplicates()` |
| `CASE WHEN` | `np.select(...)` or `np.where(...)` |
| `COALESCE(a, b)` | `a.fillna(b)` or `a.combine_first(b)` |
| `NULLIF(a, b)` | `a.where(a != b)` |
| window `SUM() OVER(PARTITION BY g)` | `df.groupby('g')['x'].transform('sum')` |
| `ROW_NUMBER()` | `df.groupby('g').cumcount() + 1` |
| `LAG/LEAD` | `df.groupby('g')['x'].shift(1) / shift(-1)` |
| `PIVOT` | `df.pivot_table(...)` |

When an OA question feels like SQL, write the SQL in your head first, then translate row by row through this table.

---

## 15. The McKinsey API-Pagination Pattern (memorize the skeleton)

A recurring McKinsey HackerRank question: hit a paginated REST endpoint, read `total_pages`, loop, aggregate. The shape is always the same:

```python
import requests

def fetch_all(base_url, **params):
    page = 1
    rows = []
    while True:
        resp = requests.get(base_url, params={**params, 'page': page}).json()
        rows.extend(resp['data'])               # accumulate this page's records
        if page >= resp['total_pages']:         # stop condition from the payload
            break
        page += 1
    return rows

# Then it's normal dict/pandas work:
records = fetch_all('https://jsonmock.hackerrank.com/api/...')
df = pd.DataFrame(records)
answer = df[df['country'] == 'UK']['name'].tolist()
```

Read `total_pages` from page 1, loop until you've consumed it, then aggregate. Many variants just want a filtered/summed field out of the combined pages.

### Defensive variant for production-flavored questions

```python
def fetch_all_safe(base_url, max_pages=1000, timeout=10):
    rows, page = [], 1
    while page <= max_pages:
        r = requests.get(base_url, params={'page': page}, timeout=timeout)
        r.raise_for_status()
        body = r.json()
        rows.extend(body.get('data', []))
        if page >= body.get('total_pages', page):
            break
        page += 1
    return rows
```

Adds: timeout, error raising, defensive `get` with defaults, max-page guard. Useful for OAs that grade error handling.

---

## 16. OA Gotchas Checklist (read the night before)

1. **`== np.nan` is always False.** Use `.isna()` / `.notna()`.
2. **Chained assignment doesn't write reliably.** Always `df.loc[mask, 'col'] = ...`.
3. **A column with any NaN becomes `float`.** An "id" can print as `1.0`; cast back with `.astype('Int64')` (nullable int) or fill first.
4. **`groupby` silently drops NaN keys.** Pass `dropna=False` if counts look short.
5. **`mean()/sum()` skip NaN; `count()` excludes NaN; `size()` includes it.** Pick deliberately.
6. **Merges explode on duplicate/many-to-many keys.** Use `validate=`; check key dtypes match.
7. **After `groupby().agg(...)` the keys are in the index** — `.reset_index()` before returning/printing, or the expected output won't match.
8. **`apply(axis=1)` is slow** — a vectorized version usually exists; loops are why solutions TLE.
9. **`sort_values` returns a copy** (no `inplace`); reassign it.
10. **HackerRank checks exact output** — column order, names, index, dtypes, and rounding all matter. Match the sample output precisely (`round()`, `reset_index(drop=True)`).
11. **`pivot` raises on duplicate index/col pairs; `pivot_table` aggregates.** Default to `pivot_table`.
12. **String columns with hidden whitespace/case** break joins, dedup, and `==`. Normalize with `.str.strip().str.lower()` first.
13. **`.str.extract()` returns NaN for non-matches** — not an error. Check for nulls after.
14. **Object-dtype columns use ~50 bytes/value.** A million-row object column is ~50MB. Cast to `category` or proper dtype to shrink 10–50×.
15. **`pd.to_datetime` without `errors='coerce'`** throws on bad input. Always coerce on OA-grade input.

---

## 17. Practice Tasks (mirror the OA — try before checking)

1. Fill missing `salary` with the **median** salary of each `dept`. *(transform + fillna)*
2. From `raw` column `['aQx 12','aub 6 5']`, output a clean DataFrame with one column of letters and one of the **first** number. *(str.replace, str.extract)*
3. Given `customers` and `orders`, list customers with **zero** orders. *(left merge + indicator)*
4. For each `dept`, return the **top 2** earners. *(sort + groupby.head)*
5. Add a `pct_of_dept` column = each salary as a share of its dept total. *(transform('sum'))*
6. Flatten a nested JSON of `users -> orders -> items` into one row per item. *(json_normalize record_path/meta)*
7. From a paginated API, return all university names in a given country. *(fetch_all + filter)*
8. Pivot `(region, month, sales)` into a region×month matrix with 0 for missing. *(pivot_table fill_value=0)*

If you can do all 8 cold in ~60 minutes total, you're ready for the pandas half.

---

## 18. Pandas vs PySpark — When the OA Switches Engines

A growing minority of OAs (Databricks, big-data shops) phrase questions in PySpark. The concepts are identical; the API differs. The big mental shifts:

- **Spark is lazy.** Nothing runs until an *action* (`.show()`, `.collect()`, `.count()`, `.write()`). You're building a query plan.
- **No row order without `orderBy`.** Spark is distributed; row order isn't preserved.
- **No index.** Spark DataFrames have no concept of an index. Group keys come back as plain columns.
- **No `.loc` / `.iloc`.** Everything is `select()` + `filter()`.
- **`withColumn` replaces assignment.** Spark DataFrames are immutable — every "edit" returns a new DataFrame.
- **`apply(axis=1)` becomes UDFs.** Python UDFs are slow (Python-JVM serialization); prefer built-in `F.*` functions or **Pandas UDFs** (which batch via Arrow).

```python
from pyspark.sql import functions as F, Window
```

### Side-by-side translation

| Task | pandas | PySpark |
|---|---|---|
| Select cols | `df[['a','b']]` | `df.select('a', 'b')` |
| Filter | `df[df.a > 5]` | `df.filter(F.col('a') > 5)` |
| New column | `df['c'] = df.a + df.b` | `df.withColumn('c', F.col('a') + F.col('b'))` |
| Rename | `df.rename(columns={'old':'new'})` | `df.withColumnRenamed('old','new')` |
| CASE WHEN | `np.select(...)` | `F.when(cond, v1).when(...).otherwise(v2)` |
| Fill NA | `df.fillna(0)` | `df.fillna(0)` / `df.na.fill(0)` |
| Drop NA | `df.dropna(subset=['a'])` | `df.dropna(subset=['a'])` |
| Group + agg | `df.groupby('g')['x'].sum()` | `df.groupby('g').agg(F.sum('x'))` |
| Named agg | `df.groupby('g').agg(total=('x','sum'))` | `df.groupby('g').agg(F.sum('x').alias('total'))` |
| Join | `a.merge(b, on='id')` | `a.join(b, on='id', how='inner')` |
| Anti-join | `merge(..., indicator=True)` filter `left_only` | `a.join(b, on='id', how='left_anti')` |
| Union all | `pd.concat([a, b])` | `a.unionByName(b)` |
| Distinct | `df.drop_duplicates()` | `df.dropDuplicates()` |
| Sort | `df.sort_values('x')` | `df.orderBy('x')` (or `.sort('x')`) |
| Top N | `df.nlargest(5, 'x')` | `df.orderBy(F.desc('x')).limit(5)` |
| `LAG` | `df.groupby('g')['x'].shift(1)` | `F.lag('x').over(Window.partitionBy('g').orderBy('t'))` |
| `ROW_NUMBER` | `df.groupby('g').cumcount() + 1` | `F.row_number().over(Window.partitionBy('g').orderBy('x'))` |
| `transform('sum')` | `df.groupby('g')['x'].transform('sum')` | `F.sum('x').over(Window.partitionBy('g'))` |
| Explode list col | `df.explode('items')` | `df.withColumn('item', F.explode('items'))` |
| Parse JSON col | `df['j'].apply(json.loads)` | `F.from_json('j', schema)` |
| To action | `df.head()` | `df.show()` or `df.limit(n).toPandas()` |

### When the prompt is Spark

If you see `SparkSession`, `F.col`, `.show()`, or "this dataset doesn't fit in memory" — it's a Spark question. Translate your pandas logic through the table above. Otherwise default to pandas.

### Pandas inside Spark — the bridge

You can convert between the two:

```python
spark_df = spark.createDataFrame(pandas_df)          # pandas -> Spark
pandas_df = spark_df.toPandas()                       # Spark -> pandas (CAREFUL: pulls all rows to driver)
```

For OAs, if you're comfortable in pandas and the dataset is small, `toPandas()` on a filtered Spark DataFrame is a legitimate shortcut. In production, `toPandas()` on a 1B-row DataFrame OOMs the driver — bad pattern. See [Spark Fundamentals](./07-spark-fundamentals.md) for the production discussion.

### Pandas UDFs — the perf escape hatch

When you must run Python logic inside Spark, **never use a regular UDF**. Use a **Pandas UDF** (Arrow-batched):

```python
from pyspark.sql.functions import pandas_udf

@pandas_udf('double')
def normalize(s: pd.Series) -> pd.Series:
    return (s - s.mean()) / s.std()

df.withColumn('z', normalize('x'))
```

10–100× faster than `udf(...)` because it batches whole columns into pandas Series via Arrow rather than row-at-a-time Python-JVM serialization. The bridge between these two worlds is [Apache Arrow](../python/03-arrays-and-buffers.md).

---

## 19. Performance Notes (when the OA times out)

If a solution works but times out:

1. **Replace `apply(axis=1)` with vectorized ops.** Often 50–100× faster.
2. **Use `np.where` / `np.select` instead of `apply` with conditionals.**
3. **Avoid loops over rows.** `for i, row in df.iterrows():` is the slowest Python pattern in pandas.
4. **Cast object columns to category** for repeated values (e.g. country codes). 10× memory shrink, faster `groupby`.
5. **Pre-sort once** before multiple `groupby.head()` operations.
6. **Use `merge` once with multi-key** instead of two sequential merges.
7. **Replace `.str.contains(...)` chains with a single regex.**
8. **`pd.eval()` / `df.eval()`** can speed up complex arithmetic on large DataFrames by using NumExpr.
9. **Read only the columns you need:** `pd.read_csv(..., usecols=[...])`, `pd.read_parquet(..., columns=[...])`.
10. **For really big work, consider [Polars](https://pola.rs/) or [DuckDB](https://duckdb.org/)** — same API shape, often 5–30× faster than pandas for OA-sized data. (Most OA graders only ship pandas — know about them, but write pandas for the test.)

---

## 20. Interview-Ready Cheat Sheet

### Top 10 idioms to drill until automatic

1. `df.groupby('g')['x'].transform('mean')` — broadcast group stat back to rows.
2. `df.fillna(df.groupby('g')['x'].transform('mean'))` — fill NA with group mean.
3. `df.sort_values(...).groupby('g').head(N)` — top-N per group.
4. `a.merge(b, on='id', how='left', indicator=True)` then filter `_merge == 'left_only'` — anti-join.
5. `pd.json_normalize(records, record_path='items', meta=[...])` — flatten nested JSON.
6. `np.select([cond1, cond2], [val1, val2], default=...)` — multi-way CASE.
7. `df['raw'].str.extract(r'(\d+)').astype(float)` — pull first number from a string column.
8. `df.pivot_table(index=..., columns=..., values=..., aggfunc=..., fill_value=0)` — long→wide.
9. `df.groupby(df['ts'].dt.to_period('M'))['x'].sum()` — monthly aggregation.
10. `while page <= total_pages: rows.extend(resp['data']); page += 1` — API pagination.

### Quick trade-off pairs

- **`apply` vs vectorized:** flexibility vs 10-100× speed.
- **`agg` vs `transform`:** shrink to one-per-group vs broadcast to original shape.
- **`merge` vs `join`:** column-based vs index-based.
- **`pivot` vs `pivot_table`:** raises on dupes vs aggregates.
- **`groupby.head(N)` vs window-rank-then-filter:** simpler vs needed when you require stable tie-breaks.
- **pandas vs Spark:** in-memory single-machine vs distributed.

### Top Q&As

**"How do I fill missing values with the mean of a group?"** → `transform('mean')` + `fillna`.

**"How do I find rows in A with no match in B?"** → left merge + `indicator=True`, filter `_merge == 'left_only'`. Or `~a['k'].isin(b['k'])`.

**"How do I get the top N per group?"** → sort then `groupby('g').head(N)`. For ties, use `rank(method='dense')` + filter.

**"Why is my pandas script slow?"** → `apply(axis=1)` or a Python `for` loop. Vectorize, or move to Polars/DuckDB.

**"What's the difference between `agg` and `transform`?"** → shape: agg returns one row per group; transform returns one row per *input* row.

**"How do I handle a nested JSON?"** → `pd.json_normalize` with `record_path` and `meta`.

**"How do I implement SQL's `COALESCE(a, b)`?"** → `a.fillna(b)` or `a.combine_first(b)`.

---

## Resources & Links

- [Pandas official docs](https://pandas.pydata.org/docs/) — surprisingly readable
- [Pandas Cookbook](https://pandas.pydata.org/docs/user_guide/cookbook.html) — recipe-style examples
- [Modern Pandas (Tom Augspurger)](https://tomaugspurger.net/posts/modern-1-intro/) — best mental-model series
- [Effective Pandas (Matt Harrison)](https://store.metasnake.com/effective-pandas-book) — opinionated style guide
- [PySpark official docs](https://spark.apache.org/docs/latest/api/python/index.html)
- [Polars User Guide](https://pola.rs/) — the modern alternative
- [DuckDB Python API](https://duckdb.org/docs/api/python/overview) — SQL on pandas DataFrames

### Companion files in this repo

- [SQL Deep Dive](./01-sql-deep-dive.md) — the SQL side; many idioms map 1:1.
- [Spark Fundamentals](./07-spark-fundamentals.md) — when pandas runs out of memory.
- [Python Internals](../python/02-python-internals.md) — why `apply` is slow.
- [Arrays & Buffers](../python/03-arrays-and-buffers.md) — NumPy/Arrow under pandas.
- [Algorithm Design](../python/09-algorithm-design.md) — DP/greedy in Python.

---

*Next: drill the 8 practice tasks above. If you can do all 8 cold in 60 min, the pandas half of any OA is yours.*
