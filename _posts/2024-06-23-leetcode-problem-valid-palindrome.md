---
layout: post
title: Leetcode Problem Solve - Valid Palindrome
date: 2024-06-24 12:00 +0900
categories: [Programming, Leetcode]
tags: [Leetcode, Algorithms]
image:
  path: /assets/img/posts/valid-palindrome/valid-palindrome-problem.png
  alt: Valid Palindrome Problem
---

## Headings

Today, let's dive into another popular problem from Leetcode, the "Valid Palindrome" problem. This problem is fundamental for understanding string manipulation and the two-pointer technique.

## Introduction

![Valid Palindrome Description](/assets/img/posts/valid-palindrome/valid-palindrome-problem.png)
_Valid Palindrome Problem Description_

```
Given a string s, determine if it is a palindrome, considering only alphanumeric characters and ignoring cases.

For example, "A man, a plan, a canal: Panama" is a palindrome, while "race a car" is not.
```

The "Valid Palindrome" problem helps us understand how to handle strings efficiently and utilize two-pointer techniques for palindrome checking.

## Analysis

To solve this problem, we need to normalize the string by removing all non-alphanumeric characters and converting it to lowercase. Then, we can use a two-pointer approach to check if the string reads the same forwards and backwards.

## Solutions

### 1. Using Regular Expressions

```python
import re

class Solution:
    def isPalindrome(self, s: str) -> bool:
        # Normalize: remove non-alphanumeric characters and convert to lowercase
        nS = re.sub('[^a-zA-Z0-9]', '', s).lower()
        i = 0
        j = len(nS) - 1

        while i < j:
            # If characters don't match, it's not a palindrome
            if nS[i] != nS[j]:
                return False
            
            i += 1
            j -= 1

        return True
```

In this approach, we use the `re` module to remove all non-alphanumeric characters and convert the string to lowercase. We then use a two-pointer technique to check if the string is a palindrome.

**Time Complexity**: `O(N)` - We iterate through the string twice (once for normalization and once for the palindrome check).

**Space Complexity**: `O(N)` - We store the normalized string.

### 2. Using String Methods

```python
class Solution:
    def isPalindrome(self, s: str) -> bool:
        # Normalize: remove non-alphanumeric characters and convert to lowercase
        nS = ''.join(char.lower() for char in s if char.isalnum())
        
        # Two-pointer technique to check palindrome
        i, j = 0, len(nS) - 1
        
        while i < j:
            if nS[i] != nS[j]:
                return False
            i += 1
            j -= 1
        
        return True
```

This approach uses string methods to filter out non-alphanumeric characters and convert the string to lowercase. It then applies the two-pointer technique to check for a palindrome.

**Time Complexity**: `O(N)` - We iterate through the string twice (once for normalization and once for the palindrome check).

**Space Complexity**: `O(N)` - We store the normalized string.

## Conclusion

In this post, we explored two different approaches to solve the "Valid Palindrome" problem. The first approach used regular expressions for normalization, while the second approach utilized string methods. Both approaches efficiently check for palindromes and meet the problem's requirements.

If you have any questions or suggestions, please feel free to leave a comment below.

Thank you for reading!
