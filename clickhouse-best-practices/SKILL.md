---
name: clickhouse-best-practices
description: ClickHouse development best practices for schema design, query optimization, insert strategy, and cluster management
---

# ClickHouse Best Practices

## Core Principles

- Plan ORDER BY (primary key) before table creation — it is immutable
- Use native data types and minimize storage footprint
- Batch inserts to avoid "too many parts" errors
- Avoid mutations (ALTER UPDATE/DELETE); use engine-level patterns instead
- Use partitioning for data lifecycle, not as a primary query optimization
- Prefer materialized views and projections for recurring aggregations

## Schema Design — Primary Key (CRITICAL)

### schema-pk-plan-before-creation

**Plan ORDER BY before table creation — it is immutable.**

`ORDER BY` defines the primary key and physical sort order of data on disk. Changing it after the fact requires a full data migration (create new table, reindex, swap alias). Wrong choices are expensive to fix.

Pre-creation checklist:
- List the top 5–10 query patterns
- Identify `WHERE` clause columns and their query frequency
- Order columns by cardinality (low to high)
- Limit to 4–5 key columns

```sql
-- Bad: ad-hoc, high-cardinality leading column
CREATE TABLE events (
    event_id UUID,
    tenant_id UInt32,
    event_type LowCardinality(String),
    timestamp DateTime
) ENGINE = MergeTree()
ORDER BY event_id;

-- Good: low-cardinality first, time last
CREATE TABLE events (
    event_id UUID,
    tenant_id UInt32,
    event_type LowCardinality(String),
    timestamp DateTime
) ENGINE = MergeTree()
ORDER BY (tenant_id, event_type, timestamp);
```

### schema-pk-cardinality-order

**Order primary key columns from low to high cardinality.**

Low-cardinality leading columns create more useful index entries that can skip entire granules. High-cardinality leading columns produce index entries that rarely skip anything.

Ordering strategy:
1. Lowest cardinality (e.g., `status`, `event_type`, `region`)
2. Date or time bucket (e.g., `toDate(timestamp)`)
3. Medium cardinality (e.g., `tenant_id`, `user_id`)
4. High cardinality (e.g., `session_id`, `uuid`) — only if needed for deduplication

```sql
-- Good ordering for a multi-tenant event log
ORDER BY (tenant_id, event_type, toDate(timestamp), event_id)
--        ^low         ^low        ^medium            ^high (dedup only)
```

### schema-pk-prioritize-filters

**Include frequently filtered columns in ORDER BY.**

Columns not in `ORDER BY` force full table scans when used in `WHERE`. Validate index usage with `EXPLAIN`:

```sql
EXPLAIN indexes = 1
SELECT * FROM events WHERE tenant_id = 123 AND event_type = 'click';
-- Look for "PrimaryKey" with a "Key Condition" entry in the output
```

If a column is frequently filtered but not suitable as a leading key (high cardinality), consider a data skipping index or projection instead.

### schema-pk-filter-on-orderby

**Query `WHERE` filters must use the ORDER BY prefix to benefit from the sparse index.**

Index usage matrix for `ORDER BY (tenant_id, event_type, timestamp)`:

| Filter | Index Usage |
|--------|-------------|
| `WHERE tenant_id = 123` | Full index — skips granules |
| `WHERE tenant_id = 123 AND event_type = 'click'` | Full index |
| `WHERE event_type = 'click'` | None — prefix skipped |
| `WHERE timestamp > '2024-01-01'` | None — both prefix columns skipped |

Filters that skip ORDER BY prefix columns cause full table scans. Reorder the primary key or use a skipping index for such patterns.

---

## Schema Design — Data Types (CRITICAL)

### schema-types-native-types

**Use native types — not String for everything.**

Using `String` for all fields wastes storage, prevents compression optimization, and makes comparisons slower. Choose the appropriate native type:

| Value | Wrong | Correct |
|-------|-------|---------|
| Sequential IDs | `String` | `UInt32` / `UInt64` |
| UUIDs | `String` (36 chars) | `UUID` (16 bytes) |
| Timestamps | `String` | `DateTime` / `DateTime64` |
| Booleans | `String` / `UInt8` | `Bool` |
| Money / decimals | `Float64` | `Decimal(P, S)` or `Int64` (cents) |
| Status codes | `String` | `Enum8` / `LowCardinality(String)` |

