# LeetCode 150 — Pattern Templates + Complete Problem Mapping

> Every pattern has a **reusable Python template** + **all 150 problems mapped** to their pattern(s).

---

## Pattern Index

| # | Pattern | Problems |
|---|---|---|
| P01 | [In-place Array Write](#p01-in-place-array-write) | #1 #2 #3 #4 #6 |
| P02 | [Greedy Scan](#p02-greedy-scan) | #5 #7 #8 #9 #10 #11 #14 #15 #51 |
| P03 | [Simulation / Direct](#p03-simulation--direct) | #12 #17 #18 #19 #20 #21 #22 #23 #24 #34 #35 #36 #37 #38 #48 #125 #131 #132 #133 |
| P04 | [Two Pointers](#p04-two-pointers) | #13 #16 #25 #26 #27 #28 #29 |
| P05 | [Sliding Window](#p05-sliding-window) | #30 #31 #32 #33 |
| P06 | [Hash Map / Set](#p06-hash-map--set) | #39 #40 #41 #42 #43 #44 #45 #46 #47 |
| P07 | [Intervals (Sort + Sweep)](#p07-intervals-sort--sweep) | #49 #50 #51 |
| P08 | [Stack](#p08-stack) | #52 #53 #54 #55 #56 |
| P09 | [Linked List Manipulation](#p09-linked-list-manipulation) | #57 #58 #59 #60 #61 #62 #63 #64 #65 #66 #67 |
| P10 | [Tree DFS (Recursion)](#p10-tree-dfs-recursion) | #68 #69 #70 #71 #72 #73 #74 #75 #76 #77 #78 #79 #80 #81 #86 #87 #88 #108 |
| P11 | [Tree BFS (Level Order)](#p11-tree-bfs-level-order) | #82 #83 #84 #85 |
| P12 | [Graph DFS](#p12-graph-dfs) | #89 #90 #91 #93 #94 |
| P13 | [Graph BFS (Shortest Path)](#p13-graph-bfs-shortest-path) | #89 #92 #95 #96 #97 |
| P14 | [Topological Sort](#p14-topological-sort) | #93 #94 |
| P15 | [Trie](#p15-trie) | #98 #99 #100 |
| P16 | [Backtracking](#p16-backtracking) | #101 #102 #103 #104 #105 #106 #107 |
| P17 | [Divide & Conquer](#p17-divide--conquer) | #108 #109 #110 #111 |
| P18 | [Kadane's Algorithm](#p18-kadanes-algorithm) | #112 #113 |
| P19 | [Binary Search](#p19-binary-search) | #114 #115 #116 #117 #118 #119 #120 |
| P20 | [Heap / Priority Queue](#p20-heap--priority-queue) | #111 #121 #122 #123 #124 |
| P21 | [Union Find](#p21-union-find) | #89 (alt) |
| P22 | [Bit Manipulation](#p22-bit-manipulation) | #125 #126 #127 #128 #129 #130 |
| P23 | [Math](#p23-math) | #131 #132 #133 #134 #135 #136 |
| P24 | [1D Dynamic Programming](#p24-1d-dynamic-programming) | #9 #10 #137 #138 #139 #140 #141 |
| P25 | [Multi-dim / 2D DP](#p25-multi-dim--2d-dp) | #142 #143 #144 #145 #146 #147 #150 |
| P26 | [State Machine DP](#p26-state-machine-dp) | #148 #149 |

---

## Complete Problem → Pattern Map (all 150)

| # | Problem | Pattern(s) |
|---|---|---|
| 1 | Merge Sorted Array | P01 In-place Array Write |
| 2 | Remove Element | P01 In-place Array Write |
| 3 | Remove Duplicates from Sorted Array | P01 In-place Array Write |
| 4 | Remove Duplicates from Sorted Array II | P01 In-place Array Write |
| 5 | Majority Element | P02 Greedy (Boyer-Moore Voting) |
| 6 | Rotate Array | P01 In-place Array Write |
| 7 | Best Time to Buy and Sell Stock | P02 Greedy Scan |
| 8 | Best Time to Buy and Sell Stock II | P02 Greedy Scan |
| 9 | Jump Game | P02 Greedy + P24 1D DP |
| 10 | Jump Game II | P02 Greedy + P24 1D DP |
| 11 | H-Index | P02 Greedy (Sort) |
| 12 | Insert Delete GetRandom O(1) | P03 Simulation (Hash Map + Array) |
| 13 | Product of Array Except Self | P04 Two Pointers (prefix/suffix) |
| 14 | Gas Station | P02 Greedy Scan |
| 15 | Candy | P02 Greedy (Two-pass) |
| 16 | Trapping Rain Water | P04 Two Pointers |
| 17 | Roman to Integer | P03 Simulation |
| 18 | Integer to Roman | P03 Simulation |
| 19 | Length of Last Word | P03 Simulation |
| 20 | Longest Common Prefix | P03 Simulation |
| 21 | Reverse Words in a String | P03 Simulation |
| 22 | Zigzag Conversion | P03 Simulation |
| 23 | Find the Index of the First Occurrence | P03 Simulation (KMP) |
| 24 | Text Justification | P03 Simulation |
| 25 | Valid Palindrome | P04 Two Pointers |
| 26 | Is Subsequence | P04 Two Pointers |
| 27 | Two Sum II | P04 Two Pointers |
| 28 | Container With Most Water | P04 Two Pointers |
| 29 | 3Sum | P04 Two Pointers (sorted) |
| 30 | Minimum Size Subarray Sum | P05 Sliding Window |
| 31 | Longest Substring Without Repeating Characters | P05 Sliding Window |
| 32 | Substring with Concatenation of All Words | P05 Sliding Window + P06 Hash Map |
| 33 | Minimum Window Substring | P05 Sliding Window + P06 Hash Map |
| 34 | Valid Sudoku | P03 Simulation + P06 Hash Set |
| 35 | Spiral Matrix | P03 Simulation |
| 36 | Rotate Image | P03 Simulation (Transpose) |
| 37 | Set Matrix Zeroes | P03 Simulation |
| 38 | Game of Life | P03 Simulation (Bit encoding) |
| 39 | Ransom Note | P06 Hash Map |
| 40 | Isomorphic Strings | P06 Hash Map |
| 41 | Word Pattern | P06 Hash Map |
| 42 | Valid Anagram | P06 Hash Map |
| 43 | Group Anagrams | P06 Hash Map |
| 44 | Two Sum | P06 Hash Map |
| 45 | Happy Number | P06 Hash Set (cycle detection) |
| 46 | Contains Duplicate II | P06 Hash Map (sliding window) |
| 47 | Longest Consecutive Sequence | P06 Hash Set |
| 48 | Summary Ranges | P03 Simulation |
| 49 | Merge Intervals | P07 Intervals |
| 50 | Insert Interval | P07 Intervals |
| 51 | Minimum Number of Arrows to Burst Balloons | P07 Intervals + P02 Greedy |
| 52 | Valid Parentheses | P08 Stack |
| 53 | Simplify Path | P08 Stack |
| 54 | Min Stack | P08 Stack |
| 55 | Evaluate Reverse Polish Notation | P08 Stack |
| 56 | Basic Calculator | P08 Stack |
| 57 | Linked List Cycle | P09 Fast & Slow Pointers |
| 58 | Add Two Numbers | P09 Linked List |
| 59 | Merge Two Sorted Lists | P09 Linked List |
| 60 | Copy List with Random Pointer | P09 Linked List + P06 Hash Map |
| 61 | Reverse Linked List II | P09 Linked List |
| 62 | Reverse Nodes in k-Group | P09 Linked List |
| 63 | Remove Nth Node From End | P09 Fast & Slow Pointers |
| 64 | Remove Duplicates from Sorted List II | P09 Linked List |
| 65 | Rotate List | P09 Linked List |
| 66 | Partition List | P09 Linked List (dummy heads) |
| 67 | LRU Cache | P09 Linked List + P06 Hash Map |
| 68 | Maximum Depth of Binary Tree | P10 Tree DFS |
| 69 | Same Tree | P10 Tree DFS |
| 70 | Invert Binary Tree | P10 Tree DFS |
| 71 | Symmetric Tree | P10 Tree DFS |
| 72 | Construct BT from Preorder + Inorder | P10 Tree DFS + P06 Hash Map |
| 73 | Construct BT from Inorder + Postorder | P10 Tree DFS + P06 Hash Map |
| 74 | Populating Next Right Pointers II | P10 Tree DFS / P11 BFS |
| 75 | Flatten Binary Tree to Linked List | P10 Tree DFS |
| 76 | Path Sum | P10 Tree DFS |
| 77 | Sum Root to Leaf Numbers | P10 Tree DFS |
| 78 | Binary Tree Maximum Path Sum | P10 Tree DFS |
| 79 | Binary Search Tree Iterator | P10 Tree DFS (iterative inorder) |
| 80 | Count Complete Tree Nodes | P10 Tree DFS + P19 Binary Search |
| 81 | Lowest Common Ancestor of Binary Tree | P10 Tree DFS |
| 82 | Binary Tree Right Side View | P11 Tree BFS |
| 83 | Average of Levels in Binary Tree | P11 Tree BFS |
| 84 | Binary Tree Level Order Traversal | P11 Tree BFS |
| 85 | Binary Tree Zigzag Level Order Traversal | P11 Tree BFS |
| 86 | Minimum Absolute Difference in BST | P10 Tree DFS (inorder) |
| 87 | Kth Smallest Element in a BST | P10 Tree DFS (inorder) |
| 88 | Validate Binary Search Tree | P10 Tree DFS (bounds) |
| 89 | Number of Islands | P12 Graph DFS / P13 BFS / P21 Union Find |
| 90 | Surrounded Regions | P12 Graph DFS |
| 91 | Clone Graph | P12 Graph DFS + P06 Hash Map |
| 92 | Evaluate Division | P13 Graph BFS |
| 93 | Course Schedule | P12 Graph DFS + P14 Topological Sort |
| 94 | Course Schedule II | P14 Topological Sort |
| 95 | Snakes and Ladders | P13 Graph BFS |
| 96 | Minimum Genetic Mutation | P13 Graph BFS |
| 97 | Word Ladder | P13 Graph BFS |
| 98 | Implement Trie | P15 Trie |
| 99 | Design Add and Search Words | P15 Trie + P12 DFS |
| 100 | Word Search II | P15 Trie + P16 Backtracking |
| 101 | Letter Combinations of a Phone Number | P16 Backtracking |
| 102 | Combinations | P16 Backtracking |
| 103 | Permutations | P16 Backtracking |
| 104 | Combination Sum | P16 Backtracking |
| 105 | N-Queens II | P16 Backtracking |
| 106 | Generate Parentheses | P16 Backtracking |
| 107 | Word Search | P16 Backtracking |
| 108 | Convert Sorted Array to BST | P17 Divide & Conquer |
| 109 | Sort List | P17 Divide & Conquer (Merge Sort) |
| 110 | Construct Quad Tree | P17 Divide & Conquer |
| 111 | Merge k Sorted Lists | P17 Divide & Conquer + P20 Heap |
| 112 | Maximum Subarray | P18 Kadane's |
| 113 | Maximum Sum Circular Subarray | P18 Kadane's |
| 114 | Search Insert Position | P19 Binary Search |
| 115 | Search a 2D Matrix | P19 Binary Search |
| 116 | Find Peak Element | P19 Binary Search |
| 117 | Search in Rotated Sorted Array | P19 Binary Search |
| 118 | Find First and Last Position of Element | P19 Binary Search |
| 119 | Find Minimum in Rotated Sorted Array | P19 Binary Search |
| 120 | Median of Two Sorted Arrays | P19 Binary Search |
| 121 | Kth Largest Element in an Array | P20 Heap |
| 122 | IPO | P20 Heap + P02 Greedy |
| 123 | Find K Pairs with Smallest Sums | P20 Heap |
| 124 | Find Median from Data Stream | P20 Heap (two heaps) |
| 125 | Add Binary | P22 Bit Manipulation |
| 126 | Reverse Bits | P22 Bit Manipulation |
| 127 | Number of 1 Bits | P22 Bit Manipulation |
| 128 | Single Number | P22 Bit Manipulation (XOR) |
| 129 | Single Number II | P22 Bit Manipulation (state machine) |
| 130 | Bitwise AND of Numbers Range | P22 Bit Manipulation |
| 131 | Palindrome Number | P23 Math |
| 132 | Plus One | P23 Math |
| 133 | Factorial Trailing Zeroes | P23 Math |
| 134 | Sqrt(x) | P19 Binary Search / P23 Math |
| 135 | Pow(x, n) | P17 Divide & Conquer / P23 Math |
| 136 | Max Points on a Line | P23 Math + P06 Hash Map |
| 137 | Climbing Stairs | P24 1D DP |
| 138 | House Robber | P24 1D DP |
| 139 | Word Break | P24 1D DP + P06 Hash Set |
| 140 | Coin Change | P24 1D DP |
| 141 | Longest Increasing Subsequence | P24 1D DP + P19 Binary Search |
| 142 | Triangle | P25 Multi-dim DP |
| 143 | Minimum Path Sum | P25 Multi-dim DP |
| 144 | Unique Paths II | P25 Multi-dim DP |
| 145 | Longest Palindromic Substring | P25 Multi-dim DP / Expand Around Center |
| 146 | Interleaving String | P25 Multi-dim DP |
| 147 | Edit Distance | P25 Multi-dim DP |
| 148 | Best Time to Buy and Sell Stock III | P26 State Machine DP |
| 149 | Best Time to Buy and Sell Stock IV | P26 State Machine DP |
| 150 | Maximal Square | P25 Multi-dim DP |

---

## Pattern Templates

---

## P01 In-place Array Write

**When:** Modify array in-place, return new length. Input is often sorted.  
**Key idea:** `slow` pointer marks the write position; `fast` scans ahead.  
**Problems:** #1 #2 #3 #4 #6

```python
# ── TEMPLATE: slow/fast write ──────────────────────────────────────────
def template_inplace_write(nums: list[int]) -> int:
    slow = 0                          # write head
    for fast in range(len(nums)):
        if keep(nums[fast], slow, nums):   # condition varies per problem
            nums[slow] = nums[fast]
            slow += 1
    return slow                       # new length

# ── VARIANT: fill from back (merge two sorted arrays) ──────────────────
def template_merge_from_back(nums1, m, nums2, n):
    p1, p2, write = m - 1, n - 1, m + n - 1
    while p2 >= 0:
        if p1 >= 0 and nums1[p1] > nums2[p2]:
            nums1[write] = nums1[p1]; p1 -= 1
        else:
            nums1[write] = nums2[p2]; p2 -= 1
        write -= 1

# ── VARIANT: allow at most k duplicates ────────────────────────────────
def template_at_most_k_dups(nums, k=1):
    slow = 0
    for num in nums:
        if slow < k or nums[slow - k] != num:
            nums[slow] = num; slow += 1
    return slow

# ── VARIANT: rotate array (triple reverse) ─────────────────────────────
def template_rotate(nums, k):
    k %= len(nums)
    nums.reverse()
    nums[:k]  = list(reversed(nums[:k]))
    nums[k:]  = list(reversed(nums[k:]))
```

**Complexity:** Time O(n) · Space O(1)

---

## P02 Greedy Scan

**When:** One or two passes, local optimal = global optimal.  
**Shapes:** Boyer-Moore voting, running max/min, two-pass (left→right then right→left).  
**Problems:** #5 #7 #8 #9 #10 #11 #14 #15 #51

```python
# ── TEMPLATE: running best ──────────────────────────────────────────────
def template_greedy_running(nums):
    best = initial_value
    state = initial_state
    for x in nums:
        state = update(state, x)      # e.g. min price seen so far
        best  = max(best, profit(state, x))
    return best

# ── VARIANT: Boyer-Moore majority vote ─────────────────────────────────
def template_majority_vote(nums):
    candidate = count = 0
    for n in nums:
        if count == 0: candidate = n
        count += (1 if n == candidate else -1)
    return candidate

# ── VARIANT: two-pass (Candy-style) ────────────────────────────────────
def template_two_pass(ratings):
    n = len(ratings)
    candy = [1] * n
    for i in range(1, n):                     # left → right
        if ratings[i] > ratings[i-1]:
            candy[i] = candy[i-1] + 1
    for i in range(n-2, -1, -1):             # right → left
        if ratings[i] > ratings[i+1]:
            candy[i] = max(candy[i], candy[i+1] + 1)
    return sum(candy)

# ── VARIANT: greedy interval (sort by end, pick greedily) ──────────────
def template_greedy_intervals(intervals):
    intervals.sort(key=lambda x: x[1])
    count = 1
    end = intervals[0][1]
    for s, e in intervals[1:]:
        if s > end: count += 1; end = e      # no overlap → new group
    return count

# ── VARIANT: jump game (track farthest reach) ──────────────────────────
def template_jump(nums):
    reach = jumps = cur_end = 0
    for i in range(len(nums) - 1):
        reach = max(reach, i + nums[i])
        if i == cur_end:
            jumps += 1; cur_end = reach
    return jumps
```

**Complexity:** Time O(n) or O(n log n) · Space O(1)

---

## P03 Simulation / Direct

**When:** No clever algorithm needed — just carefully follow the rules.  
**Problems:** #12 #17 #18 #19 #20 #21 #22 #23 #24 #34 #35 #36 #37 #38 #48 #125 #131 #132 #133

```python
# ── TEMPLATE: boundary walk (Spiral Matrix) ────────────────────────────
def template_boundary_walk(matrix):
    result = []
    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1
    while top <= bottom and left <= right:
        for c in range(left, right + 1):   result.append(matrix[top][c]);  top += 1
        for r in range(top, bottom + 1):   result.append(matrix[r][right]); right -= 1
        if top <= bottom:
            for c in range(right, left-1,-1): result.append(matrix[bottom][c]); bottom -= 1
        if left <= right:
            for r in range(bottom, top-1,-1): result.append(matrix[r][left]); left += 1
    return result

# ── VARIANT: encode next state in spare bits (Game of Life) ────────────
def template_inplace_state(board):
    m, n = len(board), len(board[0])
    def live_neighbors(r, c):
        return sum(board[r+dr][c+dc] & 1
                   for dr in [-1,0,1] for dc in [-1,0,1]
                   if (dr or dc) and 0<=r+dr<m and 0<=c+dc<n)
    for r in range(m):
        for c in range(n):
            ln = live_neighbors(r, c)
            if board[r][c] == 1 and ln in (2, 3): board[r][c] |= 2
            if board[r][c] == 0 and ln == 3:      board[r][c] |= 2
    for r in range(m):
        for c in range(n): board[r][c] >>= 1      # shift to reveal new state

# ── VARIANT: transpose then reverse rows (Rotate Image 90°) ────────────
def template_rotate_matrix(matrix):
    n = len(matrix)
    for r in range(n):
        for c in range(r + 1, n):
            matrix[r][c], matrix[c][r] = matrix[c][r], matrix[r][c]
    for row in matrix: row.reverse()

# ── VARIANT: anchor rows/cols with first row/col (Set Matrix Zeroes) ───
def template_set_zeroes(matrix):
    m, n = len(matrix), len(matrix[0])
    fr = any(matrix[0][c] == 0 for c in range(n))
    fc = any(matrix[r][0] == 0 for r in range(m))
    for r in range(1, m):
        for c in range(1, n):
            if matrix[r][c] == 0: matrix[r][0] = matrix[0][c] = 0
    for r in range(1, m):
        for c in range(1, n):
            if not matrix[r][0] or not matrix[0][c]: matrix[r][c] = 0
    if fr:
        for c in range(n): matrix[0][c] = 0
    if fc:
        for r in range(m): matrix[r][0] = 0
```

**Complexity:** Varies — usually O(n) or O(m×n) · Space O(1)

---

## P04 Two Pointers

**When:** Sorted array, palindrome, pair/triplet sum, prefix×suffix product.  
**Problems:** #13 #16 #25 #26 #27 #28 #29

```python
# ── TEMPLATE: converging pointers ──────────────────────────────────────
def template_two_ptr(arr, target):
    left, right = 0, len(arr) - 1
    while left < right:
        val = arr[left] + arr[right]
        if val == target:
            return [left, right]      # found
        elif val < target: left  += 1
        else:              right -= 1
    return []

# ── VARIANT: skip non-alnum for palindrome ─────────────────────────────
def template_palindrome(s):
    l, r = 0, len(s) - 1
    while l < r:
        while l < r and not s[l].isalnum(): l += 1
        while l < r and not s[r].isalnum(): r -= 1
        if s[l].lower() != s[r].lower(): return False
        l += 1; r -= 1
    return True

# ── VARIANT: 3Sum (fix one, two-pointer inner loop) ────────────────────
def template_3sum(nums):
    nums.sort(); result = []
    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i-1]: continue  # skip duplicate pivot
        l, r = i + 1, len(nums) - 1
        while l < r:
            s = nums[i] + nums[l] + nums[r]
            if   s == 0: result.append([nums[i], nums[l], nums[r]]); l += 1
            elif s  < 0: l += 1
            else:        r -= 1
            while l < r and nums[l] == nums[l-1]: l += 1  # skip dup left
    return result

# ── VARIANT: prefix × suffix (no division) ─────────────────────────────
def template_prefix_suffix(nums):
    n = len(nums)
    out = [1] * n
    prefix = 1
    for i in range(n):      out[i]  = prefix; prefix *= nums[i]
    suffix = 1
    for i in range(n-1,-1,-1): out[i] *= suffix; suffix *= nums[i]
    return out
```

**Complexity:** Time O(n) or O(n²) for kSum · Space O(1)

---

## P05 Sliding Window

**When:** Contiguous subarray/substring with a constraint (max length, exact count, min sum).  
**Problems:** #30 #31 #32 #33

```python
# ── TEMPLATE: variable window (shrink when violated) ───────────────────
def template_variable_window(s, constraint):
    from collections import defaultdict
    window = defaultdict(int)
    left = best = 0
    for right, ch in enumerate(s):
        window[ch] += 1                         # expand
        while violates(window, constraint):      # shrink
            window[s[left]] -= 1
            if window[s[left]] == 0: del window[s[left]]
            left += 1
        best = max(best, right - left + 1)
    return best

# ── TEMPLATE: fixed window of size k ──────────────────────────────────
def template_fixed_window(nums, k):
    window_sum = sum(nums[:k])
    best = window_sum
    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]
        best = max(best, window_sum)
    return best

# ── VARIANT: need exactly `need` distinct chars (min window substring) ─
def template_min_window(s, t):
    from collections import Counter
    need = Counter(t); missing = len(t)
    l = start = 0; end = float('inf')
    for r, ch in enumerate(s):
        if need[ch] > 0: missing -= 1
        need[ch] -= 1
        if missing == 0:                         # valid → try to shrink
            while need[s[l]] < 0: need[s[l]] += 1; l += 1
            if r - l < end - start: start, end = l, r
            need[s[l]] += 1; missing += 1; l += 1
    return '' if end == float('inf') else s[start:end+1]

# ── VARIANT: count of subarrays with property (use at most k trick) ────
# count(exactly k) = count(at most k) - count(at most k-1)
def template_exactly_k(nums, k):
    def at_most(k):
        from collections import defaultdict
        count = defaultdict(int); left = result = 0
        for right, n in enumerate(nums):
            count[n] += 1
            while len(count) > k:
                count[nums[left]] -= 1
                if count[nums[left]] == 0: del count[nums[left]]
                left += 1
            result += right - left + 1
        return result
    return at_most(k) - at_most(k - 1)
```

**Complexity:** Time O(n) · Space O(k) where k = window state size

---

## P06 Hash Map / Set

**When:** O(1) lookup, frequency count, grouping, "seen before?" check.  
**Problems:** #32 #33 #34 #39 #40 #41 #42 #43 #44 #45 #46 #47 #60 #67 #91 #136 #139

```python
# ── TEMPLATE: two-pass complement lookup ───────────────────────────────
def template_two_sum(nums, target):
    seen = {}
    for i, n in enumerate(nums):
        if target - n in seen: return [seen[target - n], i]
        seen[n] = i

# ── TEMPLATE: frequency counter ────────────────────────────────────────
def template_frequency(items):
    from collections import Counter
    freq = Counter(items)
    # use freq for anagram, ransom note, top-k, etc.
    return freq

# ── TEMPLATE: group by canonical key ───────────────────────────────────
def template_group_by_key(strs):
    from collections import defaultdict
    groups = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))          # or frozenset, or Counter tuple
        groups[key].append(s)
    return list(groups.values())

# ── TEMPLATE: "seen in sliding window" (Contains Duplicate II) ─────────
def template_window_seen(nums, k):
    seen = {}
    for i, n in enumerate(nums):
        if n in seen and i - seen[n] <= k: return True
        seen[n] = i
    return False

# ── TEMPLATE: longest streak via set ───────────────────────────────────
def template_longest_streak(nums):
    s = set(nums); best = 0
    for n in s:
        if n - 1 not in s:                # only start from sequence head
            cur = n; length = 1
            while cur + 1 in s: cur += 1; length += 1
            best = max(best, length)
    return best

# ── TEMPLATE: bijection check (isomorphic / word pattern) ──────────────
def template_bijection(seq_a, seq_b):
    a2b, b2a = {}, {}
    for a, b in zip(seq_a, seq_b):
        if a2b.get(a, b) != b or b2a.get(b, a) != a: return False
        a2b[a] = b; b2a[b] = a
    return True
```

**Complexity:** Time O(n) · Space O(n)

---

## P07 Intervals (Sort + Sweep)

**When:** Overlapping/merging intervals, scheduling, arrow problems.  
**Problems:** #49 #50 #51

```python
# ── TEMPLATE: merge overlapping intervals ──────────────────────────────
def template_merge_intervals(intervals):
    intervals.sort()
    merged = [intervals[0]]
    for s, e in intervals[1:]:
        if s <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], e)
        else:
            merged.append([s, e])
    return merged

# ── TEMPLATE: insert interval ──────────────────────────────────────────
def template_insert_interval(intervals, new):
    result = []; i = 0; n = len(intervals)
    # phase 1: add all intervals ending before new starts
    while i < n and intervals[i][1] < new[0]:
        result.append(intervals[i]); i += 1
    # phase 2: merge all overlapping with new
    while i < n and intervals[i][0] <= new[1]:
        new[0] = min(new[0], intervals[i][0])
        new[1] = max(new[1], intervals[i][1]); i += 1
    result.append(new)
    return result + intervals[i:]

# ── TEMPLATE: greedy arrow / min groups (sort by end) ──────────────────
def template_min_arrows(intervals):
    intervals.sort(key=lambda x: x[1])
    arrows = 1; end = intervals[0][1]
    for s, e in intervals[1:]:
        if s > end: arrows += 1; end = e   # need a new arrow
    return arrows

# ── TEMPLATE: meeting rooms / min rooms (event sweep) ──────────────────
def template_min_rooms(intervals):
    starts = sorted(s for s, _ in intervals)
    ends   = sorted(e for _, e in intervals)
    rooms = cur = e_ptr = 0
    for s in starts:
        if s < ends[e_ptr]: cur += 1
        else: e_ptr += 1
        rooms = max(rooms, cur)
    return rooms
```

**Complexity:** Time O(n log n) · Space O(n)

---

## P08 Stack

**When:** Matching brackets, "undo last", next-greater, expression evaluation.  
**Problems:** #52 #53 #54 #55 #56

```python
# ── TEMPLATE: balanced brackets ────────────────────────────────────────
def template_brackets(s):
    stack = []; pairs = {')':'(', ']':'[', '}':'{'}
    for ch in s:
        if ch in pairs:
            if not stack or stack[-1] != pairs[ch]: return False
            stack.pop()
        else:
            stack.append(ch)
    return not stack

# ── TEMPLATE: min stack (track running min alongside values) ───────────
def template_min_stack():
    stack = []  # stores (value, current_min)
    def push(val):
        m = min(val, stack[-1][1] if stack else val)
        stack.append((val, m))
    def get_min(): return stack[-1][1]

# ── TEMPLATE: monotonic stack — next greater element ───────────────────
def template_next_greater(nums):
    result = [-1] * len(nums)
    stack = []                              # indices, decreasing values
    for i, n in enumerate(nums):
        while stack and nums[stack[-1]] < n:
            result[stack.pop()] = n
        stack.append(i)
    return result

# ── TEMPLATE: evaluate RPN ─────────────────────────────────────────────
def template_rpn(tokens):
    stack = []
    ops = {'+': lambda a,b: a+b, '-': lambda a,b: a-b,
           '*': lambda a,b: a*b, '/': lambda a,b: int(a/b)}
    for t in tokens:
        if t in ops:
            b, a = stack.pop(), stack.pop()
            stack.append(ops[t](a, b))
        else:
            stack.append(int(t))
    return stack[0]

# ── TEMPLATE: calculator with parentheses ──────────────────────────────
def template_calculator(s):
    stack = []; num = 0; sign = 1; result = 0
    for ch in s:
        if ch.isdigit(): num = num * 10 + int(ch)
        elif ch == '+':  result += sign * num; num = 0; sign =  1
        elif ch == '-':  result += sign * num; num = 0; sign = -1
        elif ch == '(':  stack.append(result); stack.append(sign); result = 0; sign = 1
        elif ch == ')':
            result += sign * num; num = 0
            result *= stack.pop(); result += stack.pop()
    return result + sign * num
```

**Complexity:** Time O(n) · Space O(n)

---

## P09 Linked List Manipulation

**When:** Reverse, merge, cycle detection, remove nth, partition.  
**Problems:** #57–#67

```python
# ── TEMPLATE: fast & slow pointers ─────────────────────────────────────
def template_fast_slow(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    # slow is now at the middle
    return slow

# ── TEMPLATE: in-place reversal ────────────────────────────────────────
def template_reverse_list(head):
    prev = None; cur = head
    while cur:
        nxt = cur.next; cur.next = prev; prev = cur; cur = nxt
    return prev                     # new head

# ── TEMPLATE: dummy head (simplifies edge cases) ───────────────────────
def template_dummy_head(head, *args):
    dummy = ListNode(0, head)
    # manipulate via dummy.next
    return dummy.next

# ── TEMPLATE: merge two sorted lists ───────────────────────────────────
def template_merge_lists(l1, l2):
    dummy = cur = ListNode(0)
    while l1 and l2:
        if l1.val <= l2.val: cur.next = l1; l1 = l1.next
        else:                cur.next = l2; l2 = l2.next
        cur = cur.next
    cur.next = l1 or l2
    return dummy.next

# ── TEMPLATE: remove nth from end (two-pointer gap) ────────────────────
def template_remove_nth(head, n):
    dummy = ListNode(0, head)
    left = dummy; right = head
    for _ in range(n): right = right.next
    while right: left = left.next; right = right.next
    left.next = left.next.next
    return dummy.next

# ── TEMPLATE: split into two lists, then merge (partition) ─────────────
def template_partition(head, x):
    less = less_head = ListNode(0)
    more = more_head = ListNode(0)
    cur = head
    while cur:
        if cur.val < x: less.next = cur; less = less.next
        else:           more.next = cur; more = more.next
        cur = cur.next
    more.next = None; less.next = more_head.next
    return less_head.next

# ── TEMPLATE: reverse in k-groups ──────────────────────────────────────
def template_reverse_k(head, k):
    dummy = ListNode(0, head); gp = dummy
    while True:
        kth = gp
        for _ in range(k):
            kth = kth.next
            if not kth: return dummy.next
        gn = kth.next
        prev, cur = gn, gp.next
        while cur != gn:
            nxt = cur.next; cur.next = prev; prev = cur; cur = nxt
        tmp = gp.next; gp.next = kth; gp = tmp
```

**Complexity:** Time O(n) · Space O(1)

---

## P10 Tree DFS (Recursion)

**When:** Path sums, depth, construction, validation, LCA — any root-to-leaf traversal.  
**Problems:** #68–#81 #86 #87 #88 #108

```python
# ── TEMPLATE: post-order (result flows UP) ─────────────────────────────
def template_postorder(root):
    def dfs(node):
        if not node: return base_value
        left  = dfs(node.left)
        right = dfs(node.right)
        return combine(node.val, left, right)
    return dfs(root)

# ── TEMPLATE: pre-order (result flows DOWN via params) ─────────────────
def template_preorder(root):
    result = []
    def dfs(node, state):
        if not node: return
        new_state = update(state, node.val)
        if is_leaf(node): result.append(new_state)
        dfs(node.left,  new_state)
        dfs(node.right, new_state)
    dfs(root, initial_state)
    return result

# ── TEMPLATE: global best (side-effect via list cell) ──────────────────
def template_global_best(root):
    best = [float('-inf')]
    def dfs(node):
        if not node: return 0
        left  = max(dfs(node.left),  0)
        right = max(dfs(node.right), 0)
        best[0] = max(best[0], node.val + left + right)  # update global
        return node.val + max(left, right)               # return to parent
    dfs(root); return best[0]

# ── TEMPLATE: range bounds validation (BST) ────────────────────────────
def template_validate_bst(root):
    def validate(node, lo, hi):
        if not node: return True
        if not (lo < node.val < hi): return False
        return validate(node.left, lo, node.val) and \
               validate(node.right, node.val, hi)
    return validate(root, float('-inf'), float('inf'))

# ── TEMPLATE: in-order iterative (BST sorted access) ───────────────────
def template_inorder_iter(root):
    stack = []; cur = root; result = []
    while stack or cur:
        while cur: stack.append(cur); cur = cur.left
        cur = stack.pop(); result.append(cur.val)
        cur = cur.right
    return result

# ── TEMPLATE: LCA (return node, propagate match upward) ─────────────────
def template_lca(root, p, q):
    if not root or root is p or root is q: return root
    left  = template_lca(root.left,  p, q)
    right = template_lca(root.right, p, q)
    return root if left and right else left or right
```

**Complexity:** Time O(n) · Space O(h) where h = tree height

---

## P11 Tree BFS (Level Order)

**When:** Level-by-level processing, right-side view, zigzag, averages.  
**Problems:** #82 #83 #84 #85

```python
# ── TEMPLATE: level-order BFS ──────────────────────────────────────────
from collections import deque

def template_level_order(root):
    if not root: return []
    result = []; q = deque([root])
    while q:
        level = []
        for _ in range(len(q)):           # snapshot size = current level
            node = q.popleft()
            level.append(node.val)
            if node.left:  q.append(node.left)
            if node.right: q.append(node.right)
        result.append(level)
    return result

# ── VARIANT: right-side view (DFS depth trick) ─────────────────────────
def template_right_view(root):
    result = []
    def dfs(node, depth):
        if not node: return
        if depth == len(result): result.append(node.val)  # first at this depth
        dfs(node.right, depth + 1)     # visit right FIRST
        dfs(node.left,  depth + 1)
    dfs(root, 0); return result

# ── VARIANT: zigzag (flip direction each level) ────────────────────────
def template_zigzag(root):
    if not root: return []
    result = []; q = deque([root]); left_to_right = True
    while q:
        level = [q.popleft().val for _ in range(len(q))]
        # note: append children AFTER popping
        result.append(level if left_to_right else level[::-1])
        left_to_right = not left_to_right
    return result

# ── VARIANT: process with proper child appending ───────────────────────
def template_level_process(root):
    if not root: return []
    result = []; q = deque([root])
    while q:
        level_size = len(q); level_vals = []
        for _ in range(level_size):
            node = q.popleft()
            level_vals.append(node.val)
            if node.left:  q.append(node.left)
            if node.right: q.append(node.right)
        result.append(level_vals)
    return result
```

**Complexity:** Time O(n) · Space O(w) where w = max width of tree

---

## P12 Graph DFS

**When:** Connected components, path existence, cycle detection, flood fill.  
**Problems:** #89 #90 #91 #93 #94

```python
# ── TEMPLATE: grid DFS (mark visited in-place) ─────────────────────────
def template_grid_dfs(grid):
    m, n = len(grid), len(grid[0])
    def dfs(r, c):
        if r < 0 or c < 0 or r >= m or c >= n: return
        if grid[r][c] != TARGET: return
        grid[r][c] = VISITED                     # mark before recursing
        for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
            dfs(r + dr, c + dc)
    count = 0
    for r in range(m):
        for c in range(n):
            if grid[r][c] == TARGET:
                dfs(r, c); count += 1
    return count

# ── TEMPLATE: graph DFS with 3-color cycle detection ───────────────────
# state: 0 = unvisited, 1 = visiting (in stack), 2 = done
def template_cycle_dfs(n, adj):
    state = [0] * n
    def dfs(node):
        if state[node] == 1: return False    # back edge → cycle
        if state[node] == 2: return True     # already processed
        state[node] = 1
        for nb in adj[node]:
            if not dfs(nb): return False
        state[node] = 2
        return True
    return all(dfs(i) for i in range(n))

# ── TEMPLATE: graph DFS with visited set + adjacency list ──────────────
def template_graph_dfs(node, adj, visited=None):
    if visited is None: visited = set()
    visited.add(node)
    for nb in adj[node]:
        if nb not in visited:
            template_graph_dfs(nb, adj, visited)
    return visited
```

**Complexity:** Time O(V + E) · Space O(V)

---

## P13 Graph BFS (Shortest Path)

**When:** Unweighted shortest path, multi-source spread, word/gene ladder.  
**Problems:** #89 #92 #95 #96 #97

```python
# ── TEMPLATE: BFS shortest path (single source) ────────────────────────
from collections import deque

def template_bfs_shortest(start, end, adj):
    q = deque([(start, 0)])
    visited = {start}
    while q:
        node, dist = q.popleft()
        if node == end: return dist
        for nb in adj[node]:
            if nb not in visited:
                visited.add(nb); q.append((nb, dist + 1))
    return -1

# ── TEMPLATE: multi-source BFS (all sources at once) ───────────────────
def template_multi_source_bfs(grid, sources):
    q = deque(sources)
    visited = set(sources)
    steps = 0
    while q:
        for _ in range(len(q)):
            r, c = q.popleft()
            for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
                nr, nc = r + dr, c + dc
                if valid(nr, nc, grid) and (nr, nc) not in visited:
                    visited.add((nr, nc)); q.append((nr, nc))
        steps += 1
    return steps

# ── TEMPLATE: word/gene BFS (mutate one char at a time) ────────────────
def template_word_bfs(begin, end, word_set):
    q = deque([(begin, 1)]); visited = {begin}
    while q:
        word, steps = q.popleft()
        for i in range(len(word)):
            for ch in 'abcdefghijklmnopqrstuvwxyz':
                nw = word[:i] + ch + word[i+1:]
                if nw == end: return steps + 1
                if nw in word_set and nw not in visited:
                    visited.add(nw); q.append((nw, steps + 1))
    return 0

# ── TEMPLATE: BFS on weighted graph (Evaluate Division) ─────────────────
def template_weighted_bfs(src, dst, graph):
    if src not in graph or dst not in graph: return -1.0
    q = deque([(src, 1.0)]); visited = {src}
    while q:
        node, prod = q.popleft()
        if node == dst: return prod
        for nb, w in graph[node].items():
            if nb not in visited:
                visited.add(nb); q.append((nb, prod * w))
    return -1.0
```

**Complexity:** Time O(V + E) · Space O(V)

---

## P14 Topological Sort

**When:** Dependency ordering, detect cycles in directed graphs.  
**Problems:** #93 #94

```python
from collections import deque

# ── TEMPLATE: Kahn's algorithm (BFS, in-degree) ────────────────────────
def template_kahn(n, prerequisites):
    adj = [[] for _ in range(n)]
    in_degree = [0] * n
    for a, b in prerequisites:
        adj[b].append(a); in_degree[a] += 1
    q = deque(i for i in range(n) if in_degree[i] == 0)
    order = []
    while q:
        node = q.popleft(); order.append(node)
        for nb in adj[node]:
            in_degree[nb] -= 1
            if in_degree[nb] == 0: q.append(nb)
    return order if len(order) == n else []     # [] means cycle exists

# ── TEMPLATE: DFS post-order topological sort ──────────────────────────
def template_topo_dfs(n, prerequisites):
    adj = [[] for _ in range(n)]
    for a, b in prerequisites: adj[b].append(a)
    state = [0] * n; order = []
    def dfs(node):
        if state[node] == 1: return False      # cycle
        if state[node] == 2: return True
        state[node] = 1
        for nb in adj[node]:
            if not dfs(nb): return False
        state[node] = 2; order.append(node)
        return True
    if not all(dfs(i) for i in range(n)): return []
    return order[::-1]                         # reverse post-order
```

**Complexity:** Time O(V + E) · Space O(V)

---

## P15 Trie

**When:** Prefix search, autocomplete, dictionary word matching.  
**Problems:** #98 #99 #100

```python
# ── TEMPLATE: Trie node + operations ───────────────────────────────────
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end   = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def search(self, word: str) -> bool:
        node = self.root
        for ch in word:
            if ch not in node.children: return False
            node = node.children[ch]
        return node.is_end

    def starts_with(self, prefix: str) -> bool:
        node = self.root
        for ch in prefix:
            if ch not in node.children: return False
            node = node.children[ch]
        return True

# ── VARIANT: wildcard search with DFS ──────────────────────────────────
    def search_wildcard(self, word: str) -> bool:
        def dfs(node, i):
            if i == len(word): return node.is_end
            ch = word[i]
            if ch == '.':
                return any(dfs(child, i+1) for child in node.children.values())
            if ch not in node.children: return False
            return dfs(node.children[ch], i+1)
        return dfs(self.root, 0)

# ── VARIANT: Trie + backtracking (Word Search II) ──────────────────────
def template_trie_board_search(board, words):
    root = TrieNode()
    for w in words:
        node = root
        for ch in w: node = node.children.setdefault(ch, TrieNode())
        node.is_end = True; node.word = w   # store word at leaf

    m, n = len(board), len(board[0]); result = []
    def dfs(node, r, c):
        ch = board[r][c]
        if ch not in node.children: return
        nxt = node.children[ch]
        if nxt.is_end: result.append(nxt.word); nxt.is_end = False
        board[r][c] = '#'
        for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
            nr, nc = r+dr, c+dc
            if 0<=nr<m and 0<=nc<n and board[nr][nc]!='#': dfs(nxt, nr, nc)
        board[r][c] = ch
    for r in range(m):
        for c in range(n): dfs(root, r, c)
    return result
```

**Complexity:** insert/search O(m) per word · Space O(ALPHABET × N × M)

---

## P16 Backtracking

**When:** Generate all combinations/permutations/subsets, constraint satisfaction.  
**Problems:** #101 #102 #103 #104 #105 #106 #107

```python
# ── TEMPLATE: choose → explore → undo ──────────────────────────────────
def template_backtrack(choices, target):
    result = []
    def backtrack(start, path, state):
        if is_done(path, state, target):
            result.append(list(path)); return
        for i in range(start, len(choices)):
            if should_skip(i, path, choices): continue  # pruning
            path.append(choices[i])
            backtrack(next_start(i), path, update(state, choices[i]))
            path.pop()                                  # undo
    backtrack(0, [], initial_state)
    return result

# ── VARIANT: combinations (start advances to avoid repeats) ────────────
def template_combinations(n, k):
    result = []
    def bt(start, path):
        if len(path) == k: result.append(list(path)); return
        for i in range(start, n + 1):
            path.append(i); bt(i + 1, path); path.pop()
    bt(1, []); return result

# ── VARIANT: permutations (use remaining list) ─────────────────────────
def template_permutations(nums):
    result = []
    def bt(path, remaining):
        if not remaining: result.append(path[:]); return
        for i, n in enumerate(remaining):
            path.append(n)
            bt(path, remaining[:i] + remaining[i+1:])
            path.pop()
    bt([], nums); return result

# ── VARIANT: combination sum (can reuse elements, prune by remainder) ──
def template_combination_sum(candidates, target):
    candidates.sort(); result = []
    def bt(start, path, rem):
        if rem == 0: result.append(list(path)); return
        for i in range(start, len(candidates)):
            if candidates[i] > rem: break   # pruning
            path.append(candidates[i])
            bt(i, path, rem - candidates[i])   # i not i+1 → reuse allowed
            path.pop()
    bt(0, [], target); return result

# ── VARIANT: N-Queens (constraint bitmask) ─────────────────────────────
def template_nqueens(n):
    cols = set(); d1 = set(); d2 = set(); count = [0]
    def bt(row):
        if row == n: count[0] += 1; return
        for col in range(n):
            if col in cols or row-col in d1 or row+col in d2: continue
            cols.add(col); d1.add(row-col); d2.add(row+col)
            bt(row + 1)
            cols.discard(col); d1.discard(row-col); d2.discard(row+col)
    bt(0); return count[0]

# ── VARIANT: grid word search ──────────────────────────────────────────
def template_word_search(board, word):
    m, n = len(board), len(board[0])
    def bt(r, c, i):
        if i == len(word): return True
        if r<0 or c<0 or r>=m or c>=n or board[r][c] != word[i]: return False
        tmp = board[r][c]; board[r][c] = '#'
        found = any(bt(r+dr, c+dc, i+1) for dr,dc in [(0,1),(0,-1),(1,0),(-1,0)])
        board[r][c] = tmp; return found
    return any(bt(r, c, 0) for r in range(m) for c in range(n))
```

**Complexity:** Time O(k^n) varies · Space O(depth)

---

## P17 Divide & Conquer

**When:** Split problem in half, solve recursively, merge results.  
**Problems:** #108 #109 #110 #111 #135

```python
# ── TEMPLATE: divide, recurse, merge ───────────────────────────────────
def template_divide_conquer(arr, lo, hi):
    if lo >= hi: return base_case(arr[lo])
    mid = (lo + hi) // 2
    left  = template_divide_conquer(arr, lo, mid)
    right = template_divide_conquer(arr, mid + 1, hi)
    return merge(left, right)

# ── VARIANT: merge sort on linked list ─────────────────────────────────
def template_sort_list(head):
    if not head or not head.next: return head
    # find middle using fast/slow
    slow, fast = head, head.next
    while fast and fast.next: slow = slow.next; fast = fast.next.next
    mid = slow.next; slow.next = None
    left  = template_sort_list(head)
    right = template_sort_list(mid)
    return merge_two_sorted(left, right)

def merge_two_sorted(l1, l2):
    dummy = cur = ListNode(0)
    while l1 and l2:
        if l1.val <= l2.val: cur.next = l1; l1 = l1.next
        else:                cur.next = l2; l2 = l2.next
        cur = cur.next
    cur.next = l1 or l2; return dummy.next

# ── VARIANT: fast exponentiation ───────────────────────────────────────
def template_fast_pow(x, n):
    if n < 0: x, n = 1/x, -n
    result = 1.0
    while n:
        if n % 2: result *= x
        x *= x; n //= 2
    return result

# ── VARIANT: merge k sorted lists (heap) ───────────────────────────────
import heapq
def template_merge_k(lists):
    dummy = cur = ListNode(0)
    heap = []
    for i, node in enumerate(lists):
        if node: heapq.heappush(heap, (node.val, i, node))
    while heap:
        val, i, node = heapq.heappop(heap)
        cur.next = node; cur = cur.next
        if node.next: heapq.heappush(heap, (node.next.val, i, node.next))
    return dummy.next
```

**Complexity:** Time O(n log n) typical · Space O(log n)

---

## P18 Kadane's Algorithm

**When:** Maximum (or minimum) subarray sum, contiguous window.  
**Problems:** #112 #113

```python
# ── TEMPLATE: classic Kadane ────────────────────────────────────────────
def template_kadane(nums):
    cur = best = nums[0]
    for n in nums[1:]:
        cur  = max(n, cur + n)    # restart vs extend
        best = max(best, cur)
    return best

# ── VARIANT: circular subarray ──────────────────────────────────────────
# max of: (a) best normal subarray, (b) total - worst normal subarray
def template_circular_kadane(nums):
    total = sum(nums)

    # max subarray (standard Kadane)
    max_cur = max_sum = nums[0]
    for n in nums[1:]:
        max_cur = max(n, max_cur + n); max_sum = max(max_sum, max_cur)

    # min subarray (Kadane for min)
    min_cur = min_sum = nums[0]
    for n in nums[1:]:
        min_cur = min(n, min_cur + n); min_sum = min(min_sum, min_cur)

    # if max_sum <= 0, all numbers are negative → return max_sum
    return max(max_sum, total - min_sum) if max_sum > 0 else max_sum

# ── VARIANT: track start/end indices ────────────────────────────────────
def template_kadane_indices(nums):
    cur = best = nums[0]
    start = end = temp_start = 0
    for i in range(1, len(nums)):
        if nums[i] > cur + nums[i]:
            cur = nums[i]; temp_start = i
        else:
            cur += nums[i]
        if cur > best:
            best = cur; start = temp_start; end = i
    return best, start, end
```

**Complexity:** Time O(n) · Space O(1)

---

## P19 Binary Search

**When:** Sorted array OR answer lies in a monotonic search space.  
**Problems:** #114 #115 #116 #117 #118 #119 #120 #134 #141

```python
# ── TEMPLATE: classic (find exact value) ───────────────────────────────
def template_binary_search(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if   nums[mid] == target: return mid
        elif nums[mid]  < target: left  = mid + 1
        else:                     right = mid - 1
    return -1

# ── TEMPLATE: find leftmost (lower bound) ──────────────────────────────
def template_lower_bound(nums, target):
    left, right = 0, len(nums)
    while left < right:
        mid = (left + right) // 2
        if nums[mid] < target: left = mid + 1
        else:                  right = mid
    return left    # insertion point / first occurrence

# ── TEMPLATE: find rightmost (upper bound - 1) ─────────────────────────
def template_upper_bound(nums, target):
    left, right = 0, len(nums)
    while left < right:
        mid = (left + right) // 2
        if nums[mid] <= target: left = mid + 1
        else:                   right = mid
    return left - 1   # last occurrence

# ── TEMPLATE: binary search on answer (min/max feasibility) ────────────
def template_bs_on_answer(nums, constraint):
    def feasible(mid):
        # can we satisfy constraint with mid as the answer?
        return True   # implement check here

    left, right = min_possible, max_possible
    while left < right:
        mid = (left + right) // 2
        if feasible(mid): right = mid      # find minimum feasible
        else:             left  = mid + 1
    return left

# ── TEMPLATE: rotated sorted array ─────────────────────────────────────
def template_rotated_search(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if nums[mid] == target: return mid
        if nums[left] <= nums[mid]:        # left half is sorted
            if nums[left] <= target < nums[mid]: right = mid - 1
            else:                               left  = mid + 1
        else:                              # right half is sorted
            if nums[mid] < target <= nums[right]: left  = mid + 1
            else:                                 right = mid - 1
    return -1
```

**Complexity:** Time O(log n) · Space O(1)

---

## P20 Heap / Priority Queue

**When:** Top-k, streaming median, greedy with priority, merge k sorted.  
**Problems:** #111 #121 #122 #123 #124

```python
import heapq

# ── TEMPLATE: min-heap of size k (top-k largest) ───────────────────────
def template_top_k_largest(nums, k):
    heap = nums[:k]
    heapq.heapify(heap)                 # min-heap of k elements
    for n in nums[k:]:
        if n > heap[0]:
            heapq.heapreplace(heap, n)  # pop min, push n
    return heap[0]                      # kth largest

# ── TEMPLATE: max-heap (negate values in Python) ───────────────────────
def template_max_heap(nums):
    heap = [-n for n in nums]
    heapq.heapify(heap)
    return -heapq.heappop(heap)         # largest element

# ── TEMPLATE: two-heap for median ──────────────────────────────────────
def template_two_heap_median():
    small = []   # max-heap (negate) — lower half
    large = []   # min-heap          — upper half

    def add(num):
        heapq.heappush(small, -num)
        # balance: ensure small[max] <= large[min]
        if large and -small[0] > large[0]:
            heapq.heappush(large, -heapq.heappop(small))
        # balance sizes: |small| == |large| or |small| = |large| + 1
        if len(small) > len(large) + 1:
            heapq.heappush(large, -heapq.heappop(small))
        elif len(large) > len(small):
            heapq.heappush(small, -heapq.heappop(large))

    def median():
        if len(small) > len(large): return float(-small[0])
        return (-small[0] + large[0]) / 2.0

    return add, median

# ── TEMPLATE: k-way merge with heap ────────────────────────────────────
def template_k_way_merge(sorted_lists):
    result = []
    heap = [(lst[0], i, 0) for i, lst in enumerate(sorted_lists) if lst]
    heapq.heapify(heap)
    while heap:
        val, i, j = heapq.heappop(heap)
        result.append(val)
        if j + 1 < len(sorted_lists[i]):
            heapq.heappush(heap, (sorted_lists[i][j+1], i, j+1))
    return result

# ── TEMPLATE: greedy with heap (IPO-style) ─────────────────────────────
def template_greedy_heap(projects, k, initial):
    projects_sorted = sorted(zip(capital, profits))
    available = []   # max-heap of profits
    w = initial; i = 0
    for _ in range(k):
        while i < len(projects_sorted) and projects_sorted[i][0] <= w:
            heapq.heappush(available, -projects_sorted[i][1]); i += 1
        if not available: break
        w += -heapq.heappop(available)
    return w
```

**Complexity:** push/pop O(log k) · heapify O(n) · Space O(k)

---

## P21 Union Find

**When:** Dynamic connectivity, grouping components, cycle detection in undirected graphs.  
**Problems:** #89 (alternative)

```python
# ── TEMPLATE: Union Find with path compression + union by rank ─────────
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank   = [0] * n
        self.count  = n            # number of components

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])   # path compression
        return self.parent[x]

    def union(self, x, y) -> bool:
        px, py = self.find(x), self.find(y)
        if px == py: return False              # already connected (cycle!)
        if self.rank[px] < self.rank[py]: px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]: self.rank[px] += 1
        self.count -= 1
        return True

    def connected(self, x, y) -> bool:
        return self.find(x) == self.find(y)

# ── USAGE ──────────────────────────────────────────────────────────────
# uf = UnionFind(n)
# for u, v in edges:
#     if not uf.union(u, v):
#         # u and v already connected → cycle
# uf.count   # number of connected components
```

**Complexity:** nearly O(1) per operation (inverse Ackermann) · Space O(n)

---

## P22 Bit Manipulation

**When:** XOR tricks, count bits, check/set/clear specific bits.  
**Problems:** #125 #126 #127 #128 #129 #130

```python
# ── TEMPLATE: XOR all elements (find single number) ────────────────────
def template_xor_single(nums):
    result = 0
    for n in nums: result ^= n    # pairs cancel to 0
    return result

# ── TEMPLATE: Brian Kernighan — count set bits ─────────────────────────
def template_count_bits(n):
    count = 0
    while n: n &= n - 1; count += 1   # n & (n-1) clears lowest set bit
    return count

# ── TEMPLATE: DP count bits for 0..n ──────────────────────────────────
def template_dp_count_bits(n):
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        dp[i] = dp[i >> 1] + (i & 1)   # dp[i] = dp[i//2] + last_bit
    return dp

# ── TEMPLATE: reverse 32 bits ──────────────────────────────────────────
def template_reverse_bits(n):
    result = 0
    for _ in range(32):
        result = (result << 1) | (n & 1); n >>= 1
    return result

# ── TEMPLATE: single number appearing once among triples ───────────────
def template_single_in_triples(nums):
    ones = twos = 0
    for n in nums:
        ones = (ones ^ n) & ~twos
        twos = (twos ^ n) & ~ones
    return ones

# ── TEMPLATE: range bitwise AND (find common prefix) ───────────────────
def template_range_and(left, right):
    shift = 0
    while left != right:
        left >>= 1; right >>= 1; shift += 1
    return left << shift

# ── COMMON BIT TRICKS ──────────────────────────────────────────────────
# x & (x-1)       → clear lowest set bit
# x & (-x)        → isolate lowest set bit
# x | (x+1)       → set lowest zero bit
# x ^ x  == 0     → self-cancellation
# (x >> k) & 1    → kth bit value
# x | (1 << k)    → set kth bit
# x & ~(1 << k)   → clear kth bit
# x ^ (1 << k)    → toggle kth bit
```

**Complexity:** Time O(1) or O(log n) · Space O(1)

---

## P23 Math

**When:** Number theory, digit manipulation, geometry.  
**Problems:** #131 #132 #133 #134 #135 #136

```python
# ── TEMPLATE: digit extraction ─────────────────────────────────────────
def template_digits(n):
    digits = []
    while n > 0:
        digits.append(n % 10); n //= 10
    return digits[::-1]   # most significant first

# ── TEMPLATE: GCD / LCM ───────────────────────────────────────────────
from math import gcd
def lcm(a, b): return a * b // gcd(a, b)

# ── TEMPLATE: count trailing zeroes (factors of 5) ─────────────────────
def template_trailing_zeroes(n):
    count = 0
    while n >= 5: n //= 5; count += n
    return count

# ── TEMPLATE: fast power / Sqrt via binary search ──────────────────────
def template_isqrt(x):
    left, right = 0, x
    while left <= right:
        mid = (left + right) // 2
        if mid * mid <= x < (mid+1)*(mid+1): return mid
        elif mid * mid < x: left = mid + 1
        else:               right = mid - 1
    return right

# ── TEMPLATE: max points on a line (slope hash) ────────────────────────
def template_max_collinear(points):
    from math import gcd; from collections import defaultdict
    n = len(points); best = 0
    for i in range(n):
        slopes = defaultdict(int)
        for j in range(i+1, n):
            dx = points[j][0] - points[i][0]
            dy = points[j][1] - points[i][1]
            g = gcd(abs(dx), abs(dy))
            slope = (dy//g, dx//g)
            slopes[slope] += 1; best = max(best, slopes[slope] + 1)
    return best
```

**Complexity:** Varies · mostly O(log n) or O(n²)

---

## P24 1D Dynamic Programming

**When:** Overlapping subproblems with 1D state (index, capacity, amount).  
**Problems:** #9 #10 #137 #138 #139 #140 #141

```python
# ── TEMPLATE: linear DP (Climbing Stairs / Fibonacci-style) ───────────
def template_linear_dp(n):
    a, b = 1, 1   # base cases
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b

# ── TEMPLATE: house robber (skip-or-take) ─────────────────────────────
def template_skip_or_take(nums):
    prev2 = prev1 = 0
    for n in nums:
        prev2, prev1 = prev1, max(prev1, prev2 + n)
    return prev1

# ── TEMPLATE: unbounded knapsack (Coin Change) ─────────────────────────
def template_coin_change(coins, amount):
    dp = [float('inf')] * (amount + 1); dp[0] = 0
    for coin in coins:
        for a in range(coin, amount + 1):
            dp[a] = min(dp[a], dp[a - coin] + 1)
    return dp[amount] if dp[amount] < float('inf') else -1

# ── TEMPLATE: word break (segment DP) ─────────────────────────────────
def template_word_break(s, wordDict):
    words = set(wordDict); n = len(s)
    dp = [False] * (n + 1); dp[0] = True
    for i in range(1, n + 1):
        for j in range(i):
            if dp[j] and s[j:i] in words: dp[i] = True; break
    return dp[n]

# ── TEMPLATE: LIS via patience sorting (binary search) ────────────────
import bisect
def template_lis(nums):
    tails = []
    for n in nums:
        pos = bisect.bisect_left(tails, n)
        if pos == len(tails): tails.append(n)
        else:                 tails[pos] = n
    return len(tails)
```

**Complexity:** Time O(n) or O(n²) · Space O(n) → O(1) with rolling vars

---

## P25 Multi-dim / 2D DP

**When:** Two sequences (edit distance, LCS), grid path, palindrome, interleaving.  
**Problems:** #142 #143 #144 #145 #146 #147 #150

```python
# ── TEMPLATE: 2D grid DP (min path sum) ───────────────────────────────
def template_grid_dp(grid):
    m, n = len(grid), len(grid[0])
    dp = [[0]*n for _ in range(m)]
    dp[0][0] = grid[0][0]
    for r in range(m):
        for c in range(n):
            if r == 0 and c == 0: continue
            from_top  = dp[r-1][c] if r > 0 else float('inf')
            from_left = dp[r][c-1] if c > 0 else float('inf')
            dp[r][c] = grid[r][c] + min(from_top, from_left)
    return dp[m-1][n-1]

# ── TEMPLATE: two-sequence DP (edit distance) ──────────────────────────
def template_edit_distance(w1, w2):
    m, n = len(w1), len(w2)
    dp = list(range(n + 1))          # rolling 1-D array
    for i in range(1, m + 1):
        prev = dp[0]; dp[0] = i
        for j in range(1, n + 1):
            tmp = dp[j]
            if w1[i-1] == w2[j-1]: dp[j] = prev
            else: dp[j] = 1 + min(prev, dp[j], dp[j-1])
            prev = tmp
    return dp[n]

# ── TEMPLATE: palindrome expand around center ──────────────────────────
def template_palindrome_expand(s):
    start = length = 0
    def expand(l, r):
        nonlocal start, length
        while l >= 0 and r < len(s) and s[l] == s[r]:
            if r - l + 1 > length: start = l; length = r - l + 1
            l -= 1; r += 1
    for i in range(len(s)): expand(i, i); expand(i, i+1)
    return s[start:start+length]

# ── TEMPLATE: maximal square DP ───────────────────────────────────────
def template_maximal_square(matrix):
    m, n = len(matrix), len(matrix[0])
    dp = [0]*(n+1); prev = best = 0
    for r in range(m):
        for c in range(1, n+1):
            tmp = dp[c]
            if matrix[r][c-1] == '1':
                dp[c] = min(dp[c], dp[c-1], prev) + 1
                best = max(best, dp[c])
            else: dp[c] = 0
            prev = tmp
    return best * best

# ── TEMPLATE: triangle bottom-up ──────────────────────────────────────
def template_triangle(triangle):
    dp = triangle[-1][:]
    for row in range(len(triangle) - 2, -1, -1):
        for col in range(len(triangle[row])):
            dp[col] = triangle[row][col] + min(dp[col], dp[col+1])
    return dp[0]
```

**Complexity:** Time O(m×n) · Space O(n) with rolling array

---

## P26 State Machine DP

**When:** Multiple discrete states with transitions (buy/sell/cooldown/hold).  
**Problems:** #148 #149

```python
# ── TEMPLATE: 2-transaction state machine (Stock III) ─────────────────
def template_stock_2_tx(prices):
    # States: buy1, sell1, buy2, sell2
    buy1 = buy2 = float('-inf')
    sell1 = sell2 = 0
    for p in prices:
        buy1  = max(buy1,  -p)
        sell1 = max(sell1,  buy1  + p)
        buy2  = max(buy2,   sell1 - p)
        sell2 = max(sell2,  buy2  + p)
    return sell2

# ── TEMPLATE: k-transaction state machine (Stock IV) ──────────────────
def template_stock_k_tx(k, prices):
    n = len(prices)
    if k >= n // 2:
        # unlimited transactions
        return sum(max(prices[i] - prices[i-1], 0) for i in range(1, n))
    buy  = [float('-inf')] * k
    sell = [0] * k
    for p in prices:
        for i in range(k):
            buy[i]  = max(buy[i],  (sell[i-1] if i > 0 else 0) - p)
            sell[i] = max(sell[i],  buy[i] + p)
    return sell[-1]

# ── TEMPLATE: generic state machine ────────────────────────────────────
# Define states as variables; for each price update all states.
# Transition: new_state = max(stay_in_state, transition_from_other_state)
#
# Example with cooldown:
def template_stock_cooldown(prices):
    held = float('-inf')   # holding stock
    sold = 0               # just sold (cooldown next)
    rest = 0               # resting (can buy)
    for p in prices:
        prev_held = held; prev_sold = sold; prev_rest = rest
        held = max(prev_held, prev_rest - p)   # buy only from rest
        sold = prev_held + p                    # sell
        rest = max(prev_rest, prev_sold)        # cooldown ends
    return max(sold, rest)
```

**Complexity:** Time O(n×k) · Space O(k)

---

## Summary Table — All 150 Problems + Pattern Tag

| # | Problem | Pattern |
|---|---|---|
| 1 | Merge Sorted Array | P01 |
| 2 | Remove Element | P01 |
| 3 | Remove Duplicates from Sorted Array | P01 |
| 4 | Remove Duplicates from Sorted Array II | P01 |
| 5 | Majority Element | P02 |
| 6 | Rotate Array | P01 |
| 7 | Best Time to Buy and Sell Stock | P02 |
| 8 | Best Time to Buy and Sell Stock II | P02 |
| 9 | Jump Game | P02 + P24 |
| 10 | Jump Game II | P02 + P24 |
| 11 | H-Index | P02 |
| 12 | Insert Delete GetRandom O(1) | P03 |
| 13 | Product of Array Except Self | P04 |
| 14 | Gas Station | P02 |
| 15 | Candy | P02 |
| 16 | Trapping Rain Water | P04 |
| 17 | Roman to Integer | P03 |
| 18 | Integer to Roman | P03 |
| 19 | Length of Last Word | P03 |
| 20 | Longest Common Prefix | P03 |
| 21 | Reverse Words in a String | P03 |
| 22 | Zigzag Conversion | P03 |
| 23 | Find the Index of the First Occurrence | P03 |
| 24 | Text Justification | P03 |
| 25 | Valid Palindrome | P04 |
| 26 | Is Subsequence | P04 |
| 27 | Two Sum II | P04 |
| 28 | Container With Most Water | P04 |
| 29 | 3Sum | P04 |
| 30 | Minimum Size Subarray Sum | P05 |
| 31 | Longest Substring Without Repeating Characters | P05 |
| 32 | Substring with Concatenation of All Words | P05 + P06 |
| 33 | Minimum Window Substring | P05 + P06 |
| 34 | Valid Sudoku | P03 + P06 |
| 35 | Spiral Matrix | P03 |
| 36 | Rotate Image | P03 |
| 37 | Set Matrix Zeroes | P03 |
| 38 | Game of Life | P03 |
| 39 | Ransom Note | P06 |
| 40 | Isomorphic Strings | P06 |
| 41 | Word Pattern | P06 |
| 42 | Valid Anagram | P06 |
| 43 | Group Anagrams | P06 |
| 44 | Two Sum | P06 |
| 45 | Happy Number | P06 |
| 46 | Contains Duplicate II | P06 |
| 47 | Longest Consecutive Sequence | P06 |
| 48 | Summary Ranges | P03 |
| 49 | Merge Intervals | P07 |
| 50 | Insert Interval | P07 |
| 51 | Minimum Number of Arrows to Burst Balloons | P07 + P02 |
| 52 | Valid Parentheses | P08 |
| 53 | Simplify Path | P08 |
| 54 | Min Stack | P08 |
| 55 | Evaluate Reverse Polish Notation | P08 |
| 56 | Basic Calculator | P08 |
| 57 | Linked List Cycle | P09 |
| 58 | Add Two Numbers | P09 |
| 59 | Merge Two Sorted Lists | P09 |
| 60 | Copy List with Random Pointer | P09 + P06 |
| 61 | Reverse Linked List II | P09 |
| 62 | Reverse Nodes in k-Group | P09 |
| 63 | Remove Nth Node From End | P09 |
| 64 | Remove Duplicates from Sorted List II | P09 |
| 65 | Rotate List | P09 |
| 66 | Partition List | P09 |
| 67 | LRU Cache | P09 + P06 |
| 68 | Maximum Depth of Binary Tree | P10 |
| 69 | Same Tree | P10 |
| 70 | Invert Binary Tree | P10 |
| 71 | Symmetric Tree | P10 |
| 72 | Construct BT from Preorder + Inorder | P10 + P06 |
| 73 | Construct BT from Inorder + Postorder | P10 + P06 |
| 74 | Populating Next Right Pointers II | P10 / P11 |
| 75 | Flatten Binary Tree to Linked List | P10 |
| 76 | Path Sum | P10 |
| 77 | Sum Root to Leaf Numbers | P10 |
| 78 | Binary Tree Maximum Path Sum | P10 |
| 79 | Binary Search Tree Iterator | P10 |
| 80 | Count Complete Tree Nodes | P10 + P19 |
| 81 | Lowest Common Ancestor of Binary Tree | P10 |
| 82 | Binary Tree Right Side View | P11 |
| 83 | Average of Levels in Binary Tree | P11 |
| 84 | Binary Tree Level Order Traversal | P11 |
| 85 | Binary Tree Zigzag Level Order Traversal | P11 |
| 86 | Minimum Absolute Difference in BST | P10 |
| 87 | Kth Smallest Element in a BST | P10 |
| 88 | Validate Binary Search Tree | P10 |
| 89 | Number of Islands | P12 / P13 / P21 |
| 90 | Surrounded Regions | P12 |
| 91 | Clone Graph | P12 + P06 |
| 92 | Evaluate Division | P13 |
| 93 | Course Schedule | P12 + P14 |
| 94 | Course Schedule II | P14 |
| 95 | Snakes and Ladders | P13 |
| 96 | Minimum Genetic Mutation | P13 |
| 97 | Word Ladder | P13 |
| 98 | Implement Trie | P15 |
| 99 | Design Add and Search Words | P15 + P12 |
| 100 | Word Search II | P15 + P16 |
| 101 | Letter Combinations of a Phone Number | P16 |
| 102 | Combinations | P16 |
| 103 | Permutations | P16 |
| 104 | Combination Sum | P16 |
| 105 | N-Queens II | P16 |
| 106 | Generate Parentheses | P16 |
| 107 | Word Search | P16 |
| 108 | Convert Sorted Array to BST | P17 |
| 109 | Sort List | P17 |
| 110 | Construct Quad Tree | P17 |
| 111 | Merge k Sorted Lists | P17 + P20 |
| 112 | Maximum Subarray | P18 |
| 113 | Maximum Sum Circular Subarray | P18 |
| 114 | Search Insert Position | P19 |
| 115 | Search a 2D Matrix | P19 |
| 116 | Find Peak Element | P19 |
| 117 | Search in Rotated Sorted Array | P19 |
| 118 | Find First and Last Position of Element | P19 |
| 119 | Find Minimum in Rotated Sorted Array | P19 |
| 120 | Median of Two Sorted Arrays | P19 |
| 121 | Kth Largest Element in an Array | P20 |
| 122 | IPO | P20 + P02 |
| 123 | Find K Pairs with Smallest Sums | P20 |
| 124 | Find Median from Data Stream | P20 |
| 125 | Add Binary | P22 |
| 126 | Reverse Bits | P22 |
| 127 | Number of 1 Bits | P22 |
| 128 | Single Number | P22 |
| 129 | Single Number II | P22 |
| 130 | Bitwise AND of Numbers Range | P22 |
| 131 | Palindrome Number | P23 |
| 132 | Plus One | P23 |
| 133 | Factorial Trailing Zeroes | P23 |
| 134 | Sqrt(x) | P19 + P23 |
| 135 | Pow(x, n) | P17 + P23 |
| 136 | Max Points on a Line | P23 + P06 |
| 137 | Climbing Stairs | P24 |
| 138 | House Robber | P24 |
| 139 | Word Break | P24 + P06 |
| 140 | Coin Change | P24 |
| 141 | Longest Increasing Subsequence | P24 + P19 |
| 142 | Triangle | P25 |
| 143 | Minimum Path Sum | P25 |
| 144 | Unique Paths II | P25 |
| 145 | Longest Palindromic Substring | P25 |
| 146 | Interleaving String | P25 |
| 147 | Edit Distance | P25 |
| 148 | Best Time to Buy and Sell Stock III | P26 |
| 149 | Best Time to Buy and Sell Stock IV | P26 |
| 150 | Maximal Square | P25 |
MDEOF
