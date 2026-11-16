# Kafka for Principal Data Engineer Interviews — Part 2: Practicals (P1–P30)

**How to use:** Write each solution yourself first (10–15 min), then compare. Each practical has **why it matters**, **edge cases**, and a **Principal add**. Python examples use `confluent-kafka` (librdkafka); a few are Java (Kafka Streams), CLI, or JSON connector configs, because interviews mix all of these. Verify option names against the docs for your client/broker version.

Common imports and config:
```python
import json, logging, signal, time
from confluent_kafka import Producer, Consumer, KafkaError, KafkaException, TopicPartition
BOOT = "localhost:9092"
log = logging.getLogger("kafka-prep")
```

---

## A. CLI essentials and topic management

### P1. Create a topic, produce and consume from the CLI, and inspect a consumer group.
```bash
# create (RF=3, min.ISR=2, 7-day retention)
kafka-topics.sh --bootstrap-server b1:9092 --create --topic orders \
  --partitions 12 --replication-factor 3 \
  --config min.insync.replicas=2 --config retention.ms=604800000

kafka-topics.sh --bootstrap-server b1:9092 --describe --topic orders      # leaders, replicas, ISR

# produce keyed messages (key:value)
kafka-console-producer.sh --bootstrap-server b1:9092 --topic orders \
  --property parse.key=true --property key.separator=:

# consume from start, print keys/partitions/offsets
kafka-console-consumer.sh --bootstrap-server b1:9092 --topic orders --from-beginning \
  --property print.key=true --property print.partition=true --property print.offset=true

# lag per partition for a group
kafka-consumer-groups.sh --bootstrap-server b1:9092 --describe --group fraud-svc
```
**Why:** first-line debugging. In `--describe`, check `CURRENT-OFFSET`, `LOG-END-OFFSET`, `LAG`, and `CONSUMER-ID` (blank = partition unassigned). **Principal add:** never use `kafka-console-consumer` with a production `group.id` accidentally (it can trigger rebalances); use a unique throwaway group.

### P2. Write a Python producer with delivery callbacks, back-pressure handling, and a clean flush.
```python
def on_delivery(err, msg):
    if err is not None:
        log.error("delivery failed key=%s: %s", msg.key(), err)      # route to retry/alert
    else:
        log.debug("ok %s[%d]@%d", msg.topic(), msg.partition(), msg.offset())

p = Producer({"bootstrap.servers": BOOT, "client.id": "orders-svc"})

def send(topic, key, value):
    while True:
        try:
            p.produce(topic, key=key, value=value, on_delivery=on_delivery)
            break
        except BufferError:               # local queue full: apply backpressure
            p.poll(0.5)                    # serve callbacks, freeing space
    p.poll(0)                              # non-blocking; triggers callbacks

for i in range(1000):
    send("orders", f"c{i % 50}".encode(), json.dumps({"id": i}).encode())
p.flush(30)                                # block until all outstanding messages are delivered/failed
```
`produce()` is asynchronous: success/failure arrives only via the callback. **Edge cases:** skipping `flush()` at shutdown loses buffered messages; `poll()` must be called regularly or callbacks never run. **Principal add:** count failures in the callback and fail the job if any message is undeliverable.

### P3. Configure a producer for strong durability and ordering.
```python
p = Producer({
  "bootstrap.servers": BOOT,
  "enable.idempotence": True,                 # PID + sequence numbers: no duplicates on retry
  "acks": "all",
  "max.in.flight.requests.per.connection": 5, # ordering preserved with idempotence
  "delivery.timeout.ms": 120000,
  "compression.type": "zstd",
  "linger.ms": 10,
})
# topic side: replication.factor=3, min.insync.replicas=2, unclean.leader.election.enable=false
```
Explain the pairing: `acks=all` is only as strong as `min.insync.replicas`. **Edge cases:** idempotence needs `acks=all`; fatal errors (e.g. fenced/auth) require recreating the producer. **Principal add:** durability configs are enforced via topic templates (platform), not left to each team.

