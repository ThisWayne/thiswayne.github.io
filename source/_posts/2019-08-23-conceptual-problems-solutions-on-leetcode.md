---
title: Conceptual Problems/Solutions on Leetcode
date: 2019-08-23 15:01:26
urlname: conceptual-problems-solutions-on-leetcode
tags:
  - Algorithms
  - Leetcode
  - Conceptual Problems
  - Coding Tips
  - Code Templates
---

This article is for helping someone who wants to practice algorithms on leetcode, and already has some basic data structure knowledge like hash table, stack, queue, etc.

After solving some problems, I think some conceptual problems/solutions that once are understood, other problems are just the same. And without knowing these solutions, some problems are just not so easy to come up with a good solution.

<!-- more -->

## Frequently Used Coding Tips, General Templates & Ideas

### 1. getting a bigger/smaller number

instead of using if/else:

```csharp
// C#
Math.Max(a, b)
Math.Min(a, b)
```

```python
# Python
max(a, b)
min(a, b)
```

### 2. check if two variables are null

If a or b is null, return false.  
If both of them are null, return true.  
Otherwise continue.  
This snippet of code is so concise and beautiful, it gets rid of so much if/else.

```csharp
// C#
if(a == null || b == null) {
    return a == b;
}
```

```python
# Python
if a is None or b is None:
    return a == b
```

### 3. BFS(Breath First Search) template

```csharp
// C#
Queue<int> queue = new Queue<int>(); // choose Stack or Queue base on the problem
while(queue.Count != 0) {
  int levelSize = queue.Count;
  while(levelSize-- > 0) {
      // do something like enqueue, dequeue...
  }
}
```

```python
# Python
# choose Stack or Queue base on the problem
from collections import deque

queue = deque()
while len(queue) != 0:
    level_size = len(queue)
    while level_size > 0:
        # queue.append(x), queue.popleft()
        level_size -= 1

stack = []
while len(stack) != 0:
    level_size = len(stack)
    while level_size > 0:
        # stack.pop(), stack.append(x)
        level_size -= 1
```

### 4. binary tree solution template

When visiting a node, usually there are three things we can do, base on the problem to design the order of steps.

