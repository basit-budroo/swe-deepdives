# DSA Patterns & LeetCode Cheatsheet (Java)

Fast-revision guide covering the most important DSA topics and LeetCode patterns for interviews/OAs. Every pattern has: **when to use it**, **template code (Java)**, and **example problems**.

## Table of Contents

1. [Arrays & Strings](#1-arrays--strings)
2. [Two Pointers](#2-two-pointers)
3. [Sliding Window](#3-sliding-window)
4. [Prefix Sum / Difference Array](#4-prefix-sum--difference-array)
5. [Binary Search](#5-binary-search)
6. [Kadane's Algorithm (Max Subarray)](#6-kadanes-algorithm-max-subarray)
7. [Sorting-based Patterns](#7-sorting-based-patterns)
8. [Linked List](#8-linked-list)
9. [Stack](#9-stack)
10. [Monotonic Stack](#10-monotonic-stack)
11. [Queue & Deque](#11-queue--deque)
12. [Monotonic Deque (Sliding Window Max/Min)](#12-monotonic-deque-sliding-window-maxmin)
13. [Hashing (HashMap/HashSet)](#13-hashing-hashmaphashset)
14. [Trees (Binary Tree / BST)](#14-trees-binary-tree--bst)
15. [Tries](#15-tries)
16. [Heaps / Priority Queue](#16-heaps--priority-queue)
17. [Graphs — BFS / DFS](#17-graphs--bfs--dfs)
18. [Union-Find (Disjoint Set)](#18-union-find-disjoint-set)
19. [Topological Sort](#19-topological-sort)
20. [Shortest Path Algorithms](#20-shortest-path-algorithms)
21. [Minimum Spanning Tree](#21-minimum-spanning-tree)
22. [Backtracking](#22-backtracking)
23. [Dynamic Programming](#23-dynamic-programming)
24. [Greedy](#24-greedy)
25. [Bit Manipulation](#25-bit-manipulation)
26. [Intervals](#26-intervals)
27. [Matrix Patterns](#27-matrix-patterns)
28. [Fast & Slow Pointers (Cycle Detection)](#28-fast--slow-pointers-cycle-detection)
29. [Top-K / K-way Merge](#29-top-k--k-way-merge)
30. [String Algorithms](#30-string-algorithms)
31. [Design Problems](#31-design-problems)
32. [Recurrence Relations & Simulation Problems](#32-recurrence-relations--simulation-problems)
33. [Segment Tree / Fenwick Tree (Range Queries)](#33-segment-tree--fenwick-tree-range-queries)
34. [Sweep Line Algorithm](#34-sweep-line-algorithm)
35. [Advanced Graph Algorithms (SCC, Bridges, Articulation Points)](#35-advanced-graph-algorithms-scc-bridges-articulation-points)
36. [Math Essentials (GCD, Sieve, Fast Exponentiation)](#36-math-essentials-gcd-sieve-fast-exponentiation)
37. [Game Theory / Minimax DP](#37-game-theory--minimax-dp)
38. [Complexity Cheatsheet](#38-complexity-cheatsheet)

### Problem Count Breakup

| # | Section | Problems |
|----:|----|----:|
| 1 | Arrays & Strings | 16 |
| 2 | Two Pointers | 6 |
| 3 | Sliding Window | 6 |
| 4 | Prefix Sum / Difference Array | 4 |
| 5 | Binary Search | 7 |
| 6 | Kadane's Algorithm (Max Subarray) | 4 |
| 7 | Sorting-based Patterns | 5 |
| 8 | Linked List | 7 |
| 9 | Stack | 7 |
| 10 | Monotonic Stack | 6 |
| 11 | Queue & Deque | 5 |
| 12 | Monotonic Deque (Sliding Window Max/Min) | 3 |
| 13 | Hashing (HashMap/HashSet) | 6 |
| 14 | Trees (Binary Tree / BST) | 10 |
| 15 | Tries | 5 |
| 16 | Heaps / Priority Queue | 6 |
| 17 | Graphs — BFS / DFS | 8 |
| 18 | Union-Find (Disjoint Set) | 6 |
| 19 | Topological Sort | 5 |
| 20 | Shortest Path Algorithms | 4 |
| 21 | Minimum Spanning Tree | 3 |
| 22 | Backtracking | 8 |
| 23 | Dynamic Programming | 22 |
| 24 | Greedy | 7 |
| 25 | Bit Manipulation | 6 |
| 26 | Intervals | 6 |
| 27 | Matrix Patterns | 7 |
| 28 | Fast & Slow Pointers (Cycle Detection) | 5 |
| 29 | Top-K / K-way Merge | 4 |
| 30 | String Algorithms | 7 |
| 31 | Design Problems | 7 |
| 32 | Recurrence Relations & Simulation Problems | 7 |
| 33 | Segment Tree / Fenwick Tree (Range Queries) | 5 |
| 34 | Sweep Line Algorithm | 6 |
| 35 | Advanced Graph Algorithms (SCC, Bridges, Articulation Points) | 4 |
| 36 | Math Essentials (GCD, Sieve, Fast Exponentiation) | 7 |
| 37 | Game Theory / Minimax DP | 6 |
| **Total** | | **243** |

---

## 1. Arrays & Strings

**When to use:** Base data structure for most problems. Know traversal, in-place modification, rotation, and duplicate-handling tricks.

**Intuition:** Arrays give O(1) random access; most "tricks" (reversal, two-pass, in-place swap) exploit that indices can be read/written directly without extra memory.
**Flow:** Identify if operation is in-place (swap/reverse) or needs an extra pass (prefix/suffix arrays) -> pick pointers/indices accordingly.
**Complexity:** Traversal/reversal O(n) time, O(1) space (in-place).

```java
// Reverse array in-place
void reverse(int[] a, int lo, int hi) {
    while (lo < hi) {
        int tmp = a[lo]; a[lo] = a[hi]; a[hi] = tmp;
        lo++; hi--;
    }
}

// Rotate array right by k (using reversal trick) - O(n) time, O(1) space
void rotate(int[] a, int k) {
    int n = a.length;
    k %= n;
    reverse(a, 0, n - 1);
    reverse(a, 0, k - 1);
    reverse(a, k, n - 1);
}

// Move Zeroes - stable partition, in-place, O(n) time, O(1) space
void moveZeroes(int[] nums) {
    int insertPos = 0;
    for (int num : nums) if (num != 0) nums[insertPos++] = num;
    while (insertPos < nums.length) nums[insertPos++] = 0;
}

// Merge Sorted Array (nums1 has extra space at the end) - merge from the back to avoid overwrite
void mergeSortedArray(int[] nums1, int m, int[] nums2, int n) {
    int i = m - 1, j = n - 1, k = m + n - 1;
    while (j >= 0) {
        if (i >= 0 && nums1[i] > nums2[j]) nums1[k--] = nums1[i--];
        else nums1[k--] = nums2[j--];
    }
}

// Product of Array Except Self - prefix * suffix products, O(n) time, O(1) extra space
int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] res = new int[n];
    res[0] = 1;
    for (int i = 1; i < n; i++) res[i] = res[i - 1] * nums[i - 1]; // prefix products
    int suffix = 1;
    for (int i = n - 1; i >= 0; i--) {
        res[i] *= suffix; // multiply by suffix product
        suffix *= nums[i];
    }
    return res;
}

// Next Permutation - find next lexicographically greater arrangement in-place
void nextPermutation(int[] nums) {
    int n = nums.length, i = n - 2;
    while (i >= 0 && nums[i] >= nums[i + 1]) i--; // find rightmost ascent
    if (i >= 0) {
        int j = n - 1;
        while (nums[j] <= nums[i]) j--; // find rightmost element > nums[i]
        int tmp = nums[i]; nums[i] = nums[j]; nums[j] = tmp;
    }
    reverse(nums, i + 1, n - 1); // reverse suffix to get smallest arrangement
}

// Trapping Rain Water - two pointers, O(n) time, O(1) space
int trap(int[] height) {
    int left = 0, right = height.length - 1;
    int leftMax = 0, rightMax = 0, water = 0;
    while (left < right) {
        if (height[left] <= height[right]) {
            leftMax = Math.max(leftMax, height[left]);
            water += leftMax - height[left];
            left++;
        } else {
            rightMax = Math.max(rightMax, height[right]);
            water += rightMax - height[right];
            right--;
        }
    }
    return water;
}

// Contains Duplicate - true if any value appears at least twice, O(n) time, O(n) space
boolean containsDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int n : nums) if (!seen.add(n)) return true;
    return false;
}

// Find the Duplicate Number - Floyd's cycle detection, O(n) time, O(1) space
int findDuplicate(int[] nums) {
    int slow = nums[0], fast = nums[nums[0]];
    while (slow != fast) {
        slow = nums[slow];
        fast = nums[nums[fast]];
    }
    slow = 0;
    while (slow != fast) {
        slow = nums[slow];
        fast = nums[fast];
    }
    return slow;
}

// First Missing Positive - place each num at index num-1, O(n) time, O(1) space
int firstMissingPositive(int[] nums) {
    int n = nums.length;
    for (int i = 0; i < n; i++) {
        while (nums[i] > 0 && nums[i] <= n && nums[nums[i] - 1] != nums[i]) {
            int tmp = nums[i];
            nums[i] = nums[tmp - 1];
            nums[tmp - 1] = tmp;
        }
    }
    for (int i = 0; i < n; i++) if (nums[i] != i + 1) return i + 1;
    return n + 1;
}

// Majority Element - Boyer-Moore voting, O(n) time, O(1) space
int majorityElement(int[] nums) {
    int count = 0, candidate = 0;
    for (int n : nums) {
        if (count == 0) candidate = n;
        count += (n == candidate) ? 1 : -1;
    }
    return candidate;
}

// Plus One - add one to a digit array, O(n) time, O(1) amortized space
int[] plusOne(int[] digits) {
    int n = digits.length;
    for (int i = n - 1; i >= 0; i--) {
        if (digits[i] < 9) { digits[i]++; return digits; }
        digits[i] = 0;
    }
    int[] res = new int[n + 1];
    res[0] = 1;
    return res;
}

// Valid Sudoku - check rows, columns, and 3x3 boxes, O(1) time, O(1) space
boolean isValidSudoku(char[][] board) {
    Set<String> seen = new HashSet<>();
    for (int i = 0; i < 9; i++) {
        for (int j = 0; j < 9; j++) {
            char c = board[i][j];
            if (c == '.') continue;
            if (!seen.add(c + " row " + i) ||
                !seen.add(c + " col " + j) ||
                !seen.add(c + " box " + i/3 + "-" + j/3)) return false;
        }
    }
    return true;
}

// Reverse String - in-place, O(n) time, O(1) space
void reverseString(char[] s) {
    int lo = 0, hi = s.length - 1;
    while (lo < hi) {
        char tmp = s[lo]; s[lo] = s[hi]; s[hi] = tmp;
        lo++; hi--;
    }
}

// Valid Palindrome - two pointers skipping non-alphanumeric, O(n) time, O(1) space
boolean isPalindrome(String s) {
    int lo = 0, hi = s.length() - 1;
    while (lo < hi) {
        while (lo < hi && !Character.isLetterOrDigit(s.charAt(lo))) lo++;
        while (lo < hi && !Character.isLetterOrDigit(s.charAt(hi))) hi--;
        if (Character.toLowerCase(s.charAt(lo)) != Character.toLowerCase(s.charAt(hi))) return false;
        lo++; hi--;
    }
    return true;
}

// Roman to Integer - subtract if next symbol is larger, O(n) time
int romanToInt(String s) {
    Map<Character, Integer> map = new HashMap<>();
    map.put('I', 1); map.put('V', 5); map.put('X', 10);
    map.put('L', 50); map.put('C', 100); map.put('D', 500); map.put('M', 1000);
    int res = 0;
    for (int i = 0; i < s.length(); i++) {
        int val = map.get(s.charAt(i));
        if (i + 1 < s.length() && val < map.get(s.charAt(i + 1))) res -= val;
        else res += val;
    }
    return res;
}

// Integer to Roman - greedy subtract largest symbol, O(1) time (bounded by 3999)
String intToRoman(int num) {
    int[] values = {1000, 900, 500, 400, 100, 90, 50, 40, 10, 9, 5, 4, 1};
    String[] symbols = {"M","CM","D","CD","C","XC","L","XL","X","IX","V","IV","I"};
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < values.length; i++) {
        while (num >= values[i]) {
            sb.append(symbols[i]);
            num -= values[i];
        }
    }
    return sb.toString();
}
```

**Key problems:** Rotate Array, Move Zeroes, Merge Sorted Array, Product of Array Except Self, Next Permutation, Trapping Rain Water, Contains Duplicate, Find the Duplicate Number, First Missing Positive, Majority Element, Plus One, Valid Sudoku, Reverse String, Valid Palindrome, Roman to Integer, Integer to Roman.

---

## 2. Two Pointers

**When to use:** Sorted array/string, pair-sum problems, removing duplicates in-place, partitioning.

**Intuition:** With sorted data, moving `lo` right increases the sum and moving `hi` left decreases it — this monotonic behavior lets you eliminate one element per step instead of checking all pairs.
**Flow:** Start pointers at both ends (or same end for fast/slow variants) -> compare against target -> move the pointer that helps you get closer -> repeat until pointers cross.
**Complexity:** O(n) or O(n log n) after sort, O(1) extra space (excluding output).

```java
// Two Sum on sorted array
int[] twoSumSorted(int[] a, int target) {
    int lo = 0, hi = a.length - 1;
    while (lo < hi) {
        int sum = a[lo] + a[hi];
        if (sum == target) return new int[]{lo, hi};
        else if (sum < target) lo++;
        else hi--;
    }
    return new int[]{-1, -1};
}

// 3Sum
List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> res = new ArrayList<>();
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) continue; // skip dup
        int lo = i + 1, hi = nums.length - 1;
        while (lo < hi) {
            int sum = nums[i] + nums[lo] + nums[hi];
            if (sum == 0) {
                res.add(Arrays.asList(nums[i], nums[lo], nums[hi]));
                while (lo < hi && nums[lo] == nums[lo + 1]) lo++;
                while (lo < hi && nums[hi] == nums[hi - 1]) hi--;
                lo++; hi--;
            } else if (sum < 0) lo++;
            else hi--;
        }
    }
    return res;
}

// 3Sum Closest - track closest sum instead of exact match
int threeSumClosest(int[] nums, int target) {
    Arrays.sort(nums);
    int closest = nums[0] + nums[1] + nums[2];
    for (int i = 0; i < nums.length - 2; i++) {
        int lo = i + 1, hi = nums.length - 1;
        while (lo < hi) {
            int sum = nums[i] + nums[lo] + nums[hi];
            if (Math.abs(sum - target) < Math.abs(closest - target)) closest = sum;
            if (sum == target) return sum;
            else if (sum < target) lo++;
            else hi--;
        }
    }
    return closest;
}

// Container With Most Water - move the shorter wall inward (only way to possibly improve)
int maxArea(int[] height) {
    int lo = 0, hi = height.length - 1, best = 0;
    while (lo < hi) {
        int area = Math.min(height[lo], height[hi]) * (hi - lo);
        best = Math.max(best, area);
        if (height[lo] < height[hi]) lo++; else hi--;
    }
    return best;
}

// Remove Duplicates from Sorted Array - two pointers, in-place, O(n) time, O(1) space
int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;
    int k = 1;
    for (int i = 1; i < nums.length; i++) {
        if (nums[i] != nums[k - 1]) nums[k++] = nums[i];
    }
    return k;
}

// Sort Colors (Dutch National Flag) - 3-way partition, O(n) time, O(1) space
void sortColors(int[] nums) {
    int low = 0, mid = 0, high = nums.length - 1;
    while (mid <= high) {
        if (nums[mid] == 0) { int t = nums[low]; nums[low] = nums[mid]; nums[mid] = t; low++; mid++; }
        else if (nums[mid] == 1) mid++;
        else { int t = nums[mid]; nums[mid] = nums[high]; nums[high] = t; high--; }
    }
}
```

**Key problems:** Two Sum II, 3Sum, 3Sum Closest, Container With Most Water, Remove Duplicates from Sorted Array, Sort Colors (Dutch flag).

---

## 3. Sliding Window

**When to use:** Contiguous subarray/substring problems — max/min length, count with a constraint (sum, distinct chars, at most K).

**Intuition:** Instead of recomputing a window's property from scratch for every start index (O(n²)), incrementally add the new right element and remove the old left element — reusing prior work.
**Flow:** Expand `right` to grow the window; when the constraint is violated (or satisfied, for min-window problems), shrink from `left` until valid again; track the best answer as you go.
**Complexity:** O(n) time (each pointer moves forward only), O(k) space for the window's frequency map.

```java
// Fixed-size window: max sum of size-k subarray
int maxSumFixedWindow(int[] a, int k) {
    int sum = 0, best = Integer.MIN_VALUE;
    for (int i = 0; i < a.length; i++) {
        sum += a[i];
        if (i >= k - 1) {
            best = Math.max(best, sum);
            sum -= a[i - k + 1];
        }
    }
    return best;
}

// Variable-size window: longest substring without repeating chars
int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastIdx = new HashMap<>();
    int left = 0, best = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastIdx.containsKey(c) && lastIdx.get(c) >= left) {
            left = lastIdx.get(c) + 1;
        }
        lastIdx.put(c, right);
        best = Math.max(best, right - left + 1);
    }
    return best;
}

// Minimum window containing all chars of target (template)
String minWindow(String s, String t) {
    Map<Character, Integer> need = new HashMap<>();
    for (char c : t.toCharArray()) need.merge(c, 1, Integer::sum);
    int required = need.size(), formed = 0;
    Map<Character, Integer> window = new HashMap<>();
    int left = 0, bestLen = Integer.MAX_VALUE, bestStart = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        window.merge(c, 1, Integer::sum);
        if (need.containsKey(c) && window.get(c).intValue() == need.get(c).intValue()) formed++;
        while (formed == required) {
            if (right - left + 1 < bestLen) { bestLen = right - left + 1; bestStart = left; }
            char lc = s.charAt(left);
            window.put(lc, window.get(lc) - 1);
            if (need.containsKey(lc) && window.get(lc) < need.get(lc)) formed--;
            left++;
        }
    }
    return bestLen == Integer.MAX_VALUE ? "" : s.substring(bestStart, bestStart + bestLen);
}

// Longest Repeating Character Replacement - window valid if (len - maxFreq) <= k
int characterReplacement(String s, int k) {
    int[] count = new int[26];
    int left = 0, maxFreq = 0, best = 0;
    for (int right = 0; right < s.length(); right++) {
        maxFreq = Math.max(maxFreq, ++count[s.charAt(right) - 'A']);
        while (right - left + 1 - maxFreq > k) count[s.charAt(left++) - 'A']--;
        best = Math.max(best, right - left + 1);
    }
    return best;
}

// Max Consecutive Ones III - flip at most k zeros, find longest window of 1s
int longestOnes(int[] nums, int k) {
    int left = 0, zeros = 0, best = 0;
    for (int right = 0; right < nums.length; right++) {
        if (nums[right] == 0) zeros++;
        while (zeros > k) { if (nums[left++] == 0) zeros--; }
        best = Math.max(best, right - left + 1);
    }
    return best;
}

// Fruit Into Baskets - longest subarray with at most 2 distinct values (sliding window + freq map)
int totalFruit(int[] fruits) {
    Map<Integer, Integer> count = new HashMap<>();
    int left = 0, best = 0;
    for (int right = 0; right < fruits.length; right++) {
        count.merge(fruits[right], 1, Integer::sum);
        while (count.size() > 2) {
            int leftFruit = fruits[left++];
            count.put(leftFruit, count.get(leftFruit) - 1);
            if (count.get(leftFruit) == 0) count.remove(leftFruit);
        }
        best = Math.max(best, right - left + 1);
    }
    return best;
}
```

**Key problems:** Longest Substring Without Repeating Characters, Minimum Window Substring, Longest Repeating Character Replacement, Max Consecutive Ones III, Subarrays with K Different Integers, Fruit Into Baskets.

---

## 4. Prefix Sum / Difference Array

**When to use:** Range sum queries, subarray sum == K, range update queries.

**Intuition:** `sum(i..j) = prefix[j] - prefix[i-1]`. Precomputing prefixes converts O(n) range-sum queries into O(1) lookups. The difference array is the reverse trick: apply O(1) range *updates* and recover the full array with one prefix-sum pass.
**Flow:** Build prefix array once -> answer each range query via subtraction; for `subarraySum == K`, use a hashmap of prefix sums seen so far so you can look up `sum - k` in O(1).
**Complexity:** O(n) to build, O(1) per query; difference-array updates are O(1) each, O(n) to materialize final array.

```java
// Subarray Sum Equals K (count)
int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1);
    int sum = 0, count = 0;
    for (int num : nums) {
        sum += num;
        count += prefixCount.getOrDefault(sum - k, 0);
        prefixCount.merge(sum, 1, Integer::sum);
    }
    return count;
}

// Difference array: range increment updates, O(1) per update
int[] rangeAdd(int n, int[][] updates) {
    int[] diff = new int[n + 1];
    for (int[] u : updates) {
        diff[u[0]] += u[2];
        diff[u[1] + 1] -= u[2];
    }
    int[] result = new int[n];
    int running = 0;
    for (int i = 0; i < n; i++) { running += diff[i]; result[i] = running; }
    return result;
}

// Range Sum Query 2D (Immutable) - 2D prefix sum
class NumMatrix {
    private final int[][] prefix;
    NumMatrix(int[][] matrix) {
        int m = matrix.length, n = matrix[0].length;
        prefix = new int[m + 1][n + 1];
        for (int i = 0; i < m; i++)
            for (int j = 0; j < n; j++)
                prefix[i + 1][j + 1] = prefix[i][j + 1] + prefix[i + 1][j] - prefix[i][j] + matrix[i][j];
    }
    int sumRegion(int row1, int col1, int row2, int col2) {
        return prefix[row2 + 1][col2 + 1] - prefix[row1][col2 + 1] - prefix[row2 + 1][col1] + prefix[row1][col1];
    }
}

// Corporate Flight Bookings - difference array for range increments
int[] corpFlightBookings(int[][] bookings, int n) {
    int[] diff = new int[n + 1];
    for (int[] b : bookings) {
        diff[b[0] - 1] += b[2];
        diff[b[1]] -= b[2];
    }
    int[] res = new int[n];
    int running = 0;
    for (int i = 0; i < n; i++) { running += diff[i]; res[i] = running; }
    return res;
}

// Contiguous Array - longest subarray with equal 0s and 1s (map value -> first index of that prefix balance)
int findMaxLength(int[] nums) {
    Map<Integer, Integer> firstIndex = new HashMap<>();
    firstIndex.put(0, -1);
    int balance = 0, best = 0;
    for (int i = 0; i < nums.length; i++) {
        balance += nums[i] == 1 ? 1 : -1;
        if (firstIndex.containsKey(balance)) best = Math.max(best, i - firstIndex.get(balance));
        else firstIndex.put(balance, i);
    }
    return best;
}
```

**Key problems:** Subarray Sum Equals K, Range Sum Query 2D, Corporate Flight Bookings, Contiguous Array.

---

## 5. Binary Search

**When to use:** Sorted array lookup, "find boundary" problems, and **binary search on answer** (monotonic predicate over an answer space).

**Intuition:** Any monotonic predicate (true/false that flips exactly once as x increases) can be binary-searched, even if there's no literal sorted array — e.g. "can I ship all packages in D days with capacity X?" is monotonic in X.
**Flow:** Define `lo`/`hi` as the answer bounds -> at each `mid`, evaluate the predicate -> shrink to the half that still contains the boundary -> converge when `lo == hi`.
**Complexity:** O(log n) iterations; each iteration's predicate check costs its own time (e.g. O(n) for the shipping-capacity check, giving O(n log(maxSum))) overall).

```java
// Standard binary search
int binarySearch(int[] a, int target) {
    int lo = 0, hi = a.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == target) return mid;
        else if (a[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

// Lower bound (first index with a[i] >= target)
int lowerBound(int[] a, int target) {
    int lo = 0, hi = a.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] < target) lo = mid + 1; else hi = mid;
    }
    return lo;
}

// Binary search on answer: minimum capacity to ship packages within D days
int shipWithinDays(int[] weights, int days) {
    int lo = Arrays.stream(weights).max().getAsInt();
    int hi = Arrays.stream(weights).sum();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (canShip(weights, days, mid)) hi = mid; else lo = mid + 1;
    }
    return lo;
}
private boolean canShip(int[] w, int days, int cap) {
    int need = 1, cur = 0;
    for (int x : w) {
        if (cur + x > cap) { need++; cur = 0; }
        cur += x;
    }
    return need <= days;
}

// Binary search on answer: Koko Eating Bananas - MINIMIZE the max eating speed s.t. all piles finish in h hours
int minEatingSpeed(int[] piles, int h) {
    int lo = 1, hi = Arrays.stream(piles).max().getAsInt();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (canFinish(piles, h, mid)) hi = mid; // mid is fast enough -> try slower (smaller)
        else lo = mid + 1; // too slow -> need faster (larger)
    }
    return lo;
}
private boolean canFinish(int[] piles, int h, int speed) {
    long hours = 0;
    for (int p : piles) hours += (p + speed - 1) / speed; // ceil division
    return hours <= h;
}

// Binary search on answer: Aggressive Cows - MAXIMIZE the minimum distance between any two placed cows
int aggressiveCows(int[] stalls, int cows) {
    Arrays.sort(stalls);
    int lo = 1, hi = stalls[stalls.length - 1] - stalls[0];
    int best = 0;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (canPlaceCows(stalls, cows, mid)) { best = mid; lo = mid + 1; } // feasible -> try a larger min-distance
        else hi = mid - 1; // infeasible -> need a smaller min-distance
    }
    return best;
}
private boolean canPlaceCows(int[] stalls, int cows, int minDist) {
    int placed = 1, lastPos = stalls[0]; // place first cow at the first stall
    for (int i = 1; i < stalls.length; i++) {
        if (stalls[i] - lastPos >= minDist) { placed++; lastPos = stalls[i]; }
    }
    return placed >= cows;
}

// Search in Rotated Sorted Array - determine which half is sorted, then decide direction
int searchRotated(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;
        if (nums[lo] <= nums[mid]) { // left half is sorted
            if (nums[lo] <= target && target < nums[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else { // right half is sorted
            if (nums[mid] < target && target <= nums[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return -1;
}

// Find Peak Element - binary search on the slope direction
int findPeakElement(int[] nums) {
    int lo = 0, hi = nums.length - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] > nums[mid + 1]) hi = mid; // peak is at mid or to the left
        else lo = mid + 1; // peak is to the right
    }
    return lo;
}

// Median of Two Sorted Arrays - binary search on partition point, O(log(min(m,n)))
double findMedianSortedArrays(int[] a, int[] b) {
    if (a.length > b.length) return findMedianSortedArrays(b, a);
    int m = a.length, n = b.length;
    int lo = 0, hi = m;
    while (lo <= hi) {
        int i = lo + (hi - lo) / 2; // partition in a
        int j = (m + n + 1) / 2 - i; // partition in b
        int aLeft = i == 0 ? Integer.MIN_VALUE : a[i - 1];
        int aRight = i == m ? Integer.MAX_VALUE : a[i];
        int bLeft = j == 0 ? Integer.MIN_VALUE : b[j - 1];
        int bRight = j == n ? Integer.MAX_VALUE : b[j];
        if (aLeft <= bRight && bLeft <= aRight) {
            if ((m + n) % 2 == 0) return (Math.max(aLeft, bLeft) + Math.min(aRight, bRight)) / 2.0;
            else return Math.max(aLeft, bLeft);
        } else if (aLeft > bRight) hi = i - 1;
        else lo = i + 1;
    }
    return -1.0;
}
```

**Key problems:** Search in Rotated Sorted Array, Find Peak Element, Median of Two Sorted Arrays, Capacity to Ship Packages, Split Array Largest Sum, Koko Eating Bananas, Aggressive Cows.

**Binary search on answer — two flavors:**
- **Minimize the maximum** (Koko Eating Bananas, Capacity to Ship Packages, Split Array Largest Sum): if `mid` is feasible, try smaller (`hi = mid`); converge to the smallest feasible value.
- **Maximize the minimum** (Aggressive Cows, Magnetic Force Between Two Balls): if `mid` is feasible, try larger (`lo = mid + 1`, tracking `best`); converge to the largest feasible value.

---

## 6. Kadane's Algorithm (Max Subarray)

**When to use:** Max/min sum contiguous subarray, and variants (max product subarray, circular subarray).

**Intuition:** If the running sum ending at `i-1` is negative, it can only drag down any subarray extended through it — so it's never beneficial to extend from there; better to restart the subarray at `i`.
**Flow:** At each index, decide: extend the previous subarray (`cur + nums[i]`) or start fresh (`nums[i]`) -> take the max -> track the best seen so far.
**Complexity:** O(n) time, O(1) space — single pass, no extra memory.

```java
int maxSubArray(int[] nums) {
    int best = nums[0], cur = nums[0];
    for (int i = 1; i < nums.length; i++) {
        cur = Math.max(nums[i], cur + nums[i]);
        best = Math.max(best, cur);
    }
    return best;
}

// Max Product Subarray (track running max & min because of negatives)
int maxProduct(int[] nums) {
    int best = nums[0], curMax = nums[0], curMin = nums[0];
    for (int i = 1; i < nums.length; i++) {
        int x = nums[i];
        int candMax = Math.max(x, Math.max(curMax * x, curMin * x));
        int candMin = Math.min(x, Math.min(curMax * x, curMin * x));
        curMax = candMax; curMin = candMin;
        best = Math.max(best, curMax);
    }
    return best;
}

// Maximum Sum Circular Subarray - answer is either standard Kadane's, or (total - min subarray)
int maxSubarraySumCircular(int[] nums) {
    int total = 0, curMax = 0, best = nums[0], curMin = 0, worst = nums[0];
    for (int x : nums) {
        curMax = Math.max(x, curMax + x); best = Math.max(best, curMax);
        curMin = Math.min(x, curMin + x); worst = Math.min(worst, curMin);
        total += x;
    }
    if (worst == total) return best; // all elements negative, can't exclude a "min" subarray
    return Math.max(best, total - worst);
}

// Best Time to Buy and Sell Stock - track min price seen so far, max profit if sold today
int maxProfit(int[] prices) {
    int minPrice = Integer.MAX_VALUE, best = 0;
    for (int p : prices) {
        minPrice = Math.min(minPrice, p);
        best = Math.max(best, p - minPrice);
    }
    return best;
}
```

**Key problems:** Maximum Subarray, Maximum Product Subarray, Maximum Sum Circular Subarray, Best Time to Buy and Sell Stock.

---

## 7. Sorting-based Patterns

**When to use:** Custom comparators, cyclic sort (range 1..n), merge intervals.

**Intuition:** When values are guaranteed to be in range `[0, n-1]` (or `[1, n]`), each value has a "correct" home index — swap elements into their correct slots instead of using a general-purpose sort, avoiding extra space.
**Flow:** At index `i`, if `nums[i]` isn't at its correct position, swap it there; otherwise move on. After one pass, any index whose value doesn't match is the anomaly (missing/duplicate).
**Complexity:** O(n) time (each swap places at least one element correctly), O(1) space.

```java
// Cyclic Sort: Find missing number in 1..n
int missingNumber(int[] nums) {
    int i = 0, n = nums.length;
    while (i < n) {
        int correct = nums[i] < n ? nums[i] : n; // place value == index
        if (nums[i] < n && nums[i] != i) {
            int tmp = nums[nums[i]]; nums[nums[i]] = nums[i]; nums[i] = tmp;
        } else i++;
    }
    for (i = 0; i < n; i++) if (nums[i] != i) return i;
    return n;
}

// Find All Duplicates in an Array - cyclic sort style, use sign as a visited marker
List<Integer> findDuplicates(int[] nums) {
    List<Integer> res = new ArrayList<>();
    for (int i = 0; i < nums.length; i++) {
        int idx = Math.abs(nums[i]) - 1;
        if (nums[idx] < 0) res.add(idx + 1); // already visited -> duplicate
        else nums[idx] = -nums[idx];
    }
    return res;
}

// Kth Largest Element - Quickselect, O(n) average time, O(1) space
int findKthLargest(int[] nums, int k) {
    int target = nums.length - k; // index of kth largest in sorted order
    int lo = 0, hi = nums.length - 1;
    while (true) {
        int pivotIdx = partition(nums, lo, hi);
        if (pivotIdx == target) return nums[pivotIdx];
        else if (pivotIdx < target) lo = pivotIdx + 1;
        else hi = pivotIdx - 1;
    }
}
private int partition(int[] nums, int lo, int hi) {
    int pivot = nums[hi], i = lo;
    for (int j = lo; j < hi; j++) {
        if (nums[j] < pivot) { int t = nums[i]; nums[i] = nums[j]; nums[j] = t; i++; }
    }
    int t = nums[i]; nums[i] = nums[hi]; nums[hi] = t;
    return i;
}

// Meeting Rooms (can attend all meetings?) - sort by start, check for overlap
boolean canAttendMeetings(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] < intervals[i - 1][1]) return false;
    }
    return true;
}
```

**Key problems:** Missing Number, Find All Duplicates, Kth Largest Element (quickselect), Sort Colors, Meeting Rooms.

---

## 8. Linked List

**When to use:** In-place reversal, merging, dummy-node tricks.

**Intuition:** Unlike arrays, you can't jump to an index — but you CAN rewire `next` pointers in O(1) per node. A dummy head node avoids messy edge-case checks when the real head might change.
**Flow:** Walk the list with one or more pointers (`prev`/`cur`/`next` for reversal, `fast`/`slow` for cycle problems) -> rewire pointers as you go -> return `dummy.next` or the new head.
**Complexity:** O(n) time, O(1) space for iterative in-place operations (O(n) if recursive, due to call stack).

```java
class ListNode { int val; ListNode next; ListNode(int v) { val = v; } }

// Reverse a linked list
ListNode reverseList(ListNode head) {
    ListNode prev = null;
    while (head != null) {
        ListNode next = head.next;
        head.next = prev;
        prev = head;
        head = next;
    }
    return prev;
}

// Reverse in a range [left, right] (1-indexed)
ListNode reverseBetween(ListNode head, int left, int right) {
    ListNode dummy = new ListNode(0); dummy.next = head;
    ListNode prev = dummy;
    for (int i = 0; i < left - 1; i++) prev = prev.next;
    ListNode cur = prev.next;
    for (int i = 0; i < right - left; i++) {
        ListNode next = cur.next;
        cur.next = next.next;
        next.next = prev.next;
        prev.next = next;
    }
    return dummy.next;
}

// Merge two sorted lists
ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), tail = dummy;
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) { tail.next = l1; l1 = l1.next; }
        else { tail.next = l2; l2 = l2.next; }
        tail = tail.next;
    }
    tail.next = (l1 != null) ? l1 : l2;
    return dummy.next;
}

// Remove Nth node from end (one pass)
ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0); dummy.next = head;
    ListNode fast = dummy, slow = dummy;
    for (int i = 0; i < n; i++) fast = fast.next;
    while (fast.next != null) { fast = fast.next; slow = slow.next; }
    slow.next = slow.next.next;
    return dummy.next;
}

// Add Two Numbers (digits stored in reverse order as linked lists)
ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), cur = dummy;
    int carry = 0;
    while (l1 != null || l2 != null || carry != 0) {
        int sum = carry + (l1 != null ? l1.val : 0) + (l2 != null ? l2.val : 0);
        carry = sum / 10;
        cur.next = new ListNode(sum % 10);
        cur = cur.next;
        if (l1 != null) l1 = l1.next;
        if (l2 != null) l2 = l2.next;
    }
    return dummy.next;
}

// Reorder List: L0->L1->...->Ln  =>  L0->Ln->L1->Ln-1... (find mid, reverse 2nd half, merge)
void reorderList(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast.next != null && fast.next.next != null) { slow = slow.next; fast = fast.next.next; }
    ListNode secondHead = slow.next;
    slow.next = null;
    ListNode secondReversed = reverseList(secondHead);
    ListNode p1 = head, p2 = secondReversed;
    while (p2 != null) {
        ListNode n1 = p1.next, n2 = p2.next;
        p1.next = p2;
        p2.next = n1;
        p1 = n1; p2 = n2;
    }
}

// Copy List with Random Pointer - HashMap old->new node, two passes
class RandomListNode { int val; RandomListNode next, random; RandomListNode(int v) { val = v; } }
RandomListNode copyRandomList(RandomListNode head) {
    if (head == null) return null;
    Map<RandomListNode, RandomListNode> map = new HashMap<>();
    RandomListNode cur = head;
    while (cur != null) { map.put(cur, new RandomListNode(cur.val)); cur = cur.next; }
    cur = head;
    while (cur != null) {
        map.get(cur).next = map.get(cur.next);
        map.get(cur).random = map.get(cur.random);
        cur = cur.next;
    }
    return map.get(head);
}

// Palindrome Linked List - find middle, reverse 2nd half, compare
boolean isPalindrome(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) { slow = slow.next; fast = fast.next.next; }
    ListNode secondHalf = reverseList(slow);
    ListNode p1 = head, p2 = secondHalf;
    while (p2 != null) {
        if (p1.val != p2.val) return false;
        p1 = p1.next; p2 = p2.next;
    }
    return true;
}
```

**Key problems:** Reverse Linked List (I/II), Merge K Sorted Lists, Add Two Numbers, LRU Cache, Reorder List, Copy List with Random Pointer, Palindrome Linked List.

---

## 9. Stack

**When to use:** Matching/parsing (parentheses), reversing order, DFS iterative, expression evaluation, backtracking undo.

**Intuition:** LIFO order naturally mirrors nested structures — the most recently opened bracket/operator must be the next one closed/resolved.
**Flow:** Push on "open"/pending items; on a "close"/resolving item, pop and check it matches; empty stack at the end means fully matched.
**Complexity:** O(n) time, O(n) space (worst case all pushed, e.g. all-open-brackets string).

```java
boolean isValidParentheses(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');
    for (char c : s.toCharArray()) {
        if (!pairs.containsKey(c)) stack.push(c);
        else if (stack.isEmpty() || stack.pop() != pairs.get(c)) return false;
    }
    return stack.isEmpty();
}

// Basic Calculator II (no parens): + - * /
int calculate(String s) {
    Deque<Integer> stack = new ArrayDeque<>();
    int num = 0; char sign = '+';
    for (int i = 0; i <= s.length(); i++) {
        char c = i < s.length() ? s.charAt(i) : '+';
        if (Character.isDigit(c)) num = num * 10 + (c - '0');
        else if (c != ' ') {
            if (sign == '+') stack.push(num);
            else if (sign == '-') stack.push(-num);
            else if (sign == '*') stack.push(stack.pop() * num);
            else if (sign == '/') stack.push(stack.pop() / num);
            sign = c; num = 0;
        }
    }
    int total = 0;
    for (int x : stack) total += x;
    return total;
}

// Min Stack - auxiliary stack tracks running minimum, O(1) all ops
class MinStack {
    private final Deque<int[]> stack = new ArrayDeque<>(); // {val, minSoFar}
    void push(int val) {
        int min = stack.isEmpty() ? val : Math.min(val, stack.peek()[1]);
        stack.push(new int[]{val, min});
    }
    void pop() { stack.pop(); }
    int top() { return stack.peek()[0]; }
    int getMin() { return stack.peek()[1]; }
}

// Evaluate Reverse Polish Notation
int evalRPN(String[] tokens) {
    Deque<Integer> stack = new ArrayDeque<>();
    Set<String> ops = Set.of("+", "-", "*", "/");
    for (String t : tokens) {
        if (!ops.contains(t)) stack.push(Integer.parseInt(t));
        else {
            int b = stack.pop(), a = stack.pop();
            switch (t) {
                case "+": stack.push(a + b); break;
                case "-": stack.push(a - b); break;
                case "*": stack.push(a * b); break;
                case "/": stack.push(a / b); break;
            }
        }
    }
    return stack.pop();
}

// Decode String, e.g. "3[a2[c]]" -> "accaccacc"
String decodeString(String s) {
    Deque<Integer> countStack = new ArrayDeque<>();
    Deque<StringBuilder> strStack = new ArrayDeque<>();
    StringBuilder cur = new StringBuilder();
    int num = 0;
    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) num = num * 10 + (c - '0');
        else if (c == '[') { countStack.push(num); strStack.push(cur); num = 0; cur = new StringBuilder(); }
        else if (c == ']') {
            StringBuilder decoded = strStack.pop();
            int repeat = countStack.pop();
            for (int i = 0; i < repeat; i++) decoded.append(cur);
            cur = decoded;
        } else cur.append(c);
    }
    return cur.toString();
}

// Simplify Path (Unix-style canonical path)
String simplifyPath(String path) {
    Deque<String> stack = new ArrayDeque<>();
    for (String part : path.split("/")) {
        if (part.isEmpty() || part.equals(".")) continue;
        if (part.equals("..")) { if (!stack.isEmpty()) stack.pop(); }
        else stack.push(part);
    }
    StringBuilder sb = new StringBuilder();
    for (String part : stack) sb.insert(0, "/" + part);
    return sb.length() == 0 ? "/" : sb.toString();
}

// Asteroid Collision - stack simulates surviving asteroids moving right, colliding with new left-movers
int[] asteroidCollision(int[] asteroids) {
    Deque<Integer> stack = new ArrayDeque<>();
    for (int a : asteroids) {
        boolean alive = true;
        while (alive && a < 0 && !stack.isEmpty() && stack.peek() > 0) {
            if (stack.peek() < -a) stack.pop(); // top explodes
            else if (stack.peek() == -a) { stack.pop(); alive = false; } // both explode
            else alive = false; // current explodes
        }
        if (alive) stack.push(a);
    }
    int[] res = new int[stack.size()];
    for (int i = res.length - 1; i >= 0; i--) res[i] = stack.pop();
    return res;
}
```

**Key problems:** Valid Parentheses, Min Stack, Evaluate RPN, Basic Calculator I/II, Decode String, Simplify Path, Asteroid Collision.

---

## 10. Monotonic Stack

**When to use:** Next/previous greater or smaller element, histogram-area style problems.

**Intuition:** Maintain the stack so it's always monotonic (increasing or decreasing). When a new element breaks the monotonic property, it means the new element is the "answer" for everything it pops — each element is pushed/popped once, giving amortized O(1) per element instead of O(n) nested scans.
**Flow:** For each element, pop from the stack while it violates monotonicity (recording the popped index's answer using the current element) -> then push the current index.
**Complexity:** O(n) time (amortized, each element pushed/popped once), O(n) space.

```java
// Next Greater Element (indices), monotonic decreasing stack
int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] res = new int[n];
    Arrays.fill(res, -1);
    Deque<Integer> stack = new ArrayDeque<>(); // holds indices
    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            res[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return res;
}

// Largest Rectangle in Histogram
int largestRectangleArea(int[] heights) {
    Deque<Integer> stack = new ArrayDeque<>();
    int best = 0, n = heights.length;
    for (int i = 0; i <= n; i++) {
        int h = (i == n) ? 0 : heights[i];
        while (!stack.isEmpty() && heights[stack.peek()] >= h) {
            int height = heights[stack.pop()];
            int width = stack.isEmpty() ? i : i - stack.peek() - 1;
            best = Math.max(best, height * width);
        }
        stack.push(i);
    }
    return best;
}

// Daily Temperatures
int[] dailyTemperatures(int[] temps) {
    int n = temps.length;
    int[] res = new int[n];
    Deque<Integer> stack = new ArrayDeque<>();
    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && temps[stack.peek()] < temps[i]) {
            int idx = stack.pop();
            res[idx] = i - idx;
        }
        stack.push(i);
    }
    return res;
}

// Remove K Digits - greedily remove digits to form the smallest number, using a monotonic increasing stack
String removeKdigits(String num, int k) {
    Deque<Character> stack = new ArrayDeque<>();
    for (char c : num.toCharArray()) {
        while (k > 0 && !stack.isEmpty() && stack.peek() > c) { stack.pop(); k--; }
        stack.push(c);
    }
    while (k-- > 0 && !stack.isEmpty()) stack.pop();
    StringBuilder sb = new StringBuilder();
    while (!stack.isEmpty()) sb.append(stack.pollLast());
    while (sb.length() > 1 && sb.charAt(0) == '0') sb.deleteCharAt(0); // strip leading zeros
    return sb.length() == 0 ? "0" : sb.toString();
}

// Stock Span Problem - number of consecutive days (ending today) with price <= today's price
class StockSpanner {
    private final Deque<int[]> stack = new ArrayDeque<>(); // {price, span}
    int next(int price) {
        int span = 1;
        while (!stack.isEmpty() && stack.peek()[0] <= price) span += stack.pop()[1];
        stack.push(new int[]{price, span});
        return span;
    }
}
```

**Key problems:** Next Greater Element I/II, Daily Temperatures, Largest Rectangle in Histogram, Trapping Rain Water, Remove K Digits, Stock Span Problem.

---

## 11. Queue & Deque

**When to use:** BFS level processing, sliding window problems, task scheduling.

**Intuition:** FIFO order processes nodes in the order discovered — guarantees that when a node is first visited via BFS on an unweighted graph, it's via the shortest path (fewest edges).
**Flow:** Enqueue start node(s) -> repeatedly dequeue, process, and enqueue unvisited neighbors -> track distance/level via queue size snapshots or a stored depth value.
**Complexity:** O(V + E) time, O(V) space for the visited set and queue.

```java
// BFS template on grid
int bfsShortestPath(int[][] grid, int[] start, int[] end) {
    int rows = grid.length, cols = grid[0].length;
    boolean[][] visited = new boolean[rows][cols];
    Queue<int[]> queue = new ArrayDeque<>();
    queue.offer(new int[]{start[0], start[1], 0});
    visited[start[0]][start[1]] = true;
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    while (!queue.isEmpty()) {
        int[] cur = queue.poll();
        if (cur[0] == end[0] && cur[1] == end[1]) return cur[2];
        for (int[] d : dirs) {
            int nr = cur[0] + d[0], nc = cur[1] + d[1];
            if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && !visited[nr][nc] && grid[nr][nc] == 0) {
                visited[nr][nc] = true;
                queue.offer(new int[]{nr, nc, cur[2] + 1});
            }
        }
    }
    return -1;
}

// Rotting Oranges - multi-source BFS, all initially-rotten oranges are level-0 sources
int orangesRotting(int[][] grid) {
    int rows = grid.length, cols = grid[0].length, fresh = 0;
    Queue<int[]> queue = new ArrayDeque<>();
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++) {
            if (grid[i][j] == 2) queue.offer(new int[]{i, j});
            else if (grid[i][j] == 1) fresh++;
        }
    int minutes = 0;
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    while (!queue.isEmpty() && fresh > 0) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] cur = queue.poll();
            for (int[] d : dirs) {
                int nr = cur[0] + d[0], nc = cur[1] + d[1];
                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] == 1) {
                    grid[nr][nc] = 2; fresh--;
                    queue.offer(new int[]{nr, nc});
                }
            }
        }
        minutes++;
    }
    return fresh == 0 ? minutes : -1;
}

// Walls and Gates - multi-source BFS from all gates, fill each room with distance to nearest gate
void wallsAndGates(int[][] rooms) {
    int rows = rooms.length, cols = rooms[0].length;
    Queue<int[]> queue = new ArrayDeque<>();
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            if (rooms[i][j] == 0) queue.offer(new int[]{i, j});
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    while (!queue.isEmpty()) {
        int[] cur = queue.poll();
        for (int[] d : dirs) {
            int nr = cur[0] + d[0], nc = cur[1] + d[1];
            if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && rooms[nr][nc] == Integer.MAX_VALUE) {
                rooms[nr][nc] = rooms[cur[0]][cur[1]] + 1;
                queue.offer(new int[]{nr, nc});
            }
        }
    }
}

// 01 Matrix - multi-source BFS from all 0-cells, fill distance to nearest 0 for each cell
int[][] updateMatrix(int[][] mat) {
    int rows = mat.length, cols = mat[0].length;
    int[][] dist = new int[rows][cols];
    boolean[][] visited = new boolean[rows][cols];
    Queue<int[]> queue = new ArrayDeque<>();
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            if (mat[i][j] == 0) { queue.offer(new int[]{i, j}); visited[i][j] = true; }
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    while (!queue.isEmpty()) {
        int[] cur = queue.poll();
        for (int[] d : dirs) {
            int nr = cur[0] + d[0], nc = cur[1] + d[1];
            if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && !visited[nr][nc]) {
                dist[nr][nc] = dist[cur[0]][cur[1]] + 1;
                visited[nr][nc] = true;
                queue.offer(new int[]{nr, nc});
            }
        }
    }
    return dist;
}

// Open the Lock - BFS over lock-combination states, avoiding deadends
int openLock(String[] deadends, String target) {
    Set<String> dead = new HashSet<>(Arrays.asList(deadends));
    Set<String> visited = new HashSet<>();
    Queue<String> queue = new ArrayDeque<>();
    String start = "0000";
    if (dead.contains(start)) return -1;
    queue.offer(start); visited.add(start);
    int steps = 0;
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            String cur = queue.poll();
            if (cur.equals(target)) return steps;
            for (int pos = 0; pos < 4; pos++) {
                char c = cur.charAt(pos);
                for (int d : new int[]{1, -1}) {
                    char nc = (char) ('0' + ((c - '0' + d + 10) % 10));
                    String next = cur.substring(0, pos) + nc + cur.substring(pos + 1);
                    if (!dead.contains(next) && !visited.contains(next)) {
                        visited.add(next);
                        queue.offer(next);
                    }
                }
            }
        }
        steps++;
    }
    return -1;
}
```

**Key problems:** Rotting Oranges, Walls and Gates, 01 Matrix, Open the Lock, Sliding Window Maximum (see below).

---

## 12. Monotonic Deque (Sliding Window Max/Min)

**When to use:** Max/min in every window of size K in O(n).

**Intuition:** Keep the deque monotonically decreasing (for max) so the front is always the current window's max. Any element smaller than a newer element can never become the answer while the newer one is still in the window — safe to discard it.
**Flow:** Push new index, popping from the back any smaller elements first; pop from the front any indices that fell out of the window; front of deque = window's max.
**Complexity:** O(n) time (amortized), O(k) space.

```java
int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> deque = new ArrayDeque<>(); // stores indices, decreasing values
    int[] res = new int[nums.length - k + 1];
    for (int i = 0; i < nums.length; i++) {
        while (!deque.isEmpty() && deque.peekFirst() <= i - k) deque.pollFirst();
        while (!deque.isEmpty() && nums[deque.peekLast()] <= nums[i]) deque.pollLast();
        deque.offerLast(i);
        if (i >= k - 1) res[i - k + 1] = nums[deque.peekFirst()];
    }
    return res;
}

// Shortest Subarray with Sum at Least K - monotonic deque over prefix sums
int shortestSubarray(int[] nums, int k) {
    int n = nums.length;
    long[] prefix = new long[n + 1];
    for (int i = 0; i < n; i++) prefix[i + 1] = prefix[i] + nums[i];
    Deque<Integer> deque = new ArrayDeque<>(); // indices with increasing prefix[]
    int best = Integer.MAX_VALUE;
    for (int i = 0; i <= n; i++) {
        while (!deque.isEmpty() && prefix[i] - prefix[deque.peekFirst()] >= k) {
            best = Math.min(best, i - deque.pollFirst());
        }
        while (!deque.isEmpty() && prefix[deque.peekLast()] >= prefix[i]) deque.pollLast();
        deque.offerLast(i);
    }
    return best == Integer.MAX_VALUE ? -1 : best;
}

// Constrained Subsequence Sum - dp[i] = nums[i] + max(0, dp[j] for j in [i-k, i-1]), via monotonic deque
int constrainedSubsetSum(int[] nums, int k) {
    int n = nums.length;
    int[] dp = new int[n];
    Deque<Integer> deque = new ArrayDeque<>(); // indices, decreasing dp values
    int best = Integer.MIN_VALUE;
    for (int i = 0; i < n; i++) {
        while (!deque.isEmpty() && deque.peekFirst() < i - k) deque.pollFirst();
        dp[i] = nums[i] + (deque.isEmpty() ? 0 : Math.max(0, dp[deque.peekFirst()]));
        while (!deque.isEmpty() && dp[deque.peekLast()] <= dp[i]) deque.pollLast();
        deque.offerLast(i);
        best = Math.max(best, dp[i]);
    }
    return best;
}
```

**Key problems:** Sliding Window Maximum, Shortest Subarray with Sum at Least K, Constrained Subsequence Sum.

---

## 13. Hashing (HashMap/HashSet)

**When to use:** O(1) lookup, frequency counting, grouping, detecting duplicates.

**Intuition:** Trade space for time — a hash-based structure turns "does this exist?" or "how many times has this appeared?" into O(1) average lookups instead of O(n) linear scans.
**Flow:** Pick a key that captures the property you're grouping/comparing by (sorted chars for anagrams, value itself for existence) -> populate the map/set in one pass -> query/aggregate in a second pass.
**Complexity:** O(n) time average, O(n) space.

```java
// Group Anagrams
List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();
    for (String s : strs) {
        char[] key = s.toCharArray();
        Arrays.sort(key);
        map.computeIfAbsent(new String(key), k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(map.values());
}

// Longest Consecutive Sequence
int longestConsecutive(int[] nums) {
    Set<Integer> set = new HashSet<>();
    for (int n : nums) set.add(n);
    int best = 0;
    for (int n : set) {
        if (!set.contains(n - 1)) { // start of a sequence
            int len = 1;
            while (set.contains(n + len)) len++;
            best = Math.max(best, len);
        }
    }
    return best;
}

// Two Sum - single-pass hashmap of value -> index
int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        if (seen.containsKey(target - nums[i])) return new int[]{seen.get(target - nums[i]), i};
        seen.put(nums[i], i);
    }
    return new int[]{-1, -1};
}

// Top K Frequent Elements - bucket sort by frequency, O(n) time (better than heap's O(n log k))
int[] topKFrequentBucket(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);
    List<Integer>[] buckets = new List[nums.length + 1];
    for (Map.Entry<Integer, Integer> e : freq.entrySet()) {
        int f = e.getValue();
        if (buckets[f] == null) buckets[f] = new ArrayList<>();
        buckets[f].add(e.getKey());
    }
    int[] res = new int[k];
    int idx = 0;
    for (int f = buckets.length - 1; f >= 0 && idx < k; f--) {
        if (buckets[f] == null) continue;
        for (int num : buckets[f]) { if (idx < k) res[idx++] = num; }
    }
    return res;
}
```

**Key problems:** Two Sum, Group Anagrams, Longest Consecutive Sequence, Subarray Sum Equals K, LFU/LRU Cache design, Top K Frequent Elements.

---

## 14. Trees (Binary Tree / BST)

**When to use:** Hierarchical data, recursive divide & conquer, BST property for O(log n) search/insert.

**Intuition:** A tree's recursive definition (a node + two subtrees) maps naturally onto recursive solutions: solve the same problem on left and right subtrees, then combine results at the current node. BSTs add the invariant `left < node < right`, enabling pruned/binary search-like traversal.
**Flow:** Choose traversal order based on the problem — preorder (process before children, e.g. serialize), inorder (BST gives sorted order), postorder (need children's results first, e.g. height/diameter) — or level-order (BFS) for level-based problems.
**Complexity:** O(n) time for full traversal (visits every node once), O(h) space for recursion stack (h = height; O(log n) balanced, O(n) worst case skewed). BST search/insert: O(log n) average, O(n) worst case.

```java
class TreeNode { int val; TreeNode left, right; TreeNode(int v) { val = v; } }

// DFS Traversals (recursive)
void preorder(TreeNode root, List<Integer> out) {
    if (root == null) return;
    out.add(root.val);
    preorder(root.left, out);
    preorder(root.right, out);
}

// Level order (BFS)
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    Queue<TreeNode> q = new ArrayDeque<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int size = q.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode node = q.poll();
            level.add(node.val);
            if (node.left != null) q.offer(node.left);
            if (node.right != null) q.offer(node.right);
        }
        res.add(level);
    }
    return res;
}

// Diameter of Binary Tree
int diameter = 0;
int height(TreeNode node) {
    if (node == null) return 0;
    int left = height(node.left);
    int right = height(node.right);
    diameter = Math.max(diameter, left + right);
    return 1 + Math.max(left, right);
}

// Lowest Common Ancestor (general binary tree)
TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if (left != null && right != null) return root;
    return left != null ? left : right;
}

// Validate BST
boolean isValidBST(TreeNode root) { return validate(root, Long.MIN_VALUE, Long.MAX_VALUE); }
private boolean validate(TreeNode node, long lo, long hi) {
    if (node == null) return true;
    if (node.val <= lo || node.val >= hi) return false;
    return validate(node.left, lo, node.val) && validate(node.right, node.val, hi);
}

// Serialize/Deserialize Binary Tree (preorder + null markers)
String serialize(TreeNode root) {
    StringBuilder sb = new StringBuilder();
    serializeHelper(root, sb);
    return sb.toString();
}
private void serializeHelper(TreeNode node, StringBuilder sb) {
    if (node == null) { sb.append("#,"); return; }
    sb.append(node.val).append(",");
    serializeHelper(node.left, sb);
    serializeHelper(node.right, sb);
}
TreeNode deserialize(String data) {
    Deque<String> tokens = new ArrayDeque<>(Arrays.asList(data.split(",")));
    return deserializeHelper(tokens);
}
private TreeNode deserializeHelper(Deque<String> tokens) {
    String token = tokens.poll();
    if (token.equals("#")) return null;
    TreeNode node = new TreeNode(Integer.parseInt(token));
    node.left = deserializeHelper(tokens);
    node.right = deserializeHelper(tokens);
    return node;
}

// Binary Tree Right Side View - BFS, take last node of each level
List<Integer> rightSideView(TreeNode root) {
    List<Integer> res = new ArrayList<>();
    if (root == null) return res;
    Queue<TreeNode> q = new ArrayDeque<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int size = q.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = q.poll();
            if (i == size - 1) res.add(node.val);
            if (node.left != null) q.offer(node.left);
            if (node.right != null) q.offer(node.right);
        }
    }
    return res;
}

// Path Sum II - all root-to-leaf paths summing to target, backtracking
List<List<Integer>> pathSum(TreeNode root, int targetSum) {
    List<List<Integer>> res = new ArrayList<>();
    backtrackPathSum(root, targetSum, new ArrayList<>(), res);
    return res;
}
private void backtrackPathSum(TreeNode node, long remain, List<Integer> path, List<List<Integer>> res) {
    if (node == null) return;
    path.add(node.val);
    remain -= node.val;
    if (node.left == null && node.right == null && remain == 0) res.add(new ArrayList<>(path));
    else {
        backtrackPathSum(node.left, remain, path, res);
        backtrackPathSum(node.right, remain, path, res);
    }
    path.remove(path.size() - 1);
}

// Kth Smallest Element in a BST - inorder traversal gives sorted order, stop at kth
int kthSmallest(TreeNode root, int k) {
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode cur = root;
    while (cur != null || !stack.isEmpty()) {
        while (cur != null) { stack.push(cur); cur = cur.left; }
        cur = stack.pop();
        if (--k == 0) return cur.val;
        cur = cur.right;
    }
    return -1;
}

// Construct Binary Tree from Preorder and Inorder Traversal
TreeNode buildTree(int[] preorder, int[] inorder) {
    Map<Integer, Integer> inorderIdx = new HashMap<>();
    for (int i = 0; i < inorder.length; i++) inorderIdx.put(inorder[i], i);
    int[] preIdx = {0};
    return build(preorder, inorderIdx, preIdx, 0, inorder.length - 1);
}
private TreeNode build(int[] preorder, Map<Integer, Integer> inorderIdx, int[] preIdx, int lo, int hi) {
    if (lo > hi) return null;
    int rootVal = preorder[preIdx[0]++];
    TreeNode root = new TreeNode(rootVal);
    int mid = inorderIdx.get(rootVal);
    root.left = build(preorder, inorderIdx, preIdx, lo, mid - 1);
    root.right = build(preorder, inorderIdx, preIdx, mid + 1, hi);
    return root;
}

// Vertical Order Traversal - track (column, row, val), group by column, sort by row then value
List<List<Integer>> verticalTraversal(TreeNode root) {
    // node -> {col, row, val}
    List<int[]> nodes = new ArrayList<>();
    collectVertical(root, 0, 0, nodes);
    nodes.sort((a, b) -> a[0] != b[0] ? a[0] - b[0] : (a[1] != b[1] ? a[1] - b[1] : a[2] - b[2]));
    List<List<Integer>> res = new ArrayList<>();
    int prevCol = Integer.MIN_VALUE;
    for (int[] n : nodes) {
        if (n[0] != prevCol) { res.add(new ArrayList<>()); prevCol = n[0]; }
        res.get(res.size() - 1).add(n[2]);
    }
    return res;
}
private void collectVertical(TreeNode node, int col, int row, List<int[]> nodes) {
    if (node == null) return;
    nodes.add(new int[]{col, row, node.val});
    collectVertical(node.left, col - 1, row + 1, nodes);
    collectVertical(node.right, col + 1, row + 1, nodes);
}
```

**Key problems:** Max Depth, Diameter of Binary Tree, LCA, Validate BST, Serialize/Deserialize, Binary Tree Right Side View, Path Sum II, Kth Smallest in BST, Construct Tree from Preorder+Inorder, Vertical Order Traversal.

---

## 15. Tries

**When to use:** Prefix matching, autocomplete, word search in dictionaries.

**Intuition:** Store strings as paths in a tree where shared prefixes share nodes — this makes prefix lookup proportional to the prefix length, not the number of stored words.
**Flow:** `insert`: walk/create a child node per character, mark the last node as end-of-word. `search`/`startsWith`: walk the same path; fail early if a character's child doesn't exist.
**Complexity:** O(L) time per insert/search (L = word/prefix length), O(N*L) space in the worst case (N words, average length L, no shared prefixes) but typically much less due to sharing.

```java
class Trie {
    class Node {
        Node[] children = new Node[26];
        boolean isEnd;
    }
    private final Node root = new Node();

    void insert(String word) {
        Node cur = root;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (cur.children[idx] == null) cur.children[idx] = new Node();
            cur = cur.children[idx];
        }
        cur.isEnd = true;
    }

    boolean search(String word) {
        Node node = find(word);
        return node != null && node.isEnd;
    }

    boolean startsWith(String prefix) { return find(prefix) != null; }

    private Node find(String s) {
        Node cur = root;
        for (char c : s.toCharArray()) {
            int idx = c - 'a';
            if (cur.children[idx] == null) return null;
            cur = cur.children[idx];
        }
        return cur;
    }
}

// Add and Search Word - Trie supporting '.' wildcard search
class WordDictionary {
    class Node { Node[] children = new Node[26]; boolean isEnd; }
    private final Node root = new Node();
    void addWord(String word) {
        Node cur = root;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (cur.children[idx] == null) cur.children[idx] = new Node();
            cur = cur.children[idx];
        }
        cur.isEnd = true;
    }
    boolean search(String word) { return dfsSearch(word, 0, root); }
    private boolean dfsSearch(String word, int idx, Node node) {
        if (node == null) return false;
        if (idx == word.length()) return node.isEnd;
        char c = word.charAt(idx);
        if (c == '.') {
            for (Node child : node.children) if (dfsSearch(word, idx + 1, child)) return true;
            return false;
        }
        return dfsSearch(word, idx + 1, node.children[c - 'a']);
    }
}

// Word Search II - build a Trie of all words, DFS the grid, prune branches with no matching prefix
List<String> findWords(char[][] board, String[] words) {
    TrieNode root = new TrieNode();
    for (String w : words) insertWord(root, w);
    List<String> res = new ArrayList<>();
    for (int i = 0; i < board.length; i++)
        for (int j = 0; j < board[0].length; j++)
            dfsWordSearch(board, i, j, root, res);
    return res;
}
class TrieNode { Map<Character, TrieNode> children = new HashMap<>(); String word; }
private void insertWord(TrieNode root, String word) {
    TrieNode cur = root;
    for (char c : word.toCharArray()) cur = cur.children.computeIfAbsent(c, k -> new TrieNode());
    cur.word = word;
}
private void dfsWordSearch(char[][] board, int i, int j, TrieNode node, List<String> res) {
    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length || board[i][j] == '#') return;
    char c = board[i][j];
    TrieNode next = node.children.get(c);
    if (next == null) return;
    if (next.word != null) { res.add(next.word); next.word = null; } // avoid duplicates
    board[i][j] = '#'; // mark visited
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    for (int[] d : dirs) dfsWordSearch(board, i + d[0], j + d[1], next, res);
    board[i][j] = c; // restore
}
```

**Key problems:** Implement Trie, Word Search II, Add and Search Word, Longest Word in Dictionary, Replace Words.

---

## 16. Heaps / Priority Queue

**When to use:** Top-K elements, K-way merge, scheduling, median maintenance.

**Intuition:** A heap always gives O(1) access to the min/max without fully sorting — ideal when you repeatedly need "the smallest/largest so far" as data streams in, or need to merge many sorted sources by always picking the smallest next candidate.
**Flow:** For top-K: maintain a heap of size K, pushing new elements and popping the worst one if the heap overflows. For K-way merge: seed the heap with the first element of each source; each time you pop, push that source's next element.
**Complexity:** O(log k) per insert/extract, O(n log k) overall for streaming top-K; O(n log k) for K-way merge of n total elements across k sources. Space: O(k).

```java
// Kth Largest Element in a Stream
class KthLargest {
    private final PriorityQueue<Integer> minHeap; // size k, top is kth largest
    private final int k;
    KthLargest(int k, int[] nums) {
        this.k = k;
        minHeap = new PriorityQueue<>();
        for (int n : nums) add(n);
    }
    int add(int val) {
        minHeap.offer(val);
        if (minHeap.size() > k) minHeap.poll();
        return minHeap.peek();
    }
}

// Top K Frequent Elements
int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);
    PriorityQueue<Integer> heap = new PriorityQueue<>((a, b) -> freq.get(a) - freq.get(b));
    for (int key : freq.keySet()) {
        heap.offer(key);
        if (heap.size() > k) heap.poll();
    }
    int[] res = new int[k];
    for (int i = k - 1; i >= 0; i--) res[i] = heap.poll();
    return res;
}

// Merge K Sorted Lists
ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> heap = new PriorityQueue<>((a, b) -> a.val - b.val);
    for (ListNode node : lists) if (node != null) heap.offer(node);
    ListNode dummy = new ListNode(0), tail = dummy;
    while (!heap.isEmpty()) {
        ListNode node = heap.poll();
        tail.next = node; tail = tail.next;
        if (node.next != null) heap.offer(node.next);
    }
    return dummy.next;
}

// Median Finder using two heaps
class MedianFinder {
    private final PriorityQueue<Integer> low = new PriorityQueue<>(Collections.reverseOrder()); // max-heap
    private final PriorityQueue<Integer> high = new PriorityQueue<>(); // min-heap
    void addNum(int num) {
        low.offer(num);
        high.offer(low.poll());
        if (high.size() > low.size()) low.offer(high.poll());
    }
    double findMedian() {
        return low.size() > high.size() ? low.peek() : (low.peek() + high.peek()) / 2.0;
    }
}
```

**Key problems:** Kth Largest Element, Top K Frequent Elements/Words, Merge K Sorted Lists, Find Median from Data Stream, Task Scheduler, Meeting Rooms II.

---

## 17. Graphs — BFS / DFS

**When to use:** Connectivity, shortest path (unweighted), traversal, components, cycle detection, bipartiteness.

**Intuition:** DFS explores as deep as possible before backtracking (good for connectivity/cycle detection via recursion state); BFS explores layer by layer (guarantees shortest path in unweighted graphs).
**Flow:** Build an adjacency list -> pick DFS (recursion/stack, track visited + optionally parent/state for cycle detection) or BFS (queue, track visited + distance) depending on whether you need "any path/order" (DFS) or "shortest path/levels" (BFS).
**Complexity:** O(V + E) time, O(V) space, for both DFS and BFS.

```java
// Build adjacency list
List<List<Integer>> buildGraph(int n, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    for (int[] e : edges) { graph.get(e[0]).add(e[1]); graph.get(e[1]).add(e[0]); }
    return graph;
}

// DFS - number of connected components
int countComponents(int n, int[][] edges) {
    List<List<Integer>> graph = buildGraph(n, edges);
    boolean[] visited = new boolean[n];
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (!visited[i]) { count++; dfs(graph, i, visited); }
    }
    return count;
}
private void dfs(List<List<Integer>> graph, int node, boolean[] visited) {
    visited[node] = true;
    for (int nei : graph.get(node)) if (!visited[nei]) dfs(graph, nei, visited);
}

// Detect cycle in undirected graph (DFS with parent)
boolean hasCycleUndirected(List<List<Integer>> graph, int n) {
    boolean[] visited = new boolean[n];
    for (int i = 0; i < n; i++) {
        if (!visited[i] && dfsCycle(graph, i, -1, visited)) return true;
    }
    return false;
}
private boolean dfsCycle(List<List<Integer>> graph, int node, int parent, boolean[] visited) {
    visited[node] = true;
    for (int nei : graph.get(node)) {
        if (!visited[nei]) { if (dfsCycle(graph, nei, node, visited)) return true; }
        else if (nei != parent) return true;
    }
    return false;
}

// Detect cycle in directed graph (DFS with 3-color state)
boolean hasCycleDirected(List<List<Integer>> graph, int n) {
    int[] state = new int[n]; // 0=unvisited, 1=visiting, 2=done
    for (int i = 0; i < n; i++) if (state[i] == 0 && dfsDirected(graph, i, state)) return true;
    return false;
}
private boolean dfsDirected(List<List<Integer>> graph, int node, int[] state) {
    state[node] = 1;
    for (int nei : graph.get(node)) {
        if (state[nei] == 1) return true;
        if (state[nei] == 0 && dfsDirected(graph, nei, state)) return true;
    }
    state[node] = 2;
    return false;
}

// Bipartite check (BFS coloring)
boolean isBipartite(int[][] graph) {
    int n = graph.length;
    int[] color = new int[n]; // 0 = uncolored
    for (int start = 0; start < n; start++) {
        if (color[start] != 0) continue;
        color[start] = 1;
        Queue<Integer> q = new ArrayDeque<>();
        q.offer(start);
        while (!q.isEmpty()) {
            int node = q.poll();
            for (int nei : graph[node]) {
                if (color[nei] == 0) { color[nei] = -color[node]; q.offer(nei); }
                else if (color[nei] == color[node]) return false;
            }
        }
    }
    return true;
}

// Number of Islands - DFS flood fill on a grid
int numIslandsGraph(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++)
        for (int j = 0; j < grid[0].length; j++)
            if (grid[i][j] == '1') { count++; sinkIsland(grid, i, j); }
    return count;
}
private void sinkIsland(char[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length || grid[i][j] != '1') return;
    grid[i][j] = '0';
    sinkIsland(grid, i + 1, j); sinkIsland(grid, i - 1, j);
    sinkIsland(grid, i, j + 1); sinkIsland(grid, i, j - 1);
}

// Clone Graph - DFS with a map of original node -> cloned node
class GraphNode { int val; List<GraphNode> neighbors; GraphNode(int v) { val = v; neighbors = new ArrayList<>(); } }
GraphNode cloneGraph(GraphNode node) {
    if (node == null) return null;
    return cloneDfs(node, new HashMap<>());
}
private GraphNode cloneDfs(GraphNode node, Map<GraphNode, GraphNode> visited) {
    if (visited.containsKey(node)) return visited.get(node);
    GraphNode clone = new GraphNode(node.val);
    visited.put(node, clone);
    for (GraphNode nei : node.neighbors) clone.neighbors.add(cloneDfs(nei, visited));
    return clone;
}

// Course Schedule I/II - detect cycle via topological sort (Kahn's), see Section 19 for full topoSort
boolean canFinish(int numCourses, int[][] prerequisites) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) graph.add(new ArrayList<>());
    int[] indegree = new int[numCourses];
    for (int[] p : prerequisites) { graph.get(p[1]).add(p[0]); indegree[p[0]]++; }
    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < numCourses; i++) if (indegree[i] == 0) queue.offer(i);
    int visited = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        visited++;
        for (int next : graph.get(course)) if (--indegree[next] == 0) queue.offer(next);
    }
    return visited == numCourses;
}

// Pacific Atlantic Water Flow - reverse multi-source BFS/DFS from both oceans, intersect reachable sets
List<List<Integer>> pacificAtlantic(int[][] heights) {
    int rows = heights.length, cols = heights[0].length;
    boolean[][] pacific = new boolean[rows][cols], atlantic = new boolean[rows][cols];
    for (int i = 0; i < rows; i++) { dfsWaterFlow(heights, i, 0, pacific); dfsWaterFlow(heights, i, cols - 1, atlantic); }
    for (int j = 0; j < cols; j++) { dfsWaterFlow(heights, 0, j, pacific); dfsWaterFlow(heights, rows - 1, j, atlantic); }
    List<List<Integer>> res = new ArrayList<>();
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            if (pacific[i][j] && atlantic[i][j]) res.add(Arrays.asList(i, j));
    return res;
}
private void dfsWaterFlow(int[][] heights, int i, int j, boolean[][] visited) {
    visited[i][j] = true;
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    for (int[] d : dirs) {
        int ni = i + d[0], nj = j + d[1];
        if (ni >= 0 && ni < heights.length && nj >= 0 && nj < heights[0].length &&
            !visited[ni][nj] && heights[ni][nj] >= heights[i][j]) {
            dfsWaterFlow(heights, ni, nj, visited);
        }
    }
}
```

**Key problems:** Number of Islands, Clone Graph, Course Schedule I/II, Rotting Oranges, Pacific Atlantic Water Flow, Word Ladder, Bipartite Graph Check, Redundant Connection.

---

## 18. Union-Find (Disjoint Set)

**When to use:** Dynamic connectivity, cycle detection in undirected graphs, Kruskal's MST, grouping.

**Intuition:** Represent each connected component by a "representative" (root). Two nodes are connected iff they share the same root. Path compression flattens trees during `find`, and union-by-rank keeps trees shallow — together they make near-O(1) operations.
**Flow:** `find(x)`: follow parent pointers to the root, compressing the path. `union(a, b)`: find both roots; if different, attach the smaller-rank tree under the larger one (else it's a cycle/already connected).
**Complexity:** O(1) amortized per operation (technically O(α(n)), inverse Ackermann — effectively constant), O(V) space.

```java
class UnionFind {
    int[] parent, rank;
    UnionFind(int n) {
        parent = new int[n]; rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]); // path compression
        return parent[x];
    }
    boolean union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false; // already connected -> cycle
        if (rank[ra] < rank[rb]) { int t = ra; ra = rb; rb = t; }
        parent[rb] = ra;
        if (rank[ra] == rank[rb]) rank[ra]++;
        return true;
    }
}

// Number of Provinces - count connected components via union-find over the adjacency matrix
int findCircleNum(int[][] isConnected) {
    int n = isConnected.length;
    UnionFind uf = new UnionFind(n);
    int provinces = n;
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)
            if (isConnected[i][j] == 1 && uf.union(i, j)) provinces--;
    return provinces;
}

// Accounts Merge - union accounts sharing any email, then group by root
List<List<String>> accountsMerge(List<List<String>> accounts) {
    Map<String, String> emailToName = new HashMap<>();
    Map<String, Integer> emailToId = new HashMap<>();
    int id = 0;
    for (List<String> acc : accounts) {
        String name = acc.get(0);
        for (int i = 1; i < acc.size(); i++) {
            String email = acc.get(i);
            emailToName.put(email, name);
            emailToId.putIfAbsent(email, id++);
        }
    }
    UnionFind uf = new UnionFind(id);
    for (List<String> acc : accounts) {
        int firstId = emailToId.get(acc.get(1));
        for (int i = 2; i < acc.size(); i++) uf.union(firstId, emailToId.get(acc.get(i)));
    }
    Map<Integer, TreeSet<String>> groups = new HashMap<>();
    for (String email : emailToId.keySet()) {
        int root = uf.find(emailToId.get(email));
        groups.computeIfAbsent(root, k -> new TreeSet<>()).add(email);
    }
    List<List<String>> res = new ArrayList<>();
    for (Map.Entry<Integer, TreeSet<String>> e : groups.entrySet()) {
        List<String> merged = new ArrayList<>();
        String anyEmail = e.getValue().first();
        merged.add(emailToName.get(anyEmail));
        merged.addAll(e.getValue());
        res.add(merged);
    }
    return res;
}
```

**Key problems:** Number of Provinces, Redundant Connection, Accounts Merge, Graph Valid Tree, Number of Islands II, Most Stones Removed.

---

## 19. Topological Sort

**When to use:** Ordering with dependencies (DAG), detecting cycle in directed graph.

**Intuition:** A node can only be "ready" to process once everything it depends on (its incoming edges) has already been processed. Kahn's algorithm mirrors this literally: process nodes with zero remaining dependencies (indegree 0) first, then "unlock" their neighbors.
**Flow:** Compute indegree for every node -> enqueue all indegree-0 nodes -> repeatedly dequeue a node, add to order, decrement neighbors' indegree, enqueue any that hit 0 -> if final order size < n, a cycle exists (no valid topological order).
**Complexity:** O(V + E) time, O(V) space.

```java
// Kahn's algorithm (BFS-based)
int[] topoSort(int n, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    int[] indegree = new int[n];
    for (int[] e : edges) { graph.get(e[0]).add(e[1]); indegree[e[1]]++; }
    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < n; i++) if (indegree[i] == 0) queue.offer(i);
    int[] order = new int[n];
    int idx = 0;
    while (!queue.isEmpty()) {
        int node = queue.poll();
        order[idx++] = node;
        for (int nei : graph.get(node)) {
            if (--indegree[nei] == 0) queue.offer(nei);
        }
    }
    return idx == n ? order : new int[0]; // empty => cycle detected
}
```

**Key problems:** Course Schedule I/II, Alien Dictionary, Sequence Reconstruction, Minimum Height Trees, Parallel Courses.

---

## 20. Shortest Path Algorithms

**When to use:** Weighted graphs (Dijkstra for non-negative weights, Bellman-Ford for negative weights, Floyd-Warshall for all-pairs).

**Intuition:** Dijkstra is a greedy BFS variant — always expand the currently-closest unvisited node next, which is safe because all edge weights are non-negative (no shortcut can appear later). Bellman-Ford instead relaxes all edges repeatedly (V-1 times) so it tolerates negative weights and can detect negative cycles.
**Flow (Dijkstra):** Push `(node, dist)` to a min-heap seeded with the source -> pop the closest node -> if it's stale (already finalized with smaller dist) skip -> relax all neighbors, pushing improved distances. **Flow (Bellman-Ford):** Relax every edge, V-1 times total; if any edge can still relax after that, a negative cycle exists.
**Complexity:** Dijkstra O(E log V) time with a heap, O(V) space. Bellman-Ford O(V·E) time, O(V) space. Floyd-Warshall (all-pairs) O(V³) time, O(V²) space.

```java
// Dijkstra's Algorithm
int[] dijkstra(int n, List<int[]>[] graph, int src) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]); // [node, dist]
    pq.offer(new int[]{src, 0});
    while (!pq.isEmpty()) {
        int[] cur = pq.poll();
        int node = cur[0], d = cur[1];
        if (d > dist[node]) continue;
        for (int[] edge : graph[node]) { // edge = [neighbor, weight]
            int next = edge[0], weight = edge[1];
            if (dist[node] + weight < dist[next]) {
                dist[next] = dist[node] + weight;
                pq.offer(new int[]{next, dist[next]});
            }
        }
    }
    return dist;
}

// Bellman-Ford (handles negative weights, detects negative cycle)
int[] bellmanFord(int n, int[][] edges, int src) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    for (int i = 0; i < n - 1; i++) {
        for (int[] e : edges) {
            if (dist[e[0]] != Integer.MAX_VALUE && dist[e[0]] + e[2] < dist[e[1]]) {
                dist[e[1]] = dist[e[0]] + e[2];
            }
        }
    }
    return dist;
}

// Network Delay Time - Dijkstra from source, answer = max distance among all reachable nodes
int networkDelayTime(int[][] times, int n, int k) {
    List<int[]>[] graph = new List[n + 1];
    for (int i = 1; i <= n; i++) graph[i] = new ArrayList<>();
    for (int[] t : times) graph[t[0]].add(new int[]{t[1], t[2]});
    int[] dist = dijkstra(n + 1, graph, k);
    int maxDist = 0;
    for (int i = 1; i <= n; i++) {
        if (dist[i] == Integer.MAX_VALUE) return -1; // unreachable node
        maxDist = Math.max(maxDist, dist[i]);
    }
    return maxDist;
}

// Cheapest Flights Within K Stops - modified Bellman-Ford limited to K+1 edge relaxations
int findCheapestPrice(int n, int[][] flights, int src, int dst, int k) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    for (int i = 0; i <= k; i++) {
        int[] tmp = dist.clone();
        for (int[] f : flights) {
            if (dist[f[0]] != Integer.MAX_VALUE && dist[f[0]] + f[2] < tmp[f[1]]) {
                tmp[f[1]] = dist[f[0]] + f[2];
            }
        }
        dist = tmp;
    }
    return dist[dst] == Integer.MAX_VALUE ? -1 : dist[dst];
}
```

**Key problems:** Network Delay Time, Cheapest Flights Within K Stops, Path with Minimum Effort, Swim in Rising Water.

---

## 21. Minimum Spanning Tree

**When to use:** Connect all nodes with minimum total edge weight.

**Intuition:** Kruskal's is greedy: always add the globally cheapest edge that doesn't create a cycle. Union-Find efficiently answers "would this edge create a cycle?" (i.e., are the two endpoints already connected?).
**Flow:** Sort all edges by weight -> for each edge (cheapest first), union its endpoints if they're in different components (add to MST) -> stop once you have V-1 edges.
**Complexity:** O(E log E) time (dominated by sorting), O(V) space for Union-Find.

```java
// Kruskal's Algorithm (using Union-Find)
int kruskalMST(int n, int[][] edges) {
    Arrays.sort(edges, (a, b) -> a[2] - b[2]); // sort by weight
    UnionFind uf = new UnionFind(n);
    int totalWeight = 0, edgesUsed = 0;
    for (int[] e : edges) {
        if (uf.union(e[0], e[1])) { totalWeight += e[2]; edgesUsed++; }
        if (edgesUsed == n - 1) break;
    }
    return totalWeight;
}
```

**Key problems:** Min Cost to Connect All Points, Connecting Cities With Minimum Cost, Optimize Water Distribution.

---

## 22. Backtracking

**When to use:** Generate all combinations/permutations/subsets, constraint satisfaction (N-Queens, Sudoku), path finding with pruning.

**Intuition:** Explore a decision tree where each node represents a partial solution; recurse into a choice, and undo it ("backtrack") after exploring, so the same array/list can be reused across all branches instead of copying state.
**Flow:** At each step, try every valid next choice -> add it to the current path -> recurse -> remove it (backtrack) before trying the next choice. Prune early whenever a partial state can't lead to a valid solution.
**Complexity:** Exponential in general — O(2ⁿ) for subsets, O(n!) for permutations, O(kⁿ) for k-ary choices — O(n) extra space for the recursion stack/current path (excluding output storage).

```java
// Subsets
List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> res = new ArrayList<>();
    backtrackSubsets(nums, 0, new ArrayList<>(), res);
    return res;
}
private void backtrackSubsets(int[] nums, int start, List<Integer> cur, List<List<Integer>> res) {
    res.add(new ArrayList<>(cur));
    for (int i = start; i < nums.length; i++) {
        cur.add(nums[i]);
        backtrackSubsets(nums, i + 1, cur, res);
        cur.remove(cur.size() - 1);
    }
}

// Permutations
List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> res = new ArrayList<>();
    backtrackPermute(nums, new ArrayList<>(), new boolean[nums.length], res);
    return res;
}
private void backtrackPermute(int[] nums, List<Integer> cur, boolean[] used, List<List<Integer>> res) {
    if (cur.size() == nums.length) { res.add(new ArrayList<>(cur)); return; }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true; cur.add(nums[i]);
        backtrackPermute(nums, cur, used, res);
        used[i] = false; cur.remove(cur.size() - 1);
    }
}

// Combination Sum (reuse allowed)
List<List<Integer>> combinationSum(int[] candidates, int target) {
    List<List<Integer>> res = new ArrayList<>();
    Arrays.sort(candidates);
    backtrackCombo(candidates, target, 0, new ArrayList<>(), res);
    return res;
}
private void backtrackCombo(int[] cands, int remain, int start, List<Integer> cur, List<List<Integer>> res) {
    if (remain == 0) { res.add(new ArrayList<>(cur)); return; }
    for (int i = start; i < cands.length; i++) {
        if (cands[i] > remain) break;
        cur.add(cands[i]);
        backtrackCombo(cands, remain - cands[i], i, cur, res); // i (not i+1) => reuse
        cur.remove(cur.size() - 1);
    }
}

// N-Queens (bitmask column/diag pruning)
int totalNQueens(int n) {
    return solveNQueens(n, 0, 0, 0, 0);
}
private int solveNQueens(int n, int row, int cols, int diag1, int diag2) {
    if (row == n) return 1;
    int count = 0;
    int available = ((1 << n) - 1) & ~(cols | diag1 | diag2);
    while (available != 0) {
        int bit = available & (-available);
        available -= bit;
        count += solveNQueens(n, row + 1, cols | bit, (diag1 | bit) << 1, (diag2 | bit) >> 1);
    }
    return count;
}

// Word Search - DFS/backtrack from each cell, marking visited in-place
boolean existWordSearch(char[][] board, String word) {
    for (int i = 0; i < board.length; i++)
        for (int j = 0; j < board[0].length; j++)
            if (dfsWordExist(board, i, j, word, 0)) return true;
    return false;
}
private boolean dfsWordExist(char[][] board, int i, int j, String word, int idx) {
    if (idx == word.length()) return true;
    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length || board[i][j] != word.charAt(idx)) return false;
    char tmp = board[i][j];
    board[i][j] = '#';
    boolean found = dfsWordExist(board, i + 1, j, word, idx + 1) ||
                     dfsWordExist(board, i - 1, j, word, idx + 1) ||
                     dfsWordExist(board, i, j + 1, word, idx + 1) ||
                     dfsWordExist(board, i, j - 1, word, idx + 1);
    board[i][j] = tmp;
    return found;
}

// Palindrome Partitioning - backtrack, only recurse into substrings that are palindromes
List<List<String>> partition(String s) {
    List<List<String>> res = new ArrayList<>();
    backtrackPartition(s, 0, new ArrayList<>(), res);
    return res;
}
private void backtrackPartition(String s, int start, List<String> cur, List<List<String>> res) {
    if (start == s.length()) { res.add(new ArrayList<>(cur)); return; }
    for (int end = start + 1; end <= s.length(); end++) {
        String sub = s.substring(start, end);
        if (isPalindromeStr(sub)) {
            cur.add(sub);
            backtrackPartition(s, end, cur, res);
            cur.remove(cur.size() - 1);
        }
    }
}
private boolean isPalindromeStr(String s) {
    int lo = 0, hi = s.length() - 1;
    while (lo < hi) if (s.charAt(lo++) != s.charAt(hi--)) return false;
    return true;
}

// Generate Parentheses - backtrack tracking open/close counts, prune invalid states early
List<String> generateParenthesis(int n) {
    List<String> res = new ArrayList<>();
    backtrackParens(n, 0, 0, new StringBuilder(), res);
    return res;
}
private void backtrackParens(int n, int open, int close, StringBuilder cur, List<String> res) {
    if (cur.length() == 2 * n) { res.add(cur.toString()); return; }
    if (open < n) { cur.append('('); backtrackParens(n, open + 1, close, cur, res); cur.deleteCharAt(cur.length() - 1); }
    if (close < open) { cur.append(')'); backtrackParens(n, open, close + 1, cur, res); cur.deleteCharAt(cur.length() - 1); }
}

// Sudoku Solver - backtrack trying digits 1-9 in each empty cell, validating row/col/box
void solveSudoku(char[][] board) { solveSudokuHelper(board); }
private boolean solveSudokuHelper(char[][] board) {
    for (int i = 0; i < 9; i++) {
        for (int j = 0; j < 9; j++) {
            if (board[i][j] == '.') {
                for (char c = '1'; c <= '9'; c++) {
                    if (isValidSudokuPlacement(board, i, j, c)) {
                        board[i][j] = c;
                        if (solveSudokuHelper(board)) return true;
                        board[i][j] = '.';
                    }
                }
                return false; // no valid digit -> backtrack
            }
        }
    }
    return true; // board fully filled
}
private boolean isValidSudokuPlacement(char[][] board, int row, int col, char c) {
    for (int i = 0; i < 9; i++) {
        if (board[row][i] == c || board[i][col] == c) return false;
        int boxRow = 3 * (row / 3) + i / 3, boxCol = 3 * (col / 3) + i % 3;
        if (board[boxRow][boxCol] == c) return false;
    }
    return true;
}
```

**Key problems:** Subsets I/II, Permutations I/II, Combination Sum I/II/III, N-Queens, Word Search, Palindrome Partitioning, Generate Parentheses, Sudoku Solver.

---

## 23. Dynamic Programming

**When to use:** Optimal substructure + overlapping subproblems. Identify state, transition, base case. Approach: recursion -> memoization -> tabulation -> space optimization.

**Intuition:** If a problem's optimal answer can be built from optimal answers to smaller subproblems, AND those subproblems repeat, cache results instead of recomputing them. The trick is always defining `dp[state] =` the answer for that state, then finding the recurrence connecting it to smaller states.
**Flow:** (1) Define state (what varies between subproblems, e.g. index, remaining capacity). (2) Write the recurrence/transition (how to compute `dp[state]` from smaller states). (3) Identify base cases. (4) Decide direction: top-down recursion + memo, or bottom-up tabulation. (5) Optionally compress dp array dimensions if only the last row/few states are needed.
**Complexity:** Typically O(states × transitions) time; e.g. O(n) for 1D, O(n·W) for knapsack, O(n²) for LCS/Edit Distance. Space often matches state count but can frequently be reduced by one dimension (rolling array).

### 1D DP

```java
// Climbing Stairs / Fibonacci style
int climbStairs(int n) {
    if (n <= 2) return n;
    int prev2 = 1, prev1 = 2;
    for (int i = 3; i <= n; i++) {
        int cur = prev1 + prev2;
        prev2 = prev1; prev1 = cur;
    }
    return prev1;
}

// House Robber
int rob(int[] nums) {
    int prevNo = 0, prevYes = 0;
    for (int n : nums) {
        int newYes = prevNo + n;
        int newNo = Math.max(prevNo, prevYes);
        prevNo = newNo; prevYes = newYes;
    }
    return Math.max(prevNo, prevYes);
}

// Longest Increasing Subsequence - O(n log n) with binary search
int lengthOfLIS(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    for (int num : nums) {
        int idx = Collections.binarySearch(tails, num);
        if (idx < 0) idx = -(idx + 1);
        if (idx == tails.size()) tails.add(num); else tails.set(idx, num);
    }
    return tails.size();
}

// Word Break - dp[i] = true if s[0..i) can be segmented into dictionary words
boolean wordBreak(String s, List<String> wordDict) {
    Set<String> dict = new HashSet<>(wordDict);
    int n = s.length();
    boolean[] dp = new boolean[n + 1];
    dp[0] = true;
    for (int i = 1; i <= n; i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && dict.contains(s.substring(j, i))) { dp[i] = true; break; }
        }
    }
    return dp[n];
}

// Decode Ways - dp[i] = ways to decode s[0..i), depends on 1-digit and 2-digit valid decodings
int numDecodings(String s) {
    int n = s.length();
    int[] dp = new int[n + 1];
    dp[0] = 1;
    dp[1] = s.charAt(0) != '0' ? 1 : 0;
    for (int i = 2; i <= n; i++) {
        char c1 = s.charAt(i - 1), c2 = s.charAt(i - 2);
        if (c1 != '0') dp[i] += dp[i - 1];
        int twoDigit = (c2 - '0') * 10 + (c1 - '0');
        if (twoDigit >= 10 && twoDigit <= 26) dp[i] += dp[i - 2];
    }
    return dp[n];
}
```

### 2D DP

```java
// 0/1 Knapsack
int knapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n + 1][capacity + 1];
    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= capacity; w++) {
            dp[i][w] = dp[i - 1][w];
            if (weights[i - 1] <= w) {
                dp[i][w] = Math.max(dp[i][w], dp[i - 1][w - weights[i - 1]] + values[i - 1]);
            }
        }
    }
    return dp[n][capacity];
}

// Longest Common Subsequence
int longestCommonSubsequence(String a, String b) {
    int m = a.length(), n = b.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (a.charAt(i - 1) == b.charAt(j - 1)) dp[i][j] = dp[i - 1][j - 1] + 1;
            else dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
        }
    }
    return dp[m][n];
}

// Edit Distance
int minDistance(String a, String b) {
    int m = a.length(), n = b.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (a.charAt(i - 1) == b.charAt(j - 1)) dp[i][j] = dp[i - 1][j - 1];
            else dp[i][j] = 1 + Math.min(dp[i - 1][j - 1], Math.min(dp[i - 1][j], dp[i][j - 1]));
        }
    }
    return dp[m][n];
}

// Coin Change (min coins)
int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, Integer.MAX_VALUE);
    dp[0] = 0;
    for (int i = 1; i <= amount; i++) {
        for (int c : coins) {
            if (c <= i && dp[i - c] != Integer.MAX_VALUE) dp[i] = Math.min(dp[i], dp[i - c] + 1);
        }
    }
    return dp[amount] == Integer.MAX_VALUE ? -1 : dp[amount];
}

// Unique Paths (grid DP)
int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int j = 0; j < n; j++) dp[0][j] = 1;
    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
    return dp[m - 1][n - 1];
}

// Maximal Square - dp[i][j] = side length of largest all-1s square with bottom-right corner at (i,j)
int maximalSquare(char[][] matrix) {
    int m = matrix.length, n = matrix[0].length, best = 0;
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (matrix[i - 1][j - 1] == '1') {
                dp[i][j] = Math.min(dp[i - 1][j - 1], Math.min(dp[i - 1][j], dp[i][j - 1])) + 1;
                best = Math.max(best, dp[i][j]);
            }
        }
    }
    return best * best;
}

// Interleaving String - dp[i][j] = true if s1[0..i) and s2[0..j) interleave to form s3[0..i+j)
boolean isInterleave(String s1, String s2, String s3) {
    int m = s1.length(), n = s2.length();
    if (m + n != s3.length()) return false;
    boolean[][] dp = new boolean[m + 1][n + 1];
    dp[0][0] = true;
    for (int i = 0; i <= m; i++) {
        for (int j = 0; j <= n; j++) {
            if (i > 0 && dp[i - 1][j] && s1.charAt(i - 1) == s3.charAt(i + j - 1)) dp[i][j] = true;
            if (j > 0 && dp[i][j - 1] && s2.charAt(j - 1) == s3.charAt(i + j - 1)) dp[i][j] = true;
        }
    }
    return dp[m][n];
}
```

### Interval / Partition DP

```java
// Palindrome Partitioning - Minimum Cuts
int minCut(String s) {
    int n = s.length();
    boolean[][] isPal = new boolean[n][n];
    for (int i = 0; i < n; i++) isPal[i][i] = true;
    for (int len = 2; len <= n; len++) {
        for (int i = 0; i + len - 1 < n; i++) {
            int j = i + len - 1;
            if (s.charAt(i) == s.charAt(j) && (len == 2 || isPal[i + 1][j - 1])) isPal[i][j] = true;
        }
    }
    int[] dp = new int[n];
    for (int i = 0; i < n; i++) {
        if (isPal[0][i]) { dp[i] = 0; continue; }
        dp[i] = Integer.MAX_VALUE;
        for (int j = 1; j <= i; j++) {
            if (isPal[j][i]) dp[i] = Math.min(dp[i], dp[j - 1] + 1);
        }
    }
    return dp[n - 1];
}
```

**Key problems:**
- **1D:** Climbing Stairs, House Robber I/II, Decode Ways, Word Break, LIS.
- **2D:** 0/1 Knapsack, LCS, Edit Distance, Coin Change (I & II), Unique Paths, Interleaving String.
- **Grid:** Minimum Path Sum, Maximal Square, Dungeon Game.
- **Partition/Interval:** Palindrome Partitioning II, Burst Balloons, Matrix Chain Multiplication.
- **Digit DP / Bitmask DP:** Traveling Salesman-style problems, Partition to K Equal Sum Subsets.
- **String DP:** Regular Expression Matching, Wildcard Matching, Distinct Subsequences.

---

## 24. Greedy

**When to use:** Local optimal choice leads to global optimum; interval scheduling, jump games.

**Intuition:** Greedy works only when a locally-best choice never forecloses a better global solution — provable via an exchange argument or matroid structure. If greedy doesn't obviously work, check with a small counterexample before committing (otherwise it's probably DP).
**Flow:** Sort or process elements in a specific order that exposes the greedy choice (e.g. by end time for intervals) -> at each step, take the option that's locally optimal (extend farthest reach, free the earliest resource, etc.) -> never revisit past decisions.
**Complexity:** Usually O(n log n) (dominated by sorting) or O(n), O(1) to O(n) space.

```java
// Jump Game II (min jumps to reach end)
int jump(int[] nums) {
    int jumps = 0, curEnd = 0, farthest = 0;
    for (int i = 0; i < nums.length - 1; i++) {
        farthest = Math.max(farthest, i + nums[i]);
        if (i == curEnd) { jumps++; curEnd = farthest; }
    }
    return jumps;
}

// Gas Station
int canCompleteCircuit(int[] gas, int[] cost) {
    int total = 0, tank = 0, start = 0;
    for (int i = 0; i < gas.length; i++) {
        int diff = gas[i] - cost[i];
        total += diff; tank += diff;
        if (tank < 0) { start = i + 1; tank = 0; }
    }
    return total >= 0 ? start : -1;
}

// Non-overlapping Intervals (min removals to make non-overlapping)
int eraseOverlapIntervals(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[1] - b[1]); // sort by end time
    int count = 0, prevEnd = Integer.MIN_VALUE;
    for (int[] iv : intervals) {
        if (iv[0] >= prevEnd) prevEnd = iv[1];
        else count++;
    }
    return count;
}

// Task Scheduler - schedule tasks with cooldown n, minimize total time (math on most frequent task)
int leastInterval(char[] tasks, int n) {
    int[] freq = new int[26];
    for (char t : tasks) freq[t - 'A']++;
    Arrays.sort(freq);
    int maxFreq = freq[25];
    int idleSlots = (maxFreq - 1) * n;
    for (int i = 24; i >= 0 && idleSlots > 0; i--) idleSlots -= Math.min(maxFreq - 1, freq[i]);
    idleSlots = Math.max(0, idleSlots);
    return tasks.length + idleSlots;
}

// Partition Labels - greedily extend partition to the last occurrence of every char seen so far
List<Integer> partitionLabels(String s) {
    int[] lastIdx = new int[26];
    for (int i = 0; i < s.length(); i++) lastIdx[s.charAt(i) - 'a'] = i;
    List<Integer> res = new ArrayList<>();
    int start = 0, end = 0;
    for (int i = 0; i < s.length(); i++) {
        end = Math.max(end, lastIdx[s.charAt(i) - 'a']);
        if (i == end) { res.add(end - start + 1); start = i + 1; }
    }
    return res;
}

// Candy - two greedy passes (left-to-right for increasing, right-to-left for decreasing)
int candy(int[] ratings) {
    int n = ratings.length;
    int[] candies = new int[n];
    Arrays.fill(candies, 1);
    for (int i = 1; i < n; i++) if (ratings[i] > ratings[i - 1]) candies[i] = candies[i - 1] + 1;
    for (int i = n - 2; i >= 0; i--) if (ratings[i] > ratings[i + 1]) candies[i] = Math.max(candies[i], candies[i + 1] + 1);
    int total = 0;
    for (int c : candies) total += c;
    return total;
}
```

**Key problems:** Jump Game I/II, Gas Station, Task Scheduler, Non-overlapping Intervals, Partition Labels, Candy, Minimum Number of Arrows to Burst Balloons.

---

## 25. Bit Manipulation

**When to use:** Space-optimized state, XOR tricks, subsets via bitmask, single-number problems.

**Intuition:** XOR is its own inverse (`x ^ x = 0`, `x ^ 0 = x`) so pairs cancel out, leaving only unpaired bits. Bitmasks let you represent a *set* of up to ~20-30 elements as a single int/long, enabling O(1) subset operations and dense DP states (bitmask DP).
**Flow:** Identify what property is preserved under XOR/AND/OR -- pair-cancellation (single number), toggling membership (subset iteration via `mask & -mask`), or compressing state (bitmask DP over subsets).
**Complexity:** O(n) for XOR-based scans, O(1) space; O(2ⁿ) for enumerating all subsets of an n-bit mask.

```java
// Single Number (XOR cancels pairs)
int singleNumber(int[] nums) {
    int result = 0;
    for (int n : nums) result ^= n;
    return result;
}

// Count set bits
int hammingWeight(int n) {
    int count = 0;
    while (n != 0) { n &= (n - 1); count++; } // clears lowest set bit
    return count;
}

// Check power of two
boolean isPowerOfTwo(int n) { return n > 0 && (n & (n - 1)) == 0; }

// Iterate all subsets of a bitmask
void iterateSubsets(int mask) {
    for (int sub = mask; sub > 0; sub = (sub - 1) & mask) {
        // process sub
    }
}

// Counting Bits - dp[i] = dp[i >> 1] + (i & 1), reuse previously computed results
int[] countBits(int n) {
    int[] dp = new int[n + 1];
    for (int i = 1; i <= n; i++) dp[i] = dp[i >> 1] + (i & 1);
    return dp;
}

// Sum of Two Integers without + or - (bitwise add: XOR for sum bits, AND<<1 for carry)
int getSum(int a, int b) {
    while (b != 0) {
        int carry = a & b;
        a = a ^ b;
        b = carry << 1;
    }
    return a;
}

// Reverse Bits (32-bit unsigned)
int reverseBits(int n) {
    int res = 0;
    for (int i = 0; i < 32; i++) {
        res = (res << 1) | (n & 1);
        n >>= 1;
    }
    return res;
}

// Maximum XOR of Two Numbers in an Array - build a bit-trie, greedily choose opposite bits
int findMaximumXOR(int[] nums) {
    int max = Arrays.stream(nums).max().getAsInt();
    int L = max == 0 ? 1 : Integer.toBinaryString(max).length();
    TrieBitNode root = new TrieBitNode();
    int best = 0;
    for (int num : nums) {
        TrieBitNode cur = root, curXor = root;
        int xorNum = 0;
        for (int i = L - 1; i >= 0; i--) {
            int bit = (num >> i) & 1;
            if (cur.children[bit] == null) cur.children[bit] = new TrieBitNode();
            cur = cur.children[bit];
            int toggled = 1 - bit;
            if (curXor.children[toggled] != null) { xorNum = (xorNum << 1) | 1; curXor = curXor.children[toggled]; }
            else { xorNum = xorNum << 1; curXor = curXor.children[bit]; }
        }
        best = Math.max(best, xorNum);
    }
    return best;
}
class TrieBitNode { TrieBitNode[] children = new TrieBitNode[2]; }
```

**Key problems:** Single Number I/II/III, Counting Bits, Sum of Two Integers, Reverse Bits, Subsets using bitmask, Maximum XOR of Two Numbers.

---

## 26. Intervals

**When to use:** Merging/inserting overlapping ranges, meeting room scheduling.

**Intuition:** Sorting intervals by start (or separately tracking start/end events) turns an O(n²) overlap-checking problem into a single linear scan, since overlaps can only occur between adjacent intervals once sorted.
**Flow:** Sort by start time -> scan and merge whenever the current interval's start is ≤ the last merged interval's end. For "min rooms needed", sort starts and ends separately and simulate a sweep: a room opens on each start, closes on each end.
**Complexity:** O(n log n) time (sorting dominates), O(n) space.

```java
// Merge Intervals
int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    List<int[]> res = new ArrayList<>();
    for (int[] iv : intervals) {
        if (!res.isEmpty() && iv[0] <= res.get(res.size() - 1)[1]) {
            res.get(res.size() - 1)[1] = Math.max(res.get(res.size() - 1)[1], iv[1]);
        } else res.add(iv);
    }
    return res.toArray(new int[0][]);
}

// Meeting Rooms II (min rooms needed)
int minMeetingRooms(int[][] intervals) {
    int n = intervals.length;
    int[] starts = new int[n], ends = new int[n];
    for (int i = 0; i < n; i++) { starts[i] = intervals[i][0]; ends[i] = intervals[i][1]; }
    Arrays.sort(starts); Arrays.sort(ends);
    int rooms = 0, maxRooms = 0, s = 0, e = 0;
    while (s < n) {
        if (starts[s] < ends[e]) { rooms++; s++; }
        else { rooms--; e++; }
        maxRooms = Math.max(maxRooms, rooms);
    }
    return maxRooms;
}

// Insert Interval - add a new interval into a sorted, non-overlapping list, merging as needed
int[][] insertInterval(int[][] intervals, int[] newInterval) {
    List<int[]> res = new ArrayList<>();
    int i = 0, n = intervals.length;
    while (i < n && intervals[i][1] < newInterval[0]) res.add(intervals[i++]); // before overlap
    int start = newInterval[0], end = newInterval[1];
    while (i < n && intervals[i][0] <= end) { // merge all overlapping
        start = Math.min(start, intervals[i][0]);
        end = Math.max(end, intervals[i][1]);
        i++;
    }
    res.add(new int[]{start, end});
    while (i < n) res.add(intervals[i++]); // after overlap
    return res.toArray(new int[0][]);
}

// Interval List Intersections - two-pointer merge of two sorted interval lists
int[][] intervalIntersection(int[][] a, int[][] b) {
    List<int[]> res = new ArrayList<>();
    int i = 0, j = 0;
    while (i < a.length && j < b.length) {
        int lo = Math.max(a[i][0], b[j][0]);
        int hi = Math.min(a[i][1], b[j][1]);
        if (lo <= hi) res.add(new int[]{lo, hi});
        if (a[i][1] < b[j][1]) i++; else j++;
    }
    return res.toArray(new int[0][]);
}
```

**Key problems:** Merge Intervals, Insert Interval, Meeting Rooms I/II, Non-overlapping Intervals, Employee Free Time, Interval List Intersections.

---

## 27. Matrix Patterns

**When to use:** Grid traversal, rotation, spiral order, in-place transformations, flood fill.

**Intuition:** A matrix is just a 2D array where DFS/BFS moves in 4 (or 8) directions instead of along a list — boundary bookkeeping (top/bottom/left/right shrinking, or visited tracking) is the main complexity.
**Flow:** For traversal-style problems, maintain shrinking boundaries (spiral) or use DFS/BFS with a visited set/in-place marking (flood fill, islands). For rotation, decompose into simpler operations (transpose + reverse rows = 90° rotation).
**Complexity:** O(rows × cols) time, O(1) extra space for in-place transforms, O(rows × cols) for visited tracking or BFS queue.

```java
// Spiral Order Traversal
List<Integer> spiralOrder(int[][] matrix) {
    List<Integer> res = new ArrayList<>();
    int top = 0, bottom = matrix.length - 1, left = 0, right = matrix[0].length - 1;
    while (top <= bottom && left <= right) {
        for (int j = left; j <= right; j++) res.add(matrix[top][j]);
        top++;
        for (int i = top; i <= bottom; i++) res.add(matrix[i][right]);
        right--;
        if (top <= bottom) { for (int j = right; j >= left; j--) res.add(matrix[bottom][j]); bottom--; }
        if (left <= right) { for (int i = bottom; i >= top; i--) res.add(matrix[i][left]); left++; }
    }
    return res;
}

// Rotate Image 90 degrees in-place (transpose + reverse rows)
void rotate(int[][] matrix) {
    int n = matrix.length;
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++) {
            int tmp = matrix[i][j]; matrix[i][j] = matrix[j][i]; matrix[j][i] = tmp;
        }
    for (int[] row : matrix) reverse(row, 0, row.length - 1);
}

// Flood Fill / Number of Islands (DFS)
int numIslands(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++)
        for (int j = 0; j < grid[0].length; j++)
            if (grid[i][j] == '1') { count++; floodFill(grid, i, j); }
    return count;
}
private void floodFill(char[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length || grid[i][j] != '1') return;
    grid[i][j] = '0';
    floodFill(grid, i + 1, j); floodFill(grid, i - 1, j);
    floodFill(grid, i, j + 1); floodFill(grid, i, j - 1);
}

// Set Matrix Zeroes - use first row/col as markers, in-place O(1) extra space
void setZeroes(int[][] matrix) {
    int rows = matrix.length, cols = matrix[0].length;
    boolean firstRowZero = false, firstColZero = false;
    for (int j = 0; j < cols; j++) if (matrix[0][j] == 0) firstRowZero = true;
    for (int i = 0; i < rows; i++) if (matrix[i][0] == 0) firstColZero = true;
    for (int i = 1; i < rows; i++)
        for (int j = 1; j < cols; j++)
            if (matrix[i][j] == 0) { matrix[i][0] = 0; matrix[0][j] = 0; }
    for (int i = 1; i < rows; i++)
        for (int j = 1; j < cols; j++)
            if (matrix[i][0] == 0 || matrix[0][j] == 0) matrix[i][j] = 0;
    if (firstRowZero) for (int j = 0; j < cols; j++) matrix[0][j] = 0;
    if (firstColZero) for (int i = 0; i < rows; i++) matrix[i][0] = 0;
}

// Surrounded Regions - mark 'O's connected to the border as safe, flip the rest to 'X'
void solveSurroundedRegions(char[][] board) {
    int rows = board.length, cols = board[0].length;
    for (int i = 0; i < rows; i++) {
        markSafe(board, i, 0);
        markSafe(board, i, cols - 1);
    }
    for (int j = 0; j < cols; j++) {
        markSafe(board, 0, j);
        markSafe(board, rows - 1, j);
    }
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++) {
            if (board[i][j] == 'O') board[i][j] = 'X';
            else if (board[i][j] == '#') board[i][j] = 'O';
        }
}
private void markSafe(char[][] board, int i, int j) {
    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length || board[i][j] != 'O') return;
    board[i][j] = '#'; // temp marker for "safe, connected to border"
    markSafe(board, i + 1, j); markSafe(board, i - 1, j);
    markSafe(board, i, j + 1); markSafe(board, i, j - 1);
}

// Max Area of Island - DFS flood fill, track area of each island
int maxAreaOfIsland(int[][] grid) {
    int best = 0;
    for (int i = 0; i < grid.length; i++)
        for (int j = 0; j < grid[0].length; j++)
            if (grid[i][j] == 1) best = Math.max(best, areaDfs(grid, i, j));
    return best;
}
private int areaDfs(int[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length || grid[i][j] != 1) return 0;
    grid[i][j] = 0;
    return 1 + areaDfs(grid, i + 1, j) + areaDfs(grid, i - 1, j) + areaDfs(grid, i, j + 1) + areaDfs(grid, i, j - 1);
}
```

**Key problems:** Spiral Matrix, Rotate Image, Set Matrix Zeroes, Number of Islands, Word Search, Surrounded Regions, Max Area of Island.

---

## 28. Fast & Slow Pointers (Cycle Detection)

**When to use:** Detect cycle, find cycle start, find middle of list, happy number style problems (Floyd's algorithm).

**Intuition:** If a cycle exists, a faster-moving pointer will eventually "lap" a slower one inside the cycle (like two runners on a circular track). If no cycle exists, the fast pointer simply reaches the end first.
**Flow:** Move `slow` by 1 step and `fast` by 2 steps each iteration; if they meet, a cycle exists. To find the cycle's start, reset one pointer to the head and advance both by 1 step until they meet again (a well-known mathematical property of the meeting point).
**Complexity:** O(n) time, O(1) space — the key advantage over a hashset-based cycle detector (O(n) space).

```java
// Detect Cycle in Linked List
boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}

// Find Cycle Start Node
ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            ListNode ptr = head;
            while (ptr != slow) { ptr = ptr.next; slow = slow.next; }
            return ptr;
        }
    }
    return null;
}

// Find Middle of Linked List
ListNode middleNode(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next; fast = fast.next.next;
    }
    return slow;
}

// Happy Number (cycle detection on value sequence)
boolean isHappy(int n) {
    int slow = n, fast = getNext(n);
    while (fast != 1 && slow != fast) {
        slow = getNext(slow);
        fast = getNext(getNext(fast));
    }
    return fast == 1;
}
private int getNext(int n) {
    int sum = 0;
    while (n > 0) { int d = n % 10; sum += d * d; n /= 10; }
    return sum;
}
```

**Key problems:** Linked List Cycle I/II, Find Middle of Linked List, Happy Number, Palindrome Linked List, Circular Array Loop.

---

## 29. Top-K / K-way Merge

**When to use:** Selecting top/smallest K items across multiple sorted sources, or from a single collection.

**Intuition:** When multiple sequences are individually sorted (matrix rows/columns, multiple lists), a min-heap seeded with one candidate per sequence always has the next-smallest overall element at the top — avoids merging everything at once.
**Flow:** Seed the heap with the first candidate from each sequence -> pop the minimum (that's the next smallest overall) -> push that sequence's next candidate -> repeat K times.
**Complexity:** O(k log(min(k, sources))) time typically, O(sources) space for the heap.

```java
// K-th Smallest Element in Sorted Matrix
int kthSmallest(int[][] matrix, int k) {
    int n = matrix.length;
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> matrix[a[0]][a[1]] - matrix[b[0]][b[1]]);
    for (int i = 0; i < n; i++) pq.offer(new int[]{i, 0});
    int result = -1;
    for (int i = 0; i < k; i++) {
        int[] cur = pq.poll();
        result = matrix[cur[0]][cur[1]];
        if (cur[1] + 1 < n) pq.offer(new int[]{cur[0], cur[1] + 1});
    }
    return result;
}

// Find K Pairs with Smallest Sums
List<List<Integer>> kSmallestPairs(int[] nums1, int[] nums2, int k) {
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> (nums1[a[0]] + nums2[a[1]]) - (nums1[b[0]] + nums2[b[1]]));
    for (int i = 0; i < Math.min(nums1.length, k); i++) pq.offer(new int[]{i, 0});
    List<List<Integer>> res = new ArrayList<>();
    while (k-- > 0 && !pq.isEmpty()) {
        int[] cur = pq.poll();
        res.add(Arrays.asList(nums1[cur[0]], nums2[cur[1]]));
        if (cur[1] + 1 < nums2.length) pq.offer(new int[]{cur[0], cur[1] + 1});
    }
    return res;
}
```

**Key problems:** Kth Smallest Element in Sorted Matrix, Merge K Sorted Lists, Find K Pairs with Smallest Sums, Kth Smallest Product of Two Sorted Arrays.

---

## 30. String Algorithms

**When to use:** Pattern matching, palindrome checks, anagram/permutation detection.

**Intuition:** Naive substring search re-checks overlapping characters after every mismatch (O(n·m)). KMP precomputes how much of the pattern's prefix is also a suffix (the LPS/failure array) so on a mismatch it can "skip ahead" without re-comparing already-matched characters.
**Flow:** Build the LPS array for the pattern once -> scan the text, advancing both pointers on a match; on a mismatch, jump the pattern pointer back using LPS instead of resetting to 0.
**Complexity:** O(n + m) time (n = text length, m = pattern length), O(m) space for the LPS array.

```java
// KMP Pattern Matching (build LPS array, then search)
int[] buildLPS(String pattern) {
    int n = pattern.length();
    int[] lps = new int[n];
    int len = 0;
    for (int i = 1; i < n; ) {
        if (pattern.charAt(i) == pattern.charAt(len)) { lps[i] = ++len; i++; }
        else if (len > 0) len = lps[len - 1];
        else { lps[i] = 0; i++; }
    }
    return lps;
}
int kmpSearch(String text, String pattern) {
    int[] lps = buildLPS(pattern);
    int i = 0, j = 0;
    while (i < text.length()) {
        if (text.charAt(i) == pattern.charAt(j)) { i++; j++; if (j == pattern.length()) return i - j; }
        else if (j > 0) j = lps[j - 1];
        else i++;
    }
    return -1;
}

// Expand Around Center - Longest Palindromic Substring
String longestPalindrome(String s) {
    int start = 0, maxLen = 0;
    for (int i = 0; i < s.length(); i++) {
        int len1 = expand(s, i, i);
        int len2 = expand(s, i, i + 1);
        int len = Math.max(len1, len2);
        if (len > maxLen) { maxLen = len; start = i - (len - 1) / 2; }
    }
    return s.substring(start, start + maxLen);
}
private int expand(String s, int lo, int hi) {
    while (lo >= 0 && hi < s.length() && s.charAt(lo) == s.charAt(hi)) { lo--; hi++; }
    return hi - lo - 1;
}

// Valid Anagram - frequency count comparison, O(n) time
boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] count = new int[26];
    for (char c : s.toCharArray()) count[c - 'a']++;
    for (char c : t.toCharArray()) if (--count[c - 'a'] < 0) return false;
    return true;
}

// Longest Common Prefix - compare characters column by column across all strings
String longestCommonPrefix(String[] strs) {
    if (strs.length == 0) return "";
    for (int i = 0; i < strs[0].length(); i++) {
        char c = strs[0].charAt(i);
        for (String s : strs) {
            if (i == s.length() || s.charAt(i) != c) return strs[0].substring(0, i);
        }
    }
    return strs[0];
}

// Palindromic Substrings Count - expand around every center (2n-1 centers), count valid palindromes
int countSubstrings(String s) {
    int count = 0;
    for (int center = 0; center < 2 * s.length() - 1; center++) {
        int lo = center / 2, hi = lo + center % 2;
        while (lo >= 0 && hi < s.length() && s.charAt(lo) == s.charAt(hi)) { count++; lo--; hi++; }
    }
    return count;
}
```

**Key problems:** Implement strStr() (KMP), Longest Palindromic Substring/Subsequence, Valid Anagram, Group Anagrams, Longest Common Prefix, Word Break, Palindromic Substrings Count.

---

## 31. Design Problems

**When to use:** Common OOP + data-structure combo design questions.

**Intuition:** Combine two data structures so each compensates for the other's weakness — e.g. a HashMap gives O(1) key lookup but no ordering; a doubly linked list gives O(1) reordering/removal but no O(1) lookup by key. Together: O(1) get/put with eviction ordering.
**Flow:** On `get`: look up via map, then move the node to the "most recently used" end of the list. On `put`: if at capacity, evict the "least recently used" end; insert/update the new node at the front, and record it in the map.
**Complexity:** O(1) time per operation, O(capacity) space.

```java
// LRU Cache using HashMap + Doubly Linked List
class LRUCache {
    class Node {
        int key, val;
        Node prev, next;
        Node(int k, int v) { key = k; val = v; }
    }
    private final Map<Integer, Node> map = new HashMap<>();
    private final int capacity;
    private final Node head = new Node(0, 0), tail = new Node(0, 0);

    LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail; tail.prev = head;
    }
    int get(int key) {
        if (!map.containsKey(key)) return -1;
        Node node = map.get(key);
        remove(node); insertAtFront(node);
        return node.val;
    }
    void put(int key, int value) {
        if (map.containsKey(key)) remove(map.get(key));
        else if (map.size() == capacity) {
            Node lru = tail.prev;
            remove(lru); map.remove(lru.key);
        }
        Node node = new Node(key, value);
        insertAtFront(node); map.put(key, node);
    }
    private void remove(Node node) { node.prev.next = node.next; node.next.prev = node.prev; }
    private void insertAtFront(Node node) {
        node.next = head.next; node.next.prev = node;
        head.next = node; node.prev = head;
    }
}

// LFU Cache - HashMap of key->node + HashMap of freq->doubly linked list (LRU per frequency bucket)
class LFUCache {
    class Node { int key, val, freq = 1; Node prev, next; Node(int k, int v) { key = k; val = v; } }
    class DList {
        Node head = new Node(0, 0), tail = new Node(0, 0);
        DList() { head.next = tail; tail.prev = head; }
        void addFront(Node n) { n.next = head.next; n.next.prev = n; head.next = n; n.prev = head; }
        void remove(Node n) { n.prev.next = n.next; n.next.prev = n.prev; }
        boolean isEmpty() { return head.next == tail; }
        Node removeLast() { Node n = tail.prev; remove(n); return n; }
    }
    private final int capacity;
    private int minFreq;
    private final Map<Integer, Node> keyToNode = new HashMap<>();
    private final Map<Integer, DList> freqToList = new HashMap<>();
    LFUCache(int capacity) { this.capacity = capacity; }
    int get(int key) {
        if (!keyToNode.containsKey(key)) return -1;
        Node node = keyToNode.get(key);
        update(node);
        return node.val;
    }
    void put(int key, int value) {
        if (capacity == 0) return;
        if (keyToNode.containsKey(key)) { Node n = keyToNode.get(key); n.val = value; update(n); return; }
        if (keyToNode.size() == capacity) {
            Node evict = freqToList.get(minFreq).removeLast();
            keyToNode.remove(evict.key);
        }
        Node node = new Node(key, value);
        keyToNode.put(key, node);
        freqToList.computeIfAbsent(1, k -> new DList()).addFront(node);
        minFreq = 1;
    }
    private void update(Node node) {
        freqToList.get(node.freq).remove(node);
        if (freqToList.get(node.freq).isEmpty() && minFreq == node.freq) minFreq++;
        node.freq++;
        freqToList.computeIfAbsent(node.freq, k -> new DList()).addFront(node);
    }
}

// Design Twitter - each user has a list of (timestamp, tweetId); merge followees' tweets via heap
class Twitter {
    private int timestamp = 0;
    private final Map<Integer, List<int[]>> tweets = new HashMap<>(); // user -> {time, tweetId}
    private final Map<Integer, Set<Integer>> following = new HashMap<>();
    void postTweet(int userId, int tweetId) {
        tweets.computeIfAbsent(userId, k -> new ArrayList<>()).add(new int[]{timestamp++, tweetId});
    }
    List<Integer> getNewsFeed(int userId) {
        PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> b[0] - a[0]); // {time, tweetId}
        Set<Integer> ids = new HashSet<>(following.getOrDefault(userId, new HashSet<>()));
        ids.add(userId);
        for (int uid : ids) {
            List<int[]> userTweets = tweets.getOrDefault(uid, Collections.emptyList());
            for (int i = userTweets.size() - 1; i >= Math.max(0, userTweets.size() - 10); i--) pq.offer(userTweets.get(i));
        }
        List<Integer> res = new ArrayList<>();
        while (!pq.isEmpty() && res.size() < 10) res.add(pq.poll()[1]);
        return res;
    }
    void follow(int followerId, int followeeId) { following.computeIfAbsent(followerId, k -> new HashSet<>()).add(followeeId); }
    void unfollow(int followerId, int followeeId) { if (following.containsKey(followerId)) following.get(followerId).remove(followeeId); }
}

// Insert Delete GetRandom O(1) - array + value->index map, swap-with-last for O(1) removal
class RandomizedSet {
    private final List<Integer> values = new ArrayList<>();
    private final Map<Integer, Integer> indexOf = new HashMap<>();
    private final Random rand = new Random();
    boolean insert(int val) {
        if (indexOf.containsKey(val)) return false;
        indexOf.put(val, values.size());
        values.add(val);
        return true;
    }
    boolean remove(int val) {
        if (!indexOf.containsKey(val)) return false;
        int idx = indexOf.get(val);
        int lastVal = values.get(values.size() - 1);
        values.set(idx, lastVal);
        indexOf.put(lastVal, idx);
        values.remove(values.size() - 1);
        indexOf.remove(val);
        return true;
    }
    int getRandom() { return values.get(rand.nextInt(values.size())); }
}

// Time Based Key-Value Store - map of key -> list of (timestamp, value), binary search for get
class TimeMap {
    private final Map<String, List<int[]>> store = new HashMap<>(); // value stored as index into a separate list
    private final Map<String, List<String>> values = new HashMap<>();
    void set(String key, String value, int timestamp) {
        store.computeIfAbsent(key, k -> new ArrayList<>()).add(new int[]{timestamp, 0});
        values.computeIfAbsent(key, k -> new ArrayList<>()).add(value);
    }
    String get(String key, int timestamp) {
        if (!store.containsKey(key)) return "";
        List<int[]> list = store.get(key);
        int lo = 0, hi = list.size() - 1, resIdx = -1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (list.get(mid)[0] <= timestamp) { resIdx = mid; lo = mid + 1; }
            else hi = mid - 1;
        }
        return resIdx == -1 ? "" : values.get(key).get(resIdx);
    }
}
```

**Key problems:** LRU Cache, LFU Cache, Design Twitter, Insert Delete GetRandom O(1), Design HashMap, Time-based Key-Value Store, Min Stack.

---

## 32. Recurrence Relations & Simulation Problems

**When to use:** Problems defined by a recursive rule on `n` (or state), often solvable via direct recurrence, simulation, or math derived from the recurrence (e.g., Josephus, Tower of Hanoi, tiling problems). Distinguish from DP: here the recurrence itself IS the answer — sometimes closed-form, sometimes just simulated with a data structure.

```java
// Josephus Problem — recursive recurrence:
// J(1) = 0
// J(n) = (J(n-1) + k) % n
// Returns 0-indexed position of the survivor among n people, eliminating every kth person.
int josephus(int n, int k) {
    if (n == 1) return 0;
    return (josephus(n - 1, k) + k) % n;
}
// Iterative (bottom-up) version — avoids recursion overhead, O(n) time, O(1) space
int josephusIterative(int n, int k) {
    int res = 0; // base case: J(1) = 0
    for (int i = 2; i <= n; i++) {
        res = (res + k) % i;
    }
    return res;
}

// Josephus via simulation with a circular queue — useful when you need the elimination ORDER,
// not just the survivor. O(n * k) time (or O(n log n) with a Fenwick/segment tree for large k).
List<Integer> josephusOrder(int n, int k) {
    List<Integer> people = new ArrayList<>();
    for (int i = 0; i < n; i++) people.add(i);
    List<Integer> eliminationOrder = new ArrayList<>();
    int idx = 0;
    while (!people.isEmpty()) {
        idx = (idx + k - 1) % people.size();
        eliminationOrder.add(people.remove(idx));
    }
    return eliminationOrder;
}

// Tower of Hanoi — recurrence: T(n) = 2*T(n-1) + 1, T(0) = 0 -> total moves = 2^n - 1
void hanoi(int n, char from, char to, char aux, List<String> moves) {
    if (n == 0) return;
    hanoi(n - 1, from, aux, to, moves);
    moves.add("Move disk " + n + " from " + from + " to " + to);
    hanoi(n - 1, aux, to, from, moves);
}

// Climbing Stairs / tiling recurrence (also DP, but view it as pure recurrence):
// f(n) = f(n-1) + f(n-2), f(0) = 1, f(1) = 1
int tilingWays(int n) {
    if (n <= 1) return 1;
    int a = 1, b = 1;
    for (int i = 2; i <= n; i++) {
        int c = a + b;
        a = b; b = c;
    }
    return b;
}
```

**How to spot these in an interview:**
- Problem describes a **circular elimination / counting-out game** -> Josephus recurrence `J(n) = (J(n-1) + k) % n`.
- Problem asks for **minimum moves to transform a recursive structure** (disks, recursive splits) -> derive `T(n)` recurrence from the base case + how the problem reduces to a smaller instance.
- If you only need one final answer (survivor, min moves, count) -> look for a **closed-form or O(n) recurrence** instead of full simulation.
- If you need the **full sequence of steps/order** -> simulate directly with a queue/list/circular structure, even if slower.

**Key problems:** Josephus Problem, Tower of Hanoi, Find the Winner of the Circular Game (LC 1823 — direct Josephus application), Elimination Game, N-th Tribonacci Number, Count Ways to reach Nth stair (tiling recurrences), Unique Binary Search Trees (Catalan number recurrence `C(n) = sum(C(i)*C(n-1-i))`).

---

## 33. Segment Tree / Fenwick Tree (Range Queries)

**When to use:** Range sum/min/max queries **with point or range updates** — i.e. when the array mutates between queries, so a plain prefix-sum array (O(n) rebuild per update) is too slow.

**Intuition:** A Fenwick Tree (Binary Indexed Tree) exploits binary representations of indices so every prefix sum can be decomposed into O(log n) precomputed chunks. A Segment Tree generalizes further to any associative range operation (sum, min, max, gcd) via a binary tree where each node stores the aggregate of a range, enabling O(log n) query and update.

**Flow (Fenwick Tree):** `update(i, delta)`: add `delta` at index `i`, then repeatedly jump to `i += i & (-i)` until out of bounds, updating each. `query(i)`: sum values by jumping `i -= i & (-i)` until 0.
**Flow (Segment Tree):** Build a tree bottom-up where leaves are array elements and internal nodes are the aggregate of their two children. `update`: change a leaf, then propagate the change up. `query(l, r)`: recursively combine only the nodes whose range fully lies inside `[l, r]`.

**Complexity:** Fenwick Tree — O(log n) update/query, O(n) space. Segment Tree — O(log n) update/query, O(n) build, O(n) space (O(4n) array-based implementation).

```java
// Fenwick Tree (Binary Indexed Tree) — 1-indexed internally
class FenwickTree {
    private final int[] tree;
    private final int n;
    FenwickTree(int n) { this.n = n; tree = new int[n + 1]; }

    void update(int i, int delta) { // i is 1-indexed
        for (; i <= n; i += i & (-i)) tree[i] += delta;
    }
    int query(int i) { // prefix sum [1, i]
        int sum = 0;
        for (; i > 0; i -= i & (-i)) sum += tree[i];
        return sum;
    }
    int rangeQuery(int l, int r) { return query(r) - query(l - 1); }
}

// Segment Tree for Range Sum Query with point updates
class SegmentTree {
    private final int[] tree;
    private final int n;
    SegmentTree(int[] nums) {
        n = nums.length;
        tree = new int[4 * n];
        build(nums, 0, 0, n - 1);
    }
    private void build(int[] nums, int node, int lo, int hi) {
        if (lo == hi) { tree[node] = nums[lo]; return; }
        int mid = (lo + hi) / 2;
        build(nums, 2 * node + 1, lo, mid);
        build(nums, 2 * node + 2, mid + 1, hi);
        tree[node] = tree[2 * node + 1] + tree[2 * node + 2];
    }
    void update(int idx, int val) { update(0, 0, n - 1, idx, val); }
    private void update(int node, int lo, int hi, int idx, int val) {
        if (lo == hi) { tree[node] = val; return; }
        int mid = (lo + hi) / 2;
        if (idx <= mid) update(2 * node + 1, lo, mid, idx, val);
        else update(2 * node + 2, mid + 1, hi, idx, val);
        tree[node] = tree[2 * node + 1] + tree[2 * node + 2];
    }
    int query(int l, int r) { return query(0, 0, n - 1, l, r); }
    private int query(int node, int lo, int hi, int l, int r) {
        if (r < lo || hi < l) return 0; // no overlap
        if (l <= lo && hi <= r) return tree[node]; // total overlap
        int mid = (lo + hi) / 2;
        return query(2 * node + 1, lo, mid, l, r) + query(2 * node + 2, mid + 1, hi, l, r);
    }
}
```

**Key problems:** Range Sum Query - Mutable, Count of Smaller Numbers After Self, Range Sum Query 2D - Mutable, Falling Squares, Number of Longest Increasing Subsequence (with BIT optimization).

---

## 34. Sweep Line Algorithm

**When to use:** Problems involving intervals/events on a line where you process events in sorted (usually time) order while maintaining running state — overlaps, skyline, calendar booking.

**Intuition:** Instead of comparing every pair of intervals (O(n²)), convert each interval into two "events" (start/+1 and end/-1), sort all events by position, and sweep through once — maintaining a running counter or active set gives the answer in O(n log n).

**Flow:** Convert intervals into `(position, type)` events -> sort events by position (ties usually broken by processing "end" before "start" or vice versa depending on inclusivity) -> sweep left to right, updating running state (active count, max overlap, merged range) as you consume each event.

**Complexity:** O(n log n) time (dominated by sorting events), O(n) space.

```java
// My Calendar / Max overlapping intervals — count max concurrent meetings at any point
int maxOverlap(int[][] intervals) {
    List<int[]> events = new ArrayList<>(); // {time, +1 or -1}
    for (int[] iv : intervals) {
        events.add(new int[]{iv[0], 1});
        events.add(new int[]{iv[1], -1});
    }
    // process ends (-1) before starts (+1) at the same time if intervals are [start, end)
    events.sort((a, b) -> a[0] != b[0] ? a[0] - b[0] : a[1] - b[1]);
    int active = 0, best = 0;
    for (int[] e : events) {
        active += e[1];
        best = Math.max(best, active);
    }
    return best;
}

// The Skyline Problem — sweep over building edges, track max height with a heap
List<int[]> getSkyline(int[][] buildings) {
    List<int[]> events = new ArrayList<>(); // {x, height (negative = start), }
    for (int[] b : buildings) {
        events.add(new int[]{b[0], -b[2]}); // start: negative height
        events.add(new int[]{b[1], b[2]});  // end: positive height
    }
    events.sort((a, b) -> a[0] != b[0] ? a[0] - b[0] : a[1] - b[1]);
    TreeMap<Integer, Integer> heights = new TreeMap<>(Collections.reverseOrder());
    heights.put(0, 1);
    List<int[]> res = new ArrayList<>();
    int prevMax = 0;
    for (int[] e : events) {
        int x = e[0], h = e[1];
        if (h < 0) heights.merge(-h, 1, Integer::sum);
        else {
            int cnt = heights.get(h);
            if (cnt == 1) heights.remove(h); else heights.put(h, cnt - 1);
        }
        int curMax = heights.firstKey();
        if (curMax != prevMax) { res.add(new int[]{x, curMax}); prevMax = curMax; }
    }
    return res;
}
```

**Key problems:** Meeting Rooms II, My Calendar I/II/III, The Skyline Problem, Merge Intervals, Employee Free Time, Minimum Number of Arrows to Burst Balloons.

---

## 35. Advanced Graph Algorithms (SCC, Bridges, Articulation Points)

**When to use:** Finding strongly connected components (cycles of mutual reachability in directed graphs), or finding critical edges/nodes whose removal disconnects an undirected graph.

**Intuition:** Tarjan's algorithm uses DFS discovery times and a "low-link" value (the earliest-discovered node reachable back from the subtree) to detect when a subtree can't reach anything outside itself — that's exactly when it forms an SCC (directed) or when an edge is a bridge / a node is an articulation point (undirected).

**Flow (Tarjan's SCC):** DFS while tracking `disc[]` (discovery time) and `low[]` (lowest discovery time reachable) and a stack of "on-stack" nodes; when `low[node] == disc[node]`, pop the stack to output one SCC.
**Flow (Bridges/Articulation Points):** Same DFS with `disc[]`/`low[]`; an edge `(u, v)` is a bridge if `low[v] > disc[u]` (v can't reach back past u without this edge); a node `u` is an articulation point if some child `v` has `low[v] >= disc[u]` (or `u` is the DFS root with 2+ children).

**Complexity:** O(V + E) time and space for all three (Tarjan's SCC, bridges, articulation points) — each is a single DFS pass.

```java
// Tarjan's Algorithm — Bridges in an undirected graph
int timer = 0;
void findBridges(int n, List<List<Integer>> graph) {
    int[] disc = new int[n], low = new int[n];
    boolean[] visited = new boolean[n];
    List<int[]> bridges = new ArrayList<>();
    for (int i = 0; i < n; i++) if (!visited[i]) dfsBridge(i, -1, graph, visited, disc, low, bridges);
}
private void dfsBridge(int u, int parent, List<List<Integer>> graph, boolean[] visited,
                        int[] disc, int[] low, List<int[]> bridges) {
    visited[u] = true;
    disc[u] = low[u] = timer++;
    for (int v : graph.get(u)) {
        if (v == parent) continue;
        if (!visited[v]) {
            dfsBridge(v, u, graph, visited, disc, low, bridges);
            low[u] = Math.min(low[u], low[v]);
            if (low[v] > disc[u]) bridges.add(new int[]{u, v}); // (u,v) is a bridge
        } else {
            low[u] = Math.min(low[u], disc[v]); // back edge
        }
    }
}
```

**Key problems:** Critical Connections in a Network (LC 1192 — bridges), Strongly Connected Components (Tarjan/Kosaraju), Number of Provinces, Redundant Connection II.

---

## 36. Math Essentials (GCD, Sieve, Fast Exponentiation)

**When to use:** Problems requiring number theory — reducing fractions, primality checks over a range, modular exponentiation (large powers under a modulus), combinatorics.

**Intuition:** GCD via Euclid's algorithm repeatedly reduces `(a, b) -> (b, a % b)`, converging in O(log(min(a,b))) steps since the remainder shrinks fast. Sieve of Eratosthenes marks composites in bulk (each prime crosses out its multiples) rather than testing each number for primality individually. Fast exponentiation exploits `a^n = (a^(n/2))^2` (or `* a` if odd) to compute powers in O(log n) multiplications instead of O(n).

**Flow (Sieve):** Assume all numbers are prime initially; for each number `p` starting at 2, if still marked prime, mark all multiples of `p` as composite; skip ahead — never re-check a composite.
**Flow (Fast Pow):** Recursively/iteratively halve the exponent, squaring the base each time, multiplying into the result only when the current bit of the exponent is 1.

**Complexity:** GCD O(log(min(a,b))); Sieve of Eratosthenes O(n log log n) time, O(n) space; Fast exponentiation O(log n) time, O(1) or O(log n) space (recursive).

```java
// GCD (Euclid's algorithm) and LCM
int gcd(int a, int b) { return b == 0 ? a : gcd(b, a % b); }
long lcm(int a, int b) { return (long) a / gcd(a, b) * b; }

// Sieve of Eratosthenes — all primes up to n
boolean[] sieve(int n) {
    boolean[] isComposite = new boolean[n + 1];
    for (int p = 2; (long) p * p <= n; p++) {
        if (!isComposite[p]) {
            for (int multiple = p * p; multiple <= n; multiple += p) isComposite[multiple] = true;
        }
    }
    return isComposite; // isComposite[i] == false && i >= 2  =>  i is prime
}

// Fast Exponentiation (a^n mod m)
long fastPow(long a, long n, long mod) {
    long result = 1;
    a %= mod;
    while (n > 0) {
        if ((n & 1) == 1) result = (result * a) % mod;
        a = (a * a) % mod;
        n >>= 1;
    }
    return result;
}
```

**Key problems:** Count Primes, Fraction to Recurring Decimal (GCD), Super Pow, Pow(x, n), Nth Digit, Basic combinatorics (nCr with modular inverse).

---

## 37. Game Theory / Minimax DP

**When to use:** Two-player turn-based games where both players play optimally — determine the winner, or the optimal score difference (Nim game, stone/coin picking games, predicting the winner).

**Intuition:** Model each game state as a DP state representing "the best score the current player can guarantee from here," assuming the opponent also plays optimally against them. The recurrence subtracts the opponent's best response from your own best move (a minimax formulation collapsed into a single DP value: your net advantage).

**Flow:** Define `dp[state] =` the maximum score difference (current player's score minus opponent's) achievable from `state`. At each state, try every legal move; the value of a move is `gain - dp[nextState]` (because after your move, the opponent becomes the "current player" of the recursive subproblem, and their optimal play should be subtracted from your gain). Take the max over all moves.

**Complexity:** Typically O(states × moves per state); e.g. O(n²) for interval-based coin games (`dp[i][j]` over subarray `[i, j]`), O(1) for closed-form games like Nim (`XOR of piles != 0` -> first player wins).

```java
// Predict the Winner / Stone Game — dp[i][j] = best score difference achievable from subarray [i, j]
boolean predictTheWinner(int[] nums) {
    int n = nums.length;
    int[][] dp = new int[n][n];
    for (int i = 0; i < n; i++) dp[i][i] = nums[i];
    for (int len = 2; len <= n; len++) {
        for (int i = 0; i + len - 1 < n; i++) {
            int j = i + len - 1;
            // pick nums[i] (opponent then plays optimally on [i+1, j])
            // or pick nums[j] (opponent then plays optimally on [i, j-1])
            dp[i][j] = Math.max(nums[i] - dp[i + 1][j], nums[j] - dp[i][j - 1]);
        }
    }
    return dp[0][n - 1] >= 0;
}

// Nim Game — closed form: first player loses iff all piles XOR to 0
boolean canWinNim(int n) { return n % 4 != 0; } // classic single-pile variant

boolean winnerXorNim(int[] piles) {
    int xor = 0;
    for (int p : piles) xor ^= p;
    return xor != 0; // true => first player wins
}
```

**Key problems:** Predict the Winner, Stone Game I-VII, Nim Game, Can I Win, Flip Game II, Optimal Strategy for a Game.

---

## 38. Complexity Cheatsheet

| Structure / Algorithm | Time | Space | Notes |
|---|---|---|---|
| Array access | O(1) | O(n) | |
| HashMap get/put | O(1) avg | O(n) | worst O(n) with collisions |
| Sorting (comparison) | O(n log n) | O(n) or O(log n) | Arrays.sort primitives = dual-pivot quicksort O(n log n); objects = Timsort (stable) |
| Binary Search | O(log n) | O(1) | requires sorted input |
| BFS/DFS (graph) | O(V + E) | O(V) | |
| Dijkstra (heap) | O(E log V) | O(V) | non-negative weights only |
| Bellman-Ford | O(V·E) | O(V) | handles negative weights |
| Union-Find (path compression + rank) | O(log V) amortized ~O(1) | O(V) | |
| Kadane's | O(n) | O(1) | |
| Sliding Window | O(n) | O(k) | |
| DP (typical) | O(n) to O(n²) | O(n) to O(n²) | often optimizable to O(n) space |
| Backtracking (subsets/perms) | O(2ⁿ) / O(n!) | O(n) | exponential — prune aggressively |
| KMP | O(n + m) | O(m) | |
| Segment Tree | O(log n) query/update | O(n) | O(4n) array-based implementation |
| Fenwick Tree (BIT) | O(log n) query/update | O(n) | simpler than segment tree, sum-only by default |
| Tarjan's (SCC/Bridges/Articulation) | O(V + E) | O(V) | single DFS pass |
| Sieve of Eratosthenes | O(n log log n) | O(n) | precompute primes up to n |
| Fast Exponentiation | O(log n) | O(1) / O(log n) | iterative vs recursive |
| Sweep Line | O(n log n) | O(n) | dominated by sorting events |

---

## Quick Pattern-Recognition Guide

- **"Contiguous subarray/substring with constraint"** -> Sliding Window
- **"Sorted array, find pair/triplet"** -> Two Pointers
- **"Find Kth largest/smallest / Top K"** -> Heap
- **"All permutations/combinations/subsets"** -> Backtracking
- **"Minimum/maximum path, count ways"** -> Dynamic Programming
- **"Next greater/smaller element"** -> Monotonic Stack
- **"Shortest path unweighted grid/graph"** -> BFS
- **"Detect cycle / connectivity"** -> Union-Find or DFS
- **"Range sum queries"** -> Prefix Sum
- **"Search on answer space (min/max satisfying condition)"** -> Binary Search on Answer
- **"Scheduling/interval overlap"** -> Sort + Greedy or Heap
- **"Prefix matching / autocomplete"** -> Trie
- **"Cycle in linked list / middle of list"** -> Fast & Slow Pointers
- **"Word ladder / dependency ordering"** -> Topological Sort / BFS
- **"Circular counting-out / elimination game"** -> Josephus Recurrence
- **"Recursive structure reduces to smaller identical subproblem (disks, splits)"** -> Recurrence Relation
- **"Range sum/min/max queries WITH updates in between"** -> Segment Tree / Fenwick Tree
- **"Overlapping intervals, skyline, max concurrent events"** -> Sweep Line
- **"Critical edges/nodes, strongly connected components"** -> Tarjan's (Bridges/Articulation/SCC)
- **"GCD/LCM, primes up to N, large power under a modulus"** -> Math Essentials
- **"Two players alternate turns, both play optimally"** -> Game Theory / Minimax DP

---

## Suggested Revision Order (time-boxed)

1. Arrays, Two Pointers, Sliding Window, Prefix Sum (core building blocks)
2. Binary Search + Kadane's
3. Linked List + Fast/Slow Pointers
4. Stack, Monotonic Stack, Queue/Deque
5. Hashing patterns
6. Trees + Tries + Heaps
7. Graphs (BFS/DFS, Union-Find, Topo Sort, Dijkstra)
8. Backtracking
9. Dynamic Programming (biggest ROI — practice 15-20 classic problems)
10. Greedy, Bit Manipulation, Intervals, Matrix, Design problems
11. Recurrence Relations & Simulation, Game Theory (quick wins, low volume of problems)
12. Segment Tree/Fenwick Tree, Sweep Line, Advanced Graph (SCC/Bridges), Math Essentials (only if time remains — lower frequency in most OAs/interviews but common in harder rounds)
