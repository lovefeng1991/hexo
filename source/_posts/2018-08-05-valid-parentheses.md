---
title: Leetcode20. 有效的括号
date: 2018-08-05 16:22:04
tags:
- leetcode
- 栈
categories:
- [leetcode, string, easy]
---
## 题目地址
[https://leetcode.com/problems/valid-parentheses](https://leetcode.com/problems/valid-parentheses "leetcode20")

## Code
```
function isValidParentheses(string $str) :bool {
    $stack = new SplStack();
    $length = strlen($str);
    $leftBrackets = [
        '(' => 1,
        '{' => 2,
        '[' => 3,
    ];
    $rightBrackets = [
        ')' => 1,
        '}' => 2,
        ']' => 3,
    ];

    for ($i = 0; $i < $length; $i++) {
        if (isset($leftBrackets[$str[$i]])) {
            $stack->push($str[$i]);
        } elseif (isset($rightBrackets[$str[$i]])) {
            if ($stack->isEmpty()) {
                return false;
            }

            $leftBracket = $stack->pop();

            if ($leftBrackets[$leftBracket] != $rightBrackets[$str[$i]]) {
                return false;
            }
        }
    }

    return $stack->isEmpty();
}
```