# Kafka for Principal Data Engineer Interviews — Part 1: Theory (K1–K30)

**How to use:** Answer aloud first (2 min), then compare. Each entry has: **Answer**, **Principal signal** (what lifts it above a Senior answer), **Interview example** (a concrete scenario, snippet or calculation to ground your answer), and a **Follow-up** you should expect.

**Version note:** Defaults and features below reflect Apache Kafka 3.x and 4.x. Kafka 4.0 removed ZooKeeper (KRaft only) and made the new consumer group protocol (KIP-848) generally available; confirm exact defaults and feature status against the docs for the version you will be asked about. Numbers in examples are illustrative: substitute figures from your own systems.

---

## A. Fundamentals and architecture

### K1. What is Kafka, and what problem does it solve? Explain brokers, topics, partitions, replicas.
**Answer.** Kafka is a distributed, partitioned, replicated, append-only **commit log** used as a durable event streaming platform. Producers append records to **topics**; each topic is split into **partitions** (the unit of parallelism and ordering); each partition is replicated across **brokers** (one leader serving reads/writes, followers replicating). Consumers pull records by **offset** and track their own position, so many independent consumer groups can read the same data at different speeds, and data is retained by time/size policy, not deleted on consumption.

**Principal signal.** Frame Kafka as a *log*, not a queue: it decouples producers from consumers in time and speed, enables replay, and makes the log the system of record for events. Say where it is a poor fit (request/response RPC, tiny low-volume workloads, per-message routing with complex filtering).

**Interview example.** "At an e-commerce company, the `orders` topic has 24 partitions and RF=3. Checkout publishes `OrderPlaced`. Inventory, fraud, email, and the analytics lake each consume it as separate consumer groups. If the fraud team deploys a bug and needs to reprocess yesterday's orders, they reset their group's offsets. Nobody else is affected and the producer never knows."

**Follow-up.** "Why not just use a database table as a queue?" (Throughput, retention/replay, fan-out to many consumers, ordered partitioned scalability.)

### K2. How is data stored on disk? Explain segments, offsets, and indexes.
**Answer.** Each partition is a directory of **log segments** (`00000000000000012345.log`, named by base offset), plus sparse **offset index** (`.index`) and **time index** (`.timeindex`) files. Only the active segment is written; when it reaches `segment.bytes` (1 GiB default) or `segment.ms` (7 days), it rolls. Retention and compaction operate on *closed* segments. Lookup by offset: binary-search the segment list by base offset, then the sparse index, then scan forward.

**Principal signal.** Retention granularity is the segment: with a 7-day `retention.ms` and a slow-filling topic whose active segment hasn't rolled, data can live longer than expected. Tune `segment.ms` on low-volume topics.

**Interview example.** "A low-volume audit topic had `retention.ms` = 24 h but disk still held weeks of data, because it received ~1 MB/day and a 1 GiB segment never rolled. Setting `segment.ms=3600000` made segments roll hourly so retention could delete them. You can inspect with `kafka-dump-log.sh --files 000...log --print-data-log`."

**Follow-up.** "How does a consumer find offset 5,000,000 quickly?" (Segment base offsets, sparse index, short scan.)

### K3. Explain producer internals: batching, `linger.ms`, `batch.size`, compression, `acks`.
**Answer.** `send()` appends to an in-memory **record accumulator** (`buffer.memory`, 32 MiB default) grouped per partition into batches. A sender thread ships a batch when it reaches `batch.size` (16 KiB default) or `linger.ms` expires (older default 0; check your version, since 4.0 raised it). Compression (`lz4`, `zstd`, `snappy`, `gzip`) is applied *per batch*, so bigger batches compress better. `acks=0` (fire and forget), `1` (leader only), `all` (all in-sync replicas). Since Kafka 3.0 the producer defaults to `acks=all` and idempotence enabled. Other limits: `max.in.flight.requests.per.connection` (5), `delivery.timeout.ms` (120 s), `max.block.ms` (60 s), `retries` (effectively unbounded within the delivery timeout).

**Principal signal.** Throughput comes from batching, not from more producer threads. The latency/throughput trade-off is explicit: `linger.ms` buys batch size at the cost of a few ms.

**Interview example.** "A telemetry producer sending 200-byte events was doing ~8k msg/s with one request per message. Setting `linger.ms=20`, `batch.size=131072`, `compression.type=zstd` produced batches of ~600 records, cut network bytes ~4×, and raised throughput by an order of magnitude at the price of ≤20 ms added latency. For a payments topic where latency matters more, I'd keep `linger.ms` small and rely on `acks=all` plus idempotence."

**Follow-up.** "What happens if the broker is down for 3 minutes?" (Records wait in the buffer until `delivery.timeout.ms`; then failures surface to the callback.)

### K4. What delivery semantics does Kafka offer? How does the idempotent producer work?
**Answer.** At-most-once (commit offset before processing), at-least-once (process, then commit; duplicates on retry/failure), exactly-once (idempotence + transactions, within Kafka). The **idempotent producer** gets a **producer ID (PID)** and assigns a per-partition **sequence number** to each batch; the broker rejects duplicates/out-of-order sequences, so internal retries don't create duplicates *within a partition and producer session*. It does not dedupe across producer restarts (new PID) or across different producers.

