# Data Structures & Algorithms

> **45 questions** — 21 theory, 24 practical

- Core data structures: arrays, linked lists, hashmaps, stacks, queues, trees, graphs, heaps, tries — selecting based on access patterns (random access, O(1) lookup, ordered data, priority)
- Big-O notation: time and space complexity, best/average/worst case, when Big-O misleads
- Problem-solving approach: constraint analysis (input size hints at complexity), brute force first, identify repeated work, optimize with the right data structure or pattern
- Hash maps: collisions, chaining vs open addressing, amortized O(1), degradation to O(n)
- Trees: binary trees, BSTs, balanced BSTs (AVL, red-black), tries — traversal patterns (in/pre/post/level-order), when to pick each
- Linked lists: tradeoffs vs arrays, cache locality penalty, pointer manipulation patterns, when linked structures actually win (constant-time insert/delete with known position)
- Heaps (priority queues): binary heap as array, heapify O(n), insert/extract O(log n), use cases — top-K problems, merge K sorted lists, scheduling
- Stacks and queues: nesting/matching problems, BFS/scheduling, monotonic stack/queue patterns
- Graph representations: adjacency list vs adjacency matrix, sparse vs dense tradeoffs
- Union-Find (Disjoint Set Union): path compression, union by rank, connected components, cycle detection
- Sorting: quicksort vs mergesort, stability, O(n log n) lower bound for comparison sorts, counting/radix sort for beating it
- Binary search: monotonic condition generalization, search space reduction, off-by-one pitfalls
- Space complexity analysis: recursive stack space, hidden costs (string concat, slice copies)
- Dynamic programming: overlapping subproblems, optimal substructure, recursion → memoization → tabulation → space optimization, greedy as an alternative when local choices are globally optimal
- Greedy algorithms: when local optimal choices yield global optimal, interval scheduling, activity selection — recognizing greedy vs DP problems
- Sliding window pattern: fixed and variable window, substring/subarray problems
- Two pointers pattern: sorted input, pair/triplet problems, duplicate skipping
- String patterns: palindrome checks, anagram grouping, string matching, encoding/decoding — common interview category with its own recurring techniques
- BFS/DFS on graphs: visited-set pattern, island problems, cycle detection, topological sort, backtracking for permutations/subsets
- Interval problems: merge/insert intervals, sweep line, sorting by start/end time
- Classic problems reference: arrays (Two Sum, 3Sum, Product Except Self, Buy/Sell Stock), strings (Group Anagrams, Longest Substring Without Repeating), linked lists (Merge/Reverse, LRU Cache), trees (Validate BST, Lowest Common Ancestor), graphs (Number of Islands, Course Schedule), DP (Climbing Stairs, House Robber, LCS, Coin Change), search (Rotated Array Search), stacks (Valid Parentheses), heaps (Top K Frequent)

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
<summary>3. When does Big-O analysis mislead you -- what are the situations where an algorithm with better Big-O performs worse in practice (constant factors, cache locality, input size thresholds), and how do you account for these factors when making real engineering decisions about algorithm choice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. What is a systematic approach to solving algorithm problems in an interview — why should you start with brute force, how does input size hint at the target time complexity, how do you identify repeated work in a brute-force solution and use the right data structure or pattern to eliminate it, and why does this process matter more than memorizing solutions?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>5. How do hash maps achieve amortized O(1) lookup — what happens during hash collisions, what is the difference between chaining and open addressing for resolving them, why does performance degrade toward O(n) in pathological cases, and what determines whether a hash map is the right choice vs a sorted structure like a BST?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why do we need balanced BSTs (AVL, red-black) when regular BSTs exist — what problem do they solve, how do they maintain O(log n) guarantees, how do tree traversal patterns (in-order, pre-order, post-order, level-order) map to different use cases, and when would you reach for a trie instead of a BST?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. Why do linked lists have a reputation for being slower than arrays in practice despite having O(1) insertion and deletion — what is the cache locality penalty, when do linked structures actually win over arrays (constant-time insert/delete with a known position), and what pointer manipulation patterns come up repeatedly in interview problems?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why are heaps (priority queues) implemented as arrays instead of pointer-based trees — how does the binary heap array representation work, why is heapify O(n) instead of O(n log n), and what problem categories (top-K, merge K sorted lists, scheduling) make heaps the natural choice over sorting or other structures?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What problem categories do stacks and queues naturally solve — why do nesting and matching problems (parentheses, expression evaluation) map to stacks, why does BFS use a queue, and what is the monotonic stack/queue pattern, what problems does it solve, and why does it achieve O(n) despite appearing nested?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. What are the tradeoffs between adjacency list and adjacency matrix representations for graphs — why does sparsity vs density determine the choice, how does each representation affect the time complexity of common operations (check if edge exists, iterate neighbors, iterate all edges), and which one do you default to in interviews and why?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. What problem does Union-Find (Disjoint Set Union) solve that BFS/DFS doesn't handle as efficiently — how do path compression and union by rank achieve near-O(1) amortized operations, and what are the classic use cases (connected components, cycle detection in undirected graphs) where Union-Find is the preferred approach?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Why is quicksort generally faster than mergesort in practice despite both being O(n log n) — what role do stability, cache locality, and space complexity play in choosing between them, why is O(n log n) the lower bound for comparison-based sorting, and when can you beat it with counting sort or radix sort?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why is binary search more general than just "searching a sorted array" — how does the monotonic condition generalization let you apply binary search to any problem with a search space you can halve, what are the common off-by-one pitfalls (lo <= hi vs lo < hi, mid calculation overflow), and how do you decide between left-biased and right-biased binary search?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Why is space complexity analysis often harder than time complexity — how do you account for recursive call stack depth, what are the hidden space costs in common operations (string concatenation creating copies, array slicing), and how do these hidden costs turn an apparently O(n) algorithm into O(n^2) space?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. What makes a problem solvable with dynamic programming — why do overlapping subproblems and optimal substructure matter, what is the progression from naive recursion to memoization to tabulation to space-optimized tabulation, and when should you use a greedy approach instead because local optimal choices happen to be globally optimal?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Why does the sliding window pattern turn O(n^2) brute-force substring/subarray problems into O(n) — how does a fixed window differ from a variable (shrinking) window, what invariant does the window maintain, and what signals in a problem statement tell you sliding window is the right approach?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. How does the two-pointer pattern exploit sorted input to avoid nested loops — why does it work for pair and triplet sum problems, how do you handle duplicate skipping to avoid redundant results, and what is the relationship between two pointers and binary search as alternative approaches to the same problems?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. How do BFS and DFS differ in their traversal behavior and when do you choose one over the other -- why does BFS find shortest paths in unweighted graphs while DFS is better for exhaustive exploration, how does the visited-set pattern prevent infinite loops in both, and what common mistakes do people make when implementing graph traversals?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. What is topological sort, why does it require a DAG, and how does backtracking use DFS to systematically generate permutations and subsets -- what is the "choose/explore/unchoose" pattern and why does it work?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Why do interval problems almost always start with sorting — how does sorting by start time (or end time) enable the merge intervals and insert interval patterns, what is the sweep line technique, and when does sorting by end time give you a different (and better) result than sorting by start time?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Why do greedy algorithms work for some optimization problems but fail catastrophically for others -- what properties must a problem have for a greedy approach to yield a globally optimal solution, how do you recognize whether a problem needs greedy vs dynamic programming (using examples like interval scheduling and activity selection), and what happens when you apply greedy to a problem that requires DP?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Array & HashMap Problems

