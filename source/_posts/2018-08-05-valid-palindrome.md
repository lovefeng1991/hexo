---
title: Leetcode125. 验证回文串
date: 2018-08-05 13:48:57
tags:
- leetcode
- 双指针
categories:
- [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/valid-palindrome](https://leetcode.com/problems/valid-palindrome "leetcode125")

## Code
```
function isPalindrome(string $str) :bool {
    $length = strlen($str);

    if ($length == 0) {
        return true;
    }

    $i = 0;
    $j = $length -  1;
    $str = strtolower($str);

    while ($i <= $j) {
        while ($i <= $j && !isNumericOrLetter($str[$i])) {
            $i++;
        }

        while ($i <= $j && !isNumericOrLetter($str[$j])) {
            $j--;
        }

        if ($str[$i] != $str[$j]) {
            return false;
        }

        $i++;
        $j--;
    }

    return true;
}

function isNumericOrLetter(string $char) :bool {
    if ((ord($char) <= ord('9') && ord($char) >= ord('0'))
        || (ord($char) <= ord('z') && ord($char) >= ord('a'))) {
        return true;
    }

    return false;
}
```
