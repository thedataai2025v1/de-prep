# Memory-Efficient File Processing and External Merge Sort

## 1. Interview scenario

A daily export is larger than available RAM. You must process records, sort by a key, and produce a globally sorted output.

## 2. Streaming transformation

Avoid `read()` and `list(file)`. Process one line at a time, validate, and write output incrementally.

```python
import json

def transform_jsonl(input_path, output_path, transform):
    read_count = 0
    written_count = 0
    rejected_count = 0

    with open(input_path, "r", encoding="utf-8") as source, \
         open(output_path, "w", encoding="utf-8") as target:
        for line_number, line in enumerate(source, start=1):
            read_count += 1
            try:
                record = json.loads(line)
                result = transform(record)
                target.write(json.dumps(result, ensure_ascii=False) + "\n")
                written_count += 1
            except (json.JSONDecodeError, TypeError, ValueError):
                rejected_count += 1

    return {
        "read": read_count,
        "written": written_count,
        "rejected": rejected_count,
    }
```

## 3. External merge sort

1. Read a bounded chunk that fits in memory.
2. Sort the chunk by the required key.
3. Write each sorted chunk to a temporary file.
4. Open sorted chunks and merge them using a min-heap (`heapq.merge`).
5. Stream merged output to the destination.
6. Delete temporary files only after successful completion or controlled cleanup.

```python
import heapq
import json
import tempfile
from pathlib import Path

def external_sort_jsonl(input_path, output_path, key_fn, chunk_size=100_000):
    if chunk_size <= 0:
        raise ValueError("chunk_size must be positive")

    temp_paths = []
    try:
        chunk = []
        with open(input_path, encoding="utf-8") as source:
            for line in source:
                chunk.append(json.loads(line))
                if len(chunk) >= chunk_size:
                    temp_paths.append(_write_sorted_chunk(chunk, key_fn))
                    chunk = []
        if chunk:
            temp_paths.append(_write_sorted_chunk(chunk, key_fn))

        streams = [
            _read_jsonl(path) for path in temp_paths
        ]
        merged = heapq.merge(*streams, key=key_fn)

        with open(output_path, "w", encoding="utf-8") as target:
            for record in merged:
                target.write(json.dumps(record) + "\n")
    finally:
        for path in temp_paths:
            Path(path).unlink(missing_ok=True)

def _write_sorted_chunk(records, key_fn):
    records.sort(key=key_fn)
    handle = tempfile.NamedTemporaryFile(
        mode="w", encoding="utf-8", delete=False
    )
    try:
        for record in records:
            handle.write(json.dumps(record) + "\n")
    finally:
        handle.close()
    return handle.name

def _read_jsonl(path):
    with open(path, encoding="utf-8") as source:
        for line in source:
            yield json.loads(line)
```

## 4. Complexity

For N records and chunk size B:
- In-memory chunk: O(B).
- Number of runs: approximately R = ceil(N/B).
- Sorting: approximately O(N log B) total for run generation.
- K-way merge: O(N log R) time and O(R) heap/stream state, excluding file buffers.
- Disk usage: O(N) temporary storage.

## 5. Production caveats

- This example assumes each JSON record fits in memory and is independently parseable.
- Stable sorting and deterministic tie-breakers may matter.
- Temporary disk capacity and cleanup must be monitored.
- Protect output with atomic publish/rename where supported.
- Handle corrupt chunks and interrupted runs with a manifest/checkpoint.
- For distributed data, Spark/SQL engines may be more appropriate than a single-host external sort.

## 6. Interviewer–candidate discussion

**Interviewer:** Why not use pandas?

**Candidate:** Pandas is excellent when the data fits memory or can be safely chunked. A global sort of data larger than RAM needs an external algorithm or distributed engine.

**Interviewer:** How would you avoid losing output if the process crashes?

**Candidate:** Write to a temporary destination, validate row counts/checksums, fsync where required, then atomically publish. Keep a run manifest and make temporary-run cleanup restart-safe.

## 7. Tests and edge cases

- Empty input.
- Input smaller than one chunk.
- Exact multiple of chunk size.
- Duplicate sort keys.
- Malformed JSON record.
- Disk full during run generation.
- Process crash during merge.
- Very large single record.

## 8. Follow-ups

1. Add stable tie-breaking by source sequence.
2. Implement bounded fan-in when there are too many temporary runs.
3. Compare external sort with Spark `repartitionAndSortWithinPartitions`.
4. Design restartability for a multi-stage sort.