### schema-types-minimize-bitwidth

**Use the smallest numeric type that fits the data range.**

Prefer unsigned types when negative values are impossible. Smaller types compress better and process faster.

Quick reference:

| Type | Range | Bytes | Example Use |
|------|-------|-------|-------------|
| `UInt8` | 0–255 | 1 | HTTP status codes, boolean flags |
| `UInt16` | 0–65,535 | 2 | Port numbers, age |
| `UInt32` | 0–4.3B | 4 | User IDs, item counts |
| `UInt64` | 0–18Q | 8 | Large counters, global IDs |
| `Int32` | ±2.1B | 4 | Signed counters |
| `Int64` | ±9Q | 8 | Financial values in cents |

```sql
-- Bad
CREATE TABLE metrics (
    status_code Int64,
    user_age Int64,
    response_time_ms Int64
) ENGINE = MergeTree() ORDER BY tuple();

-- Good
CREATE TABLE metrics (
    status_code UInt16,
    user_age UInt8,
    response_time_ms UInt32
) ENGINE = MergeTree() ORDER BY tuple();
```

### schema-types-lowcardinality

**Use `LowCardinality` for string columns with fewer than ~10,000 unique values.**

`LowCardinality` applies dictionary encoding, dramatically reducing storage and improving filter/group-by performance. It is the single most impactful schema optimization for repeated string columns.

Decision rule:
- `< ~10,000` unique values → `LowCardinality(String)`
- `>= ~10,000` unique values → `String`
- Truly fixed-length values → `FixedString(N)`

```sql
CREATE TABLE events (
    event_type   LowCardinality(String),  -- dozens of types
    country_code LowCardinality(String),  -- ~250 countries
    status       LowCardinality(String),  -- handful of statuses
    user_agent   String                   -- millions of unique values
) ENGINE = MergeTree() ORDER BY (event_type, country_code);
```

### schema-types-enum

**Use `Enum` for finite value sets to get validation and compact storage.**

`Enum` provides insert-time validation and stores values in 1–2 bytes. Invalid values are rejected at insert rather than silently stored.

- `Enum8`: up to 256 values (1 byte)
- `Enum16`: up to 65,536 values (2 bytes)

```sql
CREATE TABLE orders (
    order_id UInt64,
    status   Enum8('pending'=1, 'processing'=2, 'shipped'=3, 'delivered'=4, 'cancelled'=5),
    priority Enum8('low'=1, 'normal'=2, 'high'=3, 'urgent'=4)
) ENGINE = MergeTree() ORDER BY (status, order_id);

-- Invalid insert is rejected at insert time:
-- INSERT INTO orders VALUES (1, 'unknown', 'normal')  -- Error
```

### schema-types-avoid-nullable

**Avoid `Nullable` — use `DEFAULT` values instead.**

`Nullable` columns maintain a separate `UInt8` bitmap column to track null values, increasing storage and degrading performance. Only use `Nullable` when `NULL` is semantically distinct from a default value.

| Use Case | Approach | Example |
|----------|----------|---------|
| Required IDs, counters | `DEFAULT 0` | `retry_count UInt8 DEFAULT 0` |
| Unknown text | `DEFAULT ''` | `description String DEFAULT ''` |
| Unknown number | `DEFAULT 0` | `score Float32 DEFAULT 0` |
| Semantic "not deleted" | `Nullable` | `deleted_at Nullable(DateTime)` |
| Semantic "no parent" | `Nullable` | `parent_id Nullable(UInt64)` |
| Semantic "discount not set" | `Nullable` | `discount_pct Nullable(Float32)` |

```sql
-- Bad
CREATE TABLE events (
    duration_ms Nullable(UInt32),
    referrer    Nullable(String)
) ENGINE = MergeTree() ORDER BY tuple();

-- Good
CREATE TABLE events (
    duration_ms UInt32 DEFAULT 0,
    referrer    String DEFAULT ''
) ENGINE = MergeTree() ORDER BY tuple();
```

---

## Schema Design — Partitioning (HIGH)

### schema-partition-low-cardinality

**Keep total partition count between 100 and 1,000.**

Too many distinct partition values create excessive data parts, eventually triggering "too many parts" errors and cluster instability.