**Principal signal.** Exactly-once is scoped: Kafka guarantees it for read-process-write *inside Kafka*. For external sinks you need idempotent writes or transactional two-phase patterns.

**Interview example.** "Network timeout after the broker already wrote a batch: without idempotence the producer retries and the record appears twice. With it, the retry carries the same PID and sequence, the broker recognizes the duplicate and acknowledges without appending. But if my service crashes and restarts, it gets a new PID, so for an at-least-once source I still key the downstream writes on a business ID."

**Follow-up.** "Does idempotence preserve ordering with `max.in.flight=5`?" (Yes, since brokers track sequence numbers.)

### K5. How do Kafka transactions and exactly-once semantics (EOS) work?
**Answer.** A producer with a `transactional.id` registers with the **transaction coordinator**, which tracks state in the internal `__transaction_state` topic. The producer calls `beginTransaction`, writes to multiple partitions/topics, includes the consumer's offsets with `sendOffsetsToTransaction` (so offset commit and output are atomic), then `commit` or `abort`. The coordinator writes **commit/abort markers** to each partition. Consumers with `isolation.level=read_committed` only see committed records, and read up to the **last stable offset (LSO)**. The `transactional.id` fences zombie instances via epochs.

**Principal signal.** Cost and limits: added latency (commit markers), consumers with `read_committed` are blocked by the oldest open transaction (hanging transactions stall consumers), and it does not extend to external systems.

**Interview example.** "A stream job reads `payments-raw`, enriches, and writes `payments-enriched`. With EOS, the output records and the input offset commit happen in one transaction; if the job crashes mid-batch, the open transaction is aborted and the batch is reprocessed, so downstream `read_committed` readers never see partial or duplicate output. In Kafka Streams this is `processing.guarantee=exactly_once_v2`."

**Follow-up.** "What is a zombie and how is it fenced?" (Old instance with same `transactional.id`; epoch bump fences it.)

### K6. Explain consumer groups, partition assignment, and rebalancing.
**Answer.** Consumers sharing a `group.id` divide a topic's partitions among themselves; each partition is read by exactly one member per group. A **group coordinator** (a broker) manages membership via heartbeats (`session.timeout.ms` 45 s, `heartbeat.interval.ms` 3 s) and `max.poll.interval.ms` (5 min; the consumer must call `poll` within it). **Rebalances** occur on member join/leave/crash, subscription or partition-count changes. Assignors: Range, RoundRobin, Sticky, **CooperativeSticky** (incremental, avoids stop-the-world). **Static membership** (`group.instance.id`) avoids rebalances on quick restarts. Kafka 4.0's KIP-848 protocol moves assignment logic to the broker and makes rebalances incremental by design.

**Principal signal.** Rebalance storms are a top production issue: slow processing exceeding `max.poll.interval.ms`, GC pauses, rolling deploys, autoscaling flapping.

**Interview example.** "During every deploy our consumers paused ~40 s because the eager protocol revoked all partitions. We switched to `CooperativeStickyAssignor` plus static membership with stable pod names, and rolling restarts moved only the restarted pod's partitions. Max useful consumers = partition count: with 12 partitions, a 20-instance group leaves 8 idle."

**Follow-up.** "A consumer is kicked out repeatedly. What do you check?" (`max.poll.interval.ms` vs batch processing time, `max.poll.records`, GC, network, session timeout.)

### K7. How do offsets and commits work? Compare commit strategies.
**Answer.** Committed offsets are stored in the internal compacted topic `__consumer_offsets`. Commit semantic: "next offset to read" (last processed + 1). Strategies: **auto-commit** (`enable.auto.commit=true`, every 5 s; can lose or duplicate on crash), **sync manual** (after processing, safest, slower), **async manual** (faster; handle failures; no retries ordering guarantee), **commit on rebalance** (`onPartitionsRevoked`), and **transactional** (offsets in the producer transaction). `auto.offset.reset` (`latest` default, `earliest`, `none`) only applies when no valid committed offset exists.

**Principal signal.** The commit is the checkpoint; processing + side effect + commit ordering defines semantics. Decide it explicitly per pipeline.

**Interview example.** "Auto-commit committed offsets for records still in my in-memory batch; a crash lost 500 records silently. We moved to: poll → process batch → write to sink → `commitSync()`. That's at-least-once, so the sink does an upsert keyed on `(topic, partition, offset)` or a business key to make duplicates harmless."

**Follow-up.** "A new consumer group starts with `auto.offset.reset=latest`. What do you miss?" (Everything produced before it joined.)

### K8. How does replication work? ISR, leader election, high watermark, leader epoch.
**Answer.** Each partition has a leader and followers that **fetch** from the leader like consumers. The **ISR** (in-sync replicas) are followers caught up within `replica.lag.time.max.ms` (30 s). The **high watermark (HW)** is the offset replicated to all ISR members; consumers can only read up to HW. If a leader fails, the controller elects a new leader from the ISR (`unclean.leader.election.enable=false` by default, so no out-of-sync replica is promoted). **Leader epochs** let followers truncate to a consistent point after leader changes, avoiding divergent logs.

