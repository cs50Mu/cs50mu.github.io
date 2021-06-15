+++
title = "leetcode challenage"
date = "2016-02-25"
slug = "2016/02/25/leetcode-challenage"
Categories = []
+++

###Nim Game
这是最简单的题目了，但也想了很久，只想到用递归，看了答案才知道原来一条代码可以解决。。

这个问题的关键是给出一个数字，判断是否一定能赢，而不管过程。

一开始想到递归：

``` python nim-game-iter
class Solution(object):
    def canWinNim(self, n):
        """
        :type n: int
        :rtype: bool
        """
        if n == 4:
            return False
        elif n == 1 or n == 2 or n == 3:
            return True
        else:
            return not self.canWinNim(n-1) or not self.canWinNim(n-2) or not self.canWinNim(n-3)
```
提示超时，继续尝试，增加memorization，对于已经计算过的做缓存，下次遇到直接返回。

``` python nim-game-iter-memo
class Solution(object):
    def __init__(self):
        self.memo = {}
        
    def canWinNim(self, n):
        try:
            return self.memo[n]
        except KeyError:
            self.memo[n] = self.canWinNimIter(n)
            return self.memo[n]
    def canWinNimIter(self, n):
        """
        :type n: int
        :rtype: bool
        """
        if n == 4:
            return False
        elif n == 1 or n == 2 or n == 3:
            return True
        else:
            return not self.canWinNim(n-1) or not self.canWinNim(n-2) or not self.canWinNim(n-3)
```
这次不提示超时了，提示超出最大递归深度，尼玛。。

看了别人的答案，眼泪留下来。。
``` python nim-game-superman
class Solution(object):
    
    def canWinNim(self, n):
        return n % 4 != 0
```
思考：对于4个的情况，先拿肯定是输，所以能赢的情况必然是不能被4整除的情况（一个数被4除，要么余1 2 3，要么整除）。设想一次游戏，总数除4后余1，那么你先拿一个，然后剩下总数为4的倍数，不管对方拿多少个，我只要确定我下次拿的数目跟对方上次拿的数目之和等于4即可，这样直至拿完，肯定赢。

