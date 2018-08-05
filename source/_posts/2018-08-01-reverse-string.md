---
title: Leetcode344. 反转字符串
date: 2018-08-01 12:00:08
tags: leetcode
categories:
- [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/reverse-string](https://leetcode.com/problems/reverse-string "leetcode344")
## Code
```
function reverseString(string $str) :string {
    $i = 0;
    $j = strlen($str) - 1;

    while ($i < $j) {
        $temp = $str[$i];
        $str[$i] = $str[$j];
        $str[$j] = $temp;
        $i++;
        $j--;
    }

    return $str;
}
```
