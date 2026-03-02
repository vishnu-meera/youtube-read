When you're sifting through millions of numbers, desperately seeking that perfectly "balanced" subsequence, things can get tricky. But what if "balanced" doesn't just mean an equal total count of evens and odds, but an equal count of *distinct* evens and odds? That subtle shift turns a solvable puzzle into a truly hard one, pushing you beyond simple tricks and into the realm of advanced data structures.

Today, we're tackling precisely that: finding the **longest subarray where the number of distinct even numbers equals the number of distinct odd numbers**. With an input array that can stretch to 100,000 elements, any brute-force approach iterating through all possible subarrays (an O(N^2) nightmare) simply won't cut it. We need something smarter, something that can adapt on the fly.

## The Problem with Simple Thinking (and Why It Fails)

Your first thought might be to simplify the problem: assign `+1` to even numbers and `-1` to odd numbers. Then, if a subarray's sum is zero, it must be balanced, right? You could use a **prefix sum** technique, storing the first index where each sum appears in a hash map. If you encounter a sum you've seen before, the segment between the current index and the previously stored index represents a balanced subarray.

But here's the catch: the problem explicitly states "**distinct** even numbers" and "**distinct** odd numbers."

Consider the array `[1, 2, 3, 2, 5]`:

*   A simple `+1` for evens, `-1` for odds might give `[-1, +1, -1, +1, -1]`.
*   The subarray `[1, 2, 3]` would sum to `-1 + 1 - 1 = -1`.
*   However, examining the *distinct* counts:
    *   Distinct odds: `{1, 3}` (count = 2)
    *   Distinct evens: `{2}` (count = 1)
    *   This subarray is *not* balanced based on distinct counts.

The issue is that a number appearing multiple times (like `2` in `[1, 2, 3, 2, 5]`) should only contribute to the "distinct" count once within any given subarray. A naive prefix sum approach doesn't handle this dynamic "deduplication" as the subarray boundaries shift. We need a mechanism to:

1.  Track the *last seen index* of each number.
2.  When a number appears again, its *previous contribution* to the distinct count must be nullified, and its new contribution at the current index must be established.
3.  All these updates need to happen efficiently across potentially large ranges.

This complexity points us toward a more advanced data structure: the **Segment Tree with Lazy Propagation**.

## The Segment Tree: Your Dynamic Balance Keeper

Imagine you're trying to count votes in an election, but voters keep changing their minds, and you only care about unique voters in certain regions. A simple tally won't work. You need a dynamic system that can:

1.  Record new votes.
2.  Invalidate old votes from the same person if they vote again (within a specific region).
3.  Quickly tell you if any region has an equal number of votes for two parties.

A **Segment Tree** with **Lazy Propagation** acts like that dynamic system. It's a tree-based data structure that allows for efficient querying over intervals or segments and also supports updating elements in the underlying array. The "lazy" part means updates aren't immediately applied to every element in a range; they're deferred and pushed down the tree only when necessary.

For our problem, the segment tree will maintain an implicit array where each position represents an index in our original `nums` array. The values stored at these positions (and propagated up the tree) will reflect the running balance of distinct even/odd numbers.

## Building the Balance Sheet: How the Segment Tree Works

The core idea is to transform our problem into one that the segment tree can efficiently manage: a **range sum query with point updates and dynamic value changes**.

1.  **Transformation of Numbers:**
    *   For each number `num` in the input array `nums`:
        *   If `num` is even, its "raw" value is `+1`.
        *   If `num` is odd, its "raw" value is `-1`.
    *   This raw value represents its potential contribution to the `(distinct evens - distinct odds)` balance.

2.  **Segment Tree Node Structure:**
    Each node in our segment tree will represent a range `[L, R]` from the original `nums` array (or rather, the transformed values). It stores:
    *   `lazy_tag`: An integer representing a pending update value to be pushed down.
    *   `mx`: The maximum prefix sum within this node's segment, relative to its own left boundary.
    *   `mn`: The minimum prefix sum within this node's segment, relative to its own left boundary.

