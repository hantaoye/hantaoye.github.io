---
title:  "算法 链表系列笔记"
date:   2021-01-05 22:11:36 +0530
author: taoye
categories: [算法]
tags: [算法]
---

### 建议

1. 刷leetCode
2. 刷一下labuladong大佬的算法系列 （https://github.com/labuladong/fucking-algorithm）
3. 客户端不建议刷太多动态规划（个人面了20家，只问了一个动态规划）
4. 以下只是个人的算法笔记

```
# 删除重复的链表
def deleteDuplicatesListNode(head: ListNode) -> ListNode:
    if head is None:
        return head
    fast = slow = head

    while (fast is not None):
        if fast.val != slow.val:
            slow.next = fast
            slow = fast
        fast = fast.next

    slow.next = None
    return head
```

```
# 1. 判定链表中是否含有环
def hasCycle(node: LinkNode) -> bool:
    if node.next == None:
        return False

    fast = slow = node
    while (fast != None and fast.next != None):
        slow = slow.next;
        fast = fast.next.next;
        if fast == slow:
            return True
        # else:
            # fast = fast.next
            # slow = slow.next

    return False
```


```
# 2. 已知链表中含有环，返回这个环的起始位置
def detectCycleStartLoc(node: LinkNode) -> LinkNode:
    fast = slow = node
    while(fast != None and fast.next != None):
        slow = slow.next
        fast = fast.next.next
        if fast == slow:
            break
    
    if (fast == None or fast.next == None):
        return None

    fast = node
    while (fast != slow):
        fast = fast.next
        slow = slow.next
    return slow

```


```
# 3. 寻找链表的中点
def detectNodeCenter(node: LinkNode) -> LinkNode:
    fast = slow = node
    while(fast != None and fast.next != None):
        slow = slow.next
        fast = fast.next.next
    return slow

```


```
# 4. 寻找链表的倒数第 k 个元素
def detectNodeReverseIndexForK(node: LinkNode, k: int) -> LinkNode:
    fast = slow = node

    for i in range(0, k):
        if fast == None or fast.next == None:
            return None
        fast = fast.next

    while fast != None and fast.next != None:
        fast = fast.next
        slow = slow.next
    return slow
```


```
# 9. 反转链表
def reverseNode(node: LinkNode):
    if node is None or node.next is None:
        return node
    cur = node
    ne = node.next
    cur.next = None
    while ne is not None:
        temp = ne.next
        ne.next = cur
        cur = ne
        ne = temp
    return cur

# 递归
def reverseNode2(head:LinkNode)->LinkNode:
    pass
    # if node is None or node.next is None:
    #     return node
    # ListNode last = reverseNode2(head.next);
    # head.next.next = head;
    # head.next = None;
    # return last;
    
    
# 反转从位置 m 到 n 的链表。请使用一趟扫描完成反转。
def reverseBetween(head: LinkNode, m: int, n: int):
    if head is None or head.next is None:
        return head
    if n == m:
        return head
    
    sHead = head
    preHead = head
    startHead = head
    if m > 1:
        for _ in range(m - 2):
            global preHaed
            preHead = preHead.next
        startHead = preHead.next
    
    currentNode = startHead
    nextNode = currentNode.next
    currentNode.next = None
    for _ in range(m, n):
        tmp = nextNode.next
        nextNode.next = currentNode
        currentNode = nextNode
        nextNode = tmp
    
    startHead.next = nextNode
    if m <= 1:
        sHead = currentNode
    else:
        preHead.next = currentNode
    
    return sHead

```


```
# 合并两个有序链表
def mergeNode(node1:LinkNode, node2:LinkNode) -> LinkNode:
    if node1 is None:
        return node2
    if node2 is None:
        return node1
    if node1.element < node2.element:
        node1.next = mergeNode(node1.next, node2)
        return node1
    else:
        node2.next = mergeNode(node1, node2.next)
        return node2
```


```
# 俩链表是否有相同的内容（公共父视图）
def hasCommon(node1: LinkNode, node2: LinkNode):
    if node1 is None or node2 is None:
        return None
    while node1 is None:
        while node2 is node:
            if node1.element == node2.element:
                return node1
            else:
                node2 = node2.next;
        node1 = node1.next
    return None
```