# Course Schedule

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Graph, Topological Sort, Cycle Detection, BFS/DFS |
| **Companies** | Meta, Google, Amazon, Bloomberg |
| **Interview Frequency** | High |

---

## 1. Problem Statement
There are `numCourses` courses labeled 0 to numCourses-1. You are given an array `prerequisites` where `prerequisites[i] = [a, b]` means you must take course `b` first to take course `a`. Return `true` if you can finish all courses, or `false` otherwise.

```python
from typing import List
```

**Example:**
```
Input: numCourses = 2, prerequisites = [[1,0]]
Output: true
Explanation: Course 0 must be taken before Course 1. Possible order: [0, 1].
```

**Example 2:**
```
Input: numCourses = 2, prerequisites = [[1,0],[0,1]]
Output: false
Explanation: Course 0 requires Course 1, and Course 1 requires Course 0 — a cycle exists.
```

---

## 2. Approaches

### Approach 1: BFS — Kahn's Algorithm (Topological Sort) (Optimal)
Build an adjacency list and in-degree count. Start with all nodes that have 0 in-degree (no prerequisites). Process them one by one, reducing the in-degree of their dependents. If all courses are processed, return True; otherwise a cycle exists.

**Pros:**
- Naturally processes courses level by level — mirrors how prerequisites are resolved in real scheduling
- Detects cycles explicitly: if processed count < numCourses, a cycle exists

**Cons:**
- Requires an upfront pass to compute all in-degrees before processing can begin
- Queue overhead for managing zero-in-degree nodes

```python
from collections import deque
from typing import List

def canFinish(numCourses: int, prerequisites: List[List[int]]) -> bool:
    # Build adjacency list and in-degree array
    adj = [[] for _ in range(numCourses)]
    in_degree = [0] * numCourses
    
    for course, prereq in prerequisites:
        adj[prereq].append(course)
        in_degree[course] += 1
    
    # Start with courses that have no prerequisites
    queue = deque([i for i in range(numCourses) if in_degree[i] == 0])
    processed = 0
    
    while queue:
        course = queue.popleft()
        processed += 1
        for dependent in adj[course]:
            in_degree[dependent] -= 1
            if in_degree[dependent] == 0:
                queue.append(dependent)
    
    return processed == numCourses
```

| Metric | Value |
|---|---|
| **Time** | O(V + E) — each node and edge visited once |
| **Space** | O(V + E) — adjacency list + in-degree array + queue |

### Approach 2: DFS with Cycle Detection (Alternative)
Use three states: 0 = unvisited, 1 = visiting (in current path), 2 = visited. DFS from each unvisited node; if we encounter a node in state 1, we found a cycle. If all nodes reach state 2, no cycle exists.

**Pros:**
- Detects a cycle as soon as a back-edge is found — can fail early without exploring the entire graph
- No need to precompute in-degrees; adjacency list is sufficient

**Cons:**
- Risk of stack overflow on deep dependency chains (Python recursion limit ~1000)
- Harder to explain in an interview than the BFS approach

```python
from typing import List

def canFinish(numCourses: int, prerequisites: List[List[int]]) -> bool:
    adj = [[] for _ in range(numCourses)]
    for course, prereq in prerequisites:
        adj[prereq].append(course)
    
    STATE_UNVISITED, STATE_VISITING, STATE_VISITED = 0, 1, 2
    state = [STATE_UNVISITED] * numCourses
    
    def hasCycle(node: int) -> bool:
        if state[node] == STATE_VISITING:
            return True   # Back edge — cycle found
        if state[node] == STATE_VISITED:
            return False  # Already fully processed
        
        state[node] = STATE_VISITING
        for neighbor in adj[node]:
            if hasCycle(neighbor):
                return True
        
        state[node] = STATE_VISITED
        return False
    
    for i in range(numCourses):
        if state[i] == STATE_UNVISITED and hasCycle(i):
            return False  # Cycle detected
    
    return True  # No cycles — topological order exists
```

| Metric | Value |
|---|---|
| **Time** | O(V + E) |
| **Space** | O(V + E) — adjacency list + O(V) recursion stack |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| No prerequisites (`prerequisites = []`) | Return `True` — all courses can be taken |
| Single course, no prerequisites | Return `True` |
| Self-prerequisite (`[0, 0]`) | Return `False` — immediate cycle |
| All courses form a single chain | Return `True` — linear order |
| Disconnected components with a cycle in one | Return `False` — cycle anywhere means impossible |

---

## 4. Interview Talking Points
- "This is the classic **topological sort problem** — we need to check if the dependency graph is a DAG (Directed Acyclic Graph)."
- "Both approaches run in O(V + E), but BFS is usually preferred in interviews because it's easier to explain and avoids recursion depth issues."
- "DE connection: This maps directly to **dependency resolution in data pipelines** — e.g., Airflow DAGs, dbt models, or build systems where tasks must be ordered by their dependencies before execution."

---

## 5. Interview Follow-Up Questions
**Q1. What if we need to return the actual ordering?**
-> In BFS: collect `course` in a list each time it's popped from the queue. In DFS: add `node` to the result *after* processing all neighbors (post-order), then reverse the list.

**Q2. How would you handle partially ordered courses (some independent, some dependent)?**
-> The algorithm handles this naturally — independent courses start at in-degree 0 and are processed first or in parallel.

**Q3. What if some prerequisites are optional (soft dependencies)?**
-> Model soft dependencies as separate edges with a flag, or run the algorithm twice: once with only hard dependencies, then optionally reorder for soft ones.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Confusing `[a, b]` direction | `[a, b]` means b → a (b is prerequisite of a), not a → b | Build adjacency list as `adj[prereq].append(course)` |
| Not handling disconnected nodes | Some courses may have no prerequisites and no dependents | BFS: start with ALL zero in-degree nodes; DFS: loop over ALL unvisited nodes |
| Using `visited` set instead of 3-state tracking | A simple visited set can't detect cycles — only distinguishes visited vs unvisited | Use 3 states: unvisited / visiting / visited |
| Returning `True` as soon as one node is processed | Must verify ALL courses can be completed | BFS: check `processed == numCourses`; DFS: ensure no cycle was found in ANY component |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| [Course Schedule II](https://leetcode.com/problems/course-schedule-ii/) | Same problem, but return the actual ordering |
| [Alien Dictionary](https://leetcode.com/problems/alien-dictionary/) | Build graph from string ordering, then topological sort |
| [Sequence Reconstruction](https://leetcode.com/problems/sequence-reconstruction/) | Verify if a sequence is uniquely determined by subsequences |

---

*Last updated: 2025*