参考：[https://www.hrwhisper.me/leetcode-nim-game/](https://www.hrwhisper.me/leetcode-nim-game/)

### Maximum Depth of Binary Tree
找出一棵二叉树的最大深度。

思路：递归，先得到左边的深度，再得到右边的深度，最后返回最大的一个即可。

```python maximum-depth-of-binary-tree
# Definition for a binary tree node.
# class TreeNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.left = None
#         self.right = None

class Solution(object):
    def maxDepth(self, root):
        """
        :type root: TreeNode
        :rtype: int
        """
        if not root:
            return 0
        else:
            left = 1 + self.maxDepth(root.left)
            right = 1 + self.maxDepth(root.right)
            return max(left, right)
```
### Invert Binary Tree
递归，先把左右分支交换，然后依次递归反转左右分支
```python invert-binary-tree
# Definition for a binary tree node.
# class TreeNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.left = None
#         self.right = None

class Solution(object):
    def invertTree(self, root):
        """
        :type root: TreeNode
        :rtype: TreeNode
        """
        if not root:
            return None
        else:
            root.left, root.right = root.right, root.left
            self.invertTree(root.left)
            self.invertTree(root.right)
            return root
```

### Move Zeroes
维护两个指针, i用于遍历数组，j初始指向数组开始，当i遇到非零元素时，交换i和j指向的元素，并把i、j各自递增。因此，若一个数组中没有零元素，i和j是始终指向同一个元素的。
```python move zeroes
class Solution(object):
    def moveZeroes(self, nums):
        """
        :type nums: List[int]
        :rtype: void Do not return anything, modify nums in-place instead.
        """
        j = 0
        for i in range(len(nums)):
            if nums[i]:
                nums[i], nums[j] = nums[j], nums[i]
                j += 1
```

### Majority Element
Moore投票算法，维护两个变量candidate和count，candidate记录的是当前可能的候选人，count记录的是当前可能候选人的票数。遍历数组，当遇到的元素等于候选人时就把票数加1；当遇到的元素不等于候选人的时候就把票数减1；当票数为0的时候说明这个候选人没希望了，将候选人置为当前遍历到的元素，然后票数初始化为0。最后得到的candidate就是这个数组中出现次数最多的元素。
```python majority-element
class Solution(object):
    def majorityElement(self, nums):
        """
        :type nums: List[int]
        :rtype: int
        """
        candidate, count = None, 0
        for i in nums:
            if count == 0:
                candidate, count = i, 1
            elif i == candidate:
                count += 1
            else:
                count -= 1
        return candidate
```

### Reverse Linked List
迭代版本：
基本原理就是新建一个linked list，一边遍历旧的linked list，一边把读出来的元素添加到新建的linked list中，不过注意添加的方法，是从最后一个一个往开始生成的。
```python reverse linked list iter
# Definition for singly-linked list.
# class ListNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution(object):
    def reverseList(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        dummy = ListNode(0)   # dummy is indeed a dummy, lol
        while head:
            next = head.next  # save the next node for later process
            head.next = dummy.next  # 先把head接到已处理过的linked lis的前面
            dummy.next = head   # 再跟原来的dummy连起来
            head = next  #  指针指向下一个node
        return dummy.next
```
递归版本：
还是递归实现起来简单些
```python reverse linked list recursion
# Definition for singly-linked list.
# class ListNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution(object):
    def reverseList(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        return self.reverseListRecursion(head, None)
        
    def reverseListRecursion(self, head, new_head):
        if not head:
            return new_head
        next = head.next   # save the rest temporarily
        head.next = new_head   # next point to new_head
        return self.reverseListRecursion(next, head)  # doing this recursively
```

###Roman to Integer
自己想出来的，不容易啊。。

首先把罗马数字与阿拉伯数字的映射关系准备好，然后遍历罗马数字的字符，同时维护一个prev变量保存前一个遍历过的字符，当发现当前字符比前一个字符代表的阿拉伯数字小时，使用特殊的累加策略，否则就是简单的把当前字符对应的阿拉伯数字累加到总和total中。
```python Roman to integer
class Solution(object):
    def romanToInt(self, s):
        """
        :type s: str
        :rtype: int
        """
        hash = {'I':1, 'V':5, 'X':10, 'L':50, 'C':100, 'D':500, 'M':1000}
        total = 0
        prev = None
        for i in s:
            if prev and hash[i] > hash[prev]:
                total += hash[i] - 2 * hash[prev]
            else:
                total += hash[i]
            prev = i
        return total
```

###Odd Even Linked List
思路其实比较简单，就是遍历链表，奇数序元素放到奇数链表中，偶数序元素放到偶数链表中。但是，要维护的指针比较多，不注意就会搞混。。新生成的奇数链表和偶数链表都需要指针来操作。
而且，在从头生成一个指针时，需要先初始化一个DummyNode，然后再把元素一个一个接在后面，元素添加完成后，`DummyNode.next`就是这条刚生成的链表的head了。

```python Odd Even Linked List
# Definition for singly-linked list.
# class ListNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution(object):
    def oddEvenList(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        oddHead = ListNode(0)
        oddCurrent = oddHead   # pointer
        evenHead = ListNode(0)
        evenCurrent = evenHead  # pointer
        count = 1
        while head != None:
            if count % 2 != 0:
                oddCurrent.next = ListNode(head.val)
                oddCurrent = oddCurrent.next
            else:
                evenCurrent.next = ListNode(head.val)
                evenCurrent = evenCurrent.next

            head = head.next
            count += 1
            
        oddCurrent.next = evenHead.next
            
        return oddHead.next
```
###Remove Duplicates from Sorted List

维护两个指针，一个prev，一个current，分别指向遍历链表的前一个元素和当前元素，当 前一个元素跟当前元素相同时，舍弃掉当前元素。
否则，就把prev和current各自向下移动一位元素。
```python Remove Duplicates from Sorted List
# Definition for singly-linked list.
# class ListNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution(object):
    def deleteDuplicates(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        prev = None
        current = head
        while current is not None:
            if prev and prev.val == current.val:
                prev.next = current.next
                current = prev.next
            else:
                prev = current
                current = current.next
        return head
```

###Happy Number
用一个集合来放已经算过的数，如果有数重复出现，说明开始进入循环了。

```python Happy Number
class Solution(object):
    def isHappy(self, n):
        """
        :type n: int
        :rtype: bool
        """
        if n <= 0:
            return False
        number = set()
        while n != 1 and n not in number:
            number.add(n)
            sum = 0
            while n != 0:
                tmp = n % 10
                sum += tmp**2
                n /= 10
            n = sum
        return n == 1
```

###Merge Two Sorted Lists

比较两个链表当前的元素大小，把符合条件的元素加到新list中，同时把符合条件的元素所在的链表的指针往下移动一个元素，另一个链表的指针不动，
如此直到至少一个链表遍历完成，最后把剩下的链表未遍历完的元素全部追加到新链表即可。

```python Merge Two Sorted Lists
# Definition for singly-linked list.
# class ListNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution(object):
    def mergeTwoLists(self, l1, l2):
        """
        :type l1: ListNode
        :type l2: ListNode
        :rtype: ListNode
        """
        dummy = ListNode(0)
        current = dummy
        while l1 and l2:
            if l1.val >= l2.val:
                current.next = ListNode(l2.val)
                l2 = l2.next
            else:
                current.next = ListNode(l1.val)
                l1 = l1.next
            current = current.next
        if l1:
            current.next = l1
        elif l2:
            current.next = l2
        return dummy.next
```

###Intersection of Two Arrays

先对两个数组排序，然后对两个数组各自维护一个指针，同时遍历两个数组，当在一个数组中当前指针对应的值小于另一个数组中当前指针对应的值时，把指针往前移动。
当两个指针对应的元素相等时，说明找到一个interaction了。

```python Intersection of Two Arrays
class Solution(object):
    def intersection(self, nums1, nums2):
        """
        :type nums1: List[int]
        :type nums2: List[int]
        :rtype: List[int]
        """
        if not nums1 or not nums2:
            return []
        sorted_nums1 = sorted(nums1)
        sorted_nums2 = sorted(nums2)
        i = j = 0
        last = None
        res = []
        while i < len(nums1) and j < len(nums2):
            if sorted_nums1[i] < sorted_nums2[j]:
                i += 1
            elif sorted_nums1[i] > sorted_nums2[j]:
                j += 1
            elif sorted_nums1[i] == sorted_nums2[j]:
                if sorted_nums1[i] != last:
                    ans = sorted_nums1[i]
                    last = ans
                    res.append(ans)
                i += 1
                j += 1
        return res
```

###Top K Frequent Elements

使用桶排序  比较有意思，先遍历一遍找到每个元素的频率存在字典中，然后初始化一个数组，以每个元素的频率为数组的下标（index）把对应的元素存入这个数组，
最后把这个数组从后往前遍历，得到的结果就是出现频率从高到低的元素



###Binary Tree Preorder Traversal

```python Binary Tree Preorder Traversal
# Definition for a binary tree node.
# class TreeNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.left = None
#         self.right = None

# recursive version
class Solution(object):
    def preorderTraversal(self, root):
        """
        :type root: TreeNode
        :rtype: List[int]
        """
        result = []
        self.recursive_traversal(root, result)
        return result

    def recursive_traversal(self, root, list):
        if root:
            list.append(root.val)
            self.recursive_traversal(root.left, list)
            self.recursive_traversal(root.right, list)

# another recursive version
class Solution(object):
    def preorderTraversal(self, root):
        """
        :type root: TreeNode
        :rtype: List[int]
        """
        if not root:
            return []
        return [root.val] + self.preorderTraversal(root.left) + self.preorderTraversal(root.right)

# iteritive version
class Solution(object):
    
    def preorderTraversal(self, root):
        res = []
        if root == None:
            return res
        stack = [root]
        while stack:
            root = stack.pop()
            res.append(root.val)
            if root.right:
                stack.append(root.right)
            if root.left:
                stack.append(root.left)
        return res
```

###Kth Smallest Element in a BST

对BST的in-order traversal就是按顺序的遍历，所以执行一个in-order traversal，同时记录遍历到第几个就行了。

```python
# Definition for a binary tree node.
# class TreeNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.left = None
#         self.right = None

class Solution(object):
    def kthSmallest(self, root, k):
        """
        :type root: TreeNode
        :type k: int
        :rtype: int
        """
        node = root
        stack= []
        # traverse to the left-most element, aka, the smallest element
        while node:
            stack.append(node)
            node = node.left
        i = 0
        # in-order traversal using stack
        while stack and i < k:
            element = stack.pop()
            if element.right:
                temp = element.right
                while temp:
                    stack.append(temp)
                    temp = temp.left
            i += 1
        return element.val
```

###Two Sum II - Input array is sorted

维护两个指针，从两边向中间搜索

```python
class Solution(object):
    def twoSum(self, numbers, target):
        """
        :type numbers: List[int]
        :type target: int
        :rtype: List[int]
        """
        low = 1
        high = len(numbers)
        while low < high:
            s = numbers[low-1] + numbers[high-1]
            if target > s:
                low += 1
            elif target < s:
                high -= 1
            else:
                return [low, high]
        return []
```

###Linked List Random Node

蓄水池抽样（Reservoir Sampling）

```python
# Definition for singly-linked list.
# class ListNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.next = None

import random

class Solution(object):

    def __init__(self, head):
        """
        @param head The linked list's head.
        Note that the head is guaranteed to be not null, so it contains at least one node.
        :type head: ListNode
        """
        self.head = head
        

    def getRandom(self):
        """
        Returns a random node's value.
        :rtype: int
        """
        cnt = 0
        node = self.head
        while node:
            if random.randint(0, cnt) == 0:
                res = node.val
            node = node.next
            cnt += 1
        return res
        


# Your Solution object will be instantiated and called as such:
# obj = Solution(head)
# param_1 = obj.getRandom()
```
###Shuffle an Array

对于每个i，从0-i随机选择一个数r，交换

```python
class Solution(object):

    def __init__(self, nums):
        """
        
        :type nums: List[int]
        :type size: int
        """
        self.nums = nums[:]
        self.original = nums[:]
        

    def reset(self):
        """
        Resets the array to its original configuration and return it.
        :rtype: List[int]
        """
        return self.original
        

    def shuffle(self):
        """
        Returns a random shuffling of the array.
        :rtype: List[int]
        """
        for i in range(len(self.nums)):
            rand = random.randint(0, i)
            self.nums[i], self.nums[rand] = self.nums[rand], self.nums[i]
        return self.nums
        


# Your Solution object will be instantiated and called as such:
# obj = Solution(nums)
# param_1 = obj.reset()
# param_2 = obj.shuffle()
```
###Count Numbers with Unique Digits

排列组合

设i为长度为i的各个位置上数字互不相同的数。

    i==1 : 10（0~9共10个数，均不重复）
    i==2: 9 * 9 （第一个位置上除0外有9种选择，第2个位置上除第一个已经选择的数，还包括数字0，也有9种选择）
    i ==3: 9* 9 * 8 （前面两个位置同i==2，第三个位置除前两个位置已经选择的数还有8个数可以用）
    ……
    i== n: 9 * 9 * 8 *…… (9-i+2)

需要注意的是，9- i + 2 >0 即 i < 11，也就是i最大为10，正好把每个数都用了一遍。

```python
class Solution(object):
    def countNumbersWithUniqueDigits(self, n):
        """
        :type n: int
        :rtype: int
        """
        res = [1] + [9] * n
        for i in range(2, n+1):
            for j in range(9, 9-i+1, -1):
                res[i] *= j
        return sum(res)
```

###Kth Smallest Element in a Sorted Matrix

利用优先队列 heapq

先把第一行加入heapq，然后经过k次循环即可得到第k大的元素，在每次循环中，pop出最小的元素，然后push进刚pop出的元素下方的元素（即它同一列上相邻的元素）

```python
import heapq
class Solution(object):
    def kthSmallest(self, matrix, k):
        """
        :type matrix: List[List[int]]
        :type k: int
        :rtype: int
        """
        row_count = len(matrix)
        col_count = len(matrix[0])
        
        pq = []
        for i in range(col_count):
            heapq.heappush(pq, (matrix[0][i], 0, i))
            
        for _ in range(k):
            val, row, col = heapq.heappop(pq)
            next = row + 1
            if next < row_count:
                heapq.heappush(pq, (matrix[next][col], next, col))
        return val
```

###Product of Array Except Self

除自己之外的其它数的乘积可以看作由两部分组成：该数左边部分的数的积和改数右边部分的数的积。

```python
class Solution(object):
    def productExceptSelf(self, nums):
        """
        :type nums: List[int]
        :rtype: List[int]
        """
        nums_length = len(nums)
        output = [1]

# calcute left part
        pre_product = 1
        for i in range(1, nums_length):
            pre_product *= nums[i-1]
            output.append(pre_product)

# calcute right part and the final result
        after_product = 1
        for i in range(nums_length-2, -1, -1):
            after_product *= nums[i+1]
            output[i] *= after_product

        return output
```

###Binary Tree Paths

DFS 深度优先搜索，在traverse过程中记住经过的node，当到达叶子节点时，把路径打印出来

```python
# Definition for a binary tree node.
# class TreeNode:
#     def __init__(self, x):
#         self.val = x
#         self.left = None
#         self.right = None

class Solution:
    # @param {TreeNode} root
    # @return {string[]}
    def binaryTreePaths(self, root):
        self.result = []
        if not root:
            return self.result
        def dfs(node, path):
            if not node.left and not node.right:
                self.result.append(path)
            if node.left:
                dfs(node.left, path + '->' + str(node.left.val))
            if node.right:
                dfs(node.right, path + '->' + str(node.right.val))
        dfs(root, str(root.val))
        return self.result
```
