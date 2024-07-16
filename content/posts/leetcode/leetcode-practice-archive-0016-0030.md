---
title: "LeetCode 题解归档 (0016~0030)"
hideSummary: true
date: 2024-07-16T17:15:23+08:00
draft: false
tags: ["leetcode"]
series: ["leetcode"]
author: ["xubinh"]
type: posts
math: true
---

## 7. 整数反转

英文题目名称: Reverse Integer

标签: 数学

思路:

- 第一反应是使用传统的通过符号位的变化判断补码整数加减法是否溢出的方法, 但是写完狠狠报错, 然后才反应过来这题还涉及到**乘法**.
- 由于涉及十进制乘法, 判断溢出也需要以十进制的思维进行.
  - 对于 $x > 0$ 的情况, 溢出等价于
  
  $$\text{current_result} \times 10 + \text{digit} > 2147483647.$$
  
  - $x < 0$ 的情况同理.
  
  提交后发现执行效率不是很好.
- 最后通过查看题解了解到上述的条件还可以进一步优化, 优化到只需要检查 $\text{current_result}$ 就能判断溢出的程度, 根本不需要检查 $\text{digit}$.
  - 之所以能够进行优化是因为上述条件是离散的, 完全可以将所有情况列出来, 对同类情况进行合并同类项, 并对定义域进行降维.
  - 还是对于 $x > 0$ 的情况, 简单移项可知上述溢出条件又等价于
    
    $$(\text{current_result} - 214748364) \times 10 > 7 - \text{digit}.$$

    - 当 $\text{current_result} - 214748364 < 0$ 时, 上式不等号左边必然小于等于 $-10$, 因此不等式必然不成立.
    - 当 $\text{current_result} - 214748364 > 0$ 时, 上式不等号左边必然大于等于 $10$, 因此不等式必然成立.
    - 当 $\text{current_result} - 214748364 = 0$ 时, 情况比较特殊, 需要进一步分析:
      - 由于 $x$ 此时仍然不为零 (若为零则已经可以返回结果, 不需要再判断溢出), 这说明 $x$ 的十进制形式必然是 $\[\text{digit}\]463847412$ (即 $214748364$ 倒过来再加上一位).
      - 又因为 $x$ 必然没有溢出, 于是可知 $\text{digit}$ 必然小于等于 $2$.
      - 此时上式不等号左边为零, 而右边大于零, 因此不等式必然不成立.

    综上所述, $x > 0$ 的情况下原溢出条件等价于 $\text{current_result} - 214748364 > 0$.
  - $x < 0$ 的情况同理.

代码:

```cpp
class Solution {
public:
    int reverse(int x) {
        // 平凡情况:
        if (x == 0) {
            return 0;
        }

        int current_result = 0;

        // 循环不变量: `current_result` 可以进行一次扩充且保证不会溢出
        while (1) {
            // 进行一次扩充:
            int digit = x % 10;
            x /= 10;
            current_result = current_result * 10 + digit;

            // 因为保证不会溢出, 结果必然是正确答案:
            if (x == 0) {
                break;
            }

            // 在进行下一次扩充之前检查是否可能溢出:
            if (current_result > 214748364 || current_result < -214748364) {
                return 0;
            }
        }

        return current_result;
    }
};
```
