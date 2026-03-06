---
name: spark-best-practices
description: Apache Spark development best practices for application structure, DataFrame API, shuffle optimization, memory tuning, and Structured Streaming
---

# Apache Spark Best Practices

## Core Principles

- Use DataFrame/Dataset API — never RDD unless absolutely necessary
- Use built-in functions — avoid UDFs (they are black boxes to the optimizer)
- Enable Adaptive Query Execution (AQE) and let it handle partition coalescing and skew
- Use Kryo serialization for all RDD and broadcast operations
- Push filters and column projections as early as possible
- Batch writes into well-sized files (128 MB–1 GB); avoid the small files problem
- Monitor the Spark UI — check shuffle sizes, spill, GC time, and task skew

## Application Structure and Configuration (HIGH)

### app-spark-session

**Use `SparkSession.builder` as the single entry point.**

Set `appName` meaningfully for identification in the Spark UI. Externalize cluster-specific configuration — never hardcode executor counts, memory, or paths in application code.

Configuration precedence (lowest to highest):
1. `spark-defaults.conf`
2. `--conf` flags on `spark-submit`
3. `SparkConf` in code

### app-key-configs

**Always set these configurations explicitly.**

```
spark.serializer=org.apache.spark.serializer.KryoSerializer
spark.sql.adaptive.enabled=true
spark.sql.shuffle.partitions=2000  # set high and let AQE coalesce automatically
spark.default.parallelism=<2-3x total executor cores>
```

For cloud object stores (S3/GCS/ADLS), increase:
```
spark.sql.sources.parallelPartitionDiscovery.parallelism=64
spark.hadoop.mapreduce.input.fileinputformat.list-status.num-threads=16
```

### app-testable-structure

**Separate transformation logic from SparkSession creation.**

Write transformations as functions that take `DataFrame => DataFrame` so they can be tested with small in-memory DataFrames without a cluster.

---

## DataFrame API (CRITICAL)

### df-prefer-dataframe

**Always prefer DataFrame/Dataset API over RDD.**

DataFrames benefit from the Catalyst optimizer and Tungsten execution engine. RDD operations bypass all query optimization. The only valid RDD use cases are custom partitioning logic and legacy code migration.

### df-use-builtins

**Use `org.apache.spark.sql.functions` instead of UDFs.**

UDFs are black boxes to the optimizer — they prevent predicate pushdown, disable code generation, and require row-by-row serialization. If a UDF is unavoidable, prefer Pandas UDFs (vectorized) over scalar UDFs.

```python
# Bad: UDF
from pyspark.sql.functions import udf
upper_udf = udf(lambda x: x.upper())
df.select(upper_udf("name"))

# Good: built-in
from pyspark.sql.functions import upper
df.select(upper("name"))
```

### df-project-early

**Select only needed columns and filter as early as possible.**

This enables column pruning on columnar formats (Parquet/ORC) and reduces data flowing through the DAG.

```python
# Bad: filter and project late
df.join(other, "key").select("col1", "col2").filter("col1 > 10")

# Good: filter and project early
df.select("key", "col1").filter("col1 > 10").join(other.select("key", "col2"), "key")
```

### df-schema-explicit

**Define schemas explicitly — never rely on inference for production pipelines.**

Schema inference requires an extra pass over the data and can produce incorrect types.

```python
from pyspark.sql.types import StructType, StructField, StringType, LongType

schema = StructType([
    StructField("user_id", LongType(), False),
    StructField("name", StringType(), True),
])
df = spark.read.schema(schema).json("s3://bucket/data/")
```

### df-avoid-collect

**Never call `collect()` or `toPandas()` on large datasets.**

These pull all data to the driver and cause OOM. Use `df.head(1)` or `df.isEmpty` (Spark 3.3+) instead of `count()` to check emptiness.

---

## Partitioning and Bucketing (HIGH)

### partition-disk

**Partition by low-cardinality columns used in frequent filter predicates.**

