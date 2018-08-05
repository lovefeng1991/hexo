---
title: Leetcode696. 计数二进制子串
date: 2018-08-01 19:20:26
tags: leetcode
categories:
- [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/count-binary-substrings](https://leetcode.com/problems/count-binary-substrings "leetcode696")

## Code
```
function countBinarySubstrings(string $str) :int {
    $length = strlen($str);
    $result = 0;
    $preCount = $curCount = 1;

    for ($i = 1; $i < $length; $i++) {
        if ($str[$i] != $str[$i - 1]) {
            $preCount = $curCount;
            $curCount = 1;
        } else {
            $curCount++;
        }

        if ($preCount >= $curCount) {
            $result++;
        }
    }

    return $result;
}
```