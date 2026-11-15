# End-to-End Incremental ETL: Principal-Level Coding Challenge

## 1. Interview prompt

Build an incremental ETL job that:
- Reads source records changed since a watermark.
- Validates required fields.
- Deduplicates by business key and update version.
- Transforms the records.
- Upserts them into a destination.
- Advances the checkpoint safely.
- Supports retries, observability, and restart after failure.

The goal is to demonstrate correctness and operational design, not to write an entire production platform in one function.

## 2. Contracts and assumptions

Assume each source record has `id`, `updated_at`, and `amount`. The destination supports idempotent upsert by `id`. The checkpoint store can persist a watermark. For a real system, checkpoint and sink commit should be atomic or the sink must be replay-safe.

## 3. Reference implementation skeleton

```python
from decimal import Decimal, InvalidOperation

def validate_record(record):
    required = ("id", "updated_at", "amount")
    if any(record.get(field) is None for field in required):
        raise ValueError("missing required field")

    try:
        amount = Decimal(str(record["amount"]))
    except (InvalidOperation, TypeError):
        raise ValueError("invalid amount")

    return {
        "id": str(record["id"]),
        "updated_at": record["updated_at"],
        "amount": amount,
    }

def choose_latest(records):
    latest = {}
    for record in records:
        key = record["id"]
        current = latest.get(key)
        if current is None or record["updated_at"] > current["updated_at"]:
            latest[key] = record
    return list(latest.values())

def run_incremental_etl(
    source,
    sink,
    checkpoint_store,
    overlap_seconds=300,
):
    previous_watermark = checkpoint_store.read()
    query_from = previous_watermark - overlap_seconds

    raw_records = source.read_updated_since(query_from)
    valid_records = []
    rejected = 0

    for record in raw_records:
        try:
            valid_records.append(validate_record(record))
        except ValueError:
            rejected += 1

    deduplicated = choose_latest(valid_records)
    sink.upsert_many(deduplicated)

    # Advance only after durable sink write.
    new_watermark = max(
        (record["updated_at"] for record in valid_records),
        default=previous_watermark,
    )
    checkpoint_store.write(new_watermark)

    return {
        "read": len(raw_records),
        "valid": len(valid_records),
        "deduplicated": len(deduplicated),
        "rejected": rejected,
        "watermark": new_watermark,
    }
```

## 4. Important limitations of the skeleton

- `raw_records` and intermediate lists are in-memory; production code should stream or batch.
- Timestamp comparison must use a normalized comparable type (for example, timezone-aware UTC datetimes).
- `choose_latest` needs a deterministic tie-breaker when timestamps match.
- A source that supports deletes needs tombstones or change-data-capture delete events.
- If the sink commits and checkpoint write fails, the next run replays records; the sink must be idempotent.
- If the checkpoint advances before the sink commit, data can be lost.
- If the source is not snapshot-consistent, pagination and watermark boundaries need additional care.

## 5. Dry run

Previous watermark is 10:00. With a five-minute overlap, query from 09:55. The source returns records for IDs A, B, and A again. Validation rejects one malformed record. Deduplication retains the newest A and B. The sink upserts them. Only after successful sink persistence does the checkpoint advance to the maximum successfully processed update timestamp.

## 6. Complexity

For N fetched records and U unique IDs:
- Validation: O(N).
- Deduplication: average O(N) time and O(U) memory.
- Sink upsert: depends on destination/indexing and batch size.
- A streaming implementation can bound application memory by batch size B, but global latest-record selection may require a keyed state store or SQL/window operation.

## 7. Production architecture

A robust implementation typically has these stages:

1. **Extract:** bounded page/batch size, source-side filter, retry and cursor tracking.
2. **Validate:** schema and semantic rules; quarantine invalid records with reason.
3. **Transform:** deterministic, versioned transformation logic.
4. **Deduplicate:** stable identity and ordering contract.
5. **Load:** idempotent merge/upsert; partition-aware writes.
6. **Checkpoint:** advance only after durable output.
7. **Reconcile:** compare counts, checksums, and source/sink watermarks.
8. **Observe:** run ID, input/output/reject counts, lag, duration, retries, and error categories.

## 8. Interviewer–candidate discussion

**Interviewer:** What if the sink write succeeds but the checkpoint update fails?

**Candidate:** The next run re-reads an overlapping range. Idempotent upsert makes replay safe. If both stores support a shared transaction, use it; otherwise design explicitly for at-least-once execution.

**Interviewer:** What if records arrive late with an old `updated_at`?

**Candidate:** Use an overlap window based on the source's lateness contract, a monotonic change sequence where available, or CDC offsets. Periodic reconciliation is needed if lateness is unbounded.

**Interviewer:** How would you scale this to billions of rows?

**Candidate:** Push filtering and deduplication into the source/compute engine, partition by stable keys, use distributed processing such as Spark, batch sink writes, and monitor skew and small-file behavior. The control-plane invariants—idempotency, checkpoint ordering, and observability—remain the same.

## 9. Test plan

- Empty incremental range.
- Duplicate ID with newer timestamp.
- Equal timestamps with deterministic tie-breaker.
- Invalid required field and invalid decimal.
- Sink failure before checkpoint.
- Checkpoint failure after sink success.
- Late-arriving record within overlap.
- Late-arriving record outside overlap.
- Delete/tombstone.
- Timezone and timestamp precision mismatch.
- Re-run same input and verify no duplicate business effect.

## 10. Follow-up questions

1. How would you implement this using Kafka offsets and a database sink?
2. How would you handle schema evolution and backward compatibility?
3. What metrics and alerts define pipeline health?
4. How do you design backfills without corrupting the live watermark?
5. How would you test failure injection and recovery?
