# Concurrent Task Scheduler with Dependencies

## 1. Interview scenario

A pipeline consists of tasks with dependencies. Independent tasks may run concurrently, but a task must not start until all parents succeed. Build a small dependency-aware scheduler and discuss production orchestration.

## 2. Graph model

Represent dependencies as a directed acyclic graph (DAG). Validate cycles before execution. Track each task as pending, running, succeeded, or failed. A task is eligible only when all dependencies succeeded.

## 3. Topological validation

```python
from collections import deque, defaultdict

def topological_order(dependencies):
    # dependencies maps task -> iterable of prerequisite tasks
    indegree = {task: 0 for task in dependencies}
    children = defaultdict(list)

    for task, parents in dependencies.items():
        for parent in parents:
            if parent not in indegree:
                indegree[parent] = 0
            indegree[task] += 1
            children[parent].append(task)

    queue = deque(task for task, degree in indegree.items() if degree == 0)
    order = []

    while queue:
        task = queue.popleft()
        order.append(task)
        for child in children[task]:
            indegree[child] -= 1
            if indegree[child] == 0:
                queue.append(child)

    if len(order) != len(indegree):
        raise ValueError("dependency graph contains a cycle")
    return order
```

## 4. Bounded concurrent execution

For a production-grade scheduler, use a tested orchestration framework rather than treating this short example as a full scheduler. The execution loop should:
1. Validate the DAG.
2. Maintain a ready queue.
3. Submit no more than `max_workers` tasks.
4. On success, release children whose dependencies are all successful.
5. On failure, apply an explicit policy: fail-fast, continue independent branches, or retry.
6. Persist state so process restart does not lose progress.

## 5. Complexity

Topological validation: O(V + E) time and O(V + E) space for V tasks and E dependency edges. Scheduling overhead is O(V + E) excluding task runtime, retries, and persistence.

## 6. Principal-level concerns

- Idempotent tasks and restartability.
- Retry policy per task and total deadline.
- Resource pools and concurrency quotas.
- Critical-path analysis and task prioritization.
- Data dependencies vs control dependencies.
- Backfills, partitions, and parameterized DAG runs.
- State persistence and exactly-once side effects.
- Observability: queue wait, task duration, retries, failure reason, and SLA misses.

## 7. Interviewer–candidate discussion

**Interviewer:** Why not launch every task in a thread pool?

**Candidate:** That ignores dependencies and can start downstream work too early. A scheduler must only release a node when all prerequisites are successful.

**Interviewer:** What if one branch fails but another is independent?

**Candidate:** The policy is business-dependent. For independent branches, continuing can preserve useful outputs, but the overall DAG run must remain failed or partially failed and clearly report skipped descendants.

## 8. Tests and edge cases

- Linear chain.
- Multiple independent roots.
- Fan-in node with multiple parents.
- Unknown dependency.
- Cycle.
- Empty graph.
- Task failure and dependent skip.
- Retry exhaustion.
- Scheduler restart with in-flight task state.

```python
def test_topological_order():
    order = topological_order({
        "extract": [],
        "clean": ["extract"],
        "publish": ["clean"],
    })
    assert order.index("extract") < order.index("clean")
    assert order.index("clean") < order.index("publish")

def test_cycle_rejected():
    try:
        topological_order({"a": ["b"], "b": ["a"]})
    except ValueError:
        pass
    else:
        raise AssertionError("cycle should be rejected")
```

## 9. Follow-ups

1. Add priority scheduling.
2. How would you schedule tasks with CPU/memory resource requirements?
3. How do Airflow, Dagster, and a custom scheduler differ?
4. Explain critical path and how it affects pipeline SLA.
