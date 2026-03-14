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

Each data structure exists because it optimizes for a specific access pattern. Picking the right one means matching the structure to how your code actually reads, writes, and searches data.

| Data Structure | Optimized For | Key Operations |
|---|---|---|
| **Array** | Random access by index, sequential iteration | O(1) read/write by index, O(n) insert/delete in middle |
| **Linked List** | Frequent insertion/deletion at known positions | O(1) insert/delete at head/tail (with pointer), O(n) search |
| **HashMap** | Key-based lookup | Amortized O(1) get/set/delete, no ordering |
| **Stack** | Last-in-first-out (LIFO) access | O(1) push/pop/peek — undo operations, nesting, recursion |
| **Queue** | First-in-first-out (FIFO) access | O(1) enqueue/dequeue — scheduling, BFS, buffering |
| **Binary Search Tree** | Ordered data with fast search | O(log n) search/insert/delete (when balanced) |
| **Heap** | Quick access to min or max element | O(1) peek, O(log n) insert/extract — priority queues |
| **Graph** | Relationships between entities | Traversal depends on representation (adjacency list vs matrix) |
| **Trie** | Prefix-based string operations | O(L) lookup where L is string length — autocomplete, spell check |

**How to decide:** Ask what operations dominate your use case. If you need fast lookup by key, hashmap. If you need ordered iteration, BST or sorted array. If you need the smallest/largest element repeatedly, heap. If you need to model connections, graph.

**What goes wrong with the wrong choice:** Using an array when you need frequent lookups by value gives you O(n) scans instead of O(1) hashmap lookups. Using a linked list when you need random access forces O(n) traversal for every read. Using an unsorted structure when you need ordered output means sorting on every query. The wrong choice doesn't break correctness — it breaks performance, sometimes by orders of magnitude.

</details>

<details>
<summary>2. What does Big-O notation actually measure, why do we distinguish between best, average, and worst case, and how should you think about time vs space complexity tradeoffs when choosing between algorithms?</summary>

Big-O measures how an algorithm's resource usage **scales** as input size grows. It describes the upper bound of growth rate, ignoring constants and lower-order terms. O(2n + 5) simplifies to O(n) because at scale, the constant factors become irrelevant compared to the growth pattern.

**Why best/average/worst case matter:**

- **Best case** (Omega) is usually not useful for planning — it tells you the happy path but not what to expect.
- **Average case** is what you'll see in practice for typical inputs. Quicksort is O(n log n) average but O(n^2) worst.
- **Worst case** (Big-O) is what you design for when reliability matters. If your API has an SLA, you care about the worst case, not the average.

The distinction matters when choosing algorithms. HashMap lookup is O(1) average but O(n) worst case (hash collisions). If you need guaranteed performance (real-time systems), a balanced BST with O(log n) worst case might be better despite being slower on average.

**Time vs space tradeoffs:**

Most optimizations trade space for time. A classic example: caching or memoization uses extra memory to avoid recomputation. The decision depends on your constraints:

- **Memory is cheap, latency is expensive** (most web services): favor time optimization. Use hashmaps, caches, precomputed lookup tables.
- **Memory is constrained** (embedded systems, processing huge datasets): favor space-efficient algorithms even if slower. Use streaming algorithms, in-place sorting.
- **Both matter**: look for algorithms that are efficient in both, like merge sort for external sorting (O(n log n) time, sequential disk access).

**When Big-O misleads:** Constants matter at small n. An O(n log n) algorithm with huge constants can be slower than O(n^2) for small inputs — that's why many sorting libraries switch to insertion sort below ~10-20 elements. Also, Big-O ignores cache behavior: array iteration (cache-friendly) is much faster in practice than linked list traversal (cache-hostile) even at the same theoretical complexity.

</details>

<details>
<summary>3. What is a systematic approach to solving algorithm problems in an interview — why should you start with brute force, how does input size hint at the target time complexity, how do you identify repeated work in a brute-force solution and use the right data structure or pattern to eliminate it, and why does this process matter more than memorizing solutions?</summary>

**The systematic approach:**

1. **Clarify constraints**: Input size, value ranges, edge cases (empty input, single element, duplicates). This isn't just politeness — it drives your algorithm choice.
2. **Brute force first**: Write or describe the naive solution. This gives you a working baseline, shows the interviewer your understanding, and reveals where the inefficiency is.
3. **Identify the bottleneck**: Look at where time is wasted. Usually it's repeated lookups, redundant comparisons, or recomputing the same thing.
4. **Optimize with the right tool**: Pick a data structure or pattern that eliminates the bottleneck.
5. **Code it, test it**: Walk through with examples, check edge cases.

**Why start with brute force:** It proves correctness before optimization. It makes the inefficiency visible — you can point at the nested loop and say "this inner loop is scanning for something we could look up in O(1) with a hashmap." Jumping to the optimal solution without this step looks like memorization, not problem-solving.

**Input size as a complexity hint:**

| Input Size (n) | Target Complexity | Common Approaches |
|---|---|---|
| n <= 20 | O(2^n), O(n!) | Backtracking, brute force |
| n <= 500 | O(n^3) | Triple nested loops, DP with 2D table |
| n <= 10,000 | O(n^2) | Two nested loops, simple DP |
| n <= 1,000,000 | O(n log n) | Sorting, divide and conquer |
| n <= 10,000,000+ | O(n) or O(log n) | Linear scan, hashmap, binary search |

**Common bottleneck-to-fix mappings:**

- "Searching for something in an inner loop" -> hashmap for O(1) lookup
- "Checking all subarrays/substrings" -> sliding window
- "Trying all pairs in sorted data" -> two pointers
- "Recomputing overlapping results" -> memoization/DP
- "Need the min/max repeatedly" -> heap
- "Searching in sorted space" -> binary search

**Why process beats memorization:** Interviews throw variants. If you memorized Two Sum but get a rotated version, you're stuck. If you understand the process — brute force reveals an O(n) inner search, a hashmap eliminates it — you can adapt to any variant. The process transfers; the solution doesn't.

</details>

## Conceptual Depth

<details>
<summary>4. How do hash maps achieve amortized O(1) lookup — what happens during hash collisions, what is the difference between chaining and open addressing for resolving them, why does performance degrade toward O(n) in pathological cases, and what determines whether a hash map is the right choice vs a sorted structure like a BST?</summary>

**How amortized O(1) works:** A hash function converts a key into an array index. If the hash function distributes keys uniformly across buckets, each bucket holds roughly 1 entry, making lookup a constant-time array access. "Amortized" accounts for occasional resizing — when the load factor (entries/buckets) exceeds a threshold (typically 0.75), the map doubles its bucket array and rehashes everything. This O(n) resize happens infrequently enough that averaged over all operations, each one is still O(1).

**Hash collisions** happen when two different keys hash to the same bucket index. Two strategies:

