# API Pagination and Incremental Extraction

## 1. Interview scenario

Extract records from a paginated REST API every hour. The API may return thousands of pages, rate-limit clients, and occasionally fail. New and updated records must be captured without gaps.

## 2. Pagination patterns

- Offset/limit: easy, but concurrent inserts can shift pages and cause gaps/duplicates.
- Page number: similar consistency risks.
- Cursor/token: usually safer for large changing datasets.
- Keyset pagination: continue after a stable ordered key, such as `(updated_at, id)`.

Prefer opaque server cursors when offered. Do not assume a cursor is a numeric offset.

## 3. Cursor extraction skeleton

```python
def extract_pages(fetch_page, persist_page, start_cursor=None):
    cursor = start_cursor
    page_count = 0
    record_count = 0

    while True:
        response = fetch_page(cursor)
        records = response["items"]
        next_cursor = response.get("next_cursor")

        # Persist records and checkpoint atomically where possible.
        persist_page(records, next_cursor)
        page_count += 1
        record_count += len(records)

        if not next_cursor:
            break
        if next_cursor == cursor:
            raise RuntimeError("API returned a non-advancing cursor")
        cursor = next_cursor

    return {"pages": page_count, "records": record_count}
```

## 4. Incremental watermark strategy

A naive `updated_at > last_watermark` can miss records when timestamps have coarse precision or updates arrive late. A safer approach is:
1. Read from `watermark - overlap_window`.
2. Upsert by stable primary key.
3. Advance watermark only after durable sink commit.
4. Use a deterministic tie-breaker such as `(updated_at, id)`.
5. Periodically reconcile historical partitions if the source can backdate updates.

## 5. Complexity

- Time: O(N) API records plus network latency per page.
- Memory: O(P) for one page, assuming page size P.
- Persistent state: cursor/watermark plus sink data.
- Throughput is usually constrained by API quotas and sink write capacity.

## 6. Production considerations

- Set connect/read timeouts.
- Retry transient errors with backoff and jitter; honor 429/Retry-After.
- Keep page size within documented API limits.
- Store request IDs and page/cursor metadata for audit.
- Make sink writes idempotent.
- Checkpoint after durable writes, not before.
- Handle token expiration and cursor invalidation.
- Secure credentials and redact them from logs.

## 7. Interviewer–candidate discussion

**Interviewer:** What if the process crashes after writing the page but before checkpointing?

**Candidate:** The page is fetched again. Idempotent upsert or an inbox key prevents duplicate effects. This is safer than checkpointing first, which can lose data.

**Interviewer:** What if it checkpoints before the sink commit?

**Candidate:** That creates a data-loss window. Checkpoint advancement must follow durable persistence, ideally in the same transaction or through a replay-safe protocol.

## 8. Tests and edge cases

- Empty first page.
- Last page has no cursor.
- Cursor repeats.
- API timeout and retry.
- 429 with Retry-After.
- Sink fails after fetch.
- Duplicate record across pages.
- Late update behind watermark.
- Cursor expires midway.

## 9. Follow-ups

1. Design extraction for an API with only offset pagination.
2. Explain composite watermarks.
3. How would you handle deletes (tombstones)?
4. How do you reconcile source and destination counts?
