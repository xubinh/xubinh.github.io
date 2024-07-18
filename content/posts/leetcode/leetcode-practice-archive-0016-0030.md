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

## 8. 字符串转换整数 (atoi)

英文题目名称: String to Integer (atoi)

标签: 字符串

思路:

- 本题题型为模拟题, 没什么技巧.
- 一开始还以为可以使用上一题 (即 [7. 整数反转](https://leetcode.cn/problems/reverse-integer/description/)) 的用于判断是否溢出的结论速通, 后来发现上一题的结论是有前提条件的, 而该前提条件在本题中并不成立, 因此只能手动判断是否溢出.

代码:

```cpp
class Solution {
public:
    int myAtoi(string s) {
        const int N = s.length();
        int current_position = 0;

        // 去除前导空格:
        while (current_position < N && s[current_position] == ' ') {
            current_position++;
        }

        // 如果到字符串尾, 或者下一个字符不是符号或数字, 则不存在有效数字部分,
        // 直接返回:
        if (current_position == N
            || (s[current_position] != '+' && s[current_position] != '-'
                && (s[current_position] < '0' || s[current_position] > '9'))) {
            return 0;
        }

        // 读取符号:
        bool is_integer_positive = true;
        int integer_sign = 1;
        if (s[current_position] == '-') {
            is_integer_positive = false;
            integer_sign = -1;
        }
        if (s[current_position] == '+' || s[current_position] == '-') {
            current_position++;
        }

        // 确定数字部分的结束位置:
        int integer_part_end_position = current_position;
        while (integer_part_end_position < N
               && s[integer_part_end_position] >= '0'
               && s[integer_part_end_position] <= '9') {
            integer_part_end_position++;
        }

        // 去除前导零, 确定数字部分的开始位置:
        int integer_part_start_position = current_position;
        while (integer_part_start_position < integer_part_end_position
               && s[integer_part_start_position] == '0') {
            integer_part_start_position++;
        }

        // 手动检查是否存在溢出:
        bool is_overflow = false;
        if (integer_part_end_position - integer_part_start_position > 10) {
            is_overflow = true;
        } else if (integer_part_end_position - integer_part_start_position
                   < 10) {
            is_overflow = false;
        } else {
            const string extreme_integer =
                is_integer_positive ? "2147483647" : "2147483648";

            for (int compare_position_offset = 0; compare_position_offset < 10;
                 compare_position_offset++) {
                if (s[integer_part_start_position + compare_position_offset]
                    < extreme_integer[compare_position_offset]) {
                    break;
                } else if (s[integer_part_start_position
                             + compare_position_offset]
                           > extreme_integer[compare_position_offset]) {
                    is_overflow = true;
                    break;
                }
            }
        }

        if (is_overflow) {
            return is_integer_positive ? 2147483647 : (-2147483647 - 1);
        }

        int integer_result = 0;
        current_position = integer_part_start_position;
        while (current_position < integer_part_end_position) {
            integer_result = integer_result * 10
                             + integer_sign * (s[current_position] - '0');
            current_position++;
        }

        return integer_result;
    }
};
```

## 9. 回文数

英文题目名称: Palindrome Number

标签: 数学

思路:

- 首先排除一些边界情况. 根据题意规定, 负号也会被反转, 因此负数必定不是回文数. 传入的整数不具有前导零, 如果其末尾为零则必定不是回文数.
- 只考虑最严格的不先将整数转换为字符串再继续的做法. 最自然的想法是将整数的所有数字水平翻转然后看反转后的结果是否和翻转前的相同. 由于本题限制整数的范围为 32 位补码整数, 同时是翻转位数, 因此可以套用 [7. 整数反转](https://leetcode.cn/problems/reverse-integer/description/)中的结论判断翻转过程中是否有可能溢出.

代码:

```cpp
class Solution {
public:
    bool isPalindrome(int x) {
        // 平凡情况:
        if (x == 0) {
            return true;
        }

        // 根据题意规定, 负号也会被反转, 因此负数必定不是回文数:
        if (x < 0) {
            return false;
        }

        // 传入的整数不具有前导零, 如果其末尾为零则必定不是回文数:
        if (x % 10 == 0) {
            return false;
        }

        int x_duplicate = x;

        // 循环不变量: `current_result` 能够进行一次扩充且保证不会溢出
        int current_result = 0;
        while (1) {
            current_result = current_result * 10 + x % 10;
            x /= 10;

            if (x == 0) {
                break;
            }

            // 检查下一次扩充是否会溢出:
            if (current_result > 214748364) {
                return false;
            }
        }

        return current_result == x_duplicate;
    }
};
```

## 12. 整数转罗马数字

英文题目名称: Integer to Roman

标签: 哈希表, 数学, 字符串

思路:

- 可以选择模拟或硬编码两种做法. 之所以能够使用硬编码的做法是因为罗马数字表示法将千, 百, 十, 和个位的部分拆开进行表示, 每个部分有各自的编码且互不相交, 因此在整数到罗马数字之间形成了一个双射.
- 下面选择模拟做法. 模拟做法实际上就是在模拟硬编码的过程, 大致可描述为分块, 处理, 以及拼接三个步骤的循环. 首先将整数分为千位, 百位, 十位和个位的块, 然后依次对各个块进行编码得到对应的子串, 最终结果由这些子串拼接而成.

代码:

```cpp
class Solution {
public:
    string get_component(int number_of_units, char name_for_one_unit,
                         char name_for_five_units, char name_for_ten_units);
    string intToRoman(int num);
};

string Solution::get_component(int number_of_units, char name_for_one_unit,
                               char name_for_five_units,
                               char name_for_ten_units) {
    string component;

    switch (number_of_units) {
    case 0:
        break;
    case 1: // e.g. I
    case 2: // e.g. II
    case 3: // e.g. III
        component = string(number_of_units, name_for_one_unit);
        break;
    case 4: // e.g. IV
        component = {name_for_one_unit, name_for_five_units};
        break;
    case 5: // e.g. V
    case 6: // e.g. VI
    case 7: // e.g. VII
    case 8: // e.g. VIII
        component = {name_for_five_units};
        component.append(number_of_units - 5, name_for_one_unit);
        break;
    case 9: // e.g. IX
        component = {name_for_one_unit, name_for_ten_units};
        break;
    }

    return component;
}

string Solution::intToRoman(int num) {
    string result;
    int unit = 0;
    int number_of_units = 0;

    unit = 1000;
    number_of_units = num / unit;
    result.append(
        get_component(number_of_units, 'M', '\0', '\0')); // 1 <= num <= 3999
    num %= unit;

    unit = 100;
    number_of_units = num / unit;
    result.append(get_component(number_of_units, 'C', 'D', 'M'));
    num %= unit;

    unit = 10;
    number_of_units = num / unit;
    result.append(get_component(number_of_units, 'X', 'L', 'C'));
    num %= unit;

    // unit = 1;
    // number_of_units = num / unit;
    result.append(get_component(num, 'I', 'V', 'X'));
    // num %= unit;

    return result;
}
```
