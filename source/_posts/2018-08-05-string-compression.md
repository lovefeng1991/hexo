---
title: Leetcode443. 压缩字符串
date: 2018-08-05 19:27:46
tags:
- leetcode
categories:
- [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/string-compression](https://leetcode.com/problems/string-compression "leetcode443")

## Code
```
function compress(array $chars) :int {
    $count = count($chars);
    $index = 0;
    $i = 0;

    while ($i < $count) {
        $currentChar = $chars[$i];
        $n = 0;

        while ($i < $count && $chars[$i] == $currentChar) {
            $i++;
            $n++;
        }

        $chars[$index++] = $currentChar;

        if ($n == 1) {
            continue;
        }

        $nStr = (string) $n;
        $nLen = strlen($nStr);

        for ($j = 0; $j < $nLen; $j++) {
            $chars[$index++] = $nStr[$j];
        }
    }

    return $index;
}
```
