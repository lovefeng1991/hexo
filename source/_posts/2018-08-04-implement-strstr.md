---
title: Leetcode28. 实现strStr()
date: 2018-08-04 18:51:12
tags: leetcode
categories: [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/implement-strstr](https://leetcode.com/problems/implement-strstr "leetcode28")

## Code
```
function strStrN(string $haystack, string $needle) :int {
    $lengthNeedle = strlen($needle);

    if ($lengthNeedle == 0) {
        return 0;
    }

    $lengthHaystack = strlen($haystack);

    $i = $j = 0;

    while ($i <= $lengthHaystack - $lengthNeedle) {
        while ($j < $lengthNeedle && $haystack[$i + $j] != $needle[$j]) {
            break;
        }

        if ($j == $lengthNeedle) {
            return $i;
        }

        $i++;
    }

    return -1;
}
```

类似于PHP中[strpos](http://php.net/manual/zh/function.strpos.php "strpos")函数，可以使用KMP算法、BM算法来解决。