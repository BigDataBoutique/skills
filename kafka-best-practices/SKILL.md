---
name: kafka-best-practices
description: Apache Kafka best practices for topic design, producer/consumer configuration, exactly-once semantics, Kafka Streams, Connect, and cluster management
---

# Apache Kafka Best Practices

## Core Principles

- Use `acks=all` with `min.insync.replicas=2` and `replication.factor=3` for durability
- Enable idempotent producers (default since Kafka 3.0) to eliminate duplicates
- Commit offsets manually after processing — never rely on auto-commit for at-least-once
- Use Schema Registry with Avro or Protobuf — never `StringSerializer` for structured data
- Deploy in KRaft mode — ZooKeeper is removed since Kafka 4.0
- Monitor consumer lag, under-replicated partitions, and offline partitions continuously
- Right-size partitions: too many wastes resources, too few caps throughput

## Topic Design (CRITICAL)

### topic-naming

**Use a hierarchical naming scheme with dots or hyphens. Never use underscores.**

```
<domain>.<system>.<entity>
```

Examples: `ecommerce.orders.order-created`, `analytics.clickstream.page-view`

Kafka internally converts dots to underscores in metrics, and underscores in topic names collide with these metric names. Use dots or hyphens only.

### topic-partitions

**Start with 6–12 partitions for most topics. Scale based on throughput needs.**

Sizing formula: `max(target-throughput-MB/s / per-partition-throughput, max-consumers-in-largest-group)`

A single partition typically handles 10–30 MB/s depending on message size and hardware.

- Partitions can be increased but **never decreased**
- Increasing partitions changes key-to-partition mapping, breaking ordering for existing keys
- Stay under 4,000 partitions per broker, under 200,000 per cluster (KRaft improves these limits)
- Over-partitioning wastes file descriptors, memory, and increases end-to-end latency

### topic-replication

**Production: `replication.factor=3` with `min.insync.replicas=2`. Always.**

This tolerates 1 broker failure without data loss while maintaining availability.

```properties
# Topic-level
replication.factor=3
min.insync.replicas=2
```

**Never** set `min.insync.replicas` equal to `replication.factor` — this makes the topic unavailable if any single replica is down.

### topic-retention

**Set `retention.ms` explicitly per topic based on business requirements.**

```properties
# Event streams: 3-7 days typical
retention.ms=604800000

# Changelog / state topics
cleanup.policy=compact

# Compacted with time-based deletion
cleanup.policy=compact,delete
```

Use `message.timestamp.type=CreateTime` (default) for event time semantics. Use `LogAppendTime` only when producer timestamps cannot be trusted.

---

## Producer Configuration (CRITICAL)

### producer-acks

**Use `acks=all` for any data you cannot afford to lose.**

`acks=all` (equivalent to `acks=-1`) waits for all in-sync replicas. This is the default since Kafka 3.0.

| Setting | Durability | Use When |
|---------|-----------|----------|
| `acks=all` | Highest — waits for all ISR | Default. Production data. |
| `acks=1` | Leader only | Metrics/logging where occasional loss is tolerable |
| `acks=0` | None (fire-and-forget) | Non-critical telemetry only |

### producer-idempotency

**Idempotent producers are enabled by default since Kafka 3.0.**

`enable.idempotence=true` eliminates duplicates caused by producer retries using sequence numbers. No meaningful performance penalty.

Idempotency automatically sets: `acks=all`, `retries > 0`, `max.in.flight.requests.per.connection <= 5`.

### producer-batching

**Tune `linger.ms` and `batch.size` together — they are the primary throughput-vs-latency knobs.**

```properties
# Low latency
linger.ms=0
batch.size=16384

# Balanced (linger.ms=5 is the default since Kafka 4.0)
linger.ms=5-20
batch.size=65536

# Maximum throughput
linger.ms=50-200
batch.size=262144-1048576
```

