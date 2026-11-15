# Log Parsing at Scale: Robust, Streaming Python

## 1. Interview scenario

A platform emits millions of application log lines per day. Build a parser that extracts timestamp, level, service, request ID, and message; tolerates malformed records; and computes counts without loading the entire file into memory.

The interviewer is testing parsing discipline, streaming, data quality, observability, and explicit failure policy—not merely regular expressions.

## 2. Requirements and design choices

- Input may be a large file or an iterator of lines.
- A malformed line must not terminate the entire job.
- Preserve enough context to investigate rejected records.
- Keep memory proportional to the aggregation state, not file size.
- Distinguish a malformed record from a valid record with a missing optional field.

Assume this input contract:

`2026-11-15T10:12:13Z INFO billing request_id=abc-123 payment captured`

## 3. Brute-force approach

Read the whole file into a list, parse it, then aggregate. This is simple but uses O(N) memory for N lines and can fail for large inputs. It also encourages an all-or-nothing failure mode.

## 4. Optimized streaming approach

Use an iterator, parse one line at a time, and update compact counters. The following parser deliberately uses a bounded grammar rather than a complicated regex that silently accepts ambiguous input.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class LogRecord:
    timestamp: datetime
    level: str
    service: str
    request_id: Optional[str]
    message: str

def parse_log_line(line: str) -> LogRecord:
    parts = line.strip().split(" ", 4)
    if len(parts) != 5:
        raise ValueError("expected timestamp, level, service, request_id, message")

    timestamp_text, level, service, request_token, message = parts
    if not request_token.startswith("request_id="):
        raise ValueError("missing request_id token")

    timestamp = datetime.fromisoformat(timestamp_text.replace("Z", "+00:00"))
    request_id = request_token.split("=", 1)[1] or None
    if level not in {"DEBUG", "INFO", "WARN", "ERROR"}:
        raise ValueError(f"unsupported log level: {level}")

    return LogRecord(timestamp, level, service, request_id, message)

def summarize_logs(lines):
    counts = {}
    rejected = 0

    for line_number, line in enumerate(lines, start=1):
        try:
            record = parse_log_line(line)
        except (ValueError, TypeError):
            rejected += 1
            continue

        key = (record.service, record.level)
        counts[key] = counts.get(key, 0) + 1

    return counts, rejected
```

## 5. Dry run

For three lines—two valid billing INFO lines and one malformed line—the parser increments `("billing", "INFO")` twice and the rejected count once. At no point does it retain all three source lines.

## 6. Complexity

Let N be the number of lines and L the average line length.

- Time: O(total input characters), approximately O(N × L).
- Extra memory: O(K), where K is the number of distinct `(service, level)` keys.
- If unique request IDs are retained, memory can instead grow with the number of records; avoid that unless required.

## 7. Production considerations

1. Define a versioned log schema; free-form logs are expensive to evolve.
2. Send rejects to a quarantine/dead-letter sink with source file, line number, and reason.
3. Track parsed, rejected, and unknown-level metrics. A sudden rejection-rate increase is an operational signal.
4. Do not log secrets or entire sensitive payloads in error messages.
5. If timestamps are used for ordering, normalize timezone handling and define late-arrival policy.
6. For JSON logs, validate required keys and types explicitly; do not rely on `dict.get()` for mandatory fields.

## 8. Interviewer–candidate discussion

**Interviewer:** Would you use one giant regular expression?

**Candidate:** Only if the grammar is stable and the expression remains reviewable. Here, tokenization plus explicit validation makes the contract clearer. For JSON logs, I would parse JSON and validate the schema.

**Interviewer:** What if the file is 200 GB?

**Candidate:** Keep the input streaming, partition work by file or byte ranges only when records can be split safely, and write aggregates incrementally. If aggregation cardinality is unbounded, use external aggregation or a distributed engine rather than an in-memory dictionary.

## 9. Edge cases and tests

- Empty line and whitespace-only line.
- Invalid timestamp or timezone.
- Message containing spaces.
- Missing request ID.
- Unsupported severity.
- Unicode message.
- Very long line and unexpectedly high-cardinality service names.

```python
def test_parse_log_line():
    record = parse_log_line(
        "2026-11-15T10:12:13Z INFO billing request_id=abc-123 payment captured"
    )
    assert record.service == "billing"
    assert record.level == "INFO"
    assert record.request_id == "abc-123"
    assert record.message == "payment captured"

def test_malformed_line_is_counted():
    counts, rejected = summarize_logs([
        "2026-11-15T10:12:13Z INFO billing request_id=x ok",
        "not a valid log",
    ])
    assert counts[("billing", "INFO")] == 1
    assert rejected == 1
```

## 10. Follow-up questions

1. How would you parse JSON logs with schema evolution?
2. How would you parallelize parsing without corrupting records at chunk boundaries?
3. How would you handle duplicate log delivery?
4. What metrics would you expose to detect a producer-side format change?