- Target partition sizes of 128 MB–1 GB per partition file
- For time-series data, partition by date or month — not by second or minute
- Avoid partitioning on high-cardinality columns (creates millions of tiny files)

### partition-shuffle

**Set `spark.sql.shuffle.partitions` high and let AQE coalesce.**

The default of 200 is rarely optimal. With AQE enabled, set it to 2000+ and let AQE automatically coalesce small partitions.

```
spark.sql.adaptive.enabled=true
spark.sql.adaptive.coalescePartitions.enabled=true
spark.sql.adaptive.advisoryPartitionSizeInBytes=64MB  # default; increase for larger datasets
spark.sql.shuffle.partitions=2000
```

### partition-coalesce-vs-repartition

**Use `coalesce(n)` to reduce partitions, `repartition(n)` to increase.**

`coalesce` avoids a full shuffle when reducing partitions. `repartition` triggers a full shuffle but ensures even distribution.

### bucketing-for-joins

**Bucket tables that are frequently joined on the same key.**

Bucketing eliminates shuffle at join time. Bucket count should match across tables being joined.

```python
df.write.bucketBy(256, "user_id").sortBy("user_id").saveAsTable("events_bucketed")
```

Bucketing only works with `saveAsTable`, not `write.parquet()`.

---

## Shuffle Optimization (CRITICAL)

### shuffle-broadcast-joins

**Broadcast the small side of a join to eliminate shuffle.**

```python
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_df), "key")
```

```sql
SELECT /*+ BROADCAST(dim) */ * FROM fact JOIN dim ON fact.dim_id = dim.id
```

- Default auto-broadcast threshold: 10 MB (`spark.sql.autoBroadcastJoinThreshold`)
- Increase for larger lookup tables (up to 1 GB if memory permits)
- If a broadcast exceeds executor memory, the job fails with OOM

### shuffle-avoid-groupbykey

**Use `reduceByKey` or `aggregateByKey` instead of `groupByKey`.**

`groupByKey` shuffles all values to the reducer without map-side pre-aggregation. `reduceByKey` combines values locally first, reducing shuffle volume dramatically.

### shuffle-aqe

**Rely on Adaptive Query Execution for runtime optimization.**

AQE (enabled by default since Spark 3.2) handles:
- **Partition coalescing**: merges small post-shuffle partitions
- **Skew join handling**: splits skewed partitions automatically
- **Join strategy switching**: converts sort-merge join to broadcast join at runtime when one side is small

Key skew settings:
```
spark.sql.adaptive.skewJoin.enabled=true
spark.sql.adaptive.skewJoin.skewedPartitionFactor=5.0
spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes=256MB
```

### shuffle-dynamic-partition-pruning

**Ensure filter predicates align with partition columns for Dynamic Partition Pruning.**

DPP (enabled by default since Spark 3.0) pushes dimension table filters down to prune fact table partitions at runtime in star-schema joins.

---

## Memory Management (HIGH)

### memory-executor-sizing

**Size executors with 4–8 cores and 4–8 GB per core.**

- Avoid very large executors (>5 cores leads to GC pressure; >64 GB heaps cause long GC pauses)
- Leave headroom for OS and YARN/K8s overhead

```
spark.executor.memory=8g
spark.executor.memoryOverhead=2g
spark.executor.cores=4
spark.driver.memory=4g
```

### memory-fraction-tuning

**Tune the unified memory region based on workload.**

```
spark.memory.fraction=0.6          # fraction of (heap - 300MB) for execution + storage
spark.memory.storageFraction=0.5   # protected floor for cached data within the region
```

- Increase `memory.fraction` for shuffle/join-heavy workloads
- Decrease for workloads with many user objects on heap

### memory-offheap

**Enable off-heap memory for large shuffle or join workloads.**

```
spark.memory.offHeap.enabled=true
spark.memory.offHeap.size=4g
```

Reduces GC pressure by moving shuffle and join data structures off the JVM heap.

### memory-gc-tuning

**Monitor GC time in the Spark UI Executors tab.**

