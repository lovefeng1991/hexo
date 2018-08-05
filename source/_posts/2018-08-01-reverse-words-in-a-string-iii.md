---
title: Leetcode557. 反转字符串中的单词III
date: 2018-08-01 13:02:49
tags: leetcode
categories:
- [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/reverse-words-in-a-string-iii](https://leetcode.com/problems/reverse-words-in-a-string-iii "leetcode557")

## Code
```
function reverseWords(string $str) :string {
    $length = strlen($str);
    $start = 0;

    for ($i = 0; $i <= $length; $i++) {
        if ($i == $length || $str[$i] == ' ') {
            $str = doReverse($str, $start, $i - 1);
            $start = $i + 1;
        }
    }

    return $str;
}

function doReverse(string $str, int $start, int $end) :string {
    while ($start < $end) {
        $temp = $str[$start];
        $str[$start] = $str[$end];
        $str[$end] = $temp;
        $start++;
        $end--;
    }

    return $str;
}
```
