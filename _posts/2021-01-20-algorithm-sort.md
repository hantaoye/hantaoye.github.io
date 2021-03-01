---
title:  "算法 排序笔记"
date:   2021-01-20 22:11:36 +0530
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
# 冒泡

def bubbleSort(array: list):
    for i in range(len(array)):
        for j in range(i + 1, len(array)):
            if array[j] < array[i]:
                temp = array[j]
                array[j] = array[i]
                array[i] = temp
    return array

# 插入排序 - 新搞个数组最简单，下面是没搞数组的
# for i in range(10, 0, -1) 倒叙的for循环
def insertSort(array: list):
    for i in range(1, len(array)):
        vale = array[i]
        k = 0
        for j in range(i - 1, -1, -1):
            if array[j] > vale:
                array[j + 1] = array[j]
            else:
                k = j + 1
                break
        array[k] = vale
    return array
    
# 选择排序
def selectionSort(arr):
    for i in range(len(arr) - 1):
        # 记录最小数的索引
        minIndex = i
        for j in range(i + 1, len(arr)):
            if arr[j] < arr[minIndex]:
                minIndex = j
        # i 不是最小数时，将 i 和最小数进行交换
        if i != minIndex:
            arr[i], arr[minIndex] = arr[minIndex], arr[i]
    return arr


# 桶排序
# 1. 取出待排序数组里最大和最小数， 然后创建 一个长度为：最大-最小数的数组，初始值为0
# 2. 遍历待排序数组，把（值-最小数）当成index，在桶数组index的值 +1
# 3. 创建个空数组arr1， 遍历桶，为0则不管，不为0，看数量，数量为几，arr就add 几个index，
# 4. print arr1

```

```
# !/usr/local/bin/python3

# 快排
def partition2(array, low, high) -> int: 
    i = ( low-1 )         # 最小元素索引
    pivot = arr[high]     
  
    for j in range(low , high): 
  
        # 当前元素小于或等于 pivot 
        if   arr[j] <= pivot: 
          
            i = i+1 
            arr[i],arr[j] = arr[j],arr[i] 
  
    arr[i+1],arr[high] = arr[high],arr[i+1] 
    return ( i+1 ) 



def partition(array, left, right) -> int: 
    key = array[left]
    while left < right:
        while left < right and array[right] > key:
            right -= 1
        array[left] = array[right]
        while left < right and array[left] <= key:
            left += 1
        array[right] = array[left]
    array[right] = key
    return left

def quickSort2(arr: list, left: int, right: int):
    if left >= right:
        return arr
    middle = partition2(arr, left, right)
    quickSort2(arr, left, middle - 1)
    quickSort2(arr, middle + 1, right)
    return arr


if __name__ == "__main__":
    arr = [1, 4, 100,5, 10, 4000, 5, 9, 0, -10]

    sortArr = quickSort2(arr, 0, len(arr) - 1)
    print(sortArr)
```

```
# 归并排序
def mergeSort(arr):
    import math
    if(len(arr) < 2):
        return arr
    middle = math.floor(len(arr)/2)
    left, right = arr[0:middle], arr[middle:]
    return merge(mergeSort(left), mergeSort(right))

def merge(left, right):
    result = []
    while left and right:
        if left[0] <= right[0]:
            result.append(left.pop(0))
        else:
            result.append(right.pop(0))
    while left:
        result.append(left.pop(0))
    while right:
        result.append(right.pop(0))
    return result

```