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

## 13. 罗马数字转整数

英文题目名称: Roman to Integer

标签: 哈希表, 数学, 字符串

思路:

- 罗马数字表示法使用字母表示原子整数, 同时使用不同的字母拼接方向表示原子整数之间的加减法, 进而表示整个整数.
- 在罗马数字表示法下, 向右拓展代表增加, 并且拓展的单位总是不超过主单位, 例如 `I` 向右拓展为 `II`, 其中拓展的单位为 `I`, 主单位也为 `I`; 而向左拓展代表减少, 并且拓展的单位同样不超过主单位, 例如 `V` 向左拓展为 `IV`, 其中拓展的单位为 `I`, 主单位为 `V`.
- 因此当从右往左遍历一个罗马数字时, 如果发现单位减少说明遇到了向左拓展的单位, 需要减去; 反之则需要加上. 最终所得结果即为原始罗马数字所对应的整数.

代码:

```cpp
class Solution {
public:
    int romanToInt(string s) {
        int letter_to_number[numeric_limits<unsigned char>::max()] = {};

        letter_to_number['I'] = 1;
        letter_to_number['V'] = 5;
        letter_to_number['X'] = 10;
        letter_to_number['L'] = 50;
        letter_to_number['C'] = 100;
        letter_to_number['D'] = 500;
        letter_to_number['M'] = 1000;

        int result = 0;
        int previous_integer = -1;
        int current_integer = 0;

        for (int current_character_position = s.length() - 1;
             current_character_position >= 0; current_character_position--) {
            current_integer = letter_to_number[s[current_character_position]];

            // 从右向左遍历时若发现单位减小说明是向左拓展, 需要减去;
            // 否则说明是向右拓展, 需要增加:
            result += current_integer >= previous_integer ? current_integer
                                                          : (-current_integer);
            previous_integer = current_integer;
        }

        return result;
    }
};
```

## 14. 最长公共前缀

英文题目名称: Longest Common Prefix

标签: 字典树, 字符串

思路:

- 一开始以为可以使用二分法, 写完提交却狠狠报错, 然后才反应过来公共前缀是 "all of these elements" 类型的而不是 "one of these elements" 类型的, 无法使用二分法, 只能老老实实从左往右每个位置依次检查, 没什么技术含量.

代码:

```cpp
class Solution {
public:
    string longestCommonPrefix(vector<string> &strs) {
        if (strs.size() == 1) {
            return strs[0];
        }

        int minimum_string_length = numeric_limits<int>::max();
        for (const auto &a_string : strs) {
            minimum_string_length =
                min(minimum_string_length, (int)a_string.length());
        }

        int maximum_common_length = 0;
        for (int current_compare_position = 0;
             current_compare_position < minimum_string_length;
             current_compare_position++) {
            char current_compare_character = strs[0][current_compare_position];
            bool are_all_characters_the_same = true;
            for (int string_index = 1; string_index < strs.size();
                 string_index++) {
                if (strs[string_index][current_compare_position]
                    != current_compare_character) {
                    are_all_characters_the_same = false;
                    break;
                }
            }

            if (are_all_characters_the_same == false) {
                break;
            }

            maximum_common_length++;
        }

        string result = strs[0].substr(0, maximum_common_length);

        return result;
    }
};
```

## 15. 三数之和

英文题目名称: 3Sum

标签: 数组, 双指针, 排序

思路:

- 这道题我老早以前做过, 依稀记得做法是将三指针中的一个指针固定住以将原问题转化为双指针问题.
- 对于双指针问题, 自己今天在脑海里使用一个由一维数组导出的二维矩阵 (矩阵中每个元素为一个二元组) 对其进行模拟之后有了新的体会. 假设左指针对应二维矩阵的 $i$ 下标, 右指针对应二维矩阵的 $j$ 下标, 由于一维数组已升序排序, 所导出的二维矩阵同样关于 $i$ 和 $j$ 下标升序有序. 初始时 $(i, j)$ 位于矩阵的右上角, 此后每次检查右上角二元组之和时总是会发生下列三种情况之一 (注意由矩阵的定义可知其为对称阵):
  1. 右上角二元组之和小于目标值. 此时易知矩阵第一行中的任何二元组之和均不再可能大于等于目标值 (因为二维矩阵关于 $j$ 下标升序, 而 $j$ 下标此时位于最右端, 向左走只会令二元组之和减少). 这便将第一行以及 (由对称阵性质) 第一列的元素全部排除, 于是原矩阵归约为行数少 1, 列数少 1 的新矩阵, $(i, j)$ 向下移动 1 位 (对应于左指针右移 1 位), 移动后的 $(i, j)$ 位于新矩阵的右上角.
  1. 右上角二元组之和大于目标值. 同理, 易知矩阵最后一列以及最后一行的二元组可全部被排除, 原矩阵归约为行数少 1, 列数少 1 的新矩阵, $(i, j)$ 向左移动 1 位 (对应于右指针左移 1 位), 移动后的 $(i, j)$ 位于新矩阵的右上角.
  1. 右上角二元组之和等于目标值. 同理, 易知矩阵第一行, 第一列, 最后一行, 以及最后一列可同时被排除, 原矩阵归约为行数少 2, 列数少 2 的新矩阵, $(i, j)$ 向左下方移动 1 位 (对应于左/右指针分别向右/左移 1 位), 移动后的 $(i, j)$ 位于新矩阵的右上角.

  易知归约终止的标志是 $(i, j)$ 移动至原矩阵的主对角线, 即 $i = j$.