- If GC time > 10% of task time, tune memory or GC settings
- G1GC is the default GC in JDK 17 (required since Spark 4.0): `-XX:G1HeapRegionSize=16m`
- If spill (memory or disk) appears in any stage, increase executor memory or parallelism

---

## Serialization (MEDIUM)

### serialization-kryo

**Always use Kryo serialization.**

```
spark.serializer=org.apache.spark.serializer.KryoSerializer
```

10x faster and more compact than default Java serialization. Register custom classes for best performance:

```scala
conf.registerKryoClasses(Array(classOf[MyEvent], classOf[MyUser]))
```

For DataFrames, Spark uses its own Tungsten binary format internally — Kryo matters primarily for RDD operations and broadcast variables.

---

## Caching and Persistence (MEDIUM)

### cache-when-reused

**Cache DataFrames that are reused multiple times. Unpersist when done.**

```python
filtered = df.filter("status = 'active'").select("id", "name").cache()
# ... use filtered multiple times ...
filtered.unpersist()
```

Do NOT cache DataFrames used only once — caching has overhead with no payoff.

### cache-storage-levels

**Choose storage level based on dataset size and access pattern.**

| Level | Use When |
|-------|----------|
| `MEMORY_AND_DISK` | Default (`df.cache()`). Good general choice. |
| `MEMORY_AND_DISK_SER` | Larger datasets — trades CPU for reduced memory with Kryo |
| `MEMORY_ONLY` | Small datasets needing fast access |
| `DISK_ONLY` | Recomputation is more expensive than disk I/O |

**Anti-pattern:** Caching before a `filter()` or `select()`. Always filter and project first, then cache the reduced dataset.

---

## Spark SQL (HIGH)

### sql-explain-plans

**Use `EXPLAIN COST` to verify optimizer decisions.**

```python
df.explain("cost")       # DataFrame API
spark.sql("EXPLAIN COST SELECT ...")  # SQL
```

Check for predicate pushdown, partition pruning, and expected join strategies. Compare actual vs. estimated row counts.

### sql-compute-statistics

**Run `ANALYZE TABLE` to give the optimizer accurate statistics.**

```sql
ANALYZE TABLE db.events COMPUTE STATISTICS FOR ALL COLUMNS;
```

Without statistics, the optimizer makes poor join strategy choices (e.g., sort-merge instead of broadcast).

### sql-cte-materialization

**By default, Spark inlines CTEs — each reference re-executes the subquery.**

If a CTE is used multiple times, either:
- Materialize it with `.cache()` or create a temp view and cache it
- Enable CTE materialization (Spark 3.5+): `spark.sql.optimizer.cteMaterialization.enabled=true`

### sql-window-functions

**Window functions over large partitions are expensive.**

Add `ORDER BY` only when required (e.g., `ROW_NUMBER` needs it; `SUM` over the whole partition does not). Use `ROWS BETWEEN` frame specs instead of `RANGE BETWEEN` when possible.

---

## Structured Streaming (HIGH)

### streaming-watermarks

**Always define watermarks for stateful operations.**

```python
df.withWatermark("eventTime", "10 minutes")
```

Without watermarks, state grows unboundedly and will eventually OOM. Set the delay based on your data's actual lateness characteristics.

### streaming-triggers

**Choose the right trigger mode.**

| Trigger | Use When |
|---------|----------|
| `ProcessingTime("30 seconds")` | Standard micro-batch; good default |
| `AvailableNow()` | Process all available data then stop; ideal for scheduled batch-style streaming |
| `Continuous("1 second")` | Experimental low-latency mode (limited operator support) |
| Default (no trigger) | Process as fast as possible |

### streaming-checkpoints

**Always set `checkpointLocation` on a reliable distributed filesystem.**

```python
query = df.writeStream \
    .format("parquet") \
    .option("checkpointLocation", "s3://bucket/checkpoints/my-query") \
    .start()
```

Checkpoint format is tied to the query plan — changing the query structure may require clearing checkpoints.