<details>
<summary>22. Solve Two Sum in TypeScript — given an array of integers and a target, return the indices of two numbers that add up to the target. Show the brute-force O(n^2) approach first, then optimize to O(n) using a hashmap. Explain why the hashmap approach works and what tradeoff you're making (space for time)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Solve 3Sum in TypeScript — given an array of integers, find all unique triplets that sum to zero. Explain why sorting + two pointers is the standard approach, how you skip duplicates to avoid redundant triplets, and why the time complexity is O(n^2) — is it possible to do better?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Solve Group Anagrams in TypeScript — given an array of strings, group the anagrams together. Show how to design a hashmap key that identifies anagrams (sorted characters vs character frequency count), explain the tradeoffs between key strategies, and analyze the time complexity</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Solve Product of Array Except Self in TypeScript — given an array, return a new array where each element is the product of all other elements, without using division. Explain the prefix and suffix product approach, why it works, and how to optimize from O(n) extra space to O(1) extra space (excluding the output array)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Solve Top K Frequent Elements in TypeScript — given an array and k, return the k most frequent elements. Show at least two approaches (hashmap + min-heap, and bucket sort), explain why the heap approach is O(n log k) vs the bucket sort approach is O(n), and when each is preferable</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Solve Best Time to Buy and Sell Stock in TypeScript — given an array of prices, find the maximum profit from a single buy and sell. Show the single-pass O(n) approach tracking the minimum price seen so far, explain why this is a greedy solution and why DP would be overkill for the single-transaction variant</summary>

<!-- Answer will be added later -->

</details>

## Practical — String, Search & Stack Problems

