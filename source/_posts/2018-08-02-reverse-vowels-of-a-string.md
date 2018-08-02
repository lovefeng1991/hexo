---
title: Leetcode345. 反转字符串中的元音字母
date: 2018-08-02 17:28:13
tags:
- leetcode
- 双指针
categories:
- leetcode
- string
- easy
---
## 题目地址
[https://leetcode.com/problems/reverse-vowels-of-a-string](https://leetcode.com/problems/reverse-vowels-of-a-string "leetcode345")

## Code
```
function reverseVowels(string $str) :string {
    $vowels = [
        'A' => 1,
        'a' => 1,
        'E' => 1,
        'e' => 1,
        'I' => 1,
        'i' => 1,
        'O' => 1,
        'o' => 1,
        'U' => 1,
        'u' => 1,
    ];
    $i = 0;
    $j = strlen($str) - 1;

    while ($i < $j) {
        while ($i < $j && !isset($vowels[$str[$i]])) {
            $i++;
        }

        while ($i < $j && !isset($vowels[$str[$j]])) {
            $j--;
        }

        list($str[$i], $str[$j]) = [$str[$j], $str[$i]];
        $i++;
        $j--;
    }

    return $str;
}
```

## 解释
标准双指针解法