A batch sends when either `linger.ms` elapses or `batch.size` is reached, whichever comes first. Larger batches compress better.

- `buffer.memory=33554432` (32 MB default) — increase if producers block waiting for buffer space. Monitor `buffer-available-bytes` metric.
- `delivery.timeout.ms=120000` (2 min default) — upper bound on time to report success or failure after `send()`.

### producer-compression

**Use `lz4` for best throughput/compression trade-off. Use `zstd` for best compression ratio.**

```properties
compression.type=lz4
```

| Codec | Throughput | Compression Ratio | Best For |
|-------|-----------|-------------------|----------|
| `lz4` | Highest | Good | General purpose (recommended default) |
| `zstd` | Medium | Best | Bandwidth-constrained, large messages |
| `snappy` | High | Good | Balanced alternative |
| `gzip` | Lowest | Good | Legacy compatibility |

Set broker-side `compression.type=producer` (default) to avoid recompression.

### producer-keys

**Keys determine partition assignment and ordering — choose keys that distribute evenly.**

- Messages with the same key go to the same partition and are ordered
- Good keys: `user_id`, `device_id`, `order_id` — distribute evenly
- Null keys use sticky partitioning (since Kafka 2.4) — good for maximum throughput when ordering doesn't matter
- **Changing partition count changes key-to-partition mapping** — plan partition count upfront for keyed topics

### producer-async

**Use async `send()` with callbacks. Never ignore callback errors.**

```java
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        log.error("Send failed for topic={} partition={}",
            record.topic(), record.partition(), exception);
        // Handle error: retry, dead-letter, alert
    }
});
```

Synchronous `send().get()` blocks the calling thread and destroys throughput. Use it only when you need to guarantee ordering of individual sends.

---

## Consumer Configuration (CRITICAL)

### consumer-offset-management

**Disable auto-commit. Commit offsets manually after processing.**

```java
// Configuration
props.put("enable.auto.commit", "false");

// Processing loop
ConsumerRecords<K, V> records = consumer.poll(Duration.ofMillis(100));
for (ConsumerRecord<K, V> record : records) {
    process(record);
}
consumer.commitSync();  // commit after processing
```

- `commitSync()` — blocks until commit succeeds. Guaranteed persistence.
- `commitAsync()` — higher throughput but implement callback to log failures.
- `auto.offset.reset=earliest` for data pipelines; `latest` for real-time monitoring where historical data is irrelevant.

### consumer-poll-tuning

**Tune poll settings to match processing capacity.**

```properties
max.poll.records=500           # default; reduce for expensive processing, increase for lightweight
max.poll.interval.ms=300000    # 5 min default; consumer is kicked if processing takes longer
session.timeout.ms=45000       # default since KIP-735
heartbeat.interval.ms=3000     # default; must be < session.timeout.ms / 3
```

Rule: `heartbeat.interval.ms` < `session.timeout.ms / 3` to allow at least 3 heartbeats before timeout.

Note: With the new consumer group protocol (KIP-848, GA in Kafka 4.0), `session.timeout.ms` and `heartbeat.interval.ms` are controlled server-side via `group.consumer.session.timeout.ms` and `group.consumer.heartbeat.interval.ms`.

### consumer-rebalancing

**Use `CooperativeStickyAssignor` to avoid stop-the-world rebalancing.**

```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

For rolling deployments, use static group membership to avoid rebalances on transient restarts:

```properties
group.instance.id=<unique-per-instance>  # e.g., hostname
session.timeout.ms=300000                # higher timeout with static membership
```

Consumer group protocol v2 (KIP-848, GA in Kafka 4.0) provides server-side rebalancing that is faster and more efficient. In the new protocol, `session.timeout.ms` and `heartbeat.interval.ms` are controlled by broker-side settings.

### consumer-fetch-tuning

**Tune fetch settings for throughput vs. latency.**

```properties
# Low latency (defaults)
fetch.min.bytes=1
fetch.max.wait.ms=500