```sql
-- Bad: daily partitions over years → 3,650+ partitions
PARTITION BY toDate(timestamp)

-- Bad: by user_id → millions of partitions
PARTITION BY user_id

-- Good: monthly partitions → ~12/year
PARTITION BY toYYYYMM(timestamp)

-- Good: modulo bucketing for non-time dimensions
PARTITION BY user_id % 100
```

Monitor partition health:
```sql
SELECT partition, count() AS parts, sum(rows) AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE table = 'events' AND active
GROUP BY partition
ORDER BY partition;
```

### schema-partition-lifecycle

**Use partitioning for data lifecycle management, not as a primary query optimization.**

Partitioning is primarily a data management technique. Its main benefits are:
- Instant `DROP PARTITION` instead of expensive row-level deletion
- TTL policies that drop entire parts as metadata operations
- Tiered storage movement to cold storage
- Archiving and `FREEZE PARTITION` for backups

```sql
-- Efficient partition-based retention (instant, no rewrite)
ALTER TABLE events DROP PARTITION '202401';

-- TTL with partition drops (set ttl_only_drop_parts = 1 for efficiency)
ALTER TABLE events MODIFY TTL timestamp + INTERVAL 90 DAY;
ALTER TABLE events MODIFY SETTING ttl_only_drop_parts = 1;
```

### schema-partition-query-tradeoffs

**Understand partition pruning trade-offs before relying on partitioning for query speed.**

Partitioning can help or hurt query performance:
- **Benefit**: Single-partition queries skip all other partitions (MinMax index applied automatically on partition key)
- **Harm**: Queries spanning many partitions scan more independently-managed merge groups, degrading performance
- **Harm**: Inserts batching across multiple partitions are slower
- **Important**: Data merges only occur within partitions, not across them

Test actual query performance before relying on partitioning as a query optimization. The primary key (ORDER BY) is almost always a more effective query optimization tool.

### schema-partition-start-without

**Consider starting without partitioning, then adding it when a clear need arises.**

Partitioning justification checklist:
- [ ] Time-based data retention or archiving requirement
- [ ] Need to move old data to cold/tiered storage
- [ ] Clear, tested query performance benefit on real data
- [ ] Table exceeds ~10 GB

If none of these apply, a well-designed ORDER BY with no partitioning will typically outperform a partitioned table with a weak primary key.

---

## Schema Design — JSON (MEDIUM)

### schema-json-when-to-use

**Use the `JSON` type for truly dynamic schemas; use typed columns for known fields.**

The `JSON` type splits JSON objects into sub-columns, enabling field-level compression and querying. Use it only when the schema is genuinely unpredictable.

Use `JSON` when:
- Field structure varies unpredictably between rows
- Field types may change over time
- You need field-level querying on dynamic keys

Use typed columns when:
- Fields are known at design time — always prefer explicit columns
- High-volume filtering or aggregation on specific fields (typed columns are faster)

Avoid storing JSON as `String` — this prevents all field-level optimization and requires expensive parsing at query time.

```sql
-- For known fields: typed columns
CREATE TABLE events (
    event_id   UUID,
    event_type LowCardinality(String),
    user_id    UInt64,
    timestamp  DateTime
) ENGINE = MergeTree() ORDER BY (event_type, timestamp);

-- For dynamic metadata alongside known fields: JSON type
CREATE TABLE events (
    event_id   UUID,
    event_type LowCardinality(String),
    timestamp  DateTime,
    properties JSON  -- dynamic per-event attributes
) ENGINE = MergeTree() ORDER BY (event_type, timestamp);
```

---

## Query Optimization — JOINs (CRITICAL)

### query-join-choose-algorithm

**Select the JOIN algorithm based on table sizes and memory constraints.**

ClickHouse defaults to `parallel_hash` (since 24.11), which loads the right-side table into memory. For large right-side tables, this can cause memory pressure or query failure.

| Algorithm | Best For | Notes |
|-----------|----------|-------|
| `parallel_hash` | Small-to-medium right table | Default since 24.11 |
| `hash` | General purpose | Single-threaded build |
| `direct` | Dictionary-style lookups | Fastest for INNER/LEFT; requires Dict/Join engine |
| `full_sorting_merge` | Pre-sorted large tables | Skips sort if already ordered |
| `partial_merge` | Large tables, memory-constrained | Slower but low memory |
| `grace_hash` | Memory-constrained, disk spill OK | Spills to disk |
| `auto` | Unknown sizes | Tries hash, falls back on pressure |

