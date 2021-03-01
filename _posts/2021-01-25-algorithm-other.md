---
title:  "算法 滑动窗口、BFS/DFS、LRU/LFU、topk等笔记"
date:   2021-01-25 10:11:36 +0530
author: taoye
categories: [算法]
tags: [算法]
---

### 建议

1. 刷leetCode
2. 刷一下labuladong大佬的算法系列 （https://github.com/labuladong/fucking-algorithm）
3. 客户端不建议刷太多动态规划（个人面了20家，只问了一个动态规划）
4. 以下只是个人的算法笔记


### 滑动窗口

```
# !/usr/local/bin/python3

# 滑动窗口

import sys

# 最小覆盖子串
# 给定一个字符串s和t，在s中查找包含t的最小子串， 没有则返回空字符串
def minWindow(s: str, t: str) -> str:
    need = {}
    for c in t:
        if c in need:
            need[c] = need.get(c) + 1
        else:
            need[c] = 1

    window = {}
    left = 0
    vaild = 0
    start = 0
    length = sys.maxsize

    for right in range(len(s)):
        c = s[right]
        if c in need:
            window[c] = window[c] + 1 if c in window else 1
            if window[c] == need[c]:
                vaild += 1
        
        while vaild == len(need):
            c2 = s[left]
            if c2 in need:
                if window[c2] == need[c2]:
                    vaild -= 1
                window[c2] -= 1
            
            # 此处需要+1，因为从0开始的， 比如0-3，长度是4，但3-0=3，需要加1
            if (right + 1) - left < length:
                length = (right + 1) - left
                start = left
            
            left += 1
    
    return s[start:start+length] if length != sys.maxsize else ""


# 字符串排列
# 给到两个字符串 s1，s2，判断s1是否包含s2的排列
# 相当给你一个 S 和一个 T，请问你 S 中是否存在一个子串，包含 T 中所有字符且不包含其他字符？
def checkArrange(s1: str, s2: str) -> str:
    need = {}
    window = {}
    for c in s2:
        need[c] = need[c] + 1 if c in need else 1

    left = 0
    right = 0
    vaild = 0

    while(right < len(s1)):
        c = s1[right]
        right += 1

        if c in need:
            window[c] = window[c] + 1 if c in window else 1
            if window[c] == need[c]:
                vaild += 1
        # 只要长度=s2，此处用>=和=一样
        while (right - left >= len(s2)):
            if vaild == len(need):
                return True
 
            c1 = s1[left]
            if c1 in need:
                if need[c1] == window[c1]:
                    vaild -= 1

                window[c1] -= 1

            left += 1


# 找所有字母异位词
# 给字符s、t，，找到 S 中所有 T 的排列(不限顺序)，返回它们的起始索引

def findIndex(s: str, t: str) -> list:
    need = {}
    window = {}
    for c in t:
        need[c] = need[c] + 1 if c in need else 1

    left = 0
    vaild = 0
    index = []

    for right in range(len(s)):
        c = s[right]
        if c in need:
            window[c] = window[c] + 1 if c in window else 1
            if window[c] == need[c]:
                vaild += 1
        
        while right + 1 - left >= len(t):
            if vaild == len(need):
                index.append(left)

            c1 = s[left]
            if c1 in need:
                if need[c1] == window[c1]:
                    vaild -= 1
                window[c1] -= 1
            left += 1
    return index


# 最长无重复子串
def subLengthForStr(s: str) -> int:
    window = ""
    left = 0
    right = 0
    length = 0
    while right < len(s):
        c = s[right]
        right += 1
        if c not in window:
            window += c
        else:
            if length < len(window):
                length = len(window)
            window += c

            while left < right:
                c1 = s[left]
                left += 1
                if c1 in window:
                    window = window[1:]
                if c1 == c:
                    break
            
    return length

def subLengthForStr2(s) -> int:
    window = {}
    left = right = 0
    length = 0
    while right < len(s):
        c = s[right]
        right += 1
        window[c] = window[c] + 1 if  c in window else 1
        while window[c] > 1:
            d = s[left]
            left += 1
            window[d] -= 1

        length = max((right - left), length)

    return length

if __name__ == "__main__":
    s = 'dafdadaf.dsa,fdsa/fsdfdsfda.123456z7[]8f.cid9-=a cf,lksfs/ayt,ytyt/fd,af.sdae,.klfnbcudo'
    # s = "fdlaababcdefghijklmnopqrstuvwxyz1234,ab4"
    # t = './,'
    # result = minWindow(s, t)
    # print(result)

    # s2 = 'ttyy/f,'
    # isExist = checkArrange(s, s2)
    # print(isExist)

    # s3 = 'da'
    # index = findIndex(s, s3)
    # print(index)

    print(subLengthForStr(s))
    print(subLengthForStr2(s))


```

### LRU