**Principal signal.** Be precise: producers with `acks=all` get acknowledged when all *current ISR* have the record, so durability depends on `min.insync.replicas`, not just RF.

**Interview example.** "RF=3 with ISR={1,2,3}. Broker 1 (leader) dies after acknowledging offset 100 to a producer using `acks=all`; brokers 2 and 3 already have it, so the new leader serves offset 100. With `acks=1` the leader could have acked and died before replicating, losing that record."

**Follow-up.** "Why can't consumers read beyond HW?" (Data not yet on all ISR could vanish on leader failover.)

### K9. How do you configure for durability vs availability?
**Answer.** Durability set: `replication.factor=3`, `min.insync.replicas=2`, producer `acks=all`, idempotence on, `unclean.leader.election.enable=false`, `retries` high with `delivery.timeout.ms`. Trade-off: with RF=3 and min.ISR=2, you tolerate one broker loss while still accepting writes; if two replicas are down, writes fail (`NotEnoughReplicas`) rather than risk loss, so you choose consistency over availability for that partition. Rack awareness (`broker.rack`) spreads replicas across failure domains.

**Principal signal.** Tie the config to a business decision: which topics may drop events (clickstream: RF=2, acks=1 maybe) vs which cannot (payments).

**Interview example.** "`payments`: RF=3, min.ISR=2, acks=all, rack-aware across 3 AZs, so an AZ loss keeps writes flowing. `clickstream-debug`: RF=2, acks=1, short retention, because losing a few events beats paying cross-AZ replication cost. We document that decision per topic tier in our platform standards."

**Follow-up.** "RF=3, min.ISR=3: what's wrong?" (Any single broker loss blocks writes: no fault tolerance for availability.)

### K10. What changed with KRaft? Compare to ZooKeeper.
**Answer.** Historically, ZooKeeper stored metadata (brokers, topics, ACLs, controller election). **KRaft** replaces it with a Raft-based quorum of controller nodes storing metadata in an internal log (`__cluster_metadata`) that brokers replicate. Benefits: simpler ops (one system), faster controller failover, much faster metadata propagation and higher partition scalability (millions in principle), and no ZK dependency. Kafka 4.0 is KRaft-only; ZooKeeper mode is removed. Controller quorum can be dedicated (`process.roles=controller`) or combined with brokers (small/dev clusters).

**Principal signal.** Migration planning: 3.x clusters on ZK must migrate via the supported migration path (≥3.6/3.7 recommended) before upgrading to 4.0.

**Interview example.** "We had a cluster with ~150k partitions where controller failover on ZK took minutes because the new controller had to load all metadata. After migrating to KRaft, failover is seconds, since controllers already hold the replicated metadata log. For production we run 3 dedicated controllers on separate nodes (5 for very large or highly critical clusters)."

**Follow-up.** "How many controller nodes and why odd numbers?" (Raft majority; 3 tolerates 1 failure, 5 tolerates 2.)

## B. Partitioning, ordering, retention, performance

### K11. How do partitions affect ordering and parallelism? How do you pick keys and partition counts?
**Answer.** Ordering is guaranteed **only within a partition**. Records with the same key go to the same partition (default partitioner hashes the key with murmur2 modulo partition count), so key = ordering scope. Partition count caps consumer parallelism and throughput, and **changing it re-maps keys** (new records for a key may land elsewhere, breaking per-key ordering across the change). Too many partitions increase metadata, open files, rebalance time, end-to-end latency, and failover time.

**Principal signal.** Size for *future* peak, since you can't decrease partitions. A rough method: partitions ≥ max(target throughput / per-partition producer throughput, target throughput / per-partition consumer throughput).

**Interview example.** "Target 300 MB/s on a topic. Measured: one partition handles ~20 MB/s producing and one consumer thread ~10 MB/s processing → need ≥ 30 partitions for consumers. I'd choose 36-48 for headroom and divisibility across consumer counts. Key = `customer_id` so each customer's events stay ordered; if one customer is 20% of traffic, that partition becomes hot, so I'd consider a composite key like `customer_id + shard` only where per-customer ordering isn't required."

**Follow-up.** "Can you increase partitions safely?" (Yes technically; keyed ordering and compacted-topic semantics are affected.)

### K12. Explain retention policies and log compaction.
**Answer.** `cleanup.policy=delete` removes old segments by `retention.ms` (7 days default) or `retention.bytes`. `cleanup.policy=compact` keeps at least the **latest record per key**; the cleaner rewrites older segments, removing superseded records. A **tombstone** (key with null value) marks deletion and is retained for `delete.retention.ms` (24 h) before removal. Compaction never touches the active segment, is not immediate, and offsets are preserved (gaps appear). `compact,delete` combines both. `min.cleanable.dirty.ratio` (0.5) governs when cleaning triggers.

