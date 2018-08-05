---
title: Leetcode859. 亲密字符串
date: 2018-08-06 00:19:52
tags:
- leetcode
categories:
- [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/buddy-strings](https://leetcode.com/problems/buddy-strings "leetcode859")

## Code
```
function buddyStrings(string $a, string $b) :bool {
    $aLength = strlen($a);
    $bLength = strlen($b);

    if ($aLength != $bLength) {
        return false;
    }

    if ($aLength < 2) {
        return false;
    }

    $canChange = false;
    $hashMap = [];
    $diff = 0;
    $indexOne = -1;
    $indexTwo = -1;

    for ($i = 0; $i < $aLength; $i++) {
        if (isset($hashMap[$a[$i]])) {
            ++$hashMap[$a[$i]];
        } else {
            $hashMap[$a[$i]] = 1;
        }

        if ($hashMap[$a[$i]] >= 2) {
            $canChange = true;
        }

        if ($a[$i] != $b[$i]) {
            $diff++;

            if ($indexOne == -1) {
                $indexOne = $i;
            } elseif ($indexTwo == -1) {
                $indexTwo = $i;
            }
        }
    }

    return ($diff == 0 && $canChange)
        || ($diff == 2 && $a[$indexOne] == $b[$indexTwo] && $a[$indexTwo] == $b[$indexOne]);
}
```