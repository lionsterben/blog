---
layout:     post
title:      "leetcode001"
subtitle:   " \"enjoy this world\""
date:       2018-07-18 23:27:00
author:     "Dawei"
tags:
    - leetcode
---
Longest Substring Without Repeating Characters
==
### The problem is in [here](https://leetcode.com/problems/longest-substring-without-repeating-characters/description/ )

题目描述：这个问题是求一个字符串最长非重复字串（必须是相连的）

思路：一看到这个问题，我就想到用递归的方法求，毕竟能够分解成更小的问题。于是我写出了以下的代码
```python
class Solution(object):
    def isUnique(self,str):
        a = set()
        for i in str:
            a.add(i)
        return len(a) == len(str)
    
    def shit(self,s):
        if len(s) == 0:
            return ""
        elif len(s) == 1:
            return s
        else:
            temp = self.shit(s[1:])
            length = len(temp)
            if s[1:].startswith(temp):
                if s[0] not in temp:
                    return s[0] + temp
                else:
                    return temp
            else:
                if self.isUnique(s[:length+1]):
                    return s[:length+1]
                elif self.isUnique(s[:length]):
                    return s[:length]
                else:
                    return temp
                
    def lengthOfLongestSubstring(self, s):
        """
        :type s: str
        :rtype: int
        """
        return len(self.shit(s))
```
整体思路就是假设我们求出s[1:]的最长非重复子串，然后分为两种情况，如果这个子串是s[1:]的开始部分，直接判断s[0]能不能加上去构成更长的子串，反之，判断s[0]加上之后能不能构成比s[1:]的子串相等或者更长的子串。写出后就觉得时间复杂度有点高是O(n<sup>2</sup>)，而且这样写每层字符串都会复制一份，果然，评判时候最后一个例子爆内存了。。

于是观察了别人的思路，写出了下面的代码
```python
class Solution(object):
                
    def lengthOfLongestSubstring(self, s):
        """
        :type s: str
        :rtype: int
        """
        map_str = {}
        start = 0
        max_len = 0
        for i in range(len(s)):
            if s[i] not in map_str or map_str[s[i]] < start:
                map_str[s[i]] = i
                # print(i)
                # print(s[i])
                # print(start)
                # print(map_str)
                if (i - start) + 1 > max_len:
                    max_len = i - start + 1
            else:
                temp = s[i]
                # print(i)
                # print(temp)
                start = map_str[s[i]] + 1
                map_str[s[i]] = i
        return max_len
```
- 代码看起来清爽不少，逻辑是最长子串肯定对应一个末尾字符，我们遍历s每个字符作为末尾字符的子串然后找到最长子串就行了。在遍历每个字符时，维护一个start，表示在当前末尾字符下非重复子串的开始字符。
- 其中维护一个hashmap，对应每个字符在字符串的位置。在遍历字符时，如果这个字符在hashmap或者在start到end的子串里不存在（也就是遇到非重复字符了）就把这个字符加入hashmap里，计算是不是最长子串。如果这个字符在目前子串里重复了，将start改为重复的字符在hashmap的位置加1（是当前字符作为末尾字符对应的最长子串），将重复字符在hashmap的位置更新。遍历完成，最大长度也就出来了。
    
        
        