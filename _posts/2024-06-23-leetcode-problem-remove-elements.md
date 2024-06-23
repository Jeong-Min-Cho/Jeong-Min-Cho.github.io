---
layout: post
title: Leetcode Problem Solve - Remove Element
date: 2024-06-23 12:00 +0900
categories: [Programming, Leetcode]
tags: [Leetcode, Algorithms]
image:
  path: /assets/img/posts/remove-element/remove-element-problem.png
  alt: Remove Element Problem
---

## Headings

After diving into the Two Sum problem, today let's explore another classic problem from Leetcode, the "Remove Element" problem. This problem is straightforward but can be solved using different approaches, each with its own trade-offs.

## Introduction

![Remove Element Description](/assets/img/posts/remove-element/remove-element-problem.png)
_Remove Element Problem Description_

```
Given an integer array nums and an integer val, remove all occurrences of val in nums in-place. The relative order of the elements may be changed.

Since it is impossible to change the length of the array in some languages, you must instead have the result be placed in the first part of the array nums. More formally, if there are k elements after removing the duplicates, then the first k elements of nums should hold the final result. It does not matter what you leave beyond the first k elements.

Return k after placing the final result in the first k slots of nums.

Do not allocate extra space for another array. You must do this by modifying the input array in-place with O(1) extra memory.
```

The Remove Element problem is a fundamental problem that helps understand the concepts of in-place array manipulation and efficient space usage.

## Analysis

We need to remove all occurrences of a specific value from the array and return the new length of the array. Importantly, we need to do this in-place with O(1) extra memory, meaning we can't use additional arrays or significant extra space.

## Solutions

### 1. Simple List Comprehension

```python
def removeElement(self, nums: List[int], val: int) -> int:
    newArr = []

    for x in nums:
        if x != val:
            newArr.append(x)
    
    nums[:] = newArr
    return len(nums)
```

In this approach, we create a new array `newArr` and append all elements from `nums` that are not equal to `val`. After that, we copy `newArr` back to `nums` and return its length. 

However, this solution uses extra space proportional to the number of elements in `nums`, which does not meet the problem's constraint of using O(1) extra memory.

**Time Complexity**: `O(N)`

**Space Complexity**: `O(N)` - due to the additional array `newArr`.

### 2. Two-Pointer Approach

```python
class Solution:
    def removeElement(self, nums: List[int], val: int) -> int:
        pOne = 0
        pTwo = len(nums) - 1 

        while pOne <= pTwo:
            # Adjust the goal point
            if nums[pTwo] == val:
                pTwo -= 1
                continue
            
            if nums[pOne] == val:
                nums[pOne], nums[pTwo] = nums[pTwo], nums[pOne]
                pTwo -= 1
            
            pOne += 1

        return pTwo + 1
```

This approach utilizes a two-pointer technique to solve the problem in-place. We initialize two pointers: `pOne` at the start and `pTwo` at the end of the array. The idea is to swap elements when `nums[pOne]` is equal to `val` and adjust the pointers accordingly.

1. If `nums[pTwo]` equals `val`, we decrement `pTwo`.
2. If `nums[pOne]` equals `val`, we swap it with `nums[pTwo]` and decrement `pTwo`.
3. In both cases, we increment `pOne` to continue the check.

This method ensures we only use O(1) extra space.

**Time Complexity**: `O(N)`

**Space Complexity**: `O(1)`

## Conclusion

In this post, we explored two different approaches to solve the Remove Element problem. The first approach was straightforward but did not meet the space complexity constraint. The second approach, using a two-pointer technique, was more efficient and met all the problem requirements.

If you have any questions or suggestions, please feel free to leave a comment below.

Thank you for reading!