**Principal signal.** Compacted topics are *tables*: great for changelogs, config distribution, and rebuilding state. They are not a way to get immediate dedup.

**Interview example.** "`customer-profile` is compacted, keyed by `customer_id`. A new service can consume from the beginning and reconstruct current state of all 20M customers by reading ~20M records, not the full history. A GDPR delete is a tombstone; consumers must treat null values as deletes, and the data physically disappears only after compaction plus `delete.retention.ms`."

**Follow-up.** "Does compaction guarantee only one record per key at read time?" (No: only eventually; the active segment and not-yet-cleaned segments may hold duplicates.)

### K13. Why is Kafka fast? Page cache, sequential I/O, zero-copy, batching.
**Answer.** (1) **Sequential disk I/O**: append-only writes and sequential reads, which are fast even on spinning disks and excellent on SSD/NVMe. (2) **OS page cache**: Kafka relies on it rather than a heap cache, so reads of recent data are served from memory, with no GC overhead and warm restarts. (3) **Zero-copy** (`sendfile`) transfers from page cache to the socket without copying through user space (not possible with TLS in the broker, since encryption needs user-space processing). (4) **Batching and compression** end to end: the same batch format on the wire and on disk. (5) Partition parallelism.

**Principal signal.** Operationally this means: leave RAM to the OS (small broker heap, ~6 GB, large page cache), and consumers lagging beyond page cache cause disk reads that slow everyone.

**Interview example.** "A cluster slowed whenever a batch job replayed 3 days of data: those reads missed page cache, hit disk, and evicted hot pages used by real-time consumers, so tail latency rose for everyone. Mitigations: consumer quotas, replaying from tiered storage/another cluster, or scheduling replays off-peak."

**Follow-up.** "Does enabling TLS affect throughput? Why?" (Loses zero-copy; adds CPU encryption cost.)

### K14. How does the broker process requests? Describe the threading model and what causes latency.
**Answer.** Network **acceptor/processor threads** read requests into a request queue; **I/O (request handler) threads** (`num.io.threads`) process them (appending to logs, serving fetches); **purgatory** holds delayed produce (waiting for acks) and fetch (waiting for `fetch.min.bytes`/`fetch.max.wait.ms`) requests; background threads handle replication, log cleaning/flush, and retention. Latency sources: request queue saturation, slow disk (fsync/flush), replication lag when `acks=all`, GC, page cache misses, noisy tenants, large messages.

**Principal signal.** Use request metrics (`RequestQueueTimeMs`, `LocalTimeMs`, `RemoteTimeMs`, `ResponseQueueTimeMs`) to decide where the time goes, rather than guessing.

**Interview example.** "p99 produce latency spiked. `RemoteTimeMs` was high, meaning time waiting for followers: one follower sat on a degraded disk. `RequestQueueTimeMs` was normal, so adding I/O threads wouldn't help. We replaced the disk, and ISR shrink/expand events stopped."

**Follow-up.** "What is purgatory?" (A structure that holds requests until their completion conditions or timeouts are met.)

### K15. What causes consumer lag, and how do you diagnose and fix it?
**Answer.** Lag = log end offset − committed (or current) offset per partition. Causes: slow processing (DB/API latency), too few consumers/partitions, hot partitions, rebalances, GC, large `max.poll.records` batches, producer burst, broker issues, downstream backpressure, or a stuck/poison message. Diagnose: lag per partition (skewed vs uniform), consumer processing time, rebalance frequency, fetch settings, broker request metrics. Fix: scale consumers (≤ partitions), optimize processing (batching, async I/O, bulk writes), add partitions with care, fix skew, tune `fetch.min.bytes`/`max.poll.records`.

**Principal signal.** Alert on **time lag** (how stale, in seconds) and its trend, not only message count, because count alone doesn't tell you whether you'll meet an SLA.

**Interview example.** "Lag grew on only partition 7. A single hot customer key accounted for 40% of its traffic and one consumer processed it sequentially. Adding consumers wouldn't help. We split that customer's key space with a sub-key where ordering permitted and moved heavy enrichment to batched async calls. In another case, uniform lag across all partitions meant under-provisioned consumers, so we scaled out."

**Follow-up.** "Lag is zero but the business says data is late. Why?" (Producer-side delay, wrong timestamp semantic, downstream batch latency, consumer reading the wrong topic/cluster.)

## C. Ecosystem: schemas, Connect, CDC, stream processing

### K16. Why use Schema Registry? Explain serialization formats and compatibility modes.
**Answer.** Schema Registry stores versioned schemas (Avro, Protobuf, JSON Schema) per **subject** (e.g. `orders-value`) and enforces **compatibility** at registration time. Messages carry a small header: magic byte `0` + 4-byte schema ID, then the payload; consumers fetch (and cache) the schema by ID. Compatibility modes: **BACKWARD** (default; new schema can read data written with old; allows deleting fields and adding optional fields with defaults; upgrade consumers first), **FORWARD** (old schema can read new data; upgrade producers first), **FULL**, and **\*_TRANSITIVE** variants (check against all prior versions), and **NONE**.

