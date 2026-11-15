# Retries, Exponential Backoff, Jitter, and Circuit Breakers

## 1. Interview scenario

A data pipeline calls a flaky downstream API. Temporary timeouts occur. Design retries that improve resilience without creating a retry storm or duplicating side effects.

## 2. Retry policy

Retry only failures classified as transient. Use a maximum attempt count, an overall deadline, exponential backoff, and jitter. Respect `Retry-After` where the service provides it. Do not retry validation errors or permanent authorization failures by default.

```python
import random
import time

def call_with_retry(
    operation,
    retryable,
    max_attempts=4,
    base_delay=0.25,
    max_delay=5.0,
    sleep=time.sleep,
    rng=random.random,
):
    last_error = None

    for attempt in range(max_attempts):
        try:
            return operation()
        except Exception as exc:
            last_error = exc
            if attempt == max_attempts - 1 or not retryable(exc):
                raise

            ceiling = min(max_delay, base_delay * (2 ** attempt))
            delay = rng() * ceiling  # full jitter in [0, ceiling)
            sleep(delay)

    raise last_error
```

## 3. Dry run

With base delay 0.25 seconds, the backoff ceilings are 0.25, 0.5, and 1.0 seconds. Jitter randomizes the actual delay below each ceiling so many workers do not retry simultaneously.

## 4. Complexity and operational behavior

- Operation time dominates.
- At most `max_attempts` calls are made.
- Worst-case waiting is bounded by the sum of configured delay ceilings, excluding operation duration.
- Always add an overall deadline; attempt count alone may exceed the caller's latency budget.

## 5. Circuit breaker

A circuit breaker has three states:

- **Closed:** requests flow; failures are counted.
- **Open:** requests fail fast for a cool-down period.
- **Half-open:** allow a small probe set; success closes the circuit, failure reopens it.

Retries handle an individual request. A circuit breaker protects a struggling dependency from sustained load. They complement each other but must be configured together to avoid repeated load during an outage.

## 6. Production considerations

- Use idempotency keys for retried writes.
- Avoid retrying both at every layer; nested retries multiply traffic.
- Add jitter and concurrency limits.
- Emit attempt count, retry reason, latency, and breaker-state metrics.
- Use a deadline propagated across service boundaries.
- Treat cancellation and process shutdown distinctly from transient network errors.

## 7. Tests

```python
def test_retries_then_succeeds():
    calls = [0]
    delays = []

    def operation():
        calls[0] += 1
        if calls[0] < 3:
            raise TimeoutError("temporary")
        return "ok"

    result = call_with_retry(
        operation,
        retryable=lambda exc: isinstance(exc, TimeoutError),
        max_attempts=4,
        sleep=delays.append,
        rng=lambda: 0.5,
    )
    assert result == "ok"
    assert calls[0] == 3
    assert len(delays) == 2
```

## 8. Interviewer–candidate discussion

**Interviewer:** Should every exception be retried?

**Candidate:** No. Retry classification is part of the API contract. Retrying malformed input or a 401 response is wasteful. Timeouts and selected 5xx responses may be transient, but a timeout on a write is ambiguous: the server may already have committed it.

**Interviewer:** What if the API has a 30-second SLA?

**Candidate:** Set an end-to-end deadline below the SLA, budget time for the operation and retry delays, and stop retrying when the remaining budget cannot support another attempt.

## 9. Follow-ups

1. Implement a circuit breaker with a fake clock.
2. How do retries interact with Kafka offset commits?
3. How would you honor HTTP 429 and `Retry-After`?
4. How do you prevent retry amplification across a microservice call chain?
