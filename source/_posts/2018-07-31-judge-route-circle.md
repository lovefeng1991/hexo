---
title: 657. 判断路线成圈
date: 2018-07-31 23:41:53
tags: leetcode
categories:
- leetcode
- string
- easy
---
## 题目地址
[https://leetcode.com/problems/judge-route-circle](https://leetcode.com/problems/judge-route-circle "leetcode657")

## Code
```
function judgeCircle(string $moves) :bool {
    $length = strlen($moves);
    $x = $y = 0;

    for ($i = 0; $i < $length; $i++) {
        if ($moves[$i] == 'U') {
            $y++;
        } elseif ($moves[$i] == 'D') {
            $y--;
        } elseif ($moves[$i] == 'R') {
            $x++;
        } elseif ($moves[$i] == 'L') {
            $x--;
        } else {
            continue;
        }
    }

    return $x == 0 && $y == 0;
}
```