- **Chaining**: Each bucket holds a linked list (or tree at high collision counts — Java's HashMap switches to a red-black tree at 8 entries per bucket). On collision, append to the list. Lookup walks the list at that bucket. Simple, but cache-unfriendly due to pointer chasing.
- **Open addressing**: On collision, probe for the next empty slot using a strategy (linear probing, quadratic probing, double hashing). All data lives in the array itself — better cache locality, but deletion is tricky (requires tombstone markers) and performance degrades faster as load factor increases.

**O(n) degradation:** If the hash function is poor (many keys map to the same bucket) or an adversary crafts inputs to cause collisions, all entries end up in one bucket. Chaining degrades to a linked list scan — O(n). This is why languages use randomized hash seeds (to prevent hash flooding attacks) and why Java switches chaining to balanced trees after a threshold.

**HashMap vs BST (sorted structure):**

| Need | HashMap | BST (balanced) |
|---|---|---|
| Lookup by key | O(1) average | O(log n) |
| Ordered iteration | Not supported | Natural (in-order traversal) |
| Range queries ("all keys between A and B") | Not supported | O(log n + k) where k = results |
| Worst-case guarantee | O(n) unless mitigated | O(log n) always |
| Find min/max | O(n) | O(log n) |

**Rule of thumb:** Use a hashmap when you only need key-value lookup and don't care about order. Use a BST (or sorted structure) when you need ordered operations — range queries, finding the closest key, iterating in sorted order.

</details>

<details>
<summary>5. Why do we need balanced BSTs (AVL, red-black) when regular BSTs exist — what problem do they solve, how do they maintain O(log n) guarantees, how do tree traversal patterns (in-order, pre-order, post-order, level-order) map to different use cases, and when would you reach for a trie instead of a BST?</summary>

**The problem with regular BSTs:** If you insert sorted data into a plain BST, it degenerates into a linked list — every node is a right child. Search becomes O(n) instead of O(log n). The tree's performance depends entirely on insertion order, which you often can't control.

**How balanced BSTs fix this:**

- **AVL trees** enforce a strict balance invariant: for every node, the heights of left and right subtrees differ by at most 1. After each insert/delete, rotations restore balance. This gives O(log n) worst case for all operations but requires more rotations on write.
- **Red-black trees** use a looser invariant (no two consecutive red nodes, equal black-height on all paths). Less strictly balanced than AVL, so lookups are slightly slower in theory, but fewer rotations on insert/delete. This makes them better for write-heavy workloads — that's why most standard library implementations (C++ `std::map`, Java `TreeMap`) use red-black trees.

Both guarantee O(log n) search, insert, and delete regardless of insertion order.

**Tree traversal patterns:**

| Traversal | Order | Use Case |
|---|---|---|
| **In-order** (left, root, right) | Sorted output | Print BST in sorted order, validate BST |
| **Pre-order** (root, left, right) | Root first | Serialize/copy a tree (root before children) |
| **Post-order** (left, right, root) | Children first | Delete a tree (children before parent), evaluate expression trees |
| **Level-order** (BFS) | Breadth-first | Find depth, print level by level, shortest path in tree |

**When to use a trie instead of a BST:**

Tries are purpose-built for string prefix operations. A BST stores whole keys and compares them — looking up a string of length L costs O(L log n) because each of the O(log n) comparisons takes O(L). A trie stores one character per node along the path, so lookup is always O(L) regardless of how many strings are stored. Use a trie for:

- **Autocomplete/prefix search**: "find all words starting with 'pre'" is a natural trie traversal.
- **Spell checking**: Check if a word exists in a dictionary.
- **IP routing tables**: Longest prefix match.

The tradeoff: tries use more memory (one node per character, with pointers for each possible child) compared to a BST that stores whole keys.

</details>

<details>
<summary>6. Why are heaps (priority queues) implemented as arrays instead of pointer-based trees — how does the binary heap array representation work, why is heapify O(n) instead of O(n log n), and what problem categories (top-K, merge K sorted lists, scheduling) make heaps the natural choice over sorting or other structures?</summary>

**Why arrays instead of pointers:** A binary heap is a **complete** binary tree — every level is full except possibly the last, which fills left to right. This property means there are no gaps, so it maps perfectly to a contiguous array with no wasted space. For a node at index `i`:

- Left child: `2i + 1`
- Right child: `2i + 2`
- Parent: `Math.floor((i - 1) / 2)`

No pointers needed — just arithmetic. This gives excellent cache locality (contiguous memory), zero pointer overhead, and simpler implementation.

**Why heapify is O(n), not O(n log n):**

The naive analysis says: n nodes, each might sift down O(log n) levels = O(n log n). But most nodes are near the bottom. In a complete binary tree, ~n/2 nodes are leaves (sift down 0 levels), ~n/4 are one level up (sift down 1 level), ~n/8 are two levels up (sift down 2 levels), and so on. The sum is:

`n/2 * 0 + n/4 * 1 + n/8 * 2 + ... = n * sum(k/2^(k+1))` which converges to O(n).

The key insight: **most nodes do very little work** because they're already near the bottom. Building bottom-up exploits this — you start sifting from the last non-leaf node upward.

**Problem categories where heaps shine:**

- **Top-K problems**: Maintain a min-heap of size K. For each element, if it's larger than the heap's minimum, swap it in. Final heap contains the K largest. O(n log k) — much better than sorting O(n log n) when k is small.
- **Merge K sorted lists**: Put the head of each list into a min-heap. Extract the minimum, advance that list, insert the next element. O(n log k) where n is total elements.
- **Scheduling / event-driven simulation**: Process the next event by deadline/priority. Insert new events as they arrive. Heap gives O(log n) for both operations.
- **Median finding (two heaps)**: A max-heap for the lower half and min-heap for the upper half gives O(log n) insertion and O(1) median access.

**Why not just sort?** Sorting gives you everything in order but costs O(n log n) upfront and doesn't handle dynamic data. A heap handles a stream of inserts and extracts efficiently. If you only need the top K from a fixed dataset and K is close to n, sorting might be simpler.

</details>

<details>
<summary>7. What problem categories do stacks and queues naturally solve — why do nesting and matching problems (parentheses, expression evaluation) map to stacks, why does BFS use a queue, and what is the monotonic stack/queue pattern, what problems does it solve, and why does it achieve O(n) despite appearing nested?</summary>

**Why stacks solve nesting/matching problems:**

Nesting has a LIFO structure — the most recently opened bracket must be the first one closed. When you see `({[`, the `[` must close before the `{`, which must close before the `(`. A stack naturally mirrors this: push on open, pop and check on close. Expression evaluation works the same way — in `2 * (3 + 4)`, you push partial results and operators, evaluate when you hit a closing paren. Undo/redo, function call stacks, and DFS all share this "most recent first" pattern.

**Why BFS uses a queue:**

BFS explores nodes level by level — process all nodes at distance 1 before distance 2. FIFO ordering guarantees this: when you discover neighbors, they go to the back of the queue, so they're processed after all current-level nodes. This is why BFS finds shortest paths in unweighted graphs — the first time you reach a node, it's via the fewest edges.

**Monotonic stack/queue pattern:**

A monotonic stack maintains elements in sorted order (increasing or decreasing). When a new element arrives, you pop all elements that violate the monotonic property before pushing. This pattern solves "next greater/smaller element" problems.

```typescript
// Next Greater Element: for each element, find the first larger element to its right
function nextGreaterElement(nums: number[]): number[] {
  const result = new Array(nums.length).fill(-1);
  const stack: number[] = []; // stores indices, values are decreasing

  for (let i = 0; i < nums.length; i++) {
    // Pop all elements smaller than current — current IS their next greater
    while (stack.length > 0 && nums[stack[stack.length - 1]] < nums[i]) {
      result[stack.pop()!] = nums[i];
    }
    stack.push(i);
  }
  return result;
}
```

**Why it's O(n) despite the inner while loop:** Each element is pushed onto the stack exactly once and popped at most once. The total number of push + pop operations across the entire loop is at most 2n. The inner loop doesn't restart from scratch — it continues from where it left off. This amortized analysis is the key: the while loop looks nested, but total work is bounded.

**Problems solved by monotonic stacks:** next greater/smaller element, largest rectangle in histogram, trapping rain water, daily temperatures, stock span.

</details>

<details>
<summary>8. Why is binary search more general than just "searching a sorted array" — how does the monotonic condition generalization let you apply binary search to any problem with a search space you can halve, what are the common off-by-one pitfalls (lo <= hi vs lo < hi, mid calculation overflow), and how do you decide between left-biased and right-biased binary search?</summary>

**The generalization:** Binary search works whenever you have a **monotonic condition** — a predicate that is false for some prefix of the search space and true for the rest (or vice versa). The sorted array is just one instance. The real pattern is: "given a search space, can I check a midpoint and definitively eliminate half?"

Examples beyond sorted arrays:
- **Finding square root**: Is `mid * mid <= target`? Binary search over integers.
- **Minimum capacity to ship packages in D days**: Can we ship everything with capacity `mid`? Binary search over possible capacities.
- **Finding the first bad version**: Is version `mid` bad? Binary search over version numbers.

**Off-by-one pitfalls:**

1. **`lo <= hi` vs `lo < hi`:** Use `lo <= hi` when the answer could be at any position and you're returning from inside the loop (classic "find exact target"). Use `lo < hi` when you're narrowing to a single position and the answer is `lo` after the loop (classic "find boundary"). Mixing these up causes infinite loops or missed elements.

2. **Mid calculation:** `Math.floor((lo + hi) / 2)` can overflow in languages with fixed-size integers (not an issue in JS/TS with normal numbers, but matters in Java/C++). Safe version: `lo + Math.floor((hi - lo) / 2)`.

3. **Boundary updates:** After checking `mid`, you must move `lo` or `hi` past `mid` to make progress. If `lo = mid` (instead of `mid + 1`) with `lo < hi` and floor division, the range never shrinks when `hi = lo + 1` — infinite loop.

**Left-biased vs right-biased:**

```typescript
// Left-biased: find first position where condition is true
// lo converges to the leftmost valid answer
function lowerBound(nums: number[], target: number): number {
  let lo = 0, hi = nums.length;
  while (lo < hi) {
    const mid = lo + Math.floor((hi - lo) / 2);
    if (nums[mid] < target) lo = mid + 1;
    else hi = mid; // mid might be the answer, don't skip it
  }
  return lo;
}

// Right-biased: find last position where condition is true
// hi converges to the rightmost valid answer
function upperBound(nums: number[], target: number): number {
  let lo = 0, hi = nums.length;
  while (lo < hi) {
    const mid = lo + Math.floor((hi - lo) / 2);
    if (nums[mid] <= target) lo = mid + 1; // mid satisfied, but look further right
    else hi = mid;
  }
  return lo - 1; // last position where nums[i] <= target
}
```

**When to use which:** Left-biased when you want the first occurrence or the leftmost boundary. Right-biased when you want the last occurrence or the rightmost boundary. The difference is always just whether `==` goes into the `lo = mid + 1` branch (right-biased) or the `hi = mid` branch (left-biased).

</details>

<details>
<summary>9. What makes a problem solvable with dynamic programming — why do overlapping subproblems and optimal substructure matter, what is the progression from naive recursion to memoization to tabulation to space-optimized tabulation, and when should you use a greedy approach instead because local optimal choices happen to be globally optimal?</summary>

**Two conditions for DP:**

1. **Optimal substructure**: The optimal solution to the problem can be built from optimal solutions to its subproblems. For example, the shortest path from A to C through B = shortest path A to B + shortest path B to C.
2. **Overlapping subproblems**: The same subproblems get solved multiple times. Fibonacci's recursive tree calls `fib(3)` multiple times. Without overlap, you just have divide-and-conquer (merge sort) — no need to cache.

If you have optimal substructure but NO overlap, divide-and-conquer suffices. If subproblems overlap, DP avoids redundant computation by caching results.

**The DP progression:**

```typescript
// 1. Naive recursion — exponential, recalculates everything
function fib(n: number): number {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2); // O(2^n) — fib(3) computed many times
}

// 2. Memoization (top-down) — add a cache, same recursive structure
function fibMemo(n: number, memo = new Map<number, number>()): number {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n)!;
  const result = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, result);
  return result; // O(n) time, O(n) space
}

// 3. Tabulation (bottom-up) — iterative, fill table from base cases up
function fibTab(n: number): number {
  if (n <= 1) return n;
  const dp = new Array(n + 1);
  dp[0] = 0;
  dp[1] = 1;
  for (let i = 2; i <= n; i++) dp[i] = dp[i - 1] + dp[i - 2];
  return dp[n]; // O(n) time, O(n) space, no recursion overhead
}

// 4. Space-optimized — only keep what you need
function fibOpt(n: number): number {
  if (n <= 1) return n;
  let prev2 = 0, prev1 = 1;
  for (let i = 2; i <= n; i++) [prev2, prev1] = [prev1, prev2 + prev1];
  return prev1; // O(n) time, O(1) space
}
```

**When to use each step:**
- Start with recursion to understand the recurrence relation.
- Add memoization if it's correct — this is often enough for an interview.
- Convert to tabulation when you want to eliminate recursion stack overhead or when the recursive structure is straightforward to invert.
- Optimize space when the recurrence only depends on a fixed number of previous states.

**When greedy works instead:**

Greedy applies when making the locally optimal choice at each step leads to the globally optimal solution — no need to explore all subproblems. Examples:
- **Activity selection**: Always pick the activity that finishes earliest. Provably optimal.
- **Huffman coding**: Always merge the two lowest-frequency nodes.
- **Coin change with standard denominations** (1, 5, 10, 25): Picking the largest coin first works because each denomination divides the larger ones.

Greedy fails when local choices don't guarantee global optimality. Coin change with denominations [1, 3, 4] and target 6: greedy picks 4+1+1=3 coins, but optimal is 3+3=2 coins. When you can't prove the greedy choice property, use DP.

</details>

<details>
<summary>10. Why does the sliding window pattern turn O(n^2) brute-force substring/subarray problems into O(n) — how does a fixed window differ from a variable (shrinking) window, what invariant does the window maintain, and what signals in a problem statement tell you sliding window is the right approach?</summary>

**Why it turns O(n^2) into O(n):**

The brute force for "find the best subarray/substring" checks all possible start/end combinations — O(n^2) pairs. Sliding window avoids this by maintaining a window `[left, right]` and only moving each pointer forward, never backward. Each element enters the window once (right moves) and leaves once (left moves), so total work is O(2n) = O(n).

**Fixed window vs variable window:**

- **Fixed window**: Window size is known upfront (e.g., "maximum sum of k consecutive elements"). Slide right by one, add the new element, remove the leftmost. Simple and always O(n).

```typescript
// Maximum sum subarray of size k
function maxSumFixed(nums: number[], k: number): number {
  let windowSum = 0;
  for (let i = 0; i < k; i++) windowSum += nums[i];
  let maxSum = windowSum;
  for (let i = k; i < nums.length; i++) {
    windowSum += nums[i] - nums[i - k]; // slide: add right, remove left
    maxSum = Math.max(maxSum, windowSum);
  }
  return maxSum;
}
```

- **Variable (shrinking) window**: Window size adjusts dynamically based on a condition. Expand right to explore, shrink left when the window violates the invariant. This is the more common and harder pattern.

```typescript
// Minimum length subarray with sum >= target
function minSubArrayLen(target: number, nums: number[]): number {
  let left = 0, sum = 0, minLen = Infinity;
  for (let right = 0; right < nums.length; right++) {
    sum += nums[right];
    while (sum >= target) { // shrink while invariant is satisfied
      minLen = Math.min(minLen, right - left + 1);
      sum -= nums[left++];
    }
  }
  return minLen === Infinity ? 0 : minLen;
}
```

**The invariant:** The window always represents a valid candidate (or is being adjusted to become one). For "longest substring without repeats," the invariant is "no duplicate characters in the window." When right expansion breaks it, left shrinks until it's restored.

**Signals that sliding window applies:**

- Problem mentions **contiguous** subarray or substring (not subsequence).
- Looking for **longest/shortest/maximum/minimum** over contiguous elements.
- There's a **constraint** that can be maintained incrementally (sum, character count, distinct count) as the window moves — not something that requires recalculating from scratch.
- The constraint is **monotonic with window size**: making the window bigger can only increase (or maintain) a sum/count, making it smaller can only decrease it. This monotonicity is why the left pointer never needs to move backward.

</details>

<details>
<summary>11. How does the two-pointer pattern exploit sorted input to avoid nested loops — why does it work for pair and triplet sum problems, how do you handle duplicate skipping to avoid redundant results, and what is the relationship between two pointers and binary search as alternative approaches to the same problems?</summary>

**Why it works on sorted input:**

With sorted data, two pointers at opposite ends can converge toward the answer. If `nums[lo] + nums[hi]` is too small, moving `lo` right increases the sum. If too large, moving `hi` left decreases it. Each step eliminates one candidate, so you scan all pairs in O(n) instead of O(n^2).

```typescript
// Two Sum on sorted array — O(n)
function twoSumSorted(nums: number[], target: number): [number, number] {
  let lo = 0, hi = nums.length - 1;
  while (lo < hi) {
    const sum = nums[lo] + nums[hi];
    if (sum === target) return [lo, hi];
    if (sum < target) lo++;
    else hi--;
  }
  return [-1, -1];
}
```

For **triplet sum** (3Sum), fix one element with an outer loop and run two pointers on the remaining sorted subarray. This gives O(n) per fixed element, O(n^2) total — covered in detail in question 14.

**Duplicate skipping:**

When the problem asks for unique results (like 3Sum), you skip over consecutive identical values to avoid producing the same combination twice:

```typescript
// After finding a valid pair at lo, hi:
while (lo < hi && nums[lo] === nums[lo + 1]) lo++;   // skip duplicate lo values
while (lo < hi && nums[hi] === nums[hi - 1]) hi--;   // skip duplicate hi values
lo++;
hi--;
```

The same applies to the outer loop in 3Sum — if `nums[i] === nums[i - 1]`, skip `i` entirely.

**Two pointers vs binary search:**

Both solve "find a complement in sorted data" problems. For two-sum on a sorted array:
- **Two pointers**: O(n) time, O(1) space. One pass from both ends.
- **Binary search**: For each element, binary search for its complement. O(n log n) time, O(1) space.

Two pointers is faster when you need to find all pairs or when processing the entire array. Binary search is better when you have a single query against a sorted dataset, or when the search space isn't a simple array (as covered in question 8). In practice, if the input is sorted and you're scanning for pairs, two pointers is the default choice.

</details>

<details>
<summary>12. How do BFS and DFS differ in their traversal behavior and when do you choose one over the other -- why does BFS find shortest paths in unweighted graphs while DFS is better for exhaustive exploration, how does the visited-set pattern prevent infinite loops in both, and what common mistakes do people make when implementing graph traversals?</summary>

**Core difference:**

- **BFS** (queue-based) explores level by level — all neighbors at distance 1, then distance 2, etc. It finds the shortest path in unweighted graphs because the first time it reaches a node is guaranteed to be via the minimum number of edges.
- **DFS** (stack/recursion-based) goes as deep as possible before backtracking. It's natural for exhaustive exploration — generating all permutations, detecting cycles, topological sort, finding connected components.

**When to choose which:**

| Use Case | BFS | DFS |
|---|---|---|
| Shortest path (unweighted) | Yes — guaranteed | No — may find longer path first |
| Level-order traversal | Yes | No |
| Detect if path exists | Either works | Either works |
| Topological sort | Kahn's algorithm (BFS variant) | Natural with post-order |
| Cycle detection | Possible but awkward | Natural (back-edge detection) |
| Memory on wide graphs | High (stores entire frontier) | Lower (stores single path) |
| Memory on deep graphs | Lower | High (recursion stack risk) |

**The visited-set pattern:**

Graphs can have cycles. Without tracking visited nodes, BFS/DFS revisit the same node infinitely. The pattern:

```typescript
// BFS with visited set
function bfs(graph: Map<number, number[]>, start: number): void {
  const visited = new Set<number>([start]);
  const queue = [start];

  while (queue.length > 0) {
    const node = queue.shift()!;
    for (const neighbor of graph.get(node) ?? []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);  // mark BEFORE enqueueing to prevent duplicates in queue
        queue.push(neighbor);
      }
    }
  }
}
```

**Common mistakes:**

1. **Marking visited too late (BFS)**: If you mark a node as visited when you dequeue it (instead of when you enqueue it), the same node can be enqueued multiple times by different neighbors. This wastes time and can cause incorrect distance calculations.

2. **Forgetting visited on DFS for graphs**: Trees don't need a visited set (no cycles), but graphs do. Copying tree DFS code to a graph problem causes infinite recursion.

3. **Using `queue.shift()` in JS for BFS**: `Array.shift()` is O(n) because it reindexes the array. For large graphs, use a proper queue (linked list or index-based). For interview-sized inputs it's fine, but worth mentioning.

4. **Not handling disconnected components**: If the graph isn't fully connected, a single BFS/DFS from one node won't visit everything. You need an outer loop over all nodes: `for each node, if not visited, start BFS/DFS`.

5. **Confusing directed vs undirected cycle detection in DFS**: In undirected graphs, a back edge to any visited node (except the parent) means a cycle. In directed graphs, you need to distinguish between nodes in the current recursion stack vs already-completed nodes — visiting a completed node isn't a cycle, only visiting one currently on the stack is.

</details>

## Practical — Array & HashMap Problems

<details>
<summary>13. Solve Two Sum in TypeScript — given an array of integers and a target, return the indices of two numbers that add up to the target. Show the brute-force O(n^2) approach first, then optimize to O(n) using a hashmap. Explain why the hashmap approach works and what tradeoff you're making (space for time)</summary>

**Brute force — O(n^2) time, O(1) space:**

```typescript
function twoSumBrute(nums: number[], target: number): [number, number] {
  for (let i = 0; i < nums.length; i++) {
    for (let j = i + 1; j < nums.length; j++) {
      if (nums[i] + nums[j] === target) return [i, j];
    }
  }
  return [-1, -1];
}
```

The bottleneck: for each element, we scan the rest of the array looking for its complement (`target - nums[i]`). That inner scan is the repeated work we can eliminate.

**Optimized — O(n) time, O(n) space:**

```typescript
function twoSum(nums: number[], target: number): [number, number] {
  const seen = new Map<number, number>(); // value -> index

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (seen.has(complement)) return [seen.get(complement)!, i];
    seen.set(nums[i], i);
  }
  return [-1, -1];
}
```

**Why it works:** Instead of scanning forward for the complement, we store every number we've seen in a hashmap. When we reach a new number, we check in O(1) if its complement was already encountered. The single pass handles everything — we never need to look ahead because if `(a, b)` is a valid pair, we'll find it when we process whichever comes second.

**The tradeoff:** We're trading O(n) space (the hashmap) for O(n) time improvement (eliminating the inner loop). This is the canonical example of the space-for-time tradeoff described in question 2. In most real-world contexts, the extra memory is negligible compared to the performance gain.

</details>

<details>
<summary>14. Solve 3Sum in TypeScript — given an array of integers, find all unique triplets that sum to zero. Explain why sorting + two pointers is the standard approach, how you skip duplicates to avoid redundant triplets, and why the time complexity is O(n^2) — is it possible to do better?</summary>

**Why sorting + two pointers:** Sorting costs O(n log n) but enables the two-pointer technique from question 11. For each fixed element `nums[i]`, run two pointers on the remaining array to find pairs summing to `-nums[i]`. This reduces the problem to n instances of two-sum on sorted data, each O(n).

```typescript
function threeSum(nums: number[]): number[][] {
  nums.sort((a, b) => a - b);
  const result: number[][] = [];

  for (let i = 0; i < nums.length - 2; i++) {
    if (nums[i] > 0) break; // sorted — no way three positives sum to zero
    if (i > 0 && nums[i] === nums[i - 1]) continue; // skip duplicate outer element

    let lo = i + 1, hi = nums.length - 1;
    while (lo < hi) {
      const sum = nums[i] + nums[lo] + nums[hi];
      if (sum === 0) {
        result.push([nums[i], nums[lo], nums[hi]]);
        while (lo < hi && nums[lo] === nums[lo + 1]) lo++;   // skip duplicate lo
        while (lo < hi && nums[hi] === nums[hi - 1]) hi--;   // skip duplicate hi
        lo++;
        hi--;
      } else if (sum < 0) {
        lo++;
      } else {
        hi--;
      }
    }
  }
  return result;
}
```

**Duplicate skipping explained:**

Three levels of deduplication:
1. **Outer loop**: `if (nums[i] === nums[i - 1]) continue` — don't fix the same value twice as the first element.
2. **After finding a match**: Skip `lo` past identical values and `hi` past identical values. Without this, `[-2, 0, 0, 2, 2]` would produce `[-2, 0, 2]` multiple times.
3. **Early termination**: `if (nums[i] > 0) break` — if the smallest remaining element is positive, no triplet can sum to zero.

**Time complexity:** O(n log n) for sort + O(n) outer loop * O(n) two-pointer inner = O(n^2). This dominates, so total is O(n^2).

**Can we do better than O(n^2)?** Not significantly. No known algorithm solves 3Sum faster than O(n^2) for the general case — O(n^2) is considered near-optimal. The sorting + two-pointer approach matches this bound and is the standard interview solution.

</details>

<details>
<summary>15. Solve Top K Frequent Elements in TypeScript — given an array and k, return the k most frequent elements. Show at least two approaches (hashmap + min-heap, and bucket sort), explain why the heap approach is O(n log k) vs the bucket sort approach is O(n), and when each is preferable</summary>

Both approaches start the same way: build a frequency map in O(n).

**Approach 1: Hashmap + Min-Heap — O(n log k)**

Maintain a min-heap of size k. For each unique element, if the heap has fewer than k elements, push. Otherwise, if the current frequency exceeds the heap's minimum, replace it. After processing all elements, the heap contains the top k.

```typescript
// Simplified min-heap with sift-up/sift-down (no sort() — real O(log k) operations)
function topKFrequentHeap(nums: number[], k: number): number[] {
  const freq = new Map<number, number>();
  for (const n of nums) freq.set(n, (freq.get(n) ?? 0) + 1);

  // Min-heap of [element, frequency], ordered by frequency
  const heap: [number, number][] = [];

  function siftUp(i: number): void {
    while (i > 0) {
      const parent = Math.floor((i - 1) / 2);
      if (heap[parent][1] <= heap[i][1]) break;
      [heap[parent], heap[i]] = [heap[i], heap[parent]];
      i = parent;
    }
  }

  function siftDown(i: number): void {
    const n = heap.length;
    while (2 * i + 1 < n) {
      let smallest = 2 * i + 1;
      if (smallest + 1 < n && heap[smallest + 1][1] < heap[smallest][1]) smallest++;
      if (heap[i][1] <= heap[smallest][1]) break;
      [heap[i], heap[smallest]] = [heap[smallest], heap[i]];
      i = smallest;
    }
  }

  for (const [num, count] of freq) {
    if (heap.length < k) {
      heap.push([num, count]);
      siftUp(heap.length - 1);
    } else if (count > heap[0][1]) {
      heap[0] = [num, count]; // replace min
      siftDown(0);
    }
  }

  return heap.map(([num]) => num);
}
```

Why O(n log k): n unique elements processed, each heap insert or replace is O(log k) via sift-up/sift-down. Since k is often much smaller than n, this is significantly better than sorting all elements O(n log n).

**Approach 2: Bucket Sort — O(n)**

The maximum possible frequency is n (all elements identical). Create an array of n+1 buckets where `buckets[i]` holds elements with frequency i. Then iterate from the highest bucket downward, collecting k elements.

```typescript
function topKFrequentBucket(nums: number[], k: number): number[] {
  const freq = new Map<number, number>();
  for (const n of nums) freq.set(n, (freq.get(n) ?? 0) + 1);

  // buckets[i] = elements that appear exactly i times
  const buckets: number[][] = Array.from({ length: nums.length + 1 }, () => []);
  for (const [num, count] of freq) buckets[count].push(num);

  const result: number[] = [];
  for (let i = buckets.length - 1; i >= 0 && result.length < k; i--) {
    result.push(...buckets[i]);
  }
  return result.slice(0, k);
}
```

Why O(n): Building the frequency map is O(n). Creating and filling buckets is O(n) (one entry per unique element). Collecting results is O(n) worst case. No sorting or heap operations.

**When to use which:**

- **Bucket sort** is better when you need guaranteed O(n) and can afford O(n) extra space for the bucket array. It's simpler to implement and faster.
- **Heap** is better when the data is streaming (you don't know all elements upfront) or when k is very small relative to n — the heap uses only O(k) space vs O(n) for buckets. It also generalizes to "top K by any metric" problems.
- In interviews, the bucket sort approach is usually preferred for this specific problem because it's O(n) and demonstrates creative use of the frequency-as-index insight.

</details>

## Practical — String, Search & Stack Problems

<details>
<summary>16. Solve Longest Substring Without Repeating Characters in TypeScript — given a string, find the length of the longest substring without duplicate characters. Show the variable-size sliding window approach with a hashmap/set, explain how the window shrinks when a duplicate is found, and walk through an example to show why the algorithm is O(n) despite the inner loop</summary>

This is the classic variable-window problem from question 10 applied to strings. The invariant: the window `[left, right]` contains no duplicate characters.

```typescript
function lengthOfLongestSubstring(s: string): number {
  const lastSeen = new Map<string, number>(); // char -> most recent index
  let maxLen = 0;
  let left = 0;

  for (let right = 0; right < s.length; right++) {
    const char = s[right];
    if (lastSeen.has(char) && lastSeen.get(char)! >= left) {
      // Duplicate found within current window — jump left past the previous occurrence
      left = lastSeen.get(char)! + 1;
    }
    lastSeen.set(char, right);
    maxLen = Math.max(maxLen, right - left + 1);
  }
  return maxLen;
}
```

**How the window shrinks:** When `right` lands on a character already in the window, we move `left` to one position past that character's last occurrence. The check `lastSeen.get(char)! >= left` is critical — without it, we'd react to characters from before the current window (already excluded by a previous shrink).

**Walkthrough with `"abcabcbb"`:**

| right | char | lastSeen | left | window | maxLen |
|---|---|---|---|---|---|
| 0 | a | {a:0} | 0 | "a" | 1 |
| 1 | b | {a:0,b:1} | 0 | "ab" | 2 |
| 2 | c | {a:0,b:1,c:2} | 0 | "abc" | 3 |
| 3 | a | {a:3,b:1,c:2} | 1 | "bca" | 3 |
| 4 | b | {a:3,b:4,c:2} | 2 | "cab" | 3 |
| 5 | c | {a:3,b:4,c:5} | 3 | "abc" | 3 |
| 6 | b | {a:3,b:6,c:5} | 5 | "cb" | 3 |
| 7 | b | {a:3,b:7,c:5} | 7 | "b" | 3 |

Result: 3 (`"abc"`).

**Why O(n):** The `right` pointer advances exactly n times. The `left` pointer only moves forward — it never goes backward. Using a hashmap (instead of a set with a while loop to shrink), we jump `left` directly to the right position in O(1), so there's no inner loop at all. Even with the set-based approach where `left` advances one step at a time, each character is added and removed from the set at most once, making total operations at most 2n = O(n).

</details>

<details>
<summary>17. Solve Search in Rotated Sorted Array in TypeScript — given a rotated sorted array and a target, find the target's index in O(log n). Show the modified binary search approach, explain how you determine which half is sorted at each step, and walk through the off-by-one decisions (why <= vs <, why mid vs mid+1/mid-1 for boundary updates)</summary>

**Key insight:** In a rotated sorted array, at least one half (left or right of mid) is always sorted. Determine which half is sorted, then check if the target falls within that sorted range. If yes, search there. If no, search the other half.

```typescript
function search(nums: number[], target: number): number {
  let lo = 0, hi = nums.length - 1;

  while (lo <= hi) {
    const mid = lo + Math.floor((hi - lo) / 2);

    if (nums[mid] === target) return mid;

    // Determine which half is sorted
    if (nums[lo] <= nums[mid]) {
      // Left half [lo..mid] is sorted
      if (nums[lo] <= target && target < nums[mid]) {
        hi = mid - 1; // target is in the sorted left half
      } else {
        lo = mid + 1; // target is in the right half
      }
    } else {
      // Right half [mid..hi] is sorted
      if (nums[mid] < target && target <= nums[hi]) {
        lo = mid + 1; // target is in the sorted right half
      } else {
        hi = mid - 1; // target is in the left half
      }
    }
  }
  return -1;
}
```

**Off-by-one decisions explained:**

1. **`lo <= hi` (not `lo < hi`):** We use `lo <= hi` because we return from inside the loop when we find the target. If we used `lo < hi`, we'd miss the case where `lo === hi` and that single element is the target.

2. **`nums[lo] <= nums[mid]` (not `<`):** The `=` handles the case where `lo === mid` (only 1-2 elements remaining). Without `=`, we'd incorrectly treat a single-element left half as unsorted.

3. **`target < nums[mid]` (strict `<` on the mid side):** We already checked `nums[mid] === target` above, so mid itself is excluded. The range we're checking is `[lo, mid-1]`.

4. **`target <= nums[hi]` (inclusive `<=` on the boundary):** The target could equal the boundary element. If `target === nums[hi]`, it's in the right half.

5. **`hi = mid - 1` and `lo = mid + 1` (skip mid):** Since we already checked `nums[mid] === target` at the top, mid is never the answer in subsequent checks. Moving past it prevents infinite loops.

**Walkthrough with `[4,5,6,7,0,1,2]`, target = 0:**

| lo | hi | mid | nums[mid] | Sorted half | Action |
|---|---|---|---|---|---|
| 0 | 6 | 3 | 7 | Left [4,5,6,7] | 0 not in [4,7], so lo=4 |
| 4 | 6 | 5 | 1 | Right [1,2] | 0 < 1, not in [1,2], so hi=4 |
| 4 | 4 | 4 | 0 | Found! | return 4 |

</details>

## Practical — Linked List, Tree & Graph Problems

<details>
<summary>18. Design and implement an LRU Cache in TypeScript — support get(key) and put(key, value) both in O(1) time with a fixed capacity. Show the implementation using a hashmap and a doubly-linked list, explain why this specific combination of data structures is required (why not just a hashmap? why not just a linked list?), and handle the eviction logic when capacity is exceeded</summary>

**Why this combination:**

- **HashMap alone** gives O(1) lookup but has no ordering — you can't know which key was least recently used without scanning all entries.
- **Linked list alone** maintains order (move to front on access, evict from tail) but lookup is O(n) — you'd scan the list to find a key.
- **Together**: the hashmap maps keys to linked-list nodes for O(1) lookup, and the doubly-linked list maintains recency order for O(1) move-to-front and O(1) eviction from tail.

A doubly-linked list (not singly) is required because removing a node from the middle requires updating the previous node's `next` pointer — only possible in O(1) if you have a `prev` pointer.

```typescript
class DLLNode {
  constructor(
    public key: number,
    public value: number,
    public prev: DLLNode | null = null,
    public next: DLLNode | null = null,
  ) {}
}

class LRUCache {
  private capacity: number;
  private map = new Map<number, DLLNode>();
  // Sentinel nodes simplify edge cases (no null checks for head/tail)
  private head = new DLLNode(0, 0);
  private tail = new DLLNode(0, 0);

  constructor(capacity: number) {
    this.capacity = capacity;
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  get(key: number): number {
    const node = this.map.get(key);
    if (!node) return -1;
    this.moveToFront(node);
    return node.value;
  }

  put(key: number, value: number): void {
    const existing = this.map.get(key);
    if (existing) {
      existing.value = value;
      this.moveToFront(existing);
      return;
    }

    const node = new DLLNode(key, value);
    this.map.set(key, node);
    this.addToFront(node);

    if (this.map.size > this.capacity) {
      const lru = this.tail.prev!; // least recently used = just before tail sentinel
      this.remove(lru);
      this.map.delete(lru.key);
    }
  }

  private addToFront(node: DLLNode): void {
    node.next = this.head.next;
    node.prev = this.head;
    this.head.next!.prev = node;
    this.head.next = node;
  }

  private remove(node: DLLNode): void {
    node.prev!.next = node.next;
    node.next!.prev = node.prev;
  }

  private moveToFront(node: DLLNode): void {
    this.remove(node);
    this.addToFront(node);
  }
}
```

**Key design details:**

- **Sentinel nodes** (`head` and `tail` are dummy nodes): Every real node is always between `head` and `tail`, so `addToFront`, `remove`, and eviction never deal with null edges. This eliminates 4-5 edge-case checks.
- **Storing key in the node**: When evicting the LRU node from the tail, we need its key to delete it from the hashmap. Without storing the key in the node, we'd need a reverse lookup.
- **All operations are O(1)**: `get` = hashmap lookup + linked-list move. `put` = hashmap insert + linked-list insert + (possibly) linked-list remove + hashmap delete.

</details>

<details>
<summary>19. Solve Validate Binary Search Tree in TypeScript — given a binary tree, determine if it is a valid BST. Show both the range-based approach (passing min/max bounds) and the in-order traversal approach, explain why checking only immediate children is insufficient, and discuss the edge cases (integer overflow with min/max bounds, equal values)</summary>

**Why checking immediate children fails:**

```
    5
   / \
  1   6
     / \
    3   7
```

Node 6's children (3 and 7) satisfy `3 < 6 < 7`, but 3 is in the right subtree of 5 — it should be > 5. A valid BST requires every node in the left subtree to be less than the root, and every node in the right subtree to be greater. You need to propagate bounds down.

**Approach 1: Range-based (min/max bounds)**

```typescript
interface TreeNode {
  val: number;
  left: TreeNode | null;
  right: TreeNode | null;
}

function isValidBST(root: TreeNode | null): boolean {
  return validate(root, -Infinity, Infinity);
}

function validate(
  node: TreeNode | null,
  min: number,
  max: number,
): boolean {
  if (!node) return true;
  if (node.val <= min || node.val >= max) return false;
  return (
    validate(node.left, min, node.val) &&   // left subtree: upper bound is current node
    validate(node.right, node.val, max)     // right subtree: lower bound is current node
  );
}
```

**Approach 2: In-order traversal**

A valid BST's in-order traversal produces strictly increasing values. Track the previous value and verify each node is greater.

```typescript
function isValidBSTInorder(root: TreeNode | null): boolean {
  let prev = -Infinity;

  function inorder(node: TreeNode | null): boolean {
    if (!node) return true;
    if (!inorder(node.left)) return false;
    if (node.val <= prev) return false;
    prev = node.val;
    return inorder(node.right);
  }

  return inorder(root);
}
```

**Edge cases:**

- **Integer overflow with bounds**: In languages like Java/C++, initializing bounds to `Integer.MIN_VALUE`/`Integer.MAX_VALUE` fails if the tree contains those values. Solutions: use `Long` for bounds, or use `null` as initial bounds with explicit null checks. In TypeScript, `-Infinity`/`Infinity` works because no number equals them.

- **Equal values**: The BST definition matters. Strictly, most problems define BST as `left < root < right` (no duplicates). If duplicates are allowed, you need to know the convention — are equal values in the left or right subtree? The `<=` vs `<` in the comparison changes. The code above uses strict inequality (`node.val <= min` rejects equals), enforcing no duplicates.

Both approaches are O(n) time, O(h) space where h is tree height (recursion stack). The range-based approach short-circuits earlier on invalid trees since it can reject a node without visiting its entire subtree.

</details>

<details>
<summary>20. Solve Number of Islands in TypeScript — given a 2D grid of '1's (land) and '0's (water), count the number of islands. Show the BFS or DFS approach with in-place marking, explain the visited-set pattern for grid problems, and discuss why BFS vs DFS doesn't matter for correctness here but might matter for stack depth on large grids</summary>

**Approach:** Scan every cell. When you find a '1' that hasn't been visited, that's a new island — increment the count and flood-fill (BFS or DFS) to mark all connected land cells as visited so they aren't counted again.

```typescript
function numIslands(grid: string[][]): number {
  if (!grid.length) return 0;
  const rows = grid.length, cols = grid[0].length;
  let count = 0;

  function dfs(r: number, c: number): void {
    if (r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] !== '1') return;
    grid[r][c] = '0'; // mark visited by modifying in-place
    dfs(r + 1, c);
    dfs(r - 1, c);
    dfs(r, c + 1);
    dfs(r, c - 1);
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === '1') {
        count++;
        dfs(r, c);
      }
    }
  }
  return count;
}
```

**The visited-set pattern for grids:**

The grid itself serves as the visited set — we overwrite '1' with '0' (or any non-'1' value) to mark a cell as visited. This avoids allocating a separate `visited` boolean matrix. If you can't modify the input, use a `Set<string>` with keys like `"${r},${c}"` or a 2D boolean array.

The pattern for grid problems generalizes: neighbors are the 4 (or 8) adjacent cells, bounds checking replaces the adjacency list, and the "visited" check prevents revisiting.

**BFS vs DFS — correctness vs stack depth:**

Both visit exactly the same cells and produce the same count — correctness is identical. The difference is practical:

- **DFS (recursion)** is simpler to write but uses the call stack. A 300x300 grid with one giant island means 90,000 recursive calls — potential stack overflow. Node.js default stack size handles ~10,000-15,000 frames typically.
- **BFS (queue)** uses heap-allocated memory for the queue, which can grow much larger without crashing. Safer for large grids.
- **Iterative DFS (explicit stack)** avoids the recursion limit while keeping DFS traversal order. Best of both if stack depth is a concern.

```typescript
// BFS version for large grids
function numIslandsBFS(grid: string[][]): number {
  const rows = grid.length, cols = grid[0].length;
  const dirs = [[1,0],[-1,0],[0,1],[0,-1]];
  let count = 0;

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] !== '1') continue;
      count++;
      grid[r][c] = '0';
      const queue: [number, number][] = [[r, c]];
      while (queue.length) {
        const [cr, cc] = queue.shift()!; // O(n) — use index-based queue for large grids (see Q12)
        for (const [dr, dc] of dirs) {
          const nr = cr + dr, nc = cc + dc;
          if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] === '1') {
            grid[nr][nc] = '0'; // mark before enqueueing (as noted in question 12)
            queue.push([nr, nc]);
          }
        }
      }
    }
  }
  return count;
}
```

</details>

<details>
<summary>21. Solve Course Schedule in TypeScript — given a number of courses and prerequisite pairs, determine if it's possible to finish all courses (detect if the dependency graph has a cycle). Show the topological sort approach using either Kahn's algorithm (BFS with in-degree tracking) or DFS with cycle detection, explain why topological sort requires a DAG, and how to detect cycles during the traversal</summary>

**Why topological sort requires a DAG:** A topological ordering is a linear sequence where for every edge u -> v, u appears before v. This is only possible if there are no cycles — if A depends on B depends on A, neither can come first. So detecting a cycle is equivalent to asking "does a valid topological ordering exist?"

**Approach 1: Kahn's Algorithm (BFS with in-degree tracking)**

Start with all nodes that have no prerequisites (in-degree 0). Process them, decrement the in-degree of their dependents. When a dependent's in-degree hits 0, it's ready to process. If you process all nodes, no cycle exists. If some nodes never reach in-degree 0, they're in a cycle.

```typescript
function canFinish(numCourses: number, prerequisites: number[][]): boolean {
  const graph: number[][] = Array.from({ length: numCourses }, () => []);
  const inDegree = new Array(numCourses).fill(0);

  for (const [course, prereq] of prerequisites) {
    graph[prereq].push(course); // prereq -> course
    inDegree[course]++;
  }

  // Start with all courses that have no prerequisites
  const queue: number[] = [];
  for (let i = 0; i < numCourses; i++) {
    if (inDegree[i] === 0) queue.push(i);
  }

  let processed = 0;
  while (queue.length > 0) {
    const course = queue.shift()!;
    processed++;
    for (const next of graph[course]) {
      inDegree[next]--;
      if (inDegree[next] === 0) queue.push(next);
    }
  }

  return processed === numCourses; // if not all processed, cycle exists
}
```

**Approach 2: DFS with cycle detection**

Use three states per node: unvisited, in-progress (on current recursion stack), and completed. If DFS visits an in-progress node, that's a back edge — a cycle.

```typescript
function canFinishDFS(numCourses: number, prerequisites: number[][]): boolean {
  const graph: number[][] = Array.from({ length: numCourses }, () => []);
  for (const [course, prereq] of prerequisites) {
    graph[prereq].push(course);
  }

  // 0 = unvisited, 1 = in-progress, 2 = completed
  const state = new Array(numCourses).fill(0);

  function hasCycle(node: number): boolean {
    if (state[node] === 1) return true;  // back edge — cycle
    if (state[node] === 2) return false; // already fully explored
    state[node] = 1; // mark in-progress
    for (const next of graph[node]) {
      if (hasCycle(next)) return true;
    }
    state[node] = 2; // mark completed
    return false;
  }

  // Check every node (handles disconnected components)
  for (let i = 0; i < numCourses; i++) {
    if (hasCycle(i)) return false;
  }
  return true;
}
```

**Key distinction from question 12:** The three-state approach (unvisited/in-progress/completed) is essential for directed graph cycle detection. In undirected graphs, a simple visited boolean suffices (as noted in question 12). In directed graphs, visiting a completed node isn't a cycle — only visiting an in-progress node on the current path is.

Both approaches are O(V + E) time and space where V = courses and E = prerequisites. Kahn's is often preferred in interviews because it naturally produces the topological ordering (the order in which nodes are processed) and avoids recursion.

</details>

## Practical — Dynamic Programming Problems

<details>
<summary>22. Solve Climbing Stairs and House Robber in TypeScript — show the full DP progression for each: start with the naive recursive solution, add memoization, convert to bottom-up tabulation, then optimize space to O(1). Explain why these two problems share the same underlying pattern (each state depends on only the previous 1-2 states) and how recognizing this pattern transfers to other problems</summary>

Both problems follow the same DP progression shown conceptually in question 9. Here we apply it to two concrete problems.

**Climbing Stairs** — n steps, can climb 1 or 2 at a time. How many distinct ways to reach the top?

The recurrence `f(n) = f(n-1) + f(n-2)` is identical to Fibonacci (question 9), so the same four-step progression applies (naive -> memo -> tab -> space-optimized). The space-optimized version:

```typescript
function climbStairs(n: number): number {
  if (n <= 2) return n;
  let prev2 = 1, prev1 = 2;
  for (let i = 3; i <= n; i++) [prev2, prev1] = [prev1, prev2 + prev1];
  return prev1;
}
```

**House Robber** — array of house values, can't rob two adjacent houses. Maximum total value?

The recurrence: at each house, either rob it (add its value to the best from two houses back) or skip it (take the best from one house back). `dp[i] = max(dp[i-1], dp[i-2] + nums[i])`.

```typescript
// 1. Naive recursion — O(2^n)
function robRec(nums: number[], i = nums.length - 1): number {
  if (i < 0) return 0;
  return Math.max(robRec(nums, i - 1), nums[i] + robRec(nums, i - 2));
}

// 2. Memoization — O(n) time, O(n) space
function robMemo(nums: number[], i = nums.length - 1, memo = new Map<number, number>()): number {
  if (i < 0) return 0;
  if (memo.has(i)) return memo.get(i)!;
  const result = Math.max(robMemo(nums, i - 1, memo), nums[i] + robMemo(nums, i - 2, memo));
  memo.set(i, result);
  return result;
}

// 3. Tabulation — O(n) time, O(n) space
function robTab(nums: number[]): number {
  if (nums.length === 0) return 0;
  if (nums.length === 1) return nums[0];
  const dp = new Array(nums.length);
  dp[0] = nums[0];
  dp[1] = Math.max(nums[0], nums[1]);
  for (let i = 2; i < nums.length; i++) {
    dp[i] = Math.max(dp[i - 1], dp[i - 2] + nums[i]);
  }
  return dp[nums.length - 1];
}

// 4. Space-optimized — O(n) time, O(1) space
function rob(nums: number[]): number {
  let prev2 = 0, prev1 = 0;
  for (const num of nums) {
    [prev2, prev1] = [prev1, Math.max(prev1, prev2 + num)];
  }
  return prev1;
}
```

**The shared pattern:**

Both problems have the same structure: `dp[i]` depends only on `dp[i-1]` and `dp[i-2]`. This means:
1. The recurrence looks back at most 2 steps.
2. Tabulation only needs a 1D array.
3. Space optimization only needs 2 variables.

This pattern appears in many other problems:
- **Fibonacci** (covered in question 9) — same recurrence, different semantics.
- **Decode Ways** — number of ways to decode a digit string. `dp[i]` depends on `dp[i-1]` (single digit) and `dp[i-2]` (two-digit decode).
- **Min Cost Climbing Stairs** — same as climbing stairs but with costs per step.
- **Paint House** — extends to 2D (previous state per color) but same "look back 1 step" idea.

Recognizing this "bounded lookback" pattern is the key transferable skill. When you see a problem where each decision depends only on a fixed number of prior decisions, you know: it's DP, the table is 1D, and it compresses to O(1) space.

</details>

<details>
<summary>23. Solve Coin Change in TypeScript — given coin denominations and a target amount, find the minimum number of coins needed. Show the bottom-up DP approach, explain why this is an unbounded knapsack variant (each coin can be used unlimited times), walk through the recurrence relation, and explain what makes greedy fail for arbitrary denominations (why always picking the largest coin doesn't work)</summary>

**The recurrence:** For each amount `a`, try every coin denomination `c`. If `c <= a`, then `dp[a] = min(dp[a], dp[a - c] + 1)`. The "+1" accounts for using one coin of denomination `c`, and `dp[a - c]` is the optimal solution for the remaining amount.

Base case: `dp[0] = 0` (zero coins needed for amount 0). Initialize all other entries to `Infinity` (unreachable until proven otherwise).

```typescript
function coinChange(coins: number[], amount: number): number {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;

  for (let a = 1; a <= amount; a++) {
    for (const coin of coins) {
      if (coin <= a && dp[a - coin] !== Infinity) {
        dp[a] = Math.min(dp[a], dp[a - coin] + 1);
      }
    }
  }

  return dp[amount] === Infinity ? -1 : dp[amount];
}
```

**Walkthrough with coins = [1, 3, 4], amount = 6:**

| Amount | Try coin 1 | Try coin 3 | Try coin 4 | dp[a] |
|---|---|---|---|---|
| 0 | - | - | - | 0 |
| 1 | dp[0]+1=1 | - | - | 1 |
| 2 | dp[1]+1=2 | - | - | 2 |
| 3 | dp[2]+1=3 | dp[0]+1=1 | - | 1 |
| 4 | dp[3]+1=2 | dp[1]+1=2 | dp[0]+1=1 | 1 |
| 5 | dp[4]+1=2 | dp[2]+1=3 | dp[1]+1=2 | 2 |
| 6 | dp[5]+1=3 | dp[3]+1=2 | dp[2]+1=3 | **2** |

Result: 2 (coins 3 + 3).

**Why this is an unbounded knapsack variant:**

In the classic 0/1 knapsack, each item can be used once. Here, each coin denomination can be used unlimited times. The key difference in the DP: in 0/1 knapsack, you iterate items in the outer loop and amounts in reverse to prevent reuse. In unbounded knapsack (coin change), iterating amounts forward naturally allows reuse — `dp[a - c]` might already include coin `c`, and that's fine.

**Why greedy fails for arbitrary denominations:**

Greedy picks the largest coin that fits, repeating until done. With coins [1, 3, 4] and amount 6: greedy picks 4 first, then needs 2 more, which requires 1+1 = 3 coins total (4+1+1). But the optimal is 3+3 = 2 coins.

Greedy works for standard US/EU denominations (1, 5, 10, 25) because each denomination is at least 2x the previous — you never need more than one of a smaller coin to match the gap. This property (called the "canonical" coin system) doesn't hold for arbitrary denominations. With [1, 3, 4], the 4 doesn't cleanly divide the gap left after choosing it, so local optimization misses the global optimum. As noted in question 9, when you can't prove the greedy choice property, DP is the safe approach.

Time: O(amount * coins.length). Space: O(amount).

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you chose the wrong data structure for a problem and had to refactor — what was the original choice, what went wrong (performance, correctness, complexity), how did you identify the issue, and what did you switch to?</summary>

**What the interviewer is looking for:**
- Self-awareness about past mistakes and willingness to admit them.
- Ability to diagnose performance/correctness issues systematically.
- Understanding of data structure tradeoffs in practice, not just theory.
- Growth mindset — what you learned and how it changed your approach.

**Suggested structure (STAR):**

1. **Situation**: What were you building? What were the access patterns and scale?
2. **Task**: What data structure did you initially choose and why did it seem reasonable?
3. **Action**: What went wrong? How did you detect the problem (monitoring, profiling, bug reports, load testing)? What did you switch to and why?
4. **Result**: Measurable improvement. What did you learn about choosing data structures?

**Example outline to personalize:**

> "We were building an event deduplication service that needed to track recently-seen event IDs to skip duplicates. I initially used an array and checked `.includes()` for each incoming event. At low volume in dev, this was fine.
>
> In staging with realistic load (~5K events/second), the dedup check became a bottleneck — p99 latency spiked because `.includes()` is O(n) and the array grew to thousands of entries. I spotted it through latency dashboards and confirmed with profiling.
>
> I switched to a `Set` for O(1) lookup and added a TTL-based cleanup to keep the set bounded. Latency dropped from ~50ms p99 to under 1ms. The lesson was that access patterns at scale matter more than what works in dev — I now always ask 'what happens at 10x and 100x the current volume' when choosing a data structure."

**Key points to hit:**
- Be specific about the data structure choice and the access pattern mismatch.
- Show you used data/metrics to identify the issue, not just guesswork.
- Explain the tradeoff of the new choice (e.g., Set uses more memory than array for small n, but that was acceptable).
- End with a transferable lesson, not just "I fixed it."

</details>

<details>
<summary>25. Tell me about a time you optimized a slow algorithm or query in a production system — what were the symptoms, how did you identify the bottleneck, what algorithmic or data structure change did you make, and what was the measurable improvement?</summary>

**What the interviewer is looking for:**
- Systematic debugging approach — not "I just knew it was slow."
- Ability to connect symptoms to root causes using observability tools.
- Understanding of algorithmic complexity in a real-world context.
- Quantifiable results — not just "it got faster."

**Suggested structure (STAR):**

1. **Situation/Symptoms**: What was slow? How did you notice (alerts, customer complaints, dashboards)? What was the scale (request volume, data size)?
2. **Investigation**: What tools did you use (APM traces, database EXPLAIN plans, profiling, flame graphs)? How did you narrow from "it's slow" to the specific bottleneck?
3. **Root cause**: What algorithmic or data structure issue was at play? Was it an O(n^2) hidden in a loop? A missing index causing full table scans? An N+1 query pattern?
4. **Fix**: What change did you make? Why was this the right fix over alternatives?
5. **Result**: Numbers — before/after latency, throughput, error rates.

**Example outline to personalize:**

> "Our audit log query endpoint had p95 latency above 2 seconds for customers with high activity. The API returned change events for a resource, and users were reporting timeouts.
>
> I started with APM traces, which showed 80% of the time was in the database layer. Running EXPLAIN on the query revealed a sequential scan on a table with millions of rows — we were filtering by `projectKey` and `resourceId` but only had an index on `projectKey` alone.
>
> Adding a composite index on `(projectKey, resourceId, createdAt DESC)` turned the sequential scan into an index-only scan. The query went from ~1800ms to ~15ms at p95. I also added a covering index to avoid table lookups entirely for the most common query pattern.
>
> The broader lesson: as data grows, queries that were 'fast enough' in the first months can degrade silently. We added query latency monitoring per endpoint afterward to catch regressions early."

**Key points to hit:**
- Start with observable symptoms, not "I looked at the code and found something slow."
- Show the investigation funnel — broad monitoring -> specific trace -> root cause.
- Name the algorithmic issue clearly (O(n) scan vs O(log n) index lookup, N+1 queries, quadratic loop).
- Mention alternatives you considered and why you chose this fix.
- End with prevention — what did you change to avoid similar issues going forward?

</details>