**Principal signal.** Schemas are *data contracts* between teams. Governance (compatibility policy per topic tier, ownership, review of breaking changes) matters more than the serialization format.

**Interview example.** "Adding `coupon_code` as `["null","string"]` with default null is backward compatible: consumers on the new schema read old records (field absent → null). Renaming `amount` to `total` is breaking; instead add `total`, dual-write for a deprecation window, migrate consumers, and then remove `amount`, with the registry configured as `BACKWARD_TRANSITIVE` for critical topics."

**Follow-up.** "Avro vs Protobuf vs JSON?" (Avro: compact, strong evolution, schema needed at read; Protobuf: widely used for services and field numbers; JSON: readable, bloated and weak typing unless JSON Schema.)

### K17. Explain Kafka Connect: architecture, converters, SMTs, scaling, error handling.
**Answer.** Connect is a framework for scalable, fault-tolerant data movement. **Workers** run **connectors** (config) that spawn **tasks** (parallel units). Standalone vs **distributed mode** (workers form a cluster, store config/offset/status in internal Kafka topics). **Converters** (`key.converter`, `value.converter`: Avro, JSON, String) serialize between Connect's internal format and bytes; **Single Message Transforms (SMTs)** apply light per-record changes (mask, rename, route). Error handling: `errors.tolerance=all`, `errors.deadletterqueue.topic.name` (sink connectors), `errors.deadletterqueue.context.headers.enable`, `errors.retry.*`, `errors.log.enable`. Exactly-once for source connectors exists (KIP-618, 3.3+) when supported by the connector.

**Principal signal.** Connect is best for standard integrations; heavy transformation belongs in a stream processor. Operate it like a platform: connector config in Git, secrets via config providers, monitoring of task failures and lag.

**Interview example.** "S3 sink connector: `tasks.max=8` for an 8-partition topic, `flush.size` and `rotate.interval.ms` to control file size, `errors.tolerance=all` with a DLQ topic so one malformed record doesn't stop the connector. We alert on DLQ rate and on tasks in FAILED state; a failed task does not auto-restart unless we've configured restart tooling."