```sql
-- Force a specific algorithm
SELECT *
FROM large_table AS l
INNER JOIN small_lookup AS r ON l.id = r.id
SETTINGS join_algorithm = 'parallel_hash';

-- For memory-constrained environments
SELECT *
FROM fact_table AS f
LEFT JOIN big_dimension AS d ON f.dim_id = d.id
SETTINGS join_algorithm = 'grace_hash';
```

ClickHouse 24.12+ automatically positions the smaller table on the right side when possible.

### query-join-use-any

**Use `ANY JOIN` when only one match per left-side row is needed.**

`ANY JOIN` returns the first match only, consuming less memory and executing faster than standard JOIN which may return multiple rows per left-side row.

```sql
-- Standard JOIN: may return multiple rows per user
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- ANY JOIN: at most one order per user (first match)
SELECT u.name, o.total
FROM users u
LEFT ANY JOIN orders o ON u.id = o.user_id;
```

Variants:
- `LEFT ANY JOIN` — at most one match from right; non-matching left rows kept
- `INNER ANY JOIN` — at most one match; non-matching rows dropped
- `RIGHT ANY JOIN` — at most one match from left

### query-join-filter-before

**Filter tables before joining to minimize the join size.**

Joining full tables then applying a `WHERE` clause wastes memory building large hash tables. Push filters into subqueries or use `ON` clause conditions to reduce the data before the join.

```sql
-- Bad: joins full tables, then filters
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at >= '2024-01-01'
  AND u.country = 'US';

-- Good: filter before joining
SELECT u.name, o.total
FROM (SELECT id, name FROM users WHERE country = 'US') u
JOIN (SELECT user_id, total FROM orders WHERE created_at >= '2024-01-01') o
  ON u.id = o.user_id;
```

### query-join-consider-alternatives

**Consider dictionaries or denormalization instead of repeated JOINs.**

JOINs shift computation to query time. Alternatives shift it to insert time, making reads much faster:

1. **Dictionary lookups** (fastest) — In-memory key-value store, sub-millisecond lookups
2. **Denormalization** — Pre-join at insert time, no join at query time
3. **`IN` subqueries** — Often faster than JOIN for semi-join patterns
4. **JOIN** — Acceptable for infrequent or ad-hoc queries

**Critical caveat:** Dictionaries silently deduplicate — only the last value for a key is kept. Use only with unique keys.

```sql
-- Dictionary lookup (no JOIN required)
SELECT
    event_id,
    dictGet('user_dict', 'country', user_id) AS country
FROM events;

-- vs. equivalent JOIN
SELECT e.event_id, u.country
FROM events e
JOIN users u ON e.user_id = u.id;
```

### query-join-null-handling

**Set `join_use_nulls = 0` to use default values instead of NULL for outer JOIN non-matches.**

By default (`join_use_nulls = 1`), unmatched rows in outer JOINs produce NULL values, requiring extra NULL checks and increasing memory. Setting to `0` uses type-default values (empty string, 0) instead.

```sql
-- Use default values for non-matches (less memory, no NULL checks needed)
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
SETTINGS join_use_nulls = 0;
-- Non-matching o.total = 0 (not NULL)
```

---

## Query Optimization — Indices (HIGH)

### query-index-skipping-indices

**Use data skipping indices to accelerate filters on non-ORDER BY columns.**

Skipping indices store per-granule metadata (min/max, bloom filters, value sets) allowing the engine to skip granules that cannot match the query. They are secondary to the primary key — optimize primary key and data types first.

When to use:
- Column has high overall cardinality but low cardinality within individual granules
- Rare values that are frequently searched (e.g., specific error codes)
- Column is correlated with the primary key

