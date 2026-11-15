# Principal Data Engineer — DSA Interview Masterclass

**Format:** One Markdown file per problem, written as an interview conversation with baseline and optimized approaches, Python code, complexity, dry run, follow-ups and Data Engineering connections.

**Total:** 30 study-plan entries across 7 sections. Container With Water appears twice in the supplied list; both entries are retained, with the second emphasizing the two-pointer proof.

## Topic index

### Part 1: Arrays & Hashing

- [01. Two Sum](01_Arrays_and_Hashing/01_Two_Sum.md) — Easy
- [02. Product of Array Except Self](01_Arrays_and_Hashing/02_Product_of_Array_Except_Self.md) — Medium
- [03. Find the Duplicate Number](01_Arrays_and_Hashing/03_Find_the_Duplicate_Number.md) — Medium
- [04. Set Matrix Zeroes](01_Arrays_and_Hashing/04_Set_Matrix_Zeroes.md) — Medium
- [05. 3Sum](01_Arrays_and_Hashing/05_3Sum.md) — Medium
- [06. Container With Water](01_Arrays_and_Hashing/06_Container_With_Water.md) — Medium
- [07. Maximum Subarray — Kadane's Algorithm](01_Arrays_and_Hashing/07_Maximum_Subarray.md) — Medium
- [08. Spiral Matrix](01_Arrays_and_Hashing/08_Spiral_Matrix.md) — Medium
- [09. Rotate Image](01_Arrays_and_Hashing/09_Rotate_Image.md) — Medium
- [10. Merge Intervals](01_Arrays_and_Hashing/10_Merge_Intervals.md) — Medium
- [11. Group Anagrams](01_Arrays_and_Hashing/11_Group_Anagrams.md) — Medium
- [12. Valid Anagram](01_Arrays_and_Hashing/12_Valid_Anagram.md) — Easy

### Part 2: Two Pointers

- [13. Container With Water — Two-Pointer Focus](02_Two_Pointers/13_Container_With_Water_Two_Pointers.md) — Medium
- [14. Trapping Rain Water](02_Two_Pointers/14_Trapping_Rain_Water.md) — Hard
- [15. Remove Duplicates from Sorted Array](02_Two_Pointers/15_Remove_Duplicates_from_Sorted_Array.md) — Easy
- [16. Palindrome Number](02_Two_Pointers/16_Palindrome_Number.md) — Easy

### Part 3: Sliding Window

- [17. Best Time to Buy and Sell Stock](03_Sliding_Window/17_Best_Time_Buy_Sell_Stock.md) — Easy
- [18. Contains Duplicate](03_Sliding_Window/18_Contains_Duplicate.md) — Easy
- [19. Minimum Window Substring](03_Sliding_Window/19_Minimum_Window_Substring.md) — Hard
- [20. Longest Repeating Character Replacement](03_Sliding_Window/20_Longest_Repeating_Char_Replacement.md) — Medium

### Part 4: Binary Search

- [21. Binary Search](04_Binary_Search/21_Binary_Search.md) — Easy
- [22. Search in Rotated Sorted Array](04_Binary_Search/22_Search_Rotated_Sorted_Array.md) — Medium

### Part 5: Linked List

- [23. Reverse Linked List](05_Linked_List/23_Reverse_Linked_List.md) — Easy
- [24. Merge Two Sorted Lists](05_Linked_List/24_Merge_Two_Sorted_Lists.md) — Easy

### Part 6: Graphs & BFS/DFS

- [25. Number of Islands](06_Graphs_BFS_DFS/25_Number_of_Islands.md) — Medium
- [26. Clone Graph](06_Graphs_BFS_DFS/26_Clone_Graph.md) — Medium
- [27. Course Schedule](06_Graphs_BFS_DFS/27_Course_Schedule.md) — Medium

### Part 7: Trees & Heaps

- [28. Lowest Common Ancestor of a BST](07_Trees_and_Heaps/28_LCA_BST.md) — Medium
- [29. Maximum Depth of Binary Tree](07_Trees_and_Heaps/29_Max_Depth_Binary_Tree.md) — Easy
- [30. Top K Frequent Elements](07_Trees_and_Heaps/30_Top_K_Frequent_Elements.md) — Medium

## Notes on topic classification

- Container With Water appears twice in the supplied plan. Both numbered entries are retained; the second is a focused revision of the same problem.
- Best Time to Buy and Sell Stock is more precisely a one-pass greedy/prefix-minimum problem than a canonical fixed-window problem.
- Contains Duplicate is primarily a hash-set problem, not a canonical sliding-window problem.
- Palindrome Number is primarily a numeric/digit-reversal problem; it is retained under the supplied Two Pointers section.

## Recommended study routine

1. Read the problem statement and clarify constraints before looking at the optimized solution.
2. Explain the baseline approach and its complexity.
3. Derive the optimized approach and state its invariant.
4. Dry-run the code without executing it.
5. Implement from memory and test edge cases.
6. Answer follow-up questions aloud, including trade-offs and data-scale implications.

## Additional Batch 1 — Interview Foundations and Complexity

This batch adds 16 standalone Markdown lessons:
- Interview strategy, language choice and AI-assisted coding awareness.
- Fast-track topic overviews for core patterns.
- Three additional coding exercises: Move Zeros, Remove Duplicates in String, Contiguous Subarray Sum.
- Four dedicated complexity lessons: Big O, time, space and optimization.

See `00_Interview_Foundations/` for the lesson files.

## Additional Batch 2 — Coding Patterns, Arrays and Hash Tables

This batch adds 14 standalone Markdown lessons:
- Coding-pattern selection and dedicated lessons for two pointers, prefix sums, tortoise & hare, sliding windows, two-pass techniques, bit manipulation and cyclic placement.
- Array fundamentals and Task Scheduler.
- Hash-table fundamentals, Difference of Arrays, Smallest Missing Positive Integer and Boggle Board.
- Each problem-oriented lesson includes problem framing, baseline and optimized approaches, Python code, dry run, complexity, edge cases, interview discussion, follow-ups and tests.

See `01_Coding_Patterns_Arrays_Hash_Tables/` for the lesson files.

## Additional Batch 3 — Searching, Sorting and String Manipulation

This batch adds 17 standalone lessons:
- Search/sort fundamentals, binary-search boundaries and variants, Number Finder, duplicate detection and K-messed arrays.
- Rotated-array minimum, binary search on answer (Koko Eating Bananas), and Find Peak Element.
- String fundamentals, most-common words, valid palindrome, IP validation, reverse-word message transformation, serialization/deserialization, longest unique substring and longest palindromic substring.
- An advanced Minimum Window Substring companion focused on correctness invariants and edge cases.

See `02_Searching_Sorting_Strings/` for the lesson files. Existing Binary Search, Search in Rotated Sorted Array, Minimum Window Substring and Remove Duplicates in String lessons were retained rather than duplicated.

## Additional Batch 4 — Graphs, Trees and Heaps

This batch adds 16 standalone lessons:
- Graph representation, BFS/DFS, connected components, cycle detection, topological ordering, unweighted shortest paths and Dijkstra.
- Binary-tree traversals, BST validation, kth smallest, diameter and balance checking.
- Trie fundamentals, heap/priority-queue fundamentals, kth-largest selection and k-way merge.
- Existing Number of Islands, Clone Graph, Course Schedule, LCA of BST, Maximum Depth and Top K Frequent Elements lessons were retained without duplication.

See `03_Graphs_Trees_Heaps/` for the lesson files.

## Additional Batch 5 — Stacks, Queues, Linked Lists and Backtracking

This batch adds 17 standalone lessons:
- Stack/queue fundamentals, Valid Parentheses, Min Stack, Monotonic Stack / Daily Temperatures, Queue Using Two Stacks.
- Linked-list fundamentals, cycle detection and cycle entry, middle node, remove nth from end, and heap-based merge of k linked lists.
- LRU Cache, recursion fundamentals, backtracking framework, Generate Parentheses, Subsets, Permutations, Combination Sum and N-Queens.
- Existing Reverse Linked List and Merge Two Sorted Lists lessons are retained; this batch adds complementary linked-list coverage.

See `04_Stacks_Queues_Linked_Lists_Backtracking/` for the lesson files.

## Additional Batch 6 — Dynamic Programming

This batch adds 16 standalone lessons:
- DP fundamentals, memoization vs tabulation and state design.
- Climbing Stairs, House Robber, minimum Coin Change and Coin Change combinations.
- 0/1 Knapsack, Longest Common Subsequence, Edit Distance, Unique Paths, Maximum Product Subarray and Longest Increasing Subsequence.
- Partition Equal Subset Sum, Word Break, Decode Ways, and DP optimization/interview problem solving.

See `05_Dynamic_Programming/` for the lesson files.


## Additional Batch 7 — Advanced Data Structures and Algorithms

This batch adds 14 standalone lessons:
- Greedy algorithms and interval scheduling; Meeting Rooms I/II; Merge Intervals and interval removals.
- Jump Game I/II and Gas Station.
- Union-Find / DSU; Kruskal and Prim minimum spanning trees.
- Bellman–Ford and Floyd–Warshall shortest paths.
- Segment Trees and Fenwick Trees / Binary Indexed Trees.
- Advanced trie-guided word search and Bitmask Dynamic Programming.
- Lessons connect the patterns to scheduling, dependency connectivity, network design, mutable range aggregation, and small assignment problems.

See `06_Advanced_Data_Structures_Algorithms/` for the lesson files.


## Additional Batch 8 — Advanced Problem-Solving Patterns and Streaming

This batch adds 12 standalone lessons:
- Monotonic queue / sliding-window maximum; sweep-line algorithms and event ordering.
- Coordinate compression and range-query indexing.
- Randomized Quickselect and reservoir sampling.
- Streaming median with two heaps and top-k frequency analysis.
- Probabilistic structures: Bloom Filter and Count-Min Sketch.
- K-way merge of sorted streams and Python iterator/chunked processing.
- A principal-level event-log analytics scenario covering sorted partition merge, exact aggregation, top-k, memory/cardinality, late events, skew, and replay safety.

See `07_Advanced_Problem_Solving_and_Streaming/` for the lesson files.

## Batch 9 — Practical Python Data Engineering Scenarios

Location: `08_Practical_Python_Data_Engineering_Scenarios/`

1. Log Parsing at Scale: Robust, Streaming Python
2. Deduplication and Idempotent Event Processing
3. LFU Cache and Cache Design Trade-offs
4. Rate Limiter: Token Bucket and Sliding Window
5. Retries, Exponential Backoff, Jitter, and Circuit Breakers
6. Bounded Producer–Consumer Pipeline and Backpressure
7. Python Concurrency: Threads vs Processes vs asyncio
8. Memory-Efficient File Processing and External Merge Sort
9. API Pagination and Incremental Extraction
10. Building a Reusable Data Quality Validation Framework
11. Concurrent Task Scheduler with Dependencies
12. End-to-End Incremental ETL: Principal-Level Coding Challenge

Batch 9 emphasizes practical Python, data pipeline correctness, idempotency, backpressure, concurrency, checkpointing, operational resilience, and principal-level trade-offs.

### Principal-Level Interview Execution Notes and References

See `00_Interview_Foundations/17_Principal_Level_Interview_Execution_Notes_and_References.md` for the interview execution checklist, principal-level engineering habits, Batch 9 priority lessons, complexity-analysis checklist, practice routine, and curated external references.
