---
title: Leetcode788. 旋转数字
date: 2018-08-01 22:24:49
tags: leetcode
categories:
- leetcode
- string
- easy
---
## 题目地址
[https://leetcode.com/problems/rotated-digits](https://leetcode.com/problems/rotated-digits "leetcode788")

## Code1
```
function rotatedDigits(int $n) :int {
    $result = 0;

    for ($i = 1; $i <= $n; $i++) {
        if (isValid($i)) {
            $result++;
        }
    }

    return $result;
}

function isValid(int $n) :bool {
    $valid = false;

    while ($n > 0) {
        if ($n % 10 == 2 || $n % 10 == 5 || $n % 10 == 6 || $n % 10 == 9) {
            $valid = true;
        }

        if ($n % 10 == 3 || $n % 10 == 4 || $n % 10 == 7) {
            return false;
        }

        $n = intdiv($n, 10);
    }

    return $valid;
}
```

## Code2
```
function rotatedDigits(int $n) :int {
    $result = 0;
    $dp = array_fill(0, $n + 1, 0);

    // $dp[$i]=0，无效；$dp[$i]=1，有效，旋转后相等；$dp[$i]=2，有效，旋转后不相等；
    for ($i = 0; $i <= $n; $i++) {
        if ($i < 10) {
            if ($i == 0 || $i == 1 || $i == 8) {
                $dp[$i] = 1;
            } elseif ($i == 2 || $i == 5 || $i == 6 || $i == 9) {
                $dp[$i] = 2;
                $result++;
            }
        } else {
            $a = $dp[intdiv($i, 10)];
            $b = $dp[$i % 10];

            if ($a == 1 && $b == 1) {
                $dp[$i] = 1;
            } elseif ($a >= 1 && $b >= 1) {
                $dp[$i] = 2;
                $result++;
            }
        }
    }

    return $result;
}
```