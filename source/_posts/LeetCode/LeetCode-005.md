---
title: "LeetCode-005,如何一步步思考到动态规划"
date: 2023-05-21
updated: 2023-05-22
categories: 技术
tags: 
  - 力扣之旅
  - java
  - 数据结构
---

### Q:

给你一个字符串 `s`，找到 `s` 中最长的回文子串。

如果字符串的反序与原始字符串相同，则该字符串称为回文字符串。

### Input:

```text
输入：s = "babad"
输出："bab"
解释："aba" 同样是符合题意的答案。
```

### Solution:

如果是第一次做类似的题目，其实一下子是并不容易想到可以使用动态规划。

但是，通过一步步的思考和优化，使用动态规划其实是一种比较自然的想法。

##### 首先：

##### 根据题目的要求，一种最简单的做法应该是：

1. 设计一个方法 `judgeIsPalindrome` ，基于双指针，我们可以得到一个 O(n) 的判断 `String(i,j)` 是否是回文串的方法。

2. 枚举所有的长度 `k`，从大到小判断所有的 `String(i, i+k-1)`，判断是否是回文串，如果是，则得到题目要求。

这个做法很简单，但是存在一个问题就是，需要枚举所有的 `k`，最坏的情况需要枚举 `N` 种，而对每种长度，最坏需要计算 `N` 次才能得到结果。所以整体复杂度很高，无法通过。需要寻找一种更优的做法。

##### 根据以上的思路，一种想法就是并不需要枚举每一个 `k`，而是改为使用二分法搜索最大的 `k`，这样复杂度会变为 `log N`。有兴趣的可以自己尝试一下这个思路。

##### 上面这个思路是有可能通过的，但是仍然可以优化。我们可以注意到，上面的做法还有一个问题就是，每次都需要重新判断新长度的子串是否是回文，而没有利用到之前计算过的结果。

假设我们对一个特定的长度 `k_i`，我们已经知道了 String 中所有长度为 `k_i` 的子串是否是回文串。那么当我们知道 `String(i, i+k_i-1)` 是否是回文的时候，对于 `String(i-1, i+k_i)` 其实这个结果也是可以在 `O(1)` 的时间内得到：

```
dp[i][j] = dp[i+1][j-1] ^ String[i] == String[j]
```

即只有 `String(i, i+k_i-1)` 是回文并且 `String[i] == String[j]`，那么 `String(i-1, i+k_i)` 才能是回文串。

下面可以写出 DP 代码：

```java
public String longestPalindrome(String s) {
    int len = s.length();
    if (len < 2) {
        return s;
    }
    boolean[][] dp = new boolean[len][len];
    for (int i = 0; i < len; i++) {
        dp[i][i] = true;
    }
    int maxLen = 1;
    int start = 0;
    for (int j = 1; j < len; j++) {
        for (int i = j - 1; i >= 0; i--) {
            if (s.charAt(i) == s.charAt(j)) {
                if (j - i < 3) {
                    dp[i][j] = true;
                } else {
                    dp[i][j] = dp[i + 1][j - 1];
                }
            } else {
                dp[i][j] = false;
            }

            if (dp[i][j]) {
                int curLen = j - i + 1;
                if (curLen > maxLen) {
                    maxLen = curLen;
                    start = i;
                }
            }
        }
    }
    return s.substring(start, start + maxLen);
}
```
