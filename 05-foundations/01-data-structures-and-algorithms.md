# Data Structures & Algorithms

> **25 questions** — 12 theory, 11 practical, 2 experience

- Core data structures: arrays, linked lists, hashmaps, stacks, queues, trees, graphs, heaps, tries — selecting based on access patterns (random access, O(1) lookup, ordered data, priority)
- Big-O notation: time and space complexity, best/average/worst case, when Big-O misleads
- Problem-solving approach: constraint analysis (input size hints at complexity), brute force first, identify repeated work, optimize with the right data structure or pattern
- Hash maps: collisions, chaining vs open addressing, amortized O(1), degradation to O(n)
- Trees: binary trees, BSTs, balanced BSTs (AVL, red-black), tries — traversal patterns (in/pre/post/level-order), when to pick each
- Heaps (priority queues): binary heap as array, heapify O(n), insert/extract O(log n), use cases — top-K problems, merge K sorted lists, scheduling
- Stacks and queues: nesting/matching problems, BFS/scheduling, monotonic stack/queue patterns
- Binary search: monotonic condition generalization, search space reduction, off-by-one pitfalls
- Dynamic programming: overlapping subproblems, optimal substructure, recursion → memoization → tabulation → space optimization, greedy as an alternative when local choices are globally optimal
- Sliding window pattern: fixed and variable window, substring/subarray problems
- Two pointers pattern: sorted input, pair/triplet problems, duplicate skipping
- BFS/DFS on graphs: visited-set pattern, island problems, cycle detection, topological sort, backtracking for permutations/subsets

---

## Foundational

<details>
<summary>1. What are the core data structures (arrays, linked lists, hashmaps, stacks, queues, trees, graphs, heaps, tries) and why does each one exist — what access pattern is each optimized for, how do you decide which data structure fits a given problem, and what goes wrong when you pick the wrong one?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What does Big-O notation actually measure, why do we distinguish between best, average, and worst case, and how should you think about time vs space complexity tradeoffs when choosing between algorithms?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What is a systematic approach to solving algorithm problems in an interview — why should you start with brute force, how does input size hint at the target time complexity, how do you identify repeated work in a brute-force solution and use the right data structure or pattern to eliminate it, and why does this process matter more than memorizing solutions?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. How do hash maps achieve amortized O(1) lookup — what happens during hash collisions, what is the difference between chaining and open addressing for resolving them, why does performance degrade toward O(n) in pathological cases, and what determines whether a hash map is the right choice vs a sorted structure like a BST?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why do we need balanced BSTs (AVL, red-black) when regular BSTs exist — what problem do they solve, how do they maintain O(log n) guarantees, how do tree traversal patterns (in-order, pre-order, post-order, level-order) map to different use cases, and when would you reach for a trie instead of a BST?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why are heaps (priority queues) implemented as arrays instead of pointer-based trees — how does the binary heap array representation work, why is heapify O(n) instead of O(n log n), and what problem categories (top-K, merge K sorted lists, scheduling) make heaps the natural choice over sorting or other structures?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. What problem categories do stacks and queues naturally solve — why do nesting and matching problems (parentheses, expression evaluation) map to stacks, why does BFS use a queue, and what is the monotonic stack/queue pattern, what problems does it solve, and why does it achieve O(n) despite appearing nested?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why is binary search more general than just "searching a sorted array" — how does the monotonic condition generalization let you apply binary search to any problem with a search space you can halve, what are the common off-by-one pitfalls (lo <= hi vs lo < hi, mid calculation overflow), and how do you decide between left-biased and right-biased binary search?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What makes a problem solvable with dynamic programming — why do overlapping subproblems and optimal substructure matter, what is the progression from naive recursion to memoization to tabulation to space-optimized tabulation, and when should you use a greedy approach instead because local optimal choices happen to be globally optimal?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why does the sliding window pattern turn O(n^2) brute-force substring/subarray problems into O(n) — how does a fixed window differ from a variable (shrinking) window, what invariant does the window maintain, and what signals in a problem statement tell you sliding window is the right approach?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How does the two-pointer pattern exploit sorted input to avoid nested loops — why does it work for pair and triplet sum problems, how do you handle duplicate skipping to avoid redundant results, and what is the relationship between two pointers and binary search as alternative approaches to the same problems?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. How do BFS and DFS differ in their traversal behavior and when do you choose one over the other -- why does BFS find shortest paths in unweighted graphs while DFS is better for exhaustive exploration, how does the visited-set pattern prevent infinite loops in both, and what common mistakes do people make when implementing graph traversals?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Array & HashMap Problems