# Higher throughput
fetch.min.bytes=65536
fetch.max.wait.ms=500
```

Increasing `fetch.min.bytes` accumulates more data per fetch request, reducing request rate at the cost of latency.

---

## Exactly-Once Semantics (HIGH)

### eos-idempotent

**Idempotent producers provide per-partition exactly-once with no performance penalty.**

Enabled by default since Kafka 3.0. Eliminates duplicates from retries using producer sequence numbers.

### eos-transactions

**Use transactions for cross-partition atomic consume-transform-produce.**

```java
producer.initTransactions();
while (true) {
    ConsumerRecords<K, V> records = consumer.poll(Duration.ofMillis(100));
    producer.beginTransaction();
    for (ConsumerRecord<K, V> record : records) {
        producer.send(transform(record));
    }
    producer.sendOffsetsToTransaction(currentOffsets, consumer.groupMetadata());
    producer.commitTransaction();
}
```

- Set `transactional.id=<unique-stable-id>` — unique per producer instance, stable across restarts
- Consumers must set `isolation.level=read_committed` to see only committed messages
- `transaction.timeout.ms=60000` (default) — must be less than broker's `transaction.max.timeout.ms` (default 15 min)
- Transactional producers are **not thread-safe** — use one per thread

### eos-external-systems

**Exactly-once applies within Kafka only.**

For external systems, use:
- Idempotent writes (upsert by key) in the target system
- Outbox pattern: write to database + Kafka in a single database transaction
- Store Kafka offsets alongside results in the same external transaction

---

## Schema Management (HIGH)

### schema-registry

**Always use a Schema Registry in production.**

Use Confluent Schema Registry, Apicurio, or Karapace. One schema per subject (`<topic>-key` and `<topic>-value`).

### schema-format-selection

**Use Avro for most Kafka architectures. Use Protobuf if you also use gRPC.**

| Format | Best For | Trade-offs |
|--------|----------|------------|
| Avro | Kafka ecosystem default | Compact binary, requires schema for read/write |
| Protobuf | Cross-language, gRPC shops | Slightly larger wire format, excellent tooling |
| JSON Schema | Debugging ease, low barrier | Larger wire format, slower serialization |

### schema-compatibility

**Use `BACKWARD` (default) or `FULL` compatibility. Use `FULL` for key schemas.**

- `BACKWARD`: new schema can read old data — safe for adding optional fields or removing fields with defaults
- `FORWARD`: old schema can read new data — safe for removing optional fields
- `FULL`: both backward and forward compatible — safest, most restrictive
- Use transitive variants (`BACKWARD_TRANSITIVE`, `FULL_TRANSITIVE`) to check against all prior versions

### schema-evolution-rules

**Follow these rules for safe schema evolution.**

- Always provide default values for new fields
- Never rename fields — add a new field and deprecate the old one
- Never change field types
- Never remove required fields without a default

---

## Performance Tuning (HIGH)

### perf-tuning-matrix

**Choose settings based on your latency vs. throughput requirements.**

| Goal | `linger.ms` | `batch.size` | `compression` | `fetch.min.bytes` | `acks` |
|------|-------------|-------------|---------------|-------------------|--------|
| Minimum latency | 0 | 16384 | none | 1 | 1 |
| Balanced | 5–20 (5 is default since Kafka 4.0) | 65536 | lz4 | 1024 | all |
| Maximum throughput | 50–200 | 262144+ | lz4/zstd | 65536+ | all |

### perf-broker-tuning

**Key broker settings for performance.**

```properties
num.io.threads=8                 # increase for high I/O load (match to disk count)
num.network.threads=3            # increase for high connection count
num.replica.fetchers=1           # increase to 2-4 if replication can't keep up
```

**Never set `log.flush.interval.messages` or `log.flush.interval.ms`.** Let the OS handle page cache flushing. Kafka's durability comes from replication, not fsync. Setting these causes 100x throughput reduction.

---

## Kafka Streams (HIGH)

### streams-topology

**Keep topologies simple. Use named processors for debuggability.**

```java
StreamsBuilder builder = new StreamsBuilder();
builder.stream("input-topic", Named.as("source-input"))
    .filter((k, v) -> v != null, Named.as("filter-nulls"))
    .mapValues(v -> transform(v), Named.as("transform-values"))
    .to("output-topic", Named.as("sink-output"));