1. visit a node, do something(like adding the node's value into the result collection)
2. visit the left child node
3. visit the right child node

### 5. iterate an array over and over again

Iterate each item in an array from start to end again and again without worrying the index out of bound problem at the end of the array.

```csharp
// C#
index = (index + 1) % length;
index = (index + length - 1) % length; // iterate from end to start
```

```python
# Python
length = len(arr)
index = (index + 1) % length
index = (index + length - 1) % length; # iterate from end to start
```

### 6. sorted data

Searching for something and the data is sorted or some math problem which is from 1 to N. Binary Search!

### 7. int overflow in binary search

Avoid integer overflow when adding two large integer in binary search

```csharp
// C#
// don't do this when left + right will overflow
mid = (left + right) / 2
// do this
mid = left + (right - left) / 2
// proof:
// mid = (left + right) / 2 = left / 2 +  right / 2 = left - left / 2 + right / 2 = left + (right - left) / 2
```

### 8. recusive solution might not be space complexity O(1)

In the Discuss page of the problem, some solution's code might looks concise by using recursion, but keep in mind that the stack takes memory space still, so usually the space complexity is not O(1).

## Bit Manupliation

### [191. Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/)

```csharp
// C#
// The basic solution is to iterate over bits by bit shifting (n >> 1) then check if rightmost bit is one (n & 1).
// The trick here is that n & (n - 1) can eliminate the rightmost 1 in the n's binary bits, so it doesn't have to iterate all bits of n.
// Now the time complexity is depends on how many of 1 in n, not how many bits of n.
public class Solution {
    public int HammingWeight(uint n) {
        int count = 0;
        while(n != 0) {
            n = n & (n - 1);
            count++;
        }
        return count;
    }
}
```

```python
# Python
class Solution:
    def hammingWeight(self, n: int) -> int:
        count = 0
        while n != 0:
            n = n & (n - 1)
            count += 1
        return count
```

## Array

### [448. Find All Numbers Disappeared in an Array](https://leetcode.com/problems/find-all-numbers-disappeared-in-an-array/)

```csharp
// C#
// If modifying the iput array is allowed, we can use some trick to manipulate the array to solve some problem without extra space.
// Like using positive/negative in the array to mark visited
public class Solution {
    public IList<int> FindDisappearedNumbers(int[] nums) {
        for(int i = 0; i < nums.Length; i++) {
            int index = Math.Abs(nums[i]) - 1;
            if(nums[index] > 0) {
                nums[index] = -nums[index];
            }
        }
        IList<int> result = new List<int>();
        for(int i = 0; i < nums.Length; i++) {
            if(nums[i] > 0) {
                result.Add(i + 1);
            }
        }
        return result;
    }
}
```

```python
# Python
class Solution:
    def findDisappearedNumbers(self, nums: List[int]) -> List[int]:
        for i in range(0, len(nums)):
            index = abs(nums[i]) - 1
            if nums[index] > 0:
                nums[index] = -nums[index]

        result = []
        for i in range(0, len(nums)):
            if nums[i] > 0:
                result.append(i + 1)
        return result
```

## Key/Value

### [1. Two Sum](https://leetcode.com/problems/two-sum/)

```csharp
// C#
// using key as distance to target, finding if there is a num equals to the distance to target
public class Solution {
    public int[] TwoSum(int[] nums, int target) {
        Dictionary<int , int> dict = new Dictionary<int, int>();
        for(int i = 0; i < nums.Length; i++) {
            if(dict.ContainsKey(nums[i])) {
                return new int[]{dict[nums[i]], i};
            } else {
                dict[target - nums[i]] = i;
            }
        }
        return new int[2];
    }
}
```

```python
class Solution:
    def twoSum(self, nums: List[int], target: int) -> List[int]:
        dict = {}
        for (i, num) in enumerate(nums):
            if num in dict:
                return [dict[num], i]
            else:
                dict[target - num] = i
        return []
```

## Binary Search

### [704. Binary Search](https://leetcode.com/problems/binary-search/)

```csharp
// C#
public class Solution {
    public int Search(int[] nums, int target) {
        int n = nums.Length;
        if(n == 0) return -1;

        int left = 0;
        int right = n - 1;
        while(left <= right){
            int mid = left + (right - left) / 2;
            if(nums[mid] < target) left = mid + 1;
            else if (nums[mid] == target) return mid;
            else right = mid - 1;
        }

        return -1;
    }
}
```

```python
class Solution:
    def search(self, nums: List[int], target: int) -> int:
        left, right = 0, len(nums) - 1
        while left <= right:
            mid = left + (right - left) // 2
            num = nums[mid]
            if num == target: return mid
            elif num > target: right = mid - 1
            else: left = mid + 1
        return -1
```

### [744. Find Smallest Letter Greater Than Target](https://leetcode.com/problems/find-smallest-letter-greater-than-target/)

```csharp
// C#
public class Solution {
    public char NextGreatestLetter(char[] letters, char target) {
        int n = letters.Length;
        if(target >= letters[n - 1]) return letters[0];

        int left = 0;
        int right = n - 1;
        while(left < right){
            int mid = left + (right - left) / 2;
            if(letters[mid] <= target) left = mid + 1;
            else right = mid;
        }

        return letters[right];
    }
}
```

```python
class Solution:
    def nextGreatestLetter(self, letters: List[str], target: str) -> str:
        left, right = 0, len(letters) - 1
        if target >= letters[right]: return letters[0]

        while left < right:
            mid = left + (right - left) // 2
            if letters[mid] <= target: left = mid + 1
            else: right = mid
        return letters[right]
```

## Breath First Search

### [111. Minimum Depth of Binary Tree](https://leetcode.com/problems/minimum-depth-of-binary-tree/)

```csharp
// C#
public class Solution {
    public int MinDepth(TreeNode root) {
        if(root == null) return 0;
        int currDepth = 0;
        Queue<TreeNode> nodes = new Queue<TreeNode>();
        nodes.Enqueue(root);
        while(nodes.Count != 0) {
            int levelSize = nodes.Count;
            currDepth++;
            while(levelSize-- > 0) {
                TreeNode node = nodes.Dequeue();
                if(node.left == null && node.right == null) {
                    return currDepth;
                }
                if(node.left != null) nodes.Enqueue(node.left);
                if(node.right != null) nodes.Enqueue(node.right);
            }
        }
        return currDepth;
    }
}
```

```python
# Definition for a binary tree node.
# class TreeNode:
#     def __init__(self, val=0, left=None, right=None):
#         self.val = val
#         self.left = left
#         self.right = right
from collections import deque

class Solution:
    def minDepth(self, root: Optional[TreeNode]) -> int:
        if root is None: return 0

        curr_depth = 0
        nodes = deque()
        nodes.append(root)

        while len(nodes) != 0:
            level_size = len(nodes)
            curr_depth += 1

            for i in range(level_size):
                node = nodes.popleft()
                if node:
                    if node.left is None and node.right is None:
                        return curr_depth
                    nodes.append(node.left)
                    nodes.append(node.right)
        return curr_depth
```

## Linked List

### [83. Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list/)

```csharp
// C#
public class Solution {
    public ListNode DeleteDuplicates(ListNode head) {
        if(head == null) return head;

        ListNode curr = head;
        while(curr.next != null) {
            if(curr.val == curr.next.val) {
                curr.next = curr.next.next;
            } else {
                curr = curr.next;
            }
        }

        return head;
    }
}
```

```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, val=0, next=None):
#         self.val = val
#         self.next = next
class Solution:
    def deleteDuplicates(self, head: Optional[ListNode]) -> Optional[ListNode]:
        if head is None: return head

        curr = head
        while curr.next:
            if curr.val == curr.next.val:
                curr.next = curr.next.next
            else:
                curr = curr.next

        return head
```

### [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/)

```csharp
// C#
// Floyd's Tortoise and Hare
public class Solution {
    public bool HasCycle(ListNode head) {
        if (head == null || head.next == null) {
            return false;
        }

        ListNode slow = head;
        ListNode fast = head.next;
        while(slow != fast) {
            if(fast == null || fast.next == null) {
                return false;
            }
            slow = slow.next;
            fast = fast.next.next;
        }
        return true;
    }
}
```

```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution:
    def hasCycle(self, head: Optional[ListNode]) -> bool:
        if head is None or head.next is None:
            return False

        slow = head
        fast = head.next

        while slow != fast:
            if fast is None or fast.next is None:
                return False
            slow = slow.next
            fast = fast.next.next
        return True
```

### [142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/)

Using Floyd's cycle detection algoritm to solve this problem only takes O(1) space complexity.
[Why Floyd's cycle detection algorithm works? Detecting loop in a linked list.](https://www.youtube.com/watch?v=LUm2ABqAs1w)

```csharp
// C#
public class Solution {
    public ListNode DetectCycle(ListNode head) {
        if (head == null) return null;

        ListNode slow = head;
        ListNode fast = head;
        while(fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if(slow == fast) {
                fast = head;
                while(slow != fast) {
                    slow = slow.next;
                    fast = fast.next;
                }
                return fast;
            }
        }
        return null;
    }
}
```

```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution:
    def detectCycle(self, head: ListNode) -> ListNode:
        if head is None or head.next is None:
            return None

        slow = head
        fast = head

        while fast is not None and fast.next is not None:
            slow = slow.next
            fast = fast.next.next
            if slow == fast:
                fast = head
                while slow != fast:
                    slow = slow.next
                    fast = fast.next
                return fast
        return None
```

### [203. Remove Linked List Elements](https://leetcode.com/problems/remove-linked-list-elements/)

```csharp
// C#
// using a dummy head to eliminate those null edge cases
public class Solution {
    public ListNode RemoveElements(ListNode head, int val) {
        ListNode dummy = new ListNode(0);
        dummy.next = head;
        ListNode curr = dummy;
        while(curr != null && curr.next != null) {
            if(curr.next.val == val) {
                curr.next = curr.next.next;
            } else {
                curr = curr.next;
            }
        }
        return dummy.next;
    }
}
```

```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, val=0, next=None):
#         self.val = val
#         self.next = next
class Solution:
    def removeElements(self, head: Optional[ListNode], val: int) -> Optional[ListNode]:
        dummy_head = ListNode(0)
        dummy_head.next = head

        curr = dummy_head
        while curr is not None and curr.next is not None:
            if curr.next.val == val:
                curr.next = curr.next.next
            else:
                curr = curr.next
        return dummy_head.next
```

### [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)

```csharp
// C#
public class Solution {
    public ListNode ReverseList(ListNode head) {
        ListNode prev = null;
        ListNode curr = head;
        while(curr != null) {
            ListNode temp = curr.next;
            curr.next = prev;
            prev = curr;
            curr = temp;
        }
        return prev;
    }
}
```

```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, val=0, next=None):
#         self.val = val
#         self.next = next
class Solution:
    def reverseList(self, head: Optional[ListNode]) -> Optional[ListNode]:
        prev = None
        curr = head
        while curr is not None:
            temp = curr.next
            curr.next = prev
            prev = curr
            curr = temp
        return prev
```

## Sliding Window

[Sliding Window algorithm template to solve all the Leetcode substring search problem.](https://leetcode.com/problems/find-all-anagrams-in-a-string/discuss/92007/Sliding-Window-algorithm-template-to-solve-all-the-Leetcode-substring-search-problem.)

### [438. Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string/)

### [567. Permutation in String](https://leetcode.com/problems/permutation-in-string/)

## Prefix Sum

### [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/)

### [437. Path Sum III](https://leetcode.com/problems/path-sum-iii/)

## Permutation, Backtracking

### [backtracking solution template](<https://leetcode.com/problems/permutations/discuss/18239/A-general-approach-to-backtracking-questions-in-Java-(Subsets-Permutations-Combination-Sum-Palindrome-Partioning)>)

### [46. Permutations](https://leetcode.com/problems/permutations/)

```csharp
// C#
public class Solution {
    public IList<IList<int>> Permute(int[] nums) {
        IList<IList<int>> result = new List<IList<int>>();
        // for some problems, it might be good to sort first
        backtracking(nums, 0, result);
        return result;
    }

    private void backtracking(int[] nums, int startIndex, IList<IList<int>> result) {
        if(startIndex == nums.Length - 1) {
            result.Add(new List<int>(nums));
            return;
        }

        for(int i = startIndex; i < nums.Length; i++) {
            // some method to mark the current index is visited
            swap(nums, startIndex, i);
            backtracking(nums, startIndex + 1, result);
            // some method to erase the current index is visited
            swap(nums, startIndex, i);
        }
    }

    private void swap(int[] nums, int indexA, int indexB) {
        int temp = nums[indexA];
        nums[indexA] = nums[indexB];
        nums[indexB] = temp;
    }
}
```

```python
class Solution:
    def permute(self, nums: List[int]) -> List[List[int]]:
        result = []
        self.backtracking(nums, 0, result)
        return result

    def backtracking(self, nums: List[int], start_index: int, result: List[List[int]]):
        if start_index == len(nums):
            result.append(list(nums))
            return

        for i in range(start_index, len(nums)):
            self.swap(nums, start_index, i)
            self.backtracking(nums, start_index + 1, result)
            self.swap(nums, start_index, i)

    def swap(self, nums: List[int], index_a: int, index_b: int):
        temp = nums[index_a]
        nums[index_a] = nums[index_b]
        nums[index_b] = temp
```

### [1079. Letter Tile Possibilities](https://leetcode.com/problems/letter-tile-possibilities/)

### [784. Letter Case Permutation](https://leetcode.com/problems/letter-case-permutation/)

## Dynamic Programming

### [746. Min Cost Climbing Stairs](https://leetcode.com/problems/min-cost-climbing-stairs/)

### [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray/)

### [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)

### [303. Range Sum Query - Immutable](https://leetcode.com/problems/range-sum-query-immutable/)

```csharp
// C#
// save every range from index 0 to index j
// sum from index i to index j = range(0, j) - range(0, i)
public class NumArray {
    private int[] ranges;
    public NumArray(int[] nums) {
        ranges = new int[nums.Length + 1];
        int sum = 0;
        for(int i = 0; i < nums.Length; i++) {
            ranges[i + 1] = sum + nums[i];
            sum += nums[i];
        }
    }

    public int SumRange(int i, int j) {
        return ranges[j + 1] - ranges[i];
    }
}
```

```python
class NumArray:

    def __init__(self, nums: List[int]):
        self.__ranges = [0]
        sum = 0
        for (i, num) in enumerate(nums):
            self.__ranges.append(sum + num)
            sum += num

    def sumRange(self, left: int, right: int) -> int:
        return self.__ranges[right + 1] - self.__ranges[left]


# Your NumArray object will be instantiated and called as such:
# obj = NumArray(nums)
# param_1 = obj.sumRange(left,right)
```

### [413. Arithmetic Slices](https://leetcode.com/problems/arithmetic-slices/)

## Union Find

### [547. Friend Circles](https://leetcode.com/problems/friend-circles/)

[Java solution, Union Find](https://leetcode.com/problems/friend-circles/discuss/101336/Java-solution-Union-Find)
