---
title:  "算法 面试笔记"
date:   2021-01-14 22:11:36 +0530
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
#!/usr/local/bin/python3

import random
# 腾讯的算法，随机输出1-10，概率一样， 下面是我面试时候写的，还有其他解法
def getList2(n: int) -> list:
    arr = []
    def randomN(left: int, right: int, arr: list):
        if left >= right:
            arr.append(left)
            return
        temp = random.randint(left, right)
        arr.append(temp)
        if temp < right:
            randomN(temp + 1, right, arr)
        if temp > left:
            randomN(left, temp - 1, arr)
    
    randomN(1, n, arr)
    return arr


# 打家劫舍问题
def getMaxNum(nums) -> int:
    if nums == [] or nums is None:
        return 0
    if len(nums) == 1:
        return nums[0]
    sums = [0] * (len(nums) + 1)
    sums[0] = nums[0]
    sums[1] = nums[1]

    for i in range(2, len(nums)):
        result = (nums[i] + sums[i - 2])
        b = sums[i - 1]
        sums[i] = max(result, b)
    return sums[i]


if __name__ == "__main__":
    arr = getList2(10)
    print(arr)
    
    print("*"*100)
    # nums = [1, 10, 20, 100, 2, 40, 88]
    # result = getMaxNum(nums)
    # print (result)

```