```

Prefer the DSL (`KStream`, `KTable`) for most use cases. Drop to the Processor API only for complex stateful logic.

Minimize repartitioning: `selectKey()` / `map()` that changes the key triggers automatic repartitioning. Batch key changes together.

### streams-state-stores

**Use RocksDB state stores (default) for production. Configure standby replicas for fast failover.**

```properties
num.standby.replicas=1           # pre-populated standby for faster failover
state.dir=/fast-ssd/kafka-streams
statestore.cache.max.bytes=10485760  # 10 MB default; increase for write-heavy workloads
```

Caching deduplicates writes to the changelog topic, reducing I/O. Configure `state.dir` on a fast local SSD.

### streams-exactly-once

**Use `processing.guarantee=exactly_once_v2` for exactly-once processing.**

```properties
processing.guarantee=exactly_once_v2
```

This handles transactional produce + offset commit atomically. Requires broker version 2.5+. Never use the old `exactly_once` setting.

### streams-error-handling

**Configure deserialization and production exception handlers.**

```properties
default.deserialization.exception.handler=org.apache.kafka.streams.errors.LogAndContinueExceptionHandler
```

For poison pills, `LogAndContinueExceptionHandler` logs and skips. Implement a custom handler to route bad records to a dead letter topic.

### streams-scaling

**Scale by adding instances or increasing threads per instance.**

```properties
num.stream.threads=<number-of-cpu-cores>
```

Total thread count across all instances should roughly equal the input topic partition count.

---

## Kafka Connect (HIGH)

### connect-configuration

**Use distributed mode for production. Set converters explicitly per connector.**

```properties
tasks.max=<partition-count>      # more tasks = more parallelism
key.converter=io.confluent.connect.avro.AvroConverter
value.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://schema-registry:8081
value.converter.schema.registry.url=http://schema-registry:8081
```

Never rely on worker-level converter defaults. Never use `JsonConverter` with embedded schemas in production — use Schema Registry converters.

### connect-smts

**Use SMTs for lightweight transformations only.**

Good for: field renaming, routing, timestamp conversion, masking. Common SMTs: `ExtractField`, `ReplaceField`, `InsertField`, `TimestampConverter`, `RegexRouter`, `MaskField`.

Do NOT use SMTs for complex transformations (joins, aggregations, windowing) — use Kafka Streams or Flink instead.

### connect-dead-letter-queue

**Enable dead letter queues for sink connectors.**

```properties
errors.tolerance=all
errors.deadletterqueue.topic.name=<connector-name>-dlq
errors.deadletterqueue.topic.replication.factor=3
errors.deadletterqueue.context.headers.enable=true
```

Monitor DLQ topic size — it should ideally be empty. Alert on messages appearing. Include error context in headers for debugging.

### connect-worker-config

**Set replication factor for internal topics.**

```properties
config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3
```

Use `producer.override.*` and `consumer.override.*` to tune producer/consumer settings per connector.

---

## Cluster Management (HIGH)

### cluster-broker-sizing

**Kafka is I/O and memory (page cache) bound, not CPU bound.**

| Resource | Recommendation |
|----------|---------------|
| CPU | 8–16 cores (more if TLS or compression-heavy) |
| JVM Heap | 6–8 GB max (avoid long GC pauses) |
| RAM | 32–64 GB total — OS page cache is where performance comes from |
| Disk | Multiple `log.dirs` across separate disks (JBOD, no RAID). SSDs for low latency. |
| Network | 10 Gbps minimum for production |

Plan for 60–70% utilization — leave headroom for spikes and rebalancing.

### cluster-rack-awareness

**Set `broker.rack` for replica distribution across failure domains.**

```properties
broker.rack=us-east-1a  # use availability zones in cloud
```

Kafka distributes replicas across racks automatically. Combine with `min.insync.replicas=2` and `replication.factor=3` across 3 AZs for maximum resilience.

### cluster-compaction

**Configure compaction for changelog and state topics.**

```properties
cleanup.policy=compact
min.compaction.lag.ms=3600000     # 1 hour before eligible for compaction
delete.retention.ms=86400000     # 24h — how long tombstones are kept
min.cleanable.dirty.ratio=0.5    # lower = more eager compaction (more I/O)
log.cleaner.threads=1            # increase for many compacted topics
```

### cluster-isr

**Never enable `unclean.leader.election`.**

```properties
unclean.leader.election.enable=false  # default since Kafka 0.11
replica.lag.time.max.ms=30000         # increase in high-latency environments
```

Enabling unclean leader election allows an out-of-sync replica to become leader, causing **data loss**. Monitor ISR shrinks — frequent shrinks indicate overloaded brokers or network issues.

---

## KRaft Mode (HIGH)

### kraft-deployment

**All new deployments must use KRaft. ZooKeeper is removed in Kafka 4.0.**

```properties
process.roles=broker,controller       # combined mode; or separate 'broker' / 'controller'
node.id=1                             # unique per node
controller.quorum.voters=1@ctrl1:9093,2@ctrl2:9093,3@ctrl3:9093
controller.listener.names=CONTROLLER
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
```

- Use 3 or 5 controller nodes (odd number for quorum)
- Dedicated controllers recommended for clusters with >10 brokers
- Combined mode acceptable for smaller clusters

### kraft-benefits

**KRaft advantages over ZooKeeper:**

- Faster controller failover (seconds vs. minutes)
- Supports millions of partitions per cluster (vs. ~200K with ZooKeeper)
- One system to deploy, monitor, and secure instead of two
- Improved broker startup time

### kraft-migration

**Migration from ZooKeeper is one-way — test in staging first.**

Use `kafka-metadata.sh` for migration tooling. Perform online rolling migration: ZK mode -> KRaft dual-write -> KRaft mode. No rollback after finalization.

---

## Tiered Storage (MEDIUM)

### tiered-storage-config

**Use tiered storage for long-retention topics to reduce broker disk costs.**

```properties
# Broker-level
remote.log.storage.system.enable=true
remote.log.storage.manager.class.name=<implementation-class>
remote.log.metadata.manager.class.name=<implementation-class>