3.  **Core Operations:**
    *   `update(node, l, r)`: This helper function applies any `lazy_tag` from the `node` to its children. If `lazy_tag` is non-zero, it updates `mx`, `mn`, and the children's `lazy_tag`s, then clears the `node`'s `lazy_tag`. This prevents unnecessary deep traversals.
    *   `change_range(node, val, change_L, change_R, l, r)`: This is where we dynamically update the segment tree. It adds `val` to the range `[change_L, change_R]`.
        *   If the current node's range `[l, r]` is entirely outside `[change_L, change_R]`, we do nothing.
        *   If `[l, r]` is entirely within `[change_L, change_R]`, we update `mx[node]`, `mn[node]`, and `lazy[node]` by adding `val`.
        *   Otherwise (partial overlap), we first `update` the current node (push down lazy changes), then recursively call `change_range` for its left and right children. Finally, we update `mx[node]` and `mn[node]` by combining the results from its children.
    *   `left_zero(node, l, r)`: This function searches for the leftmost index within the segment tree's current range `[l, r]` where the accumulated balance (prefix sum) is 0.
        *   It first `update`s the current node.
        *   If `l == r` (leaf node): if `mn[node]` is `0`, return `l`. Otherwise, return `-1` (not found).
        *   Otherwise (non-leaf):
            *   Check the left child: if `mn[left_child]` (after lazy updates) is less than or equal to `0`, recurse left.
            *   If not, check the right child: if `mn[right_child]` is less than or equal to `0`, recurse right.
            *   If neither child contains a `0`, return `-1`.

4.  **The Sliding Window Dance (Main Logic):**
    We iterate through the `nums` array with a right pointer `r`. A `prev` hash map stores the last seen index for each number.

    *   For each `curr_num = nums[r]`:
        *   Determine its raw `value` (`+1` for even, `-1` for odd).
        *   **Handle Duplicates (Removal):** If `curr_num` was seen before (`curr_num in prev`):
            *   We must "undo" its old contribution. Call `st.change_range(1, -value, 0, prev[curr_num], 0, n - 1)`. This subtracts the `value` from all prefix sums up to the previous occurrence, effectively removing its impact on distinct counts in that range.
        *   **Add Current Contribution:** Call `st.change_range(1, value, 0, r, 0, n - 1)`. This adds the `value` to all prefix sums from index `0` up to `r`, incorporating its new contribution.
        *   Update `prev[curr_num] = r` to record its most recent position.
        *   **Find Longest Balanced Subarray:** Call `st.left_zero(1, 0, n - 1)` to find the leftmost index `L` where the prefix sum is 0. This `L` signifies the start of a balanced subarray ending at `r`.
        *   Update `res = max(res, r - L + 1)`.

### Complexity Analysis

*   **Time Complexity: O(N log N)**
    *   We iterate through the input array `nums` once (N iterations).
    *   Inside the loop, each `change_range` operation (for removing old contributions and adding new ones) and each `left_zero` query takes O(log N) time because it traverses the height of the segment tree.
    *   Total: N * O(log N) = O(N log N).
*   **Space Complexity: O(N)**
    *   The segment tree (including `lazy`, `mx`, `mn` arrays) typically requires ~4N space for N elements.
    *   The `prev` hash map stores at most N unique elements.
    *   Total: O(N).

## Putting It All Together (Code)