### P4. Control partitioning: key hashing and cross-client compatibility.
```python
# librdkafka default is consistent_random (CRC32), which differs from Java's murmur2.
# To match Java producers (so the same key maps to the same partition), set:
p = Producer({"bootstrap.servers": BOOT, "partitioner": "murmur2_random"})

# Or explicit routing with a stable hash (never Python's hash(): it is randomized per process)
import zlib
n = len(p.list_topics("orders", timeout=10).topics["orders"].partitions)
def part_for(key: bytes) -> int: return zlib.crc32(key) % n
p.produce("orders", key=b"c42", value=b"...", partition=part_for(b"c42"))
```
**Why:** mixed Python/Java producers writing the same keyed topic silently break per-key ordering if their partitioners differ. **Edge case:** adding partitions changes the mapping. **Principal add:** document the partitioner as part of the topic contract.

### P5. Tune a producer for throughput and verify with the perf tool.
```bash
kafka-producer-perf-test.sh --topic perf --num-records 5000000 --record-size 200 \
  --throughput -1 --producer-props bootstrap.servers=b1:9092 \
  linger.ms=20 batch.size=131072 compression.type=zstd acks=all
```
Compare runs varying one setting at a time (`linger.ms` 0/5/20/50; `batch.size` 16K/128K/512K; `lz4` vs `zstd`). Report records/s, MB/s, avg and p99 latency. **Principal add:** use realistic payloads (compressibility affects results), measure end-to-end including consumers, and set per-tier client profiles (latency-tier vs throughput-tier).

## B. Consumers

### P6. Write an at-least-once consumer loop with manual commits and graceful shutdown.
```python
c = Consumer({
  "bootstrap.servers": BOOT, "group.id": "fraud-svc",
  "enable.auto.commit": False, "auto.offset.reset": "earliest",
  "partition.assignment.strategy": "cooperative-sticky",
  "session.timeout.ms": 45000, "max.poll.interval.ms": 300000,
})
running = True
signal.signal(signal.SIGTERM, lambda *_: globals().update(running=False))
c.subscribe(["orders"])
try:
    while running:
        msg = c.poll(1.0)
        if msg is None: continue
        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF: continue
            raise KafkaException(msg.error())
        process(msg)                               # side effect first...
        c.commit(message=msg, asynchronous=False)  # ...then commit offset+1 (at-least-once)
finally:
    c.close()                                      # leaves the group cleanly, commits per config
```
Committing per message is safe but slow; see P8 for batching. **Principal add:** `c.close()` triggers a prompt rebalance instead of waiting for `session.timeout.ms`.

### P7. Use rebalance callbacks to commit and clean up state safely.
```python
state = {}                                   # per-partition in-memory state

def on_revoke(consumer, partitions):
    flush_state(partitions)                  # persist what you owe downstream
    consumer.commit(asynchronous=False)      # commit processed offsets before losing ownership
    for tp in partitions: state.pop((tp.topic, tp.partition), None)

def on_assign(consumer, partitions):
    log.info("assigned %s", [(tp.topic, tp.partition) for tp in partitions])

c.subscribe(["orders"], on_assign=on_assign, on_revoke=on_revoke)
```
With cooperative-sticky, only the moved partitions are revoked. **Edge cases:** callbacks run inside `poll()`, so keep them fast; offsets committed in `on_revoke` prevent reprocessing. **Principal add:** state tied to partitions (aggregations, caches) must be rebuilt or persisted on reassignment.

### P8. Consume in batches, bulk-write to a sink, then commit.
```python
BATCH, MAX_WAIT = 500, 2.0
buf = []
while running:
    msgs = c.consume(num_messages=BATCH, timeout=MAX_WAIT)   # returns up to BATCH messages
    for m in msgs:
        if m.error(): raise KafkaException(m.error())
    if not msgs: continue
    rows = [json.loads(m.value()) for m in msgs]
    sink.bulk_upsert(rows)                                   # idempotent bulk write
    c.commit(asynchronous=False)                             # commits current positions of assigned partitions
```
Batching cuts commit and DB round-trips by ~100×. Keep batch processing time well below `max.poll.interval.ms`. **Principal add:** write batches idempotently (`INSERT ... ON CONFLICT DO UPDATE`), because a crash between write and commit replays the batch.