# Topic-level
remote.storage.enable=true
local.retention.ms=86400000     # keep 1 day locally (hot tier)
retention.ms=2592000000         # keep 30 days total (local + remote)
```

- Early access since Kafka 3.6; production-ready (GA) since Kafka 3.9
- Recent data stays on local broker disks; older data moves to object storage (S3, GCS, Azure Blob)
- Set `local.retention.ms` based on hot data access pattern (typically 1–3 days)
- Not suitable for compacted topics
- Reads from remote storage are significantly slower than local disk
- Enables infinite retention without scaling broker disks

---

## Security (HIGH)

### security-authentication

**Use `SASL_SSL` for production. Never `SASL_PLAINTEXT`.**

Preferred SASL mechanisms (in order):
1. `OAUTHBEARER` — best for cloud-native, token-based auth
2. `SCRAM-SHA-512` — good when OAuth isn't feasible
3. `GSSAPI` (Kerberos) — for enterprises with existing Kerberos infrastructure
4. `PLAIN` — only acceptable over TLS, for dev environments

### security-tls

**Enable TLS for all inter-broker and client-broker communication.**

```properties
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.keystore.type=PKCS12
ssl.client.auth=required          # mutual TLS when feasible
```

TLS adds ~10–20% CPU overhead. Rotate certificates before expiration using rolling restarts.

### security-acls

**Enable ACLs with deny by default.**

```properties
# KRaft
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
allow.everyone.if.no.acl.found=false
```

- Grant specific operations (READ, WRITE, DESCRIBE) on specific resources
- Use prefixed ACLs (`--resource-pattern-type prefixed`) to avoid per-topic ACL explosion
- Separate service accounts per application — never share credentials

### security-encryption-at-rest

**Kafka does not natively encrypt data at rest.**

Use disk-level encryption (LUKS, dm-crypt) or cloud provider volume encryption (AWS EBS encryption, GCP PD encryption).

---

## Monitoring (CRITICAL)

### monitoring-critical-metrics

**These metrics must be monitored continuously. Alert on any non-zero values.**

| Metric (JMX) | Expected | Alert If |
|--------------|----------|----------|
| `UnderReplicatedPartitions` | 0 | > 0 — data at risk |
| `OfflinePartitionsCount` | 0 | > 0 — partitions unavailable |
| `ActiveControllerCount` | 1 | != 1 — no leader controller |

### monitoring-consumer-lag

**Consumer lag is the single most important consumer metric.**

Monitor via `kafka.consumer:type=consumer-fetch-manager-metrics,name=records-lag-max` or tools like Burrow, Kafka Lag Exporter.

- Alert if lag exceeds expected processing time
- Alert if lag grows continuously (falling behind)
- Differentiate between "catching up" lag (acceptable after restart) and "falling behind" lag (systemic problem)

### monitoring-producer-metrics

**Watch these producer metrics.**

| Metric | Action If |
|--------|-----------|
| `record-error-rate` | > 0 — investigate send failures |
| `batch-size-avg` | Consistently small — increase `linger.ms` |
| `buffer-available-bytes` | Low — increase `buffer.memory` |
| `request-latency-max` | High — broker overloaded or network issue |

### monitoring-stack

**Export JMX metrics via JMX Exporter (Prometheus format) or Jolokia.**

Use Grafana with Prometheus for dashboards. Set alerts on: `UnderReplicatedPartitions > 0`, `OfflinePartitions > 0`, consumer lag growing, request latency p99 > threshold, disk usage > 75%.

---

## Common Anti-Patterns

| Anti-Pattern | Problem | Solution |
|---|---|---|
| Using Kafka as a database | Not a queryable store | Materialize to a database for queries |
| One giant topic for everything | No isolation, can't tune per use case | Separate topics per event type/domain |
| Too many partitions | File descriptor exhaustion, slow elections | Start with 6–12, scale based on throughput |
| `acks=0` or `acks=1` for important data | Data loss on broker failure | `acks=all` with `min.insync.replicas=2` |
| Auto-commit with at-least-once needs | Message loss on crash | Manual offset commits after processing |
| Large messages (>1 MB) | Broker memory pressure, replication lag | Claim check pattern or increase limits explicitly |
| Synchronous `send()` in hot path | Blocks thread, destroys throughput | Async `send()` with callback |
| Ignoring `send()` callback errors | Silent data loss | Always check callback for exceptions |
| Heavy processing in poll loop | Exceeds `max.poll.interval.ms`, triggers rebalance | Offload to thread pool or tune settings |
| `log.flush.interval.messages=1` | 100x throughput reduction | Remove — replication handles durability |
| Underscores in topic names | Collision with metrics names | Use dots or hyphens only |
| Sharing transactional producers | Not thread-safe | One transactional producer per thread |
| Not monitoring consumer lag | Silent pipeline delays | Monitor continuously, alert on growth |
| `unclean.leader.election.enable=true` | Data loss | Never enable in production |
