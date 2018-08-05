---
title: Leetcode67. 二进制求和
date: 2018-08-02 16:31:48
tags: leetcode
categories:
- [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/add-binary](https://leetcode.com/problems/add-binary "leetcode67")

## Code
```
function addBinary(string $a, string $b, int $n = 2) :string {
    $i = strlen($a) - 1;
    $j = strlen($b) - 1;
    $result = '';
    $carry = 0;

    while ($i >= 0 || $j >= 0) {
        $sum = $carry;
        if ($i >= 0) {
            $sum += $a[$i--];
        }

        if ($j >= 0) {
            $sum += $b[$j--];
        }

        $result = ($sum % $n) . $result;
        $carry = intdiv($sum, $n);
    }

    if ($carry != 0) {
        $result = $carry . $result;
    }

    return $result;
}
```