### P9. Make a consumer idempotent with a dedup key (exactly-once effect).
```python
def handle(msg):
    event = json.loads(msg.value())
    with db.transaction() as tx:
        inserted = tx.execute(
            "INSERT INTO processed_events(event_id) VALUES (%s) ON CONFLICT DO NOTHING",
            (event["event_id"],)).rowcount
        if inserted:                                   # first time we see it
            tx.execute("UPDATE balances SET amt = amt + %s WHERE acct = %s",
                       (event["amount"], event["acct"]))
    # commit Kafka offset after the DB transaction
```
Alternative: store the Kafka offset in the *same* DB transaction as the effect (`consumer_offsets(topic, partition, next_offset)`) and `seek()` to it on assign, giving exactly-once effect without Kafka transactions. **Edge cases:** dedup table growth (TTL/partition by day), event IDs must be stable across producer retries. **Principal add:** state the principle: *at-least-once delivery + idempotent processing = exactly-once effect*.

### P10. Implement a transactional consume-transform-produce loop (Kafka EOS).
```python
producer = Producer({"bootstrap.servers": BOOT, "transactional.id": "enricher-0"})   # stable per instance
producer.init_transactions()
consumer = Consumer({"bootstrap.servers": BOOT, "group.id": "enricher",
                     "enable.auto.commit": False, "isolation.level": "read_committed",
                     "auto.offset.reset": "earliest"})
consumer.subscribe(["payments-raw"])

def rewind():                                       # after abort, re-read the uncommitted range
    for tp in consumer.committed(consumer.assignment(), timeout=10):
        consumer.seek(tp) if tp.offset >= 0 else consumer.seek(TopicPartition(tp.topic, tp.partition, 0))

while True:
    msgs = consumer.consume(num_messages=100, timeout=1.0)
    if not msgs: continue
    producer.begin_transaction()
    try:
        for m in msgs:
            if m.error(): raise KafkaException(m.error())
            producer.produce("payments-enriched", key=m.key(), value=enrich(m.value()))
        producer.send_offsets_to_transaction(
            consumer.position(consumer.assignment()), consumer.consumer_group_metadata())
        producer.commit_transaction()
    except Exception:
        producer.abort_transaction()
        rewind()
```
Output records and input offsets commit atomically. **Edge cases:** `transactional.id` must be unique per producer instance and stable across restarts (zombie fencing); downstream readers need `isolation.level=read_committed`; a long-open transaction blocks read_committed consumers at the LSO. **Principal add:** this is EOS *within Kafka* only: sinks outside Kafka still need idempotent writes.

### P11. Build retry topics with a dead-letter queue.
```python
RETRY = ["orders.retry.1", "orders.retry.2"]; DLQ = "orders.dlq"; MAX = len(RETRY)

def fail(msg, err):
    attempt = int(dict(msg.headers() or []).get("attempt", b"0")) + 1
    target = RETRY[attempt - 1] if attempt <= MAX else DLQ
    headers = [("attempt", str(attempt).encode()), ("error", str(err)[:500].encode()),
               ("orig", f"{msg.topic()}:{msg.partition()}:{msg.offset()}".encode()),
               ("not_before", str(int(time.time()) + 30 * attempt).encode())]   # delay metadata
    producer.produce(target, key=msg.key(), value=msg.value(), headers=headers)
    producer.flush()                                                              # then commit the original offset

# retry-topic consumer: if now < not_before -> pause partition until due, then reprocess
```
**Why:** one poison message must not block a partition (Kafka has no per-message nack). Retry topics keep the main topic flowing; the delay topics avoid hot-loop retries. **Edge cases:** ordering across retries is lost; separate *transient* (retry) from *permanent* (DLQ immediately) errors. **Principal add:** DLQ messages carry error context headers, are monitored with alerts, and have a replay tool.