<details>
<summary>28. Solve Longest Substring Without Repeating Characters in TypeScript — given a string, find the length of the longest substring without duplicate characters. Show the variable-size sliding window approach with a hashmap/set, explain how the window shrinks when a duplicate is found, and walk through an example to show why the algorithm is O(n) despite the inner loop</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Solve Valid Parentheses in TypeScript — given a string containing just the characters '(', ')', '{', '}', '[', ']', determine if the input string is valid. Show the stack-based approach, explain why a stack is the natural data structure for nesting problems, and handle the edge cases that catch people (empty stack pop, remaining elements after traversal)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Solve Search in Rotated Sorted Array in TypeScript — given a rotated sorted array and a target, find the target's index in O(log n). Show the modified binary search approach, explain how you determine which half is sorted at each step, and walk through the off-by-one decisions (why <= vs <, why mid vs mid+1/mid-1 for boundary updates)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. Solve a palindrome check problem in TypeScript -- given a string, determine the longest palindromic substring. Show the expand-around-center approach, explain why it is O(n^2) and why dynamic programming offers the same complexity with more space, and briefly discuss what other string pattern techniques (string matching with two pointers, encoding/decoding with stack or index tracking) come up repeatedly in interviews and why they form their own problem category</summary>

<!-- Answer will be added later -->

</details>

## Practical — Linked List, Tree & Graph Problems

<details>
<summary>32. Solve Merge Two Sorted Linked Lists in TypeScript — given two sorted linked lists, merge them into one sorted list. Show the iterative approach with a dummy head node, explain why the dummy head simplifies edge cases, and analyze why this is O(n + m) time and O(1) space</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>33. Solve Reverse a Linked List in TypeScript — reverse a singly linked list iteratively. Show the three-pointer approach (prev, current, next), walk through the pointer manipulation step by step, and explain why this is one of the most important linked list patterns to internalize since it appears as a subroutine in harder problems</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>34. Design and implement an LRU Cache in TypeScript — support get(key) and put(key, value) both in O(1) time with a fixed capacity. Show the implementation using a hashmap and a doubly-linked list, explain why this specific combination of data structures is required (why not just a hashmap? why not just a linked list?), and handle the eviction logic when capacity is exceeded</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>35. Solve Validate Binary Search Tree in TypeScript — given a binary tree, determine if it is a valid BST. Show both the range-based approach (passing min/max bounds) and the in-order traversal approach, explain why checking only immediate children is insufficient, and discuss the edge cases (integer overflow with min/max bounds, equal values)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>36. Solve Lowest Common Ancestor of a Binary Tree in TypeScript — given a binary tree and two nodes, find their lowest common ancestor. Show the recursive approach, explain why the recursion works (what each recursive call returns and how you combine results), and clarify the difference between solving this for a general binary tree vs a BST</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>37. Solve Number of Islands in TypeScript — given a 2D grid of '1's (land) and '0's (water), count the number of islands. Show the BFS or DFS approach with in-place marking, explain the visited-set pattern for grid problems, and discuss why BFS vs DFS doesn't matter for correctness here but might matter for stack depth on large grids</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>38. Solve Course Schedule in TypeScript — given a number of courses and prerequisite pairs, determine if it's possible to finish all courses (detect if the dependency graph has a cycle). Show the topological sort approach using either Kahn's algorithm (BFS with in-degree tracking) or DFS with cycle detection, explain why topological sort requires a DAG, and how to detect cycles during the traversal</summary>

<!-- Answer will be added later -->

</details>

## Practical — Dynamic Programming Problems

<details>
<summary>39. Solve Climbing Stairs and House Robber in TypeScript — show the full DP progression for each: start with the naive recursive solution, add memoization, convert to bottom-up tabulation, then optimize space to O(1). Explain why these two problems share the same underlying pattern (each state depends on only the previous 1-2 states) and how recognizing this pattern transfers to other problems</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>40. Solve Longest Common Subsequence in TypeScript — given two strings, find the length of their longest common subsequence. Show the 2D DP table approach, explain why this problem has optimal substructure and overlapping subproblems, walk through how to fill the table, and discuss whether space optimization is possible (reducing from O(m*n) to O(min(m,n)))</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>41. Solve Coin Change in TypeScript — given coin denominations and a target amount, find the minimum number of coins needed. Show the bottom-up DP approach, explain why this is an unbounded knapsack variant (each coin can be used unlimited times), walk through the recurrence relation, and explain what makes greedy fail for arbitrary denominations (why always picking the largest coin doesn't work)</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>42. Tell me about a time you chose the wrong data structure for a problem and had to refactor — what was the original choice, what went wrong (performance, correctness, complexity), how did you identify the issue, and what did you switch to?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>43. Tell me about a time you optimized a slow algorithm or query in a production system — what were the symptoms, how did you identify the bottleneck, what algorithmic or data structure change did you make, and what was the measurable improvement?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>44. Tell me about a time you applied a non-obvious algorithmic pattern (graph traversal, dynamic programming, topological sort) to solve a real engineering problem — not a LeetCode problem, but an actual system or feature requirement. What was the problem and why did the pattern fit?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>45. Tell me about a time you had to balance algorithmic correctness or optimality with practical constraints — maybe the optimal solution was too complex to maintain, too slow to implement under a deadline, or overkill for the actual data size. What tradeoff did you make and how did it turn out?</summary>

<!-- Answer framework will be added later -->

</details>
