# Rate Limiter: Token Bucket and Sliding Window

## 1. Interview scenario

Protect an API or ingestion service from bursts and sustained overload. Implement a limiter that allows a configured average rate while permitting a controlled burst.

## 2. Token bucket model

A bucket holds at most `capacity` tokens. Tokens refill at `refill_rate` tokens per second. A request costing `n` tokens is allowed if enough tokens exist.

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate, clock):
        if capacity <= 0 or refill_rate <= 0:
            raise ValueError("capacity and refill_rate must be positive")
        self.capacity = float(capacity)
        self.refill_rate = float(refill_rate)
        self.clock = clock
        self.tokens = float(capacity)
        self.last_time = clock()

    def allow(self, cost=1):
        if cost <= 0 or cost > self.capacity:
            raise ValueError("cost must be positive and no greater than capacity")

        now = self.clock()
        elapsed = max(0.0, now - self.last_time)
        self.tokens = min(
            self.capacity,
            self.tokens + elapsed * self.refill_rate
        )
        self.last_time = max(self.last_time, now)

        if self.tokens >= cost:
            self.tokens -= cost
            return True
        return False
```

## 3. Dry run

Capacity 5, refill rate 1 token/second. Initially 5 requests can pass. A sixth immediate request is denied. After 2 seconds, 2 tokens have replenished, so two more one-token requests can pass.

## 4. Complexity

- Time: O(1) per request.
- Space: O(1) per tracked client.
- For M clients, state is O(M); client-key cardinality must be bounded or expired.

## 5. Alternative algorithms

- Fixed window: simple counters, but allows boundary bursts.
- Sliding log: accurate but stores request timestamps; O(R) state for R requests in window.
- Sliding-window counter: approximate with lower memory.
- Token bucket: controls burst and long-term average rate.
- Leaky bucket: smooths output rate.

## 6. Distributed production design

A process-local limiter is not globally correct when requests are served by multiple instances. Use a shared atomic store (for example, a Redis Lua script) or an API gateway limiter. The refill and token deduction must be atomic. Define behavior during shared-store outage: fail open, fail closed, or use a conservative local fallback.

## 7. Tests with a fake clock

```python
def test_token_bucket():
    now = [0.0]
    bucket = TokenBucket(2, 1, clock=lambda: now[0])
    assert bucket.allow()
    assert bucket.allow()
    assert not bucket.allow()
    now[0] += 1.0
    assert bucket.allow()
    assert not bucket.allow()
```

## 8. Interviewer–candidate discussion

**Interviewer:** Why not just sleep when the limit is exceeded?

**Candidate:** Sleeping ties up workers and can amplify queueing. An API should usually reject with a retry hint or enqueue work with explicit backpressure. A client-side SDK may apply bounded retry with jitter.

**Interviewer:** How do you avoid unbounded per-user state?

**Candidate:** Expire inactive keys, cap tracked cardinality, use shared storage with TTL, and define a safe fallback when state capacity is reached.

## 9. Follow-ups

1. Implement a sliding-window counter.
2. Add separate limits per tenant and endpoint.
3. Design a globally consistent limiter across regions.
4. Explain fairness when tenants have different quotas.