- 由于题目要求不能有重复, 首先想到的是将答案放到一个 set 中, 但直觉让我觉得很不舒服, 因此思路马上转移到如何编织合适的遍历策略来让结果天然是去重的. 最终的解决方案是首先对数组进行排序 (升序), 然后主指针从左到右进行遍历, 并在固定主指针的情况下使用双指针进行遍历. 规定主指针元素总是作为最终三元组中的最左端元素, 左右指针元素分别对应三元组中的中间和最右端元素, 于是只需要在固定主指针的情况下令左右指针的遍历范围为主指针右端的区间即可保证遍历不遗漏. 而遍历的不重复则是通过每次跳过和主/左/右指针元素值相同的所有元素实现的.
- 根据官方题解评论区一位网友的评论, 作为一种优化, 如果发现从主指针 `main_pointer` 开始的连续三个元素构成的三元组之和 `nums[main_pointer] + nums[main_pointer + 1] + nums[main_pointer + 2]` 已经大于 0, 由于数组升序有序, 余下未遍历的所有三元组之和均大于 0, 因此可终止全体遍历; 另一方面如果发现主指针元素和数组最右端两个元素构成的三元组之和 `nums[main_pointer] + nums[nums.size() - 1] + nums[nums.size() - 2]` 已经小于 0, 同样由于数组升序有序, 在固定主指针的情况下余下未遍历的所有三元组之和均小于 0, 因此可终止对左右指针的遍历, 直接令主指针跳到下一个位置.

代码:

```cpp
class Solution {
public:
    vector<vector<int>> threeSum(vector<int> &nums) {
        // 使数组升序有序:
        sort(nums.begin(), nums.end());

        vector<vector<int>> result;

        for (int main_pointer = 0; main_pointer < nums.size() - 2;) {
            // 最小三元组大于 0, 可终止全体遍历:
            if (nums[main_pointer] + nums[main_pointer + 1]
                    + nums[main_pointer + 2]
                > 0) {
                break;
            }

            // 固定主指针情况下的最大三元组小于 0, 可终止对左右指针的遍历,
            // 主指针跳到下一位置:
            if (nums[main_pointer] + nums[nums.size() - 1]
                    + nums[nums.size() - 2]
                < 0) {
                main_pointer++;
                continue;
            }

            int main_value = nums[main_pointer];
            int left_pointer = main_pointer + 1;
            int right_pointer = nums.size() - 1;

            while (left_pointer < right_pointer) {
                if (nums[left_pointer] + nums[right_pointer] == -main_value) {
                    result.push_back(vector<int>{main_value, nums[left_pointer],
                                                 nums[right_pointer]});

                    left_pointer++;

                    // 去重:
                    while (left_pointer < right_pointer
                           && nums[left_pointer] == nums[left_pointer - 1]) {
                        left_pointer++;
                    }

                    right_pointer--;

                    // 去重:
                    while (left_pointer < right_pointer
                           && nums[right_pointer] == nums[right_pointer + 1]) {
                        right_pointer--;
                    }
                } else if (nums[left_pointer] + nums[right_pointer]
                           < -main_value) {
                    left_pointer++;

                    // 去重:
                    while (left_pointer < right_pointer
                           && nums[left_pointer] == nums[left_pointer - 1]) {
                        left_pointer++;
                    }
                } else {
                    right_pointer--;

                    // 去重:
                    while (left_pointer < right_pointer
                           && nums[right_pointer] == nums[right_pointer + 1]) {
                        right_pointer--;
                    }
                }
            }

            main_pointer++;

            // 去重:
            while (main_pointer < nums.size()
                   && nums[main_pointer] == main_value) {
                main_pointer++;
            }
        }

        return result;
    }
};
```
