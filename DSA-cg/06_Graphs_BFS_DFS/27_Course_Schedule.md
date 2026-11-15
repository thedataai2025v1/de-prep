# Course Schedule

**Difficulty:** Medium  
**Topics:** Graph, Topological Sort, BFS, DFS  
**Study-plan section:** Graphs BFS DFS  
**Problem ID:** 27

## 1. Problem Statement

There are numCourses courses and prerequisite pairs [a,b] meaning b must be completed before a. Return true if all courses can be completed.

**Example:**
```text
numCourses = 2, prerequisites = [[1,0]]
Output = True
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Try every possible course ordering and check whether each prerequisite appears first; this is factorial and only a conceptual baseline.

```python
def can_finish(numCourses, prerequisites):
    from itertools import permutations
    edges = set((b,a) for a,b in prerequisites)
    for order in permutations(range(numCourses)):
        pos = {course:i for i,course in enumerate(order)}
        if all(pos[u] < pos[v] for u,v in edges):
            return True
    return numCourses == 0
```

- **Complexity:** O(V! × E) conceptual brute force

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Use Kahn's topological-sort algorithm: compute indegrees, enqueue all zero-indegree nodes, and remove outgoing edges.

```python
from collections import deque
def can_finish(numCourses, prerequisites):
    graph = [[] for _ in range(numCourses)]
    indegree = [0] * numCourses
    for course, pre in prerequisites:
        graph[pre].append(course)
        indegree[course] += 1
    q = deque(i for i in range(numCourses) if indegree[i] == 0)
    completed = 0
    while q:
        node = q.popleft()
        completed += 1
        for nxt in graph[node]:
            indegree[nxt] -= 1
            if indegree[nxt] == 0:
                q.append(nxt)
    return completed == numCourses
```

- **Complexity:** O(V+E) time; O(V+E) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Use Kahn's topological-sort algorithm: compute indegrees, enqueue all zero-indegree nodes, and remove outgoing edges.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Course 0 has indegree zero, so process it and reduce course 1's indegree to zero. Process course 1; completed=2, equal to numCourses, so there is no cycle.

## 4. Interview Conversation

**Interviewer:** How do you detect a cycle in prerequisites?

**Candidate:** A directed cycle prevents a topological ordering. Kahn's algorithm processes only zero-indegree nodes; if fewer than V nodes are processed, a cycle exists.

**Interviewer:** Why does the graph direction matter?

**Candidate:** For pair [course, prerequisite], I add prerequisite→course so prerequisites are processed first.

**Interviewer:** Could DFS solve it?

**Candidate:** Yes. Three-color DFS detects a back edge to a node currently in the recursion stack.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Return an actual order using the same algorithm. Duplicate prerequisite pairs may inflate indegree unless deduplicated according to the input contract. Self-loop means impossible.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Disconnected DAG components are all processed. No prerequisites means all courses can be completed.

Additional checks:
- Minimum valid input size.
- Maximum input size and performance.
- Duplicate values / repeated characters / repeated edges where applicable.
- Boundary values and absent target/result.

## 7. Common Interview Mistakes

- Coding before confirming constraints and output semantics.
- Explaining complexity without stating what `n`, `m`, `k` or other variables represent.
- Ignoring empty inputs, duplicates, boundary conditions or mutation requirements.
- Claiming a hash operation is worst-case O(1); it is generally expected/average O(1).
- Giving an optimized algorithm without explaining its invariant or correctness argument.

## 8. Data Engineering Connection

Topological sorting models job dependencies and DAG orchestration. Real schedulers also need retries, resource constraints and failure propagation.

## 9. Final Interview Summary

**One-sentence explanation:** Use Kahn's topological-sort algorithm: compute indegrees, enqueue all zero-indegree nodes, and remove outgoing edges.

**Complexity to say aloud:** O(V+E) time; O(V+E) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #27 in the supplied study plan.*