### streaming-state

**Monitor state size and set timeouts to expire stale state.**

- Use `mapGroupsWithState` or `flatMapGroupsWithState` for custom stateful processing
- Set processing-time or event-time timeouts to expire stale state
- Monitor state size in the Structured Streaming tab of Spark UI
- For large state, consider RocksDB state backend (`spark.sql.streaming.stateStore.providerClass`)

### streaming-kafka

**For Kafka sources, set `maxOffsetsPerTrigger` to bound batch sizes.**

Set `spark.sql.shuffle.partitions` to match your Kafka topic partition count or desired parallelism. Use `foreachBatch` for complex sink logic or writing to multiple sinks from one stream.

---

## Output File Optimization (HIGH)

### output-file-sizing

**Write well-sized output files (128 MB–1 GB).**

- Use `coalesce(n)` or `repartition(n)` before writes to control file count
- Use the `REBALANCE` hint in SQL: `SELECT /*+ REBALANCE(3, col) */ * FROM t`
- Set `spark.sql.files.maxPartitionBytes=128MB` for reads

### output-format

**Use Parquet with Snappy or ZSTD compression.**

```python
df.write.format("parquet") \
    .option("compression", "zstd") \
    .partitionBy("date") \
    .save("s3://bucket/output/")
```

Parquet enables column pruning, predicate pushdown, and efficient encoding. Always define explicit schemas when reading.

---

## Common Anti-Patterns

| Anti-Pattern | Problem | Solution |
|---|---|---|
| Using RDD API | Bypasses optimizer | Use DataFrame/Dataset API |
| `collect()` on large data | Driver OOM | Use distributed operations or `take(n)` |
| `groupByKey` | No map-side reduction | Use `reduceByKey` / `aggregateByKey` |
| Not repartitioning after filter | Many empty partitions | `coalesce()` after heavy filtering |
| Caching everything | Memory pressure, eviction | Cache only reused DataFrames |
| Python scalar UDFs | 10–100x slower than built-ins | Use built-in functions or Pandas UDFs |
| Schema inference in production | Extra data pass, wrong types | Define schemas explicitly |
| `count()` to check emptiness | Full scan | Use `head(1)` or `isEmpty` |
| Hardcoded `shuffle.partitions=200` | Wrong for most workloads | Enable AQE, set high, let it coalesce |
| `SELECT *` on wide tables | Reads all columns | Project only needed columns |
| `repartition()` to reduce partitions | Unnecessary full shuffle | Use `coalesce()` instead |
| Missing `unpersist()` | Cached data never freed | Always unpersist when done |
| Accidental cross joins | O(n*m) output | Always specify join conditions |
| Ignoring Spark UI | Missed optimization opportunities | Check shuffle, spill, GC, skew |

---

## Performance Monitoring

### monitoring-spark-ui

**Always check the Spark UI for optimization opportunities.**

Key tabs:
- **Stages**: shuffle read/write sizes, GC time, task duration distribution (min/median/max — large spread indicates skew)
- **SQL**: query plans with runtime metrics — click to see actual row counts vs. estimates
- **Storage**: cached DataFrames, memory usage
- **Executors**: per-executor memory, GC time, shuffle stats
- **Structured Streaming**: input rate, processing rate, batch duration, state size

### monitoring-key-metrics

**Watch these metrics in the Spark UI.**

| Metric | Healthy | Action Required |
|--------|---------|-----------------|
| GC time | < 10% of task time | Tune memory/GC settings |
| Spill (memory/disk) | None | Increase executor memory or parallelism |
| Task skew (max/median) | < 2x | Use AQE skew handling or salt keys |
| Shuffle write | Proportional to data | Check for unnecessary shuffles |

### monitoring-event-log

**Enable event logging for post-mortem analysis.**

```
spark.eventLog.enabled=true
spark.eventLog.dir=s3://bucket/spark-logs/
```

Use the Spark History Server to analyze completed or failed jobs.
