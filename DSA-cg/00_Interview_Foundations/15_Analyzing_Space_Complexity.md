# Analyzing Space Complexity

## 1. What to count

Distinguish:
- **Input space:** memory occupied by the provided input.
- **Auxiliary space:** extra memory used by the algorithm.
- **Output space:** memory required for returned results.
- **Call stack:** recursion frames.

Interviewers often expect auxiliary space separately from output space. State your convention.

## 2. Examples

| Technique | Auxiliary space |
|---|---|
| Iterative sum | O(1) |
| Hash set of n values | O(n) |
| Copy of input list | O(n) |
| Recursive tree DFS | O(h) stack |
| BFS on a wide tree | O(w) queue |
| Merge sort | O(n) auxiliary array in common implementations |
| In-place two pointers | O(1) |

For a tree, `h` is height and `w` is maximum width. Either can be O(n) in the worst case.

## 3. Example: recursive versus iterative DFS

Recursive DFS uses O(h) call-stack space. An iterative DFS uses an explicit stack; its worst-case auxiliary space is also O(V), but it avoids language recursion limits and makes stack management explicit.

## 4. Python-specific considerations

- Lists and dictionaries have object overhead beyond raw values.
- Slicing a list creates a new list.
- `sorted(items)` creates a new list; `items.sort()` sorts in place.
- A generator can reduce materialized intermediate collections, but does not make retained state disappear.
- Recursion depth can fail even when asymptotic memory appears acceptable.

## 5. Interview conversation

**Interviewer:** Your algorithm is O(n) space. Does that include the output?

**Candidate:** I am reporting O(n) auxiliary space for the hash map, excluding the returned output. If the output is also O(n), total additional memory is still O(n) asymptotically, but I will state the distinction.

## 6. Practice

For each solution, report input, output, auxiliary and stack space separately: merge intervals, BFS, recursive DFS, group anagrams and product-except-self.