**Follow-up.** "Where are connector offsets stored?" (Source: Connect's offsets topic; sink: normal consumer-group offsets.)

### K18. Explain CDC with Debezium and the outbox pattern.
**Answer.** Debezium is a set of Connect source connectors that read the database's transaction log (MySQL binlog, Postgres WAL via a replication slot, SQL Server/Oracle CDC) and emit change events per table (default topic `server.schema.table`) in an envelope: `before`, `after`, `op` (`c/u/d/r`), `ts_ms`, `source` metadata (log position/LSN). Phases: initial **snapshot** then streaming. Deletes produce a delete event followed by a tombstone. The **outbox pattern** solves the dual-write problem: the service writes business data *and* an event row to an `outbox` table in one DB transaction; Debezium's Outbox Event Router SMT routes those rows to topics.

**Principal signal.** Operational risks: Postgres replication slots retain WAL if the connector is down (disk fill), schema changes in source tables, snapshot load on the primary, large transactions, ordering across tables, and PII in raw change streams.

**Interview example.** "Order service must update `orders` and tell other services. Writing to the DB then calling Kafka can lose the event on a crash between steps. With the outbox, one transaction inserts the order and an `OrderPlaced` row; Debezium publishes it, at-least-once, so consumers dedupe by event ID. We once had a connector down over a weekend and the Postgres slot retained WAL until disk alerts fired. We now alert on slot lag and set `max_slot_wal_keep_size`."

**Follow-up.** "How do you handle a snapshot of a 2 TB table?" (Incremental/ad-hoc snapshots with signaling, read replica where supported, chunking.)

### K19. Kafka Streams vs Flink vs Spark Structured Streaming vs ksqlDB: how do you choose?
**Answer.** **Kafka Streams**: a Java library (no cluster); state in RocksDB with **changelog topics** and standby replicas; exactly-once within Kafka; scales by partitions; best for microservice-style processing on Kafka only. **Flink**: a distributed engine with strong event-time semantics, large managed state, many sources/sinks, SQL; higher operational footprint. **Spark Structured Streaming**: micro-batch (seconds latency), unifies batch+stream, great for lakehouse ETL. **ksqlDB**: SQL on Kafka Streams, quick prototypes, limited flexibility.

**Principal signal.** Decide by latency need, state size, team skills (JVM vs Python/SQL), sink types, and operational ownership, not by popularity.

**Interview example.** "For a fraud-scoring service consuming `payments` and emitting `alerts` in <100 ms with small keyed state, I'd use Kafka Streams embedded in the service. For hourly lakehouse aggregation of the same topic, Spark Structured Streaming into Iceberg is cheaper and simpler. For a 10 TB-state, event-time-heavy sessionization with joins across streams, Flink."

**Follow-up.** "What are changelog topics for?" (They back up local state stores so state can be rebuilt on another instance.)

### K20. Stream processing patterns on Kafka: joins, co-partitioning, windows, late data.
**Answer.** **Co-partitioning** is required for stream-stream and stream-table joins: same key, same partition count, same partitioner, so matching keys land in the same task. **KStream–KTable** joins enrich events with the latest table state; **KStream–GlobalKTable** joins avoid co-partitioning (full table replicated per instance, so only for small tables); **windowed joins** need time bounds. Windows: tumbling, hopping, sliding, session; grace periods handle late events; event-time vs processing-time. Repartition topics are created when keys change (`selectKey`, `groupBy` on a new key).

**Principal signal.** Repartitioning is a hidden cost: extra topic, extra latency, extra storage.

**Interview example.** "Joining `clicks` (key `user_id`, 12 partitions) to `profiles` (key `user_id`, 8 partitions) fails co-partitioning. Fix: recreate `profiles` with 12 partitions (or repartition through a topic) with the same partitioner, or make `profiles` a GlobalKTable if it's small (say 50 MB). For late clicks arriving up to 5 minutes after the window closes, I set a 5-minute grace period."

**Follow-up.** "Why can't GlobalKTable be used for 500 GB of reference data?" (Every instance holds a full copy.)

## D. Security, multi-tenancy, DR, cost

### K21. How do you secure a Kafka cluster?
**Answer.** **Encryption in transit** (TLS, brokers and clients, inter-broker). **Authentication**: mTLS, SASL/SCRAM, SASL/OAUTHBEARER (OIDC), SASL/GSSAPI (Kerberos), SASL/PLAIN only over TLS. **Authorization**: ACLs (`kafka-acls.sh`: principal, resource, operation, allow/deny) or RBAC (Confluent). **Encryption at rest** is delegated to disk/volume encryption; **field-level/client-side encryption** for sensitive payloads. Quotas for abuse protection, network segmentation, audit logs, secrets management, and principle of least privilege (separate producers/consumers per service).

**Principal signal.** Treat topics like APIs: ownership, access request workflow, PII classification, and retention aligned with compliance (GDPR deletion in an immutable log requires design: compaction + tombstones, crypto-shredding, or short retention).

**Interview example.** "Each service has its own SCRAM user or OAuth client; ACL: `User:svc-orders` may `Write` to `orders` and `Describe`; `User:svc-fraud` may `Read` `orders` using group prefix `fraud-`. PII fields (email, PAN) are encrypted client-side with per-customer keys so deleting the key crypto-shreds the data across all retained copies."

**Follow-up.** "How do you rotate credentials without downtime?" (Multiple credentials per principal / overlapping validity, rolling restarts.)

### K22. Multi-tenancy, quotas, and topic governance.
**Answer.** Quotas: `producer_byte_rate`, `consumer_byte_rate`, `request_percentage`, `controller_mutation_rate`, set per user/client-id; throttling is applied by delaying responses. Governance: naming convention (`<domain>.<entity>.<event>.v1`), topic creation via self-service/GitOps rather than auto-create (`auto.create.topics.enable=false`), policy checks (partition max, RF, retention caps; `create.topic.policy.class.name`), ownership metadata, schema compatibility policy, cost chargeback.

**Principal signal.** Noisy-neighbor isolation is the main multi-tenant risk. Decide between *shared big cluster with quotas* and *cluster per domain/criticality tier*.

**Interview example.** "A batch replay saturated the network and hurt real-time consumers. We set `consumer_byte_rate=50 MB/s` for the analytics user, put tier-1 topics (payments) on a separate cluster from tier-3 (debug/telemetry), and enforce via Terraform that every topic declares an owner, data classification, and retention."

**Follow-up.** "Why disable auto topic creation?" (Typos create junk topics with default configs and no governance.)

### K23. Multi-cluster and DR: MirrorMaker 2, Cluster Linking, active-active.
**Answer.** Options: **MirrorMaker 2** (Connect-based replication with topic renaming `source.topic`, offset sync/checkpoints for consumer-group translation, config/ACL sync), **Confluent Cluster Linking** (byte-for-byte mirror topics, preserves offsets), **stretch cluster** across AZs (synchronous; needs low-latency links, ok across AZs, risky across regions), and application-level dual-write (avoid). Patterns: active-passive (failover; offset translation, RPO > 0 since replication is async), active-active (conflict handling, loop prevention via topic prefixing), aggregate/hub-and-spoke.

**Principal signal.** State the RPO/RTO honestly: async replication means potential data loss at failover, and consumers resuming at translated offsets may reprocess or skip, so idempotent consumers are needed.

**Interview example.** "Primary in region A, standby in region B via MM2, RPO ≈ replication lag (seconds), RTO ≈ minutes after DNS/bootstrap switch. On failover, consumer groups resume from MM2-translated checkpoints, so downstream writes are idempotent (upserts) to tolerate a short replay. We drill the failover quarterly."

**Follow-up.** "Why not one cluster stretched across two regions?" (Cross-region latency hurts `acks=all`; quorum/availability trade-offs.)

### K24. Capacity planning: how do you size a cluster?
**Answer.** Inputs: peak ingress MB/s, fan-out (consumer groups), retention, RF, message size/compression ratio, growth. Formulas: **storage** = ingress/s × retention seconds × RF ÷ compression (+ 20–30% headroom, keep disks ≤ ~60–70% full); **replication network** = ingress × (RF−1); **egress** = ingress × consumer groups (+ replication); **broker count** from the binding constraint (disk, network, CPU, partitions per broker). Keep partitions per broker bounded (guidance historically a few thousand; KRaft raises limits), plan for broker loss (N+1/N+2).

**Principal signal.** Size for failure: after losing a broker (or an AZ), the remaining ones must still carry traffic and re-replicate.

**Interview example.** "Ingress 100 MB/s, retention 3 days, RF=3, compression 3× → 100 × 259,200 s × 3 ÷ 3 ≈ 26 TB, with 30% headroom ≈ 34 TB across brokers. Fan-out of 4 consumer groups → egress 400 MB/s plus replication egress 200 MB/s ≈ 600 MB/s total outbound. At ~100 MB/s sustainable per broker NIC share, I'd want ≥ 9-10 brokers including failure headroom."

**Follow-up.** "What breaks first as you scale?" (Often network or disk throughput, then partitions per broker/controller metadata.)

### K25. How do you tune producers, consumers, and brokers for throughput vs latency?
**Answer.** **Producer:** `linger.ms`, `batch.size`, `compression.type`, `buffer.memory`, `acks`, `max.in.flight`; avoid per-record `flush()`. **Consumer:** `fetch.min.bytes`, `fetch.max.wait.ms`, `max.partition.fetch.bytes`, `max.poll.records`, number of consumers, process in batches, decouple polling from slow work (pause/resume). **Broker:** `num.network.threads`, `num.io.threads`, `num.replica.fetchers`, JBOD/disk layout, page cache, `log.flush` (leave to OS), OS limits (file descriptors, `vm.max_map_count`), JVM (G1, heap ~6 GB), rack-aware replicas, preferred leader balance.

**Principal signal.** Always state the objective: throughput, latency, or durability, because tuning for one shifts the others. Measure with a realistic load test (`kafka-producer-perf-test.sh`).

**Interview example.** "To cut p99 produce latency for a payments topic I'd set `linger.ms=0–5`, `acks=all`, no oversized batches, and keep consumers' `fetch.max.wait.ms` low. For a log-shipping topic I'd go the other way: `linger.ms=50`, `batch.size=256KB`, `zstd`, and `fetch.min.bytes=1MB`. Same cluster, different client profiles per tier."

**Follow-up.** "Should you set `log.flush.interval.messages` low for durability?" (No; rely on replication, not fsync per message.)

## E. Operations, failure modes, ecosystem decisions

### K26. Walk through failure scenarios: broker failure, ISR shrink, network partition, disk full.
**Answer.** **Broker crash:** controller reassigns leadership for its led partitions to ISR members (seconds); clients refresh metadata; partitions with a single ISR member become unavailable if min.ISR unmet. **ISR shrink:** a follower falls behind (slow disk, GC, network); `UnderReplicatedPartitions` > 0; with `acks=all` and ISR < `min.insync.replicas`, producers get `NotEnoughReplicas`. **Network partition:** brokers/clients on the minority side lose controller/quorum contact; KRaft quorum majority continues; fenced brokers stop leading. **Disk full:** broker log dir goes offline (JBOD) or the broker shuts down; fix capacity, delete/shorten retention, and reassign. **Offline partitions:** no leader; clients can't read/write.

**Principal signal.** Have runbooks and have rehearsed them; know which metric fires first for each failure.

**Interview example.** "One AZ's network flapped. `UnderReplicatedPartitions` rose, `IsrShrinksPerSec` spiked, and producers with `acks=all` and min.ISR=2 kept working because RF=3 spread across 3 AZs still had 2 healthy replicas per partition. Had we been RF=2 across 2 AZs, writes would have blocked or required relaxing durability."

**Follow-up.** "What is preferred leader election and why run it?" (Restore leader balance after failures to avoid hot brokers.)

### K27. What do you monitor? Which alerts matter most?
**Answer.** Cluster health: `OfflinePartitionsCount` (must be 0), `UnderReplicatedPartitions`, `UnderMinIsrPartitionCount`, `ActiveControllerCount` (exactly 1), ISR shrink/expand rate, request latency (p99 produce/fetch), request handler idle %, network processor idle %, disk usage and I/O wait, JVM GC, `LeaderCount` balance. Client/pipeline health: consumer lag (offset and time), rebalance rate, produce error/retry rate, DLQ rates, end-to-end latency via heartbeat/canary events, connector task status. Tools: JMX → Prometheus/Grafana, Burrow/kafka-lag-exporter/Cruise Control, Confluent Control Center, Conduktor/AKHQ.

**Principal signal.** Alert on symptoms tied to SLOs (freshness, loss risk), not on every metric. Add an **end-to-end canary** producing and consuming a synthetic record each minute.

**Interview example.** "Pages: `OfflinePartitionsCount > 0` (immediate), `UnderMinIsrPartitionCount > 0` for 5 min, time-lag SLO breach for tier-1 consumer groups. Tickets: disk >70%, lag trending up, rebalances >3/hour. Dashboards show per-topic bytes in/out and partition skew."

**Follow-up.** "Why alert on `ActiveControllerCount`?" (0 or 2 indicates a controller/quorum problem.)

### K28. Operating Kafka: rolling upgrades, partition reassignment, rebalancing storage.
**Answer.** **Rolling upgrade:** upgrade brokers one at a time, wait for ISR to recover before the next (`UnderReplicatedPartitions=0`), keep inter-broker protocol/metadata version pinned until all nodes are upgraded, then bump it; for KRaft, follow the `metadata.version` finalization procedure. **Reassignment:** `kafka-reassign-partitions.sh` (`--generate`, `--execute`, `--verify`) with throttles (`--throttle`) so replication traffic doesn't starve production; **Cruise Control** automates balance. **Adding brokers** doesn't move data automatically; you must reassign. Decommission by moving partitions off first.

**Principal signal.** Always throttle and remember to remove throttles after reassignment completes.

**Interview example.** "After adding 3 brokers to a 6-broker cluster, the new nodes were idle. We generated a reassignment plan, executed it with a 100 MB/s throttle over a weekend, and watched `UnderReplicatedPartitions` and p99 latency. Cruise Control now continuously rebalances by disk, network and leader count."

**Follow-up.** "Why can a reassignment hurt latency?" (Replica fetch traffic competes for disk/network and page cache.)

### K29. Tiered storage and cost optimization.
**Answer.** **Tiered storage** (KIP-405; production-ready from Kafka 3.9) keeps recent "hot" data on local disks and offloads closed segments to object storage (S3/GCS/Azure) via a remote storage manager plugin. Retention can then be long (weeks/months) at low cost, and brokers can be sized for throughput rather than storage. Other cost levers: compression (zstd), right-sizing partitions and RF per tier, shorter retention for derived topics, avoiding cross-AZ traffic (rack-aware **follower fetching**, `client.rack`, cuts consumer cross-AZ egress), consolidating tiny topics, compaction for state topics, using managed services where operations cost dominates.

**Principal signal.** Cross-AZ network charges are often the largest hidden cost on cloud Kafka. Quantify before optimizing.

**Interview example.** "A cluster retained 14 days on gp3 SSDs for replay. Moving to tiered storage with 1 day local and 14 days in S3 cut broker storage roughly 90% for that topic set, with slower reads only for deep replays. Enabling follower fetching (`replica.selector.class=RackAwareReplicaSelector` and `client.rack`) kept consumer traffic inside the AZ, so cross-AZ transfer charges dropped."

**Follow-up.** "What are the limitations of tiered storage?" (Not for compacted topics; feature-support and plugin maturity vary; deep reads have higher latency.)

### K30. Principal-level: Kafka vs alternatives, and how you'd build an event-streaming platform for an organization.
**Answer (alternatives).** **Pulsar** (separate compute/storage via BookKeeper, multi-tenancy built in, more moving parts), **Kinesis** (managed AWS, shard limits, simple but less ecosystem), **Google Pub/Sub** (serverless, global, no partition ordering model like Kafka's, replay features differ), **RabbitMQ** (broker with rich routing, per-message acks, not a log for replay at scale), **Redpanda** (Kafka-API-compatible, C++, simpler ops claims), **Azure Event Hubs** (Kafka-compatible endpoint). Don't use Kafka for low-volume request/response, per-message TTL/priority queues, or when the team can't operate it and a managed queue suffices.

**Answer (platform).** (1) **Define tiers and SLOs** (criticality, retention, RPO/RTO). (2) **Paved road**: self-service topics via GitOps, schema registry with compatibility policy, standard client libraries/templates (retry, DLQ, metrics, tracing), Connect/Debezium patterns. (3) **Governance**: ownership, data classification, access workflow, naming, quotas, cost chargeback. (4) **Reliability**: multi-AZ RF=3, runbooks, game days, DR with defined RPO/RTO. (5) **Observability**: lag/time-lag SLOs, lineage, end-to-end canaries. (6) **Evolution**: contracts, deprecation policy, upgrade cadence. (7) **Build vs buy**: managed (MSK, Confluent Cloud, Aiven) vs self-run, based on team size and compliance.

**Interview example.** "At an organization with 40 producing teams, we found 600 topics created ad hoc with inconsistent retention and no owners. I introduced a topic-as-code repo (owner, tier, retention, schema subject, PII flag) with CI policy checks; platform defaults gave teams a 10-minute path to a compliant topic. Within two quarters, unowned topics fell to near zero and partition count dropped ~30% after cleanup. (Use your own real numbers.)"

**Common mistakes (whole topic):** treating Kafka as a database without understanding retention and ordering limits; too many partitions "just in case"; `acks=1` on critical topics; auto-commit with batching; not planning schema evolution; no DLQ strategy; ignoring consumer lag until the business notices; CDC without monitoring replication slots; no DR drills.

---
**Next:** Part 2 has 30 coding and configuration practicals.
