---
title: Leetcode686. 重复叠加字符串匹配
date: 2018-08-05 23:18:32
tags:
- leetcode
categories:
- [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/repeated-string-match](https://leetcode-cn.com/problems/repeated-string-match "leetcode686")

## Code
```
function repeatedStringMatch(string $a, string $b) :int {
    $aLength = strlen($a);
    $bLength = strlen($b);

    for ($i = 0; $i < $aLength; $i++) {
        for ($j = 0; $j < $bLength && $a[($i + $j) % $aLength] == $b[$j]; $j++);

        if ($j == $bLength) {
            return intdiv($i + $j - 1, $aLength) + 1;
        }
    }

    return -1;
}
```

[Leetcode28](/2018/08/04/implement-strstr)的变形。