```
# !/usr/local/bin/python3


class ListNode():
    def __init__(self, key = None, value = None, previous=None, next=None):
        self.previous = previous
        self.next = next
        self.key = key
        self.value = value


class LRUCache():

    def __init__(self, capacity: int):
        self.dict = {}
        self.capacity = capacity
        self.head = ListNode()
        self.tail = ListNode()
        self.head.next = self.tail
        self.tail.previous = self.head

# 参数为key也行
    def _moveToHead(self, node:ListNode):
        if node.previous is not None:
            node.previous.next = node.next
        if node.next is not None:
            node.next.previous = node.previous

        tempNode = self.head.next
        self.head.next = node
        node.next = tempNode
        tempNode.previous = node
        node.previous = self.head

    def get(self, key: int) -> int:
        if key in self.dict:
            node = self.dict.get(key)
            self._moveToHead(node)
            return node.value
        else:
            return -1

    def put(self, key: int, value: int) -> None:
        if key in self.dict:
            node = self.dict.get(key)
            node.value = value
            self._moveToHead(node)
            self.dict[key] = node
        else:
            node = ListNode(key=key, value=value)
            self._moveToHead(node)
            self.dict[key] = node
            if (self.capacity < len(self.dict)):
                self.dict.pop(self.tail.previous.key)
                self.tail = self.tail.previous
                self.tail.next = None


class numberListNode():
    def __init__(self, nodeList: list, key = None, value=None, number=1):
        self.key = key
        self.value = value
        self.number = number
        self.nodeList = nodeList

class LFU():
    def __init__(self, capacity: int):
        self.dict = {}
        self.capacity = capacity
        self.head = ListNode()
        self.tail = ListNode()
        self.head.next = self.tail
        self.tail.previous = self.head
        self.node

    def put(self, key: int, value: int) -> None:
        pass
    
    
    def get(self, key: int) -> int:
        pass

    def _moveToHead(self, node:ListNode):
        pass
    
    
    

if __name__ == "__main__":
    a = LRUCache(10)
    a.put(1, 1)
    a.put(2, 2)
    a.get(1)
    a.put(3, 3)
    print(a.dict)
    a.get(3)
    a.get(4)
    a.put(4, 4)
    a.put(5, 5)
    a.get(4)


```

### 背包问题

```

"""
有 N 件物品和一个容量为 V 的背包，第 i 件物品的费用是 c[i] ，体积是 w[i] ，求解将哪些物品装入背包可使价值总和最大。
"""

def bags():
    # 物品容积
    w = [0, 4, 5, 5, 2, 2]
    # 物品价值
    c = [0, 6, 4, 6, 3, 6]
    # 背包大小
    W = 10
    # 初始化 dp 状态数组
    dp = [[0 for _ in range(W + 1)] for _ in range(len(w))]
    
    for i in range(1, len(w)):
        for j in range(0, W + 1):
            if (j < w[i]):
                dp[i][j] = dp[i - 1][j]
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i - 1][j - w[i]] + c[i])
    
    # debug
    for i in range(1, len(w)):
        print(dp[i])
            
    print(f'可以装最大价值是 {dp[-1][-1]}')

if __name__ == "__main__":
    bags();
```

### BFS-DFS （二叉树的最小高度）

```

class TreeNode():
    def __init__(self):
        self.left = None
        self.right = None

# 二叉树的最小高度 BFS
def minDepth(root: TreeNode) -> int:
    if root is None:
        return 0
    queue = [root]
    int depth = 1

    while len(queue) > 0:
        count = len(queue)
        tempList = []
        for i in range(count):
            node = queue[i]
            left = node.left
            right = node.right

            if left is not None:
                tempList.append(left)
            elif right is not None:
                tempList.append(right)
            else:
                return depth
        depth += 1
        queue = tempList

    return depth

# 二叉树的最小高度 DFS

def minDepthDFS(root: TreeNode) -> int:
    if root is None:
        return 0
    return min(minDepthDFS(root.left), minDepth(root.right)) + 1

```

### topK

```
import heapq

def findKthLargest(nums: List[int], k: int) -> int:
    if len(nums) < k:
        return -1
    L = []
    for index in range(k):
        heapq.heappush(L, nums[index])

    for index in range(k, len(nums)):
        top = L[0]
        if nums[index] > top:
            heapq.heapreplace(L, nums[index])
    return L[0]

```

### 表达式添加括号

```
# 为运算表达式设计优先级
# https://leetcode-cn.com/problems/different-ways-to-add-parentheses/
# 给定一个含有数字和运算符的字符串，为表达式添加括号，改变其运算优先级以求出不同的结果。你需要给出所有可能的组合的结果。有效的运算符号包含 +, - 以及 * 。
def diffWaysToCompute(input: str) -> list:
    sums = []
    for i in range(len(input)):
        s = input[i]
        if s == '+' or s == '-' or s == '*':
            print(s)
            leftStr = input[0:i]
            rightStr = input[i+1:]
            print(leftStr)
            print(rightStr)
            leftSums = diffWaysToCompute(leftStr)
            rightSums = diffWaysToCompute(rightStr)
            for left in leftSums:
                for right in rightSums:
                    if s == '+':
                        sums.append(left + right)
                    elif s == '-':
                        sums.append(left - right)
                    elif s == '*':
                        sums.append(left * right)
    if len(sums) == 0 and len(input) > 0:
        sums.append(int(input))
    return sums


if __name__ == "__main__":
    result = diffWaysToCompute('2*3-4*5')
    print(result)
```