---
title:  "算法 数组系列笔记"
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

### 题目

```
# 删除有序数组中重复的，空间复杂度为O(1)
def removeDuplicates(nums: list) -> int:
    if len(nums) <= 0: return 0
    fast = slow = 0

    while fast < len(nums):
        if nums[fast] != nums[slow]:
            slow += 1
            nums[fast], nums[slow] = nums[slow], nums[fast]
        fast += 1
    return slow + 1
```


```
# 删除0
def moveZeroes(nums: List[int]) -> None:
    """
    Do not return anything, modify nums in-place instead.
    """
    if len(nums) <= 0:
        return
    fast = slow = 0
    while fast < len(nums):
        if nums[fast] != 0:
            nums[slow] = nums[fast]
            slow += 1
        fast += 1
    while slow < len(nums):
        nums[slow] = 0
        slow += 1
```


```
# 动态规划思路
# 最长公共子序列
def _longestCommonSubsequence2(str1, str2) -> int:
    m, n = len(str1), len(str2)
    dp = [[0] * (n + 1) for _ in range(0, (m + 1))]
    # print(dp)
    for i in  range(1, m + 1):
        for j in range(1, n + 1):
            if str1[i - 1] == str2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1;
            else:
                dp[i][j] = max(dp[i][j - 1], dp[i - 1][j])

    print(dp)
    return dp[m][n]

if __name__ == "__main__":
    first_str = 'aaqtaaa'
    second_str = 'aatkpoelqz.m'
    longest = _longestCommonSubsequence2(first_str, second_str)
    print(longest)

```

```

# 5. 二分查找
def midSearch(nums: list, target: int):
    left: int = 0
    right: int = len(nums) - 1
    while left <= right:
        mid = left + int((right - left) / 2)
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        elif nums[mid] > target:
            right = mid - 1
    return -1
```

```
// 寻找左侧边界的二分搜索 （教科书方法）
int left_bound(int[] nums, int target) {
    if (nums.length == 0) return -1;
    int left = 0;
    int right = nums.length; // 注意

    while (left < right) { // 注意
        int mid = (left + right) / 2;
        if (nums[mid] == target) {
            right = mid;
        } else if (nums[mid] < target) {
            left = mid + 1;
        } else if (nums[mid] > target) {
            right = mid; // 注意
        }
    }
    // target 比所有数都大
    if (left == nums.length) return -1;
    // 类似之前算法的处理方式
    return nums[left] == target ? left : -1;
}
# 二分查找， 如果有重复的，查找左边第一个 （自己写的）
def midLeftSearch(nums: list, target: int):
    left = 0
    right = len(nums) - 1
    while left <= right:
        mid = left + int((right - left) / 2)
        if nums[mid] == target:
            if mid == 0:
                return mid
            else:
                right = mid - 1
        elif nums[mid] < target:
            left = mid + 1
        elif nums[mid] > target:
            right = mid - 1
    if right < 0 or right == len(nums) - 1:
        return -1;
    return right + 1;
    

```


```
# !/usr/local/bin/python3

from typing import List, Tuple, Dict

array = List[int]
dictionary = Dict[str, str]
customTuple = Tuple[float, str]


# 前缀和技巧：解决子数组问题
# 给定一个整数数组和一个整数K，找出数组中的连续子数组和为k的连续子数组个数


# 这个解法的时间复杂度 O(N的二次方) 空间复杂度O（N），并不是最优的解法
def subArraySum1(arr: array, k: int) -> int:
    subArraySum = [0]
    for number in arr:
        sum = subArraySum[-1] + number
        subArraySum.append(sum)
    print(subArraySum)

    result = 0
    for i in range(1, len(arr) + 1):
        for j in range(0, i):
            if subArraySum[i] - subArraySum[j] == k:
                result += 1
    print("连续子数组个数和为", k, "的数量为：", result)
    return result
                

# 时间复杂度降到了O(N)
def subArraySum2(arr: array, k: int) -> int:
    sum = 0
    result = 0
    arrMap = {0:1}
    for i in range(0, len(arr)):
        sum += arr[i]
        left = sum - k;
        if left in arrMap:
            result += arrMap[left]
        

        if sum in arrMap:
            arrMap[sum] = arrMap[sum] + 1
        else:
            arrMap[sum] = 1
    print("连续子数组个数和为", k, "的数量为：", result)
    return result


def subarraySum111(nums: List[int], k: int) -> int:
    if nums == []:
            return 0
        
        preSum = {0:1}
        sum = 0
        result = 0
        for i in range(len(nums)):
            sum += nums[i]
            value = sum - k
            if value in preSum:
                result += preSum.get(value)
            preSum[sum] = preSum.get(sum, 0) + 1
        return result

if __name__ == "__main__":
    subArraySum1([1, 4, 5, 9, 10, 12, 9, 2, 4, 2, 2, 5, 5, 3, -12, 8, 2], 10)
    subarraySum111([1,3,5,6,0, -4 , 2, -1], 3)
```

```
# 最大子序和 -  给定一个整数数组 nums ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。
# https://leetcode-cn.com/problems/maximum-subarray/
def maxSubArray(self, nums: List[int]) -> int:
    if nums is None:
        return 0
    if len(nums) == 1:
        return nums[0]

    dp = [0 for _ in range(len(nums))]
    dp[0] = nums[0]
    for i in range(1, len(nums)):
        dp[i] = max(nums[i], dp[i - 1] + nums[i])

    result = dp[0]
    for i in dp:
        result = max(result, i)

    return result
```


```
# 6. 两数之和  给出一个按照升序排序的有序数组，找出两个数之和=目标数，可以假设答案是唯一的，函数返回index1和index2， index从1开始算（类似数组的0=index的1）
def twoSum(sum: list, target: int):
    left = 0
    right = len(sum) - 1
    while left < right:
        tempSum = sum[left] + sum[right]
        if tempSum == target:
            return left + 1, right + 1
        elif tempSum < target:
            left += 1
        elif tempSum > target:
            right -= 1
    return -1, -1
    

# 7. 反转数组
def reverseSum(sum: list):
    left = 0
    right = len(sum) - 1
    while left < right:
        num1 = sum[left]
        num2 = sum[right]
        sum[left] = num2
        sum[right] = num1
        left += 1
        right -= 1
    return sum

```

```
# 给定一个数组，找出两数之和=sum， 
# 解法、1 俩for暴力破解  2. 加一个map，值为key，下标为value，（需考虑数组中数据重复的情况） 3. 先排序、后用双指针
```


```
# 287. 寻找重复数
# 给定一个包含 n + 1 个整数的数组 nums ，其数字都在 1 到 n 之间（包括 1 和 n），可知至少存在一个重复的整数。
 def findDuplicate(nums: List[int]) -> int:
    size = len(nums)
    left = 1
    right = size - 1

    while left < right:
        mid = left + (right - left) // 2

        cnt = 0
        for num in nums:
            if num <= mid:
                cnt += 1
        # 根据抽屉原理，小于等于 4 的数的个数如果严格大于 4 个，
        # 此时重复元素一定出现在 [1, 4] 区间里

        if cnt > mid:
            # 重复的元素一定出现在 [left, mid] 区间里
            right = mid
        else:
            # if 分析正确了以后，else 搜索的区间就是 if 的反面
            # [mid + 1, right]
            left = mid + 1
    return left
```


```
数组内查找只出现一次的元素，其他都是重复两个
def singleNumber(nums:list)->int:
    res = 0
    for num in nums:
        res ^= num
    return res
    

```