### P12. Handle poison pills and deserialization failures without crashing the consumer.
```python
def safe_decode(msg):
    try:
        return json.loads(msg.value())
    except (ValueError, TypeError) as e:
        producer.produce("orders.dlq", key=msg.key(), value=msg.value(),
                         headers=[("error", f"decode: {e}".encode()),
                                  ("orig", f"{msg.topic()}:{msg.partition()}:{msg.offset()}".encode())])
        metrics.inc("dlq_decode_errors")
        return None

event = safe_decode(msg)
if event is not None: process(event)
c.commit(message=msg, asynchronous=False)      # commit past the bad record either way
```
Deserialize *after* `poll()` in your own code (not inside the client's deserializer) so a bad record can't throw out of the poll loop and leave you stuck re-reading it. **Principal add:** alert when the DLQ rate crosses a threshold; a spike usually means a producer or schema change, not random corruption.

## C. Schemas and replay

### P13. Produce and consume Avro with Schema Registry.
```python
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer, AvroDeserializer
from confluent_kafka.serialization import SerializationContext, MessageField, StringSerializer

SCHEMA = """{"type":"record","name":"Order","namespace":"com.acme","fields":[
  {"name":"id","type":"string"},
  {"name":"amount","type":"double"},
  {"name":"coupon","type":["null","string"],"default":null}]}"""

sr = SchemaRegistryClient({"url": "http://sr:8081"})
ser = AvroSerializer(sr, SCHEMA, lambda obj, ctx: obj)       # registers/looks up the schema, prepends magic byte + ID
key_ser = StringSerializer("utf_8")
p.produce("orders", key=key_ser("o-1"),
          value=ser({"id": "o-1", "amount": 12.5, "coupon": None},
                    SerializationContext("orders", MessageField.VALUE)))

deser = AvroDeserializer(sr, from_dict=lambda d, ctx: d)      # fetches writer schema by ID
order = deser(msg.value(), SerializationContext(msg.topic(), MessageField.VALUE))
```
Wire format: `0x00 | 4-byte schema ID | Avro payload`. **Edge cases:** disable auto-registration in production (`auto.register.schemas=False`) so schemas are registered through CI, not by whichever producer runs first. **Principal add:** subject naming strategy (`TopicNameStrategy` vs `RecordNameStrategy`) matters when a topic holds multiple event types.

### P14. Check and enforce schema compatibility before deploy.
```bash
# set policy per subject
curl -s -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility":"BACKWARD_TRANSITIVE"}' http://sr:8081/config/orders-value

# test a candidate schema against the latest registered version
curl -s -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema":"{\"type\":\"record\",\"name\":\"Order\",\"namespace\":\"com.acme\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"coupon\",\"type\":[\"null\",\"string\"],\"default\":null},{\"name\":\"channel\",\"type\":[\"null\",\"string\"],\"default\":null}]}"}' \
  http://sr:8081/compatibility/subjects/orders-value/versions/latest
# -> {"is_compatible": true}
```
Adding an optional field with a default is backward compatible; removing a required field without default or changing a type is not. **Principal add:** run this check in the producer team's CI pipeline so incompatible schemas fail the build, not production consumers.

### P15. Replay a topic from a point in time.
```python
from datetime import datetime, timezone
ts_ms = int(datetime(2026, 11, 15, 0, 0, tzinfo=timezone.utc).timestamp() * 1000)
md = c.list_topics("orders", timeout=10).topics["orders"]
query = [TopicPartition("orders", p, ts_ms) for p in md.partitions]     # offset field holds the timestamp here
start = c.offsets_for_times(query, timeout=10)                          # earliest offset with timestamp >= ts
c.assign(start)                                                         # manual assignment: no group rebalancing
# CLI equivalent for an existing group (stop the consumers first):
# kafka-consumer-groups.sh --bootstrap-server b1:9092 --group fraud-svc --topic orders \
#   --reset-offsets --to-datetime 2026-11-15T00:00:00.000 --execute
```
If no record exists at/after the timestamp for a partition, the returned offset is end (-1). **Principal add:** replay consumers must be idempotent and rate-limited (quotas) so they don't starve live consumers; consider replaying into a separate topic/cluster.

### P16. Compute consumer lag programmatically (offset lag and time lag).
```python
def lag_report(group, topic):
    c = Consumer({"bootstrap.servers": BOOT, "group.id": group, "enable.auto.commit": False})
    parts = [TopicPartition(topic, p) for p in c.list_topics(topic, timeout=10).topics[topic].partitions]
    out = {}
    for tp in c.committed(parts, timeout=10):
        lo, hi = c.get_watermark_offsets(tp, timeout=10)
        cur = tp.offset if tp.offset >= 0 else lo
        out[tp.partition] = {"lag": hi - cur}
    c.close()
    return out
# time lag: read the record at the committed offset and compare its timestamp with now()
```
Offset lag alone can mislead (10M messages may be 2 seconds of data on a hot topic, or 1,000 messages an hour of data on a slow topic). **Principal add:** export both offset and time lag; alert on time lag against the SLA.

## D. Administration

### P17. Manage topics with the AdminClient (create, inspect, add partitions).
```python
from confluent_kafka.admin import AdminClient, NewTopic, NewPartitions, ConfigResource

a = AdminClient({"bootstrap.servers": BOOT})
futs = a.create_topics([NewTopic("orders", num_partitions=12, replication_factor=3,
        config={"min.insync.replicas": "2", "retention.ms": str(7 * 86400 * 1000),
                "compression.type": "producer"})])
for t, f in futs.items():
    try: f.result(); print("created", t)
    except Exception as e: print("failed", t, e)

a.create_partitions([NewPartitions("orders", 24)])        # can only increase; remaps keys
cfg = a.describe_configs([ConfigResource(ConfigResource.Type.TOPIC, "orders")])
for r, f in cfg.items(): print({k: v.value for k, v in f.result().items()})
```
**Principal add:** topic creation is done via GitOps with policy checks (owner, RF≥3, retention caps); broker `auto.create.topics.enable=false`. Increasing partitions on keyed/compacted topics is a migration, not a flag change.

### P18. Create a compacted topic and handle tombstones.
```bash
kafka-topics.sh --bootstrap-server b1:9092 --create --topic customer-profile --partitions 6 \
  --replication-factor 3 --config cleanup.policy=compact \
  --config min.cleanable.dirty.ratio=0.1 --config segment.ms=3600000 \
  --config delete.retention.ms=86400000
```
```python
p.produce("customer-profile", key=b"c42", value=json.dumps({"name": "Asha"}).encode())  # upsert
p.produce("customer-profile", key=b"c42", value=None)                                    # tombstone = delete

table = {}                                   # consumer materializes current state
for msg in consume_from_beginning("customer-profile"):
    if msg.value() is None: table.pop(msg.key(), None)
    else: table[msg.key()] = json.loads(msg.value())
```
Keys must never be null on compacted topics. Compaction is eventual, so readers must tolerate duplicates and old values until cleaned. **Principal add:** GDPR deletion = tombstone + wait for compaction + `delete.retention.ms`, and downstream copies (lake, caches) need their own delete path.

## E. Integration patterns

### P19. Implement the transactional outbox with a polling publisher.
```sql
CREATE TABLE outbox(id BIGSERIAL PRIMARY KEY, aggregate_id TEXT, type TEXT,
                    payload JSONB, created_at TIMESTAMPTZ DEFAULT now(), published_at TIMESTAMPTZ);
-- service code: INSERT business row AND outbox row in ONE transaction
```
```python
def publish_batch(conn, producer, limit=500):
    with conn:                                   # one DB transaction
        cur = conn.cursor()
        cur.execute("""SELECT id, aggregate_id, type, payload FROM outbox
                       WHERE published_at IS NULL ORDER BY id LIMIT %s FOR UPDATE SKIP LOCKED""", (limit,))
        rows = cur.fetchall()
        for id_, agg, typ, payload in rows:
            producer.produce(f"events.{typ}", key=agg.encode(), value=json.dumps(payload).encode(),
                             headers=[("event_id", str(id_).encode())])
        if producer.flush(30) > 0: raise RuntimeError("undelivered messages")
        cur.execute("UPDATE outbox SET published_at = now() WHERE id = ANY(%s)", ([r[0] for r in rows],))
```
Result: at-least-once publish with ordering per `ORDER BY id` (consumers dedupe by `event_id`). `SKIP LOCKED` lets multiple publishers run safely. **Principal add:** prefer log-based CDC (Debezium outbox router) at scale to avoid polling load; either way, define retention/cleanup for the outbox table.

### P20. Register a Debezium PostgreSQL connector (with outbox-friendly settings).
```json
{
  "name": "pg-orders-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "plugin.name": "pgoutput",
    "database.hostname": "pg.internal", "database.port": "5432",
    "database.user": "debezium", "database.password": "${file:/secrets/pg.properties:password}",
    "database.dbname": "shop",
    "topic.prefix": "shop",
    "table.include.list": "public.orders,public.order_items",
    "slot.name": "debezium_orders", "publication.name": "dbz_pub",
    "snapshot.mode": "initial",
    "heartbeat.interval.ms": "10000",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://sr:8081",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://sr:8081",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "transforms.unwrap.drop.tombstones": "false"
  }
}
```
```bash
curl -X POST -H "Content-Type: application/json" --data @pg-orders.json http://connect:8083/connectors
curl http://connect:8083/connectors/pg-orders-cdc/status
curl -X POST http://connect:8083/connectors/pg-orders-cdc/tasks/0/restart
```
Topics: `shop.public.orders`, etc. The `heartbeat` keeps the replication slot advancing on quiet databases. **Principal add:** monitor replication-slot lag and cap WAL retention (`max_slot_wal_keep_size`); secrets via config providers, never inline.

### P21. Configure an S3 sink connector with a dead-letter queue.
```json
{
  "name": "orders-s3-sink",
  "config": {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max": "6", "topics": "orders",
    "s3.bucket.name": "lake-raw", "s3.region": "ap-south-1",
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
    "flush.size": "100000", "rotate.interval.ms": "600000",
    "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
    "path.format": "'dt'=YYYY-MM-dd/'hr'=HH", "partition.duration.ms": "3600000",
    "locale": "en-US", "timezone": "UTC", "timestamp.extractor": "Record",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://sr:8081",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "orders.s3sink.dlq",
    "errors.deadletterqueue.topic.replication.factor": "3",
    "errors.deadletterqueue.context.headers.enable": "true",
    "errors.log.enable": "true"
  }
}
```
`flush.size` plus `rotate.interval.ms` control file size/latency (avoids small files); `tasks.max` should not exceed topic partitions. Sink connectors resume from consumer-group offsets named `connect-<name>`. **Principal add:** DLQ catches conversion/transform errors; errors at the storage layer still fail the task. Alert on both.

### P22. Apply SMTs: mask PII and route by date.
```json
"transforms": "mask,route",
"transforms.mask.type": "org.apache.kafka.connect.transforms.MaskField$Value",
"transforms.mask.fields": "card_number,ssn",
"transforms.mask.replacement": "XXXX",
"transforms.route.type": "org.apache.kafka.connect.transforms.TimestampRouter",
"transforms.route.topic.format": "${topic}-${timestamp}",
"transforms.route.timestamp.format": "yyyyMMdd"
```
Other common SMTs: `ReplaceField`, `RegexRouter`, `InsertField`, `Cast`, `Filter` with predicates. **Edge case:** SMTs run per record inside the worker, so heavy logic or lookups hurt throughput and are hard to test. **Principal add:** masking at the connector protects the lake, but the raw topic still contains the PII; classify and secure the source topic too (or encrypt fields at the producer).

### P23. Spark Structured Streaming from Kafka with parsed JSON and lineage columns.
```python
from pyspark.sql import functions as F, types as T
schema = T.StructType([T.StructField("order_id", T.StringType()), T.StructField("amount", T.DoubleType())])

raw = (spark.readStream.format("kafka")
       .option("kafka.bootstrap.servers", "b1:9092")
       .option("subscribe", "orders").option("startingOffsets", "earliest")
       .option("maxOffsetsPerTrigger", 500000).option("failOnDataLoss", "true").load())

parsed = (raw.select(F.col("key").cast("string").alias("k"),
                     F.from_json(F.col("value").cast("string"), schema).alias("v"),
                     "topic", "partition", "offset", "timestamp")
          .select("k", "v.*", "topic", "partition", "offset", F.col("timestamp").alias("kafka_ts")))

(parsed.writeStream.format("delta").option("checkpointLocation", "s3://chk/orders")
 .trigger(processingTime="30 seconds").outputMode("append").toTable("bronze.orders"))
```
Spark tracks offsets in its checkpoint, not in Kafka consumer-group commits. **Principal add:** keep `topic/partition/offset` as columns for dedup and lineage; rows where `v` is null are parse failures, so route them to a quarantine table rather than dropping.

### P24. Kafka Streams (Java): enrich with a KTable and compute windowed counts with exactly-once.
```java
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "click-enricher");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "b1:9092");
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);

StreamsBuilder b = new StreamsBuilder();
KStream<String, Click> clicks   = b.stream("clicks",   Consumed.with(Serdes.String(), clickSerde));
KTable<String, Profile> profiles = b.table("profiles", Consumed.with(Serdes.String(), profileSerde));

KStream<String, Enriched> enriched = clicks.join(profiles, Enriched::new);   // needs co-partitioned topics

enriched.groupByKey()
        .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(1)))
        .count(Materialized.as("clicks-5m"))
        .toStream((wk, v) -> wk.key())
        .to("clicks-5m-counts", Produced.with(Serdes.String(), Serdes.Long()));
```
`application.id` is the consumer group and the prefix for internal topics (repartition/changelog). **Edge cases:** the default windowed count emits *updates* continuously; use `.suppress(Suppressed.untilWindowCloses(...))` for one final result per window. **Principal add:** state is local RocksDB backed by changelog topics; standby replicas shorten failover; plan state-restore time into your recovery objectives.

## F. Concurrency, security, operations, testing

### P25. Parallelize processing while preserving per-partition ordering.
```python
import queue, threading
from collections import defaultdict

class PartitionWorker(threading.Thread):
    def __init__(self, handler, done_q):
        super().__init__(daemon=True); self.q, self.handler, self.done_q = queue.Queue(maxsize=1000), handler, done_q
    def run(self):
        while True:
            msg = self.q.get()
            if msg is None: return
            self.handler(msg)                                   # sequential within a partition
            self.done_q.put((msg.topic(), msg.partition(), msg.offset() + 1))

workers, done_q, to_commit = {}, queue.Queue(), {}
while running:
    msg = c.poll(0.2)
    if msg and not msg.error():
        key = (msg.topic(), msg.partition())
        w = workers.setdefault(key, PartitionWorker(process, done_q)); 
        if not w.is_alive(): w.start()
        w.q.put(msg)                                            # blocks if worker is slow (backpressure)
    while not done_q.empty():                                   # only the main thread touches the consumer
        t, p, nxt = done_q.get(); to_commit[(t, p)] = nxt
    if to_commit:
        c.commit(offsets=[TopicPartition(t, p, o) for (t, p), o in to_commit.items()], asynchronous=True)
        to_commit.clear()
```
Order is preserved per partition (one worker each) while partitions run in parallel. **Edge cases:** on rebalance, drain/stop workers for revoked partitions; a blocking `put` can exceed `max.poll.interval.ms`, so prefer P26's pause/resume. **Principal add:** simplest scaling is more consumer instances; threading inside a consumer is for cases where partitions are few but per-message work is slow.

### P26. Apply backpressure with `pause()` / `resume()`.
```python
HIGH, LOW = 5000, 1000
paused = False
while running:
    msg = c.poll(0.5)
    if msg and not msg.error():
        work_q.put(msg)                                    # unbounded hand-off to workers
    depth = work_q.qsize()
    if not paused and depth > HIGH:
        c.pause(c.assignment()); paused = True            # stop fetching but keep polling (stays in group)
    elif paused and depth < LOW:
        c.resume(c.assignment()); paused = False
```
You must keep calling `poll()` while paused so heartbeats and rebalances still work. Re-apply the paused state after a rebalance (pause is per assigned partition). **Principal add:** pair with metrics on queue depth and pause time; sustained pausing means the consumer group is under-provisioned.

### P27. Secure access: SASL/SCRAM over TLS, plus ACLs.
```python
secure = {
  "bootstrap.servers": "b1:9093", "security.protocol": "SASL_SSL",
  "sasl.mechanisms": "SCRAM-SHA-512",
  "sasl.username": "svc-orders", "sasl.password": os.environ["KAFKA_PASSWORD"],
  "ssl.ca.location": "/etc/ssl/kafka-ca.pem",
}
p = Producer(secure)
```
```bash
# create credential (needs admin)
kafka-configs.sh --bootstrap-server b1:9093 --command-config admin.properties --alter \
  --add-config 'SCRAM-SHA-512=[password=<generated>]' --entity-type users --entity-name svc-orders
# producer ACL
kafka-acls.sh --bootstrap-server b1:9093 --command-config admin.properties --add \
  --allow-principal User:svc-orders --operation Write --operation Describe --topic orders
# consumer ACL: read topic + group (prefixed group names for a team)
kafka-acls.sh ... --add --allow-principal User:svc-fraud --operation Read --topic orders
kafka-acls.sh ... --add --allow-principal User:svc-fraud --operation Read --group fraud- --resource-pattern-type prefixed
```
Idempotent producers need `IdempotentWrite` on the cluster (or `Write` on the topic in recent versions); transactional ones need `TransactionalId` ACLs. **Principal add:** least privilege per service, access via PR-reviewed ACL-as-code, secrets from a vault, and TLS everywhere including inter-broker.

### P28. Operate the cluster: find problems, reassign partitions, rebalance leaders.
```bash
# what's broken?
kafka-topics.sh --bootstrap-server b1:9092 --describe --under-replicated-partitions
kafka-topics.sh --bootstrap-server b1:9092 --describe --unavailable-partitions

# move partitions onto new brokers 4,5,6 with throttling
cat > topics.json <<'EOF'
{"topics":[{"topic":"orders"}],"version":1}
EOF
kafka-reassign-partitions.sh --bootstrap-server b1:9092 --topics-to-move-json-file topics.json \
  --broker-list "4,5,6" --generate                     # copy the proposed plan into plan.json
kafka-reassign-partitions.sh --bootstrap-server b1:9092 --reassignment-json-file plan.json \
  --execute --throttle 100000000                       # 100 MB/s cap for replication traffic
kafka-reassign-partitions.sh --bootstrap-server b1:9092 --reassignment-json-file plan.json --verify   # also removes throttle

# restore preferred leaders after failures
kafka-leader-election.sh --bootstrap-server b1:9092 --election-type PREFERRED --all-topic-partitions
```
**Principal add:** `--verify` clears throttles once done (forgetting leaves replication permanently capped), do large moves off-peak and watch under-replicated count; Cruise Control automates this with goals (disk/network/leader balance).

### P29. Write an integration test with a real Kafka broker (pytest + testcontainers).
```python
# pip install testcontainers confluent-kafka pytest
import pytest
from testcontainers.kafka import KafkaContainer

@pytest.fixture(scope="session")
def bootstrap():
    with KafkaContainer() as k:
        yield k.get_bootstrap_server()

def test_roundtrip_and_ordering(bootstrap):
    p = Producer({"bootstrap.servers": bootstrap, "enable.idempotence": True})
    for i in range(100): p.produce("t1", key=b"k", value=str(i).encode())
    p.flush(10)
    c = Consumer({"bootstrap.servers": bootstrap, "group.id": "t", "auto.offset.reset": "earliest"})
    c.subscribe(["t1"]); got = []
    deadline = time.time() + 20
    while len(got) < 100 and time.time() < deadline:
        m = c.poll(1.0)
        if m and not m.error(): got.append(int(m.value()))
    c.close()
    assert got == list(range(100))             # same key => same partition => ordered
```
Also test: idempotent handler on duplicate delivery, DLQ routing on a poison record, and rebalancing behavior (start a second consumer). **Principal add:** keep business logic as pure functions (unit tests, no Kafka) and use containers only for contract/integration paths; add a compatibility test against your schema registry config.

### P30. Capacity estimator: storage, network, brokers, partitions.
```python
def kafka_capacity(ingress_mb_s, retention_days, rf=3, compression=3.0,
                   consumer_groups=3, headroom=0.30, broker_net_mb_s=100,
                   broker_disk_tb=8, disk_target=0.65, per_part_consumer_mb_s=10):
    sec = retention_days * 86400
    raw_tb = ingress_mb_s * sec * rf / compression / 1e6
    storage_tb = raw_tb * (1 + headroom)
    egress = ingress_mb_s * consumer_groups + ingress_mb_s * (rf - 1)      # consumers + replication
    brokers_disk = storage_tb / (broker_disk_tb * disk_target)
    brokers_net = (ingress_mb_s * rf + egress) / (broker_net_mb_s * 2)     # in+out per broker (illustrative)
    brokers = max(brokers_disk, brokers_net) + 1                           # +1 for failure headroom
    partitions = ingress_mb_s / per_part_consumer_mb_s
    return {"storage_tb": round(storage_tb, 1), "egress_mb_s": egress,
            "brokers": int(brokers + 0.999), "min_partitions": int(partitions + 0.999)}

print(kafka_capacity(100, 3))
# ≈ {'storage_tb': 33.7, 'egress_mb_s': 500, 'brokers': 8, 'min_partitions': 10}  (approximate; inputs are assumptions)
```
State your assumptions out loud (message size, compression ratio, fan-out, peak vs average, growth) and sanity-check against failure of one broker or AZ. **Principal add:** replace guesswork with a load test on target hardware, include cross-AZ traffic cost, and review quarterly against actual growth.

---

## Scorecard: self-evaluate per practical
| Dimension | Senior | Staff | Principal |
|---|---|---|---|
| Correct, runnable solution | ✔ | ✔ | ✔ |
| Names the delivery guarantee and failure behavior | some | ✔ | ✔ |
| Idempotency, ordering, rebalancing edge cases | – | ✔ | ✔ |
| Security, schema contracts, governance | – | some | ✔ |
| Observability, testing, operability | – | some | ✔ |
| Cost, scale path, build-vs-buy, org rollout | – | – | ✔ |
