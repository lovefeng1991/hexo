---
title: Leetcode13. 罗马数字转整数
date: 2018-08-01 13:24:16
tags: leetcode
categories:
- leetcode
- string
- easy
---
## 题目地址
[https://leetcode.com/problems/roman-to-integer](https://leetcode.com/problems/roman-to-integer "leetcode13")

## Code
```
function romanToInt(string $str) :int {
    $map = [
        'I' => 1,
        'V' => 5,
        'X' => 10,
        'L' => 50,
        'C' => 100,
        'D' => 500,
        'M' => 1000
    ];

    $length = strlen($str);

    if ($length == 0) {
        return 0;
    }

    $sum = $map[$str[$length - 1]];

    for ($i = $length - 2; $i >= 0; $i--) {
        if ($map[$str[$i]] < $map[$str[$i + 1]]) {
            $sum -= $map[$str[$i]];
        } else {
            $sum += $map[$str[$i]];
        }
    }

    return $sum;
}
```
