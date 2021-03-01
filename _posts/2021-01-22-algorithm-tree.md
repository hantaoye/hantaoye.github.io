---
title:  "算法 二叉树系列笔记"
date:   2021-01-22 22:11:36 +0530
author: taoye
categories: [算法]
tags: [算法]
---

### 建议

1. 刷leetCode
2. 刷一下labuladong大佬的算法系列 （https://github.com/labuladong/fucking-algorithm）
3. 客户端不建议刷太多动态规划（个人面了20家，只问了一个动态规划）
4. 以下只是个人的算法笔记

### 题目

```
class TreeNode():
    def __init__(self, left, right, value = 0):
        self.value = value
        self.left = left
        self.right = right


# 二叉树节点数
def treeCount(treeNode: TreeNode) -> int:
    if not treeNode:
        return 0
    else:
        return 1 + treeCount(treeNode.left) + treeCount(treeNode.right)


# 翻转二叉树
def reversedTreeNode(treeNode:TreeNode) -> TreeNode:
    if not treeNode:
        return
    treeNode.left, treeNode.right = treeNode.right, treeNode.left
    reversedTreeNode(treeNode.left)
    reversedTreeNode(treeNode.right)
    return treeNode
        
```


```

# 将二叉树展开为链表
def flatten(node: TreeNode):
    if not node:
        return None
    
    # 让子节点的左节点放到右节点后面
    flatten(node.left)
    flatten(node.right)

    right = node.right
    node.right = node.left
    node.left = None

    # 将原先的右子树接到当前右子树的末端
    flag = node
    while node.right is not None:
        flag = node.right

    flag.right = right
    return node

```


```

# 构造最大二叉树
def constructMaximumTree(nums: list):
    if nums is None or len(nums) == 0:
        return None

    index = 0
    maxNum = nums[0]
    for i in range(1, len(nums)):
        if maxNum < nums[i]:
            maxNum = nums[i]
            index = i

    left = constructMaximumTree(nums[0:index - 1])
    right = constructMaximumTree(nums[index+1: len(nums)])
    node = TreeNode(left = left, right = right, value=maxNum)
    
    return node
```



/// TODO:
```

# 从后序与中序遍历序列构造二叉树
def buildTree(self, inorder: List[int], postorder: List[int]) -> TreeNode:
    # 实际上inorder 和 postorder一定是同时为空的，因此你无论判断哪个都行
    if not inorder:
        return None
    root = TreeNode(postorder[-1])
    i = inorder.index(root.val)
    root.left = self.buildTree(inorder[:i], postorder[:i])
    root.right = self.buildTree(inorder[i+1:], postorder[i:-1])
    return root



# 寻找重复的子树 深度优先（DFS）
def findDuplicateSubtrees(root):
    import collections
    d = collections.defaultdict(list)

    def dfs(root):
        if not root:
            return ''
        s = ' '.join((str(root.val), dfs(root.left), dfs(root.right)))
        d[s].append(root)
        return s

    dfs(root)
    return [l[0] for l in d.values() if len(l) > 1]


# 二叉树最小深度(DFS)
def minDepth(root: TreeNode) -> int:
    if not root:
        return 0
    ans = 0
    if not root.left and not root.right: 	# 叶子节点
        ans = 1
    elif root.left and root.right:  # 左右子树均不为空
        ans = min(self.minDepth(root.left), self.minDepth(root.right)) + 1
    elif root.left:		# 左子树不为空 & 右子树为空
        ans = self.minDepth(root.left) + 1
    else:			# 左子树为空 & 右子树不为空
        ans = self.minDepth(root.right) + 1
    return ans
# BFS
def minDepth2(self, root: TreeNode) -> int:
    if not root:
        return 0
    arr = []
    depth = 1

    treeArr = [root]
    tempTreeArr = treeArr.copy()
    while len(tempTreeArr):
        for i in len(tempTreeArr):
            treeNode = tempTreeArr[i]
            treeArr.remove(treeNode)
            if treeNode.left is None and treeNode.right is None:
                return depth
            elif treeNode.left is not None:
                treeArr.append(treeNode.left)
            elif treeNode.right is not None:
                treeArr.append(treeNode.right)

        tempTreeArr = treeArr.copy()
        depth += 1
    return depth





if __name__ == "__main__":
    node1 = TreeNode(None, None)
    node2 = TreeNode(node1, None)
    node3 = TreeNode(node2, node1)
    node4 = TreeNode(node3, node3)
    
    count = treeCount(node4)
    print(count)
    
    print("-" * 100)
    a =  constructMaximumTree([3, 2, 1, 6, 0, 5])
    print(a)

```