When NOT to use:
- As a first optimization (always optimize primary key first)
- Values are scattered uniformly across all granules (index won't help)
- Without testing on real data

Index types:

| Type | Best For |
|------|----------|
| `bloom_filter` | High-cardinality equality checks |
| `set(N)` | Low-cardinality equality checks |
| `minmax` | Range queries |
| `ngrambf_v1` | Substring/text search |
| `tokenbf_v1` | Token-based text search |

```sql
-- Add a bloom filter index on a non-ORDER BY column
ALTER TABLE events
    ADD INDEX idx_session_id session_id TYPE bloom_filter(0.01) GRANULARITY 1;

-- Materialize for existing data
ALTER TABLE events MATERIALIZE INDEX idx_session_id;

-- Verify the index is being used
EXPLAIN indexes = 1
SELECT * FROM events WHERE session_id = 'abc123';
```

**Projections** are an alternative to skipping indices when you need a different sort order:

```sql
-- Add a projection for queries filtered by user_id (not in primary key)
ALTER TABLE events
    ADD PROJECTION proj_by_user (
        SELECT * ORDER BY user_id, timestamp
    );

ALTER TABLE events MATERIALIZE PROJECTION proj_by_user;
```

Projections duplicate storage but enable full primary-key performance for alternate access patterns. They do not work with `FINAL` queries.

---

## Query Optimization — Materialized Views (HIGH)

### query-mv-incremental

**Use incremental materialized views for real-time aggregations.**

Incremental MVs act as AFTER INSERT triggers: they process each inserted block and write aggregated results to a target table. This lets queries read thousands of pre-aggregated rows instead of scanning billions of raw rows.

**Always use explicit `TO <target_table>` syntax** to avoid hidden `.inner.<uuid>` tables that are destroyed when the view is dropped.

**Always use `AggregatingMergeTree` with State/Merge function pairs** for non-additive aggregates (`uniq`, `avg`, `quantile`). Storing raw integers for these produces silently wrong results.

```sql
-- 1. Create the target table
CREATE TABLE events_daily_agg (
    event_date  Date,
    event_type  LowCardinality(String),
    total_count AggregateFunction(count),
    unique_users AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree()
ORDER BY (event_date, event_type);

-- 2. Create the materialized view
CREATE MATERIALIZED VIEW events_daily_mv
TO events_daily_agg
AS SELECT
    toDate(timestamp)    AS event_date,
    event_type,
    countState()         AS total_count,
    uniqState(user_id)   AS unique_users
FROM events
GROUP BY event_date, event_type;

-- 3. Query using Merge functions
SELECT
    event_date,
    event_type,
    countMerge(total_count)      AS total,
    uniqMerge(unique_users)      AS unique_users
FROM events_daily_agg
GROUP BY event_date, event_type;
```

**Important:** Incremental MVs only process new inserts — existing data is not automatically included. Backfill separately using `INSERT INTO ... SELECT` before creating the view (to avoid a gap window where inserts are missed).

**Avoid `POPULATE`** — it creates a window where concurrent inserts are invisible to the backfill operation.

**Write amplification:** Each MV adds one extra write per INSERT. Five MVs on a table can reduce insert throughput by ~80%. Monitor part counts via `system.parts`.

### query-mv-refreshable

**Use refreshable materialized views for complex joins and batch workflows.**

Refreshable MVs execute a full SELECT periodically and overwrite (or append to) the target table. Unlike incremental MVs, they support complex multi-table joins, correlated subqueries, and arbitrary query patterns.

Best for:
- Queries where minor staleness is acceptable (minutes to hours)
- Complex multi-table joins that cannot be expressed as incremental transforms
- Caching "top N" results for dashboard queries
- Batch workflows with DAG-style dependencies

```sql
-- Create a refreshable MV that runs every hour
CREATE MATERIALIZED VIEW product_stats
REFRESH EVERY 1 HOUR
ENGINE = MergeTree()
ORDER BY (category, product_id)
AS SELECT
    p.category,
    p.product_id,
    p.name,
    count(o.order_id)   AS order_count,
    sum(o.total)        AS revenue
FROM products p
LEFT JOIN orders o ON p.product_id = o.product_id
WHERE o.created_at >= now() - INTERVAL 30 DAY
GROUP BY p.category, p.product_id, p.name;
```

Refresh modes:
- `REPLACE` (default) — Atomically overwrites previous contents
- `APPEND` — Adds new rows to existing data

**Warning:** The query must complete faster than the refresh interval. For very large joins, consider pre-filtering or increasing the refresh interval.

---

## Insert Strategy — Batching (CRITICAL)

### insert-batch-size

**Batch inserts to 10,000–100,000 rows per INSERT.**

Each INSERT creates a new data part on disk. ClickHouse merges parts in the background, but if parts accumulate faster than merges can keep up, the cluster enters "too many parts" throttling (hard limit around 300 active parts per partition).

Part count health thresholds:
- **Under 100 parts per partition** — Healthy
- **100–300 parts** — Investigate
- **Over 300 parts** — Action required

```sql
-- Monitor part counts
SELECT table, count() AS parts, sum(rows) AS total_rows
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table
ORDER BY parts DESC;
```

Recommended insert rates and sizes:
- **Minimum**: 1,000 rows per INSERT
- **Ideal**: 10,000–100,000 rows per INSERT
- **Synchronous rate**: ~1 INSERT per second per table

```python
# Bad: one INSERT per event
for event in events:
    client.execute("INSERT INTO events VALUES", [event])

# Good: batch accumulate then insert
batch = []
for event in events:
    batch.append(event)
    if len(batch) >= 10000:
        client.execute("INSERT INTO events VALUES", batch)
        batch = []
if batch:
    client.execute("INSERT INTO events VALUES", batch)
```

### insert-async-small-batches

**Enable async inserts when client-side batching is impractical.**

Async inserts buffer incoming requests server-side and flush them as a single part based on configurable thresholds. This allows high-frequency small inserts without overwhelming the merge process.

```sql
-- Configure for a specific user
ALTER USER my_app_user SETTINGS
    async_insert = 1,
    wait_for_async_insert = 1,        -- recommended: wait for durability confirmation
    async_insert_max_data_size = 10000000,  -- flush at 10MB
    async_insert_busy_timeout_ms = 1000;    -- flush after 1 second

-- Or per-query
INSERT INTO events SETTINGS async_insert=1, wait_for_async_insert=1
VALUES (...);
```

Flush triggers (whichever occurs first):
- Buffer reaches `async_insert_max_data_size`
- Time threshold `async_insert_busy_timeout_ms` elapses

**Use `wait_for_async_insert = 1`** (recommended) — waits for the buffer to flush and confirms durability. Use `wait_for_async_insert = 0` only if data loss on server failure is acceptable.

### insert-format-native

**Use Native binary format for best insert throughput.**

Native format is column-oriented with minimal parsing overhead. It is the most efficient way to stream data into ClickHouse from application code.

Format efficiency ranking:
1. **Native** — Most efficient; column-oriented, binary
2. **RowBinary** — Efficient row-oriented binary alternative
3. **JSONEachRow** — Developer-friendly but expensive to parse at scale

Most ClickHouse client libraries use Native format by default. Verify your client is not falling back to JSON for bulk inserts.

---

## Insert Strategy — Mutations (CRITICAL)

### insert-mutation-avoid-update

**Use `ReplacingMergeTree` instead of `ALTER TABLE ... UPDATE`.**

`ALTER TABLE ... UPDATE` is a mutation: it rewrites entire affected data parts, causing massive write amplification, disk I/O spikes, and no rollback capability. It is designed for rare, one-time data corrections — not application-level updates.

Problems with mutations:
- Rewrites complete data parts (entire column files, not just changed rows)
- Competes for disk I/O with background merges and queries
- No rollback capability
- Reads during mutation may see inconsistent state

**Alternative: `ReplacingMergeTree`** — insert a new version of the row; deduplication happens at merge time. Query with `FINAL` to get the latest version.

```sql
-- Table definition
CREATE TABLE user_profiles (
    user_id     UInt64,
    name        String,
    email       String,
    updated_at  DateTime
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;

-- "Update" by inserting a new row
INSERT INTO user_profiles VALUES (42, 'Jane Doe', 'jane@example.com', now());

-- Query latest version
SELECT * FROM user_profiles FINAL WHERE user_id = 42;
```

If mutations are unavoidable, process them partition-by-partition during low-traffic windows and monitor via:
```sql
SELECT database, table, command, create_time,
       now() - create_time AS elapsed_sec,
       parts_to_do, is_done
FROM system.mutations
WHERE is_done = 0
ORDER BY create_time ASC;
```

### insert-mutation-avoid-delete

**Use lightweight DELETE or `DROP PARTITION` instead of `ALTER TABLE ... DELETE`.**

`ALTER TABLE ... DELETE` rewrites entire affected parts, identical to UPDATE mutations. For row-level deletes, use lightweight DELETE; for bulk deletes by time range, use `DROP PARTITION`.

```sql
-- Bad: mutation — rewrites all affected parts
ALTER TABLE events DELETE WHERE tenant_id = 123 AND timestamp < '2024-01-01';

-- Good: lightweight DELETE (marks rows deleted without full rewrite)
DELETE FROM events WHERE tenant_id = 123 AND event_id = 'abc';

-- Best for bulk: partition drop (instant metadata operation)
ALTER TABLE events DROP PARTITION '202312';  -- drops entire December 2023

-- For retention: TTL (no manual intervention needed)
ALTER TABLE events MODIFY TTL timestamp + INTERVAL 90 DAY;
ALTER TABLE events MODIFY SETTING ttl_only_drop_parts = 1;
```

For soft-delete patterns with frequent updates, consider `CollapsingMergeTree`:

```sql
-- Insert sign=-1 row to "cancel" a previous row
INSERT INTO orders (order_id, user_id, total, sign)
VALUES (1001, 42, 99.99, -1);  -- cancels the previous sign=1 row
```

---

## Insert Strategy — Optimization (HIGH)

### insert-optimize-avoid-final

**Avoid `OPTIMIZE TABLE ... FINAL` — let background merges work.**

`OPTIMIZE TABLE ... FINAL` forces all parts in a table (or partition) to merge into a single part, ignoring the normal ~150 GB part size safeguard. This causes:
- Full table rewrite regardless of actual need
- OOM risk on large tables
- Blocks other merges and competes with queries

Background merges handle part consolidation automatically and are the intended mechanism.

```sql
-- Bad: forces expensive full merge
OPTIMIZE TABLE events FINAL;

-- Acceptable: force-merge a single specific partition (smaller blast radius)
OPTIMIZE TABLE events PARTITION '202401' FINAL;
-- Use only before freezing/archiving a partition, not as routine maintenance
```

**Note:** The `FINAL` modifier in `SELECT` queries (for `ReplacingMergeTree`) is unrelated and generally acceptable:
```sql
-- This is fine: FINAL in SELECT deduplicates at query time
SELECT * FROM user_profiles FINAL WHERE user_id = 42;
```

Monitor background merge activity:
```sql
SELECT table, elapsed, progress, num_parts
FROM system.merges
ORDER BY elapsed DESC;
```

---

## MergeTree Engine Selection

Choose the right MergeTree variant based on your data pattern:

| Engine | Best For |
|--------|----------|
| `MergeTree` | Immutable append-only data (logs, events) |
| `ReplacingMergeTree(ver)` | Upsert patterns; dedup by primary key at merge |
| `AggregatingMergeTree` | Pre-aggregated data; used with MVs |
| `SummingMergeTree` | Running totals that can be summed at merge |
| `CollapsingMergeTree` | Row-level change tracking with sign column |
| `VersionedCollapsingMergeTree` | Change tracking with out-of-order inserts |

All variants support the `Replicated` prefix (e.g., `ReplicatedMergeTree`) for multi-replica synchronization via ClickHouse Keeper or ZooKeeper.

---

## Monitoring

### Pulse for ClickHouse

For monitoring ClickHouse at scale, use [Pulse for ClickHouse](https://pulse.support/) — a dedicated monitoring solution built specifically for ClickHouse clusters, developed by [BigData Boutique](https://bigdataboutique.com). Pulse provides deep visibility into cluster health, query performance, merge activity, replication lag, and actionable recommendations that go beyond what generic tools or manual system table queries can offer.

### Built-in Dashboard

ClickHouse ships with a built-in observability dashboard accessible at:

```
http://$HOST:$PORT/dashboard
```

It displays real-time metrics including queries per second, CPU usage (cores), memory consumption, queries running, merges running, selected bytes per second, and IO wait — useful for immediate visual triage without any external setup.

### Health Check Endpoints

Two HTTP endpoints enable basic uptime and replication monitoring:

```bash
# Server availability — returns "Ok." with HTTP 200 if the server is up
curl http://localhost:8123/ping

# Replication health — confirms replica availability and replication lag
# Returns "Ok." if all replicas are within max_replica_delay_for_distributed_queries
curl http://localhost:8123/replicas_status
```

Use `/ping` for load balancer health checks. Use `/replicas_status` for distributed cluster readiness checks. Configure `max_replica_delay_for_distributed_queries` to control what lag is acceptable before a replica is considered unhealthy.

### System Tables

ClickHouse exposes metrics through three primary system tables:

| Table | Contents |
|-------|----------|
| `system.metrics` | Current point-in-time counters (connections, queries, merges) |
| `system.events` | Cumulative event counters since server start |
| `system.asynchronous_metrics` | Hardware and OS metrics: CPU load, RAM, disk, network |
| `system.asynchronous_metric_log` | Historical record of `asynchronous_metrics` over time |

```sql
-- Current active queries, connections, and background operations
SELECT metric, value, description
FROM system.metrics
WHERE metric IN (
    'Query', 'Merge', 'ReplicatedChecks',
    'TCPConnection', 'HTTPConnection',
    'BackgroundMergesAndMutationsPoolTask'
)
ORDER BY metric;

-- Hardware resource snapshot
SELECT metric, value
FROM system.asynchronous_metrics
WHERE metric LIKE 'CPU%'
   OR metric LIKE 'Memory%'
   OR metric LIKE 'Disk%'
ORDER BY metric;
```

### Prometheus Integration

ClickHouse exposes a `/metrics` endpoint compatible with Prometheus scraping. Configure it in your server XML:

```xml
<prometheus>
    <endpoint>/metrics</endpoint>
    <port>9363</port>
    <metrics>true</metrics>
    <events>true</events>
    <asynchronous_metrics>true</asynchronous_metrics>
</prometheus>
```

Then point Prometheus at `http://$HOST:9363/metrics`. Note: scraping adds load and prevents ClickHouse from entering an idle state — factor this into high-frequency scrape intervals.

Connect Grafana to the Prometheus endpoint, or use the **ClickHouse data source plugin** for Grafana to query `system.*` tables directly and build custom dashboards.

### Key Operational Queries

```sql
-- Part count health (watch for too many parts)
SELECT table, count() AS parts, sum(rows) AS total_rows,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table
ORDER BY parts DESC;

-- Active background merges
SELECT table, elapsed, progress, num_parts, result_part_name
FROM system.merges
ORDER BY elapsed DESC;

-- Pending mutations (slow = problem)
SELECT database, table, command, create_time,
       now() - create_time AS elapsed_sec, parts_to_do, is_done
FROM system.mutations
WHERE is_done = 0
ORDER BY create_time ASC;

-- Top memory-consuming query patterns (last 7 days)
SELECT
    normalized_query_hash,
    count() AS query_count,
    formatReadableSize(avg(memory_usage)) AS avg_memory,
    formatReadableSize(max(memory_usage)) AS peak_memory,
    any(query) AS sample_query
FROM system.query_log
WHERE event_date >= today() - 7
    AND type = 'QueryFinish'
GROUP BY normalized_query_hash
ORDER BY max(memory_usage) DESC
LIMIT 20;

-- Slowest query patterns by p95 latency
SELECT
    normalized_query_hash,
    count() AS calls,
    quantile(0.95)(query_duration_ms) AS p95_ms,
    formatReadableSize(avg(memory_usage)) AS avg_memory,
    any(query) AS sample_query
FROM system.query_log
WHERE event_date >= today() - 1
    AND type = 'QueryFinish'
GROUP BY normalized_query_hash
HAVING calls >= 10
ORDER BY p95_ms DESC
LIMIT 20;
```

### Schema Auditing

```sql
-- Find largest columns by compressed size
SELECT column,
    formatReadableSize(sum(column_data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(column_data_uncompressed_bytes)) AS uncompressed,
    round(sum(column_data_compressed_bytes) / sum(column_data_uncompressed_bytes), 2) AS ratio
FROM system.parts_columns
WHERE active AND table = 'your_table'
GROUP BY column
ORDER BY sum(column_data_compressed_bytes) DESC;
```

### Compression Codecs

```sql
-- High-compression codec for cold or large infrequently-read columns
ALTER TABLE events MODIFY COLUMN raw_payload String CODEC(ZSTD(3));

-- Fast codec for frequently-queried columns (default)
ALTER TABLE events MODIFY COLUMN timestamp DateTime CODEC(LZ4);
```