<details>
<summary>13. Solve Two Sum in TypeScript — given an array of integers and a target, return the indices of two numbers that add up to the target. Show the brute-force O(n^2) approach first, then optimize to O(n) using a hashmap. Explain why the hashmap approach works and what tradeoff you're making (space for time)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Solve 3Sum in TypeScript — given an array of integers, find all unique triplets that sum to zero. Explain why sorting + two pointers is the standard approach, how you skip duplicates to avoid redundant triplets, and why the time complexity is O(n^2) — is it possible to do better?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Solve Top K Frequent Elements in TypeScript — given an array and k, return the k most frequent elements. Show at least two approaches (hashmap + min-heap, and bucket sort), explain why the heap approach is O(n log k) vs the bucket sort approach is O(n), and when each is preferable</summary>

<!-- Answer will be added later -->

</details>

## Practical — String, Search & Stack Problems

<details>
<summary>16. Solve Longest Substring Without Repeating Characters in TypeScript — given a string, find the length of the longest substring without duplicate characters. Show the variable-size sliding window approach with a hashmap/set, explain how the window shrinks when a duplicate is found, and walk through an example to show why the algorithm is O(n) despite the inner loop</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Solve Search in Rotated Sorted Array in TypeScript — given a rotated sorted array and a target, find the target's index in O(log n). Show the modified binary search approach, explain how you determine which half is sorted at each step, and walk through the off-by-one decisions (why <= vs <, why mid vs mid+1/mid-1 for boundary updates)</summary>

<!-- Answer will be added later -->

</details>

## Practical — Linked List, Tree & Graph Problems

<details>
<summary>18. Design and implement an LRU Cache in TypeScript — support get(key) and put(key, value) both in O(1) time with a fixed capacity. Show the implementation using a hashmap and a doubly-linked list, explain why this specific combination of data structures is required (why not just a hashmap? why not just a linked list?), and handle the eviction logic when capacity is exceeded</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Solve Validate Binary Search Tree in TypeScript — given a binary tree, determine if it is a valid BST. Show both the range-based approach (passing min/max bounds) and the in-order traversal approach, explain why checking only immediate children is insufficient, and discuss the edge cases (integer overflow with min/max bounds, equal values)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Solve Number of Islands in TypeScript — given a 2D grid of '1's (land) and '0's (water), count the number of islands. Show the BFS or DFS approach with in-place marking, explain the visited-set pattern for grid problems, and discuss why BFS vs DFS doesn't matter for correctness here but might matter for stack depth on large grids</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Solve Course Schedule in TypeScript — given a number of courses and prerequisite pairs, determine if it's possible to finish all courses (detect if the dependency graph has a cycle). Show the topological sort approach using either Kahn's algorithm (BFS with in-degree tracking) or DFS with cycle detection, explain why topological sort requires a DAG, and how to detect cycles during the traversal</summary>

<!-- Answer will be added later -->

</details>

## Practical — Dynamic Programming Problems

<details>
<summary>22. Solve Climbing Stairs and House Robber in TypeScript — show the full DP progression for each: start with the naive recursive solution, add memoization, convert to bottom-up tabulation, then optimize space to O(1). Explain why these two problems share the same underlying pattern (each state depends on only the previous 1-2 states) and how recognizing this pattern transfers to other problems</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Solve Coin Change in TypeScript — given coin denominations and a target amount, find the minimum number of coins needed. Show the bottom-up DP approach, explain why this is an unbounded knapsack variant (each coin can be used unlimited times), walk through the recurrence relation, and explain what makes greedy fail for arbitrary denominations (why always picking the largest coin doesn't work)</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you chose the wrong data structure for a problem and had to refactor — what was the original choice, what went wrong (performance, correctness, complexity), how did you identify the issue, and what did you switch to?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Tell me about a time you optimized a slow algorithm or query in a production system — what were the symptoms, how did you identify the bottleneck, what algorithmic or data structure change did you make, and what was the measurable improvement?</summary>

<!-- Answer framework will be added later -->

</details>