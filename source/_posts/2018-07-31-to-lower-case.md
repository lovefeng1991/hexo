---
title: 709. 转换为小写字母
date: 2018-07-31 22:40:06
tags: leetcode
categories:
- leetcode
- string
- easy
---
## 题目地址
[https://leetcode.com/problems/to-lower-case/description/](https://leetcode.com/problems/to-lower-case/description/ "leetcode709")

## Code
```
function toLowerCase(string $str) :string {
    $length = strlen($str);

    for ($i = 0; $i < $length; $i++) {
        if (ord('A') <= ord($str[$i]) && ord($str[$i]) <= ord('Z')) {
            $str[$i] = chr(ord($str[$i]) - ord('A') + ord('a'));
        }
    }

    return $str;
}
```