```python
class SegmentTree:
    def __init__(self, n):
        # lazy[i]: pending updates for node i
        self.lazy = [0] * (4 * n) 
        # mx[i]: maximum prefix sum in the range of node i
        self.mx = [0] * (4 * n) 
        # mn[i]: minimum prefix sum in the range of node i
        self.mn = [0] * (4 * n) 

    def update(self, node, l, r):
        # If there are pending lazy updates for this node
        if self.lazy[node] != 0:
            # Apply lazy tag to current node's min/max sums
            self.mx[node] += self.lazy[node]
            self.mn[node] += self.lazy[node]

            # If not a leaf node, push lazy tag to children
            if l != r:
                self.lazy[node * 2] += self.lazy[node]
                self.lazy[node * 2 + 1] += self.lazy[node]
            
            # Clear current node's lazy tag
            self.lazy[node] = 0

    def change_range(self, node, val, change_L, change_R, l, r):
        # Update current node with any pending lazy tags
        self.update(node, l, r)

        # No overlap or invalid range
        if l > change_R or r < change_L or l > r:
            return

        # Full overlap: apply change directly to this node
        if change_L <= l and r <= change_R:
            self.lazy[node] += val
            self.update(node, l, r) # Apply this new lazy tag immediately
            return

        # Partial overlap: recurse on children
        mid = l + (r - l) // 2
        self.change_range(node * 2, val, change_L, change_R, l, mid)
        self.change_range(node * 2 + 1, val, change_L, change_R, mid + 1, r)

        # After children are updated, re-calculate current node's min/max from children
        self.mx[node] = max(self.mx[node * 2], self.mx[node * 2 + 1])
        self.mn[node] = min(self.mn[node * 2], self.mn[node * 2 + 1])

    def left_zero(self, node, l, r):
        # Update current node with any pending lazy tags
        self.update(node, l, r)

        # If the minimum value in this segment is positive, no zero can be found here
        if self.mn[node] > 0:
            return -1

        # If it's a leaf node and its value is 0, return its index
        if l == r:
            if self.mn[node] == 0:
                return l
            return -1 # No zero found at this leaf

        mid = l + (r - l) // 2
        
        # Check if the left child's minimum value (considering its lazy tag) is <= 0
        # If yes, a zero might be in the left child, so prioritize searching there
        self.update(node * 2, l, mid) # Update left child for accurate min value check
        if self.mn[node * 2] <= 0:
            result = self.left_zero(node * 2, l, mid)
            if result != -1: # If zero found in left child, return it
                return result

        # If not found in left child, check the right child
        self.update(node * 2 + 1, mid + 1, r) # Update right child for accurate min value check
        if self.mn[node * 2 + 1] <= 0:
            return self.left_zero(node * 2 + 1, mid + 1, r)

        return -1 # No zero found in either child

class Solution:
    def longestBalanced(self, nums: List[int]) -> int:
        n = len(nums)
        res = 0
        
        # Stores the last seen index for each number encountered so far
        # Used to handle distinct elements
        prev = {} 

        # Initialize Segment Tree with size n
        st = SegmentTree(n)

        # Iterate through the array with a right pointer 'r'
        for r in range(n):
            curr = nums[r]
            
            # Determine base value: +1 for even, -1 for odd
            val = 1 if curr % 2 == 0 else -1

            # If the current number has been seen before in the current window:
            if curr in prev:
                # Remove its *old* contribution to the prefix sums
                # We add -val to the range from index 0 up to its previous occurrence.
                # This effectively 'nullifies' the old contribution for balance calculation.
                st.change_range(1, -val, 0, prev[curr], 0, n - 1)
            
            # Add the current number's contribution to the prefix sums
            # We add 'val' to the range from index 0 up to the current right pointer 'r'.
            st.change_range(1, val, 0, r, 0, n - 1)
            
            # Update the last seen index for 'curr'
            prev[curr] = r

            # Find the leftmost index 'L' where the current prefix sum (balance) is 0.
            # This indicates a balanced subarray from L to r.
            L = st.left_zero(1, 0, n - 1)

            # If a balanced subarray starting at L was found (L != -1),
            # calculate its length and update the maximum result.
            if L != -1:
                res = max(res, r - L + 1)
        
        return res

```

## Conclusion

The "Longest Balanced Subarray II" problem showcases how a seemingly simple modification (counting *distinct* elements) can dramatically increase complexity. While a naive approach with simple prefix sums quickly fails, the combination of a **Segment Tree** and **Lazy Propagation** provides an elegant and efficient solution. By dynamically updating prefix sums to reflect only the contributions of the most recent occurrences of distinct elements, we can transform a tricky problem into one solvable within optimal time and space constraints. If you found this journey useful, remember to drop a like and subscribe for more deep dives into complex algorithms!