---
title: "LeetCode 题解归档 - 2"
hideSummary: true
date: 2024-07-16T17:15:23+08:00
draft: false
tags: ["LeetCode"]
series: ["LeetCode"]
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
  
  $$
  \begin{equation}
  \text{current_result} \times 10 + \text{digit} > 2147483647.
  \end{equation}
  $$
  
  - $x < 0$ 的情况同理.
  
  提交后发现执行效率不是很好.
- 最后通过查看题解了解到上述的条件还可以进一步优化, 优化到只需要检查 $\text{current_result}$ 就能判断溢出的程度, 根本不需要检查 $\text{digit}$.
  - 之所以能够进行优化是因为上述条件是离散的, 完全可以将所有情况列出来, 对同类情况进行合并同类项, 并对定义域进行降维.
  - 还是对于 $x > 0$ 的情况, 简单移项可知上述溢出条件又等价于
    
    $$
    \begin{equation}
    (\text{current_result} - 214748364) \times 10 > 7 - \text{digit}.
    \end{equation}
    $$

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
- 一开始还以为可以使用 [7. 整数反转](https://leetcode.cn/problems/reverse-integer/description/)中的用于判断是否溢出的结论进行速通, 后来发现该题的结论是有前提条件的, 并且在本题中并不成立, 因此只能手动判断是否溢出.

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

            int main_value = nums[main_pointer];

            // 固定主指针情况下的最大三元组小于 0, 可终止对左右指针的遍历,
            // 主指针跳到下一位置:
            if (nums[main_pointer] + nums[nums.size() - 1]
                    + nums[nums.size() - 2]
                < 0) {
                main_pointer++;

                // 去重:
                while (main_pointer < nums.size()
                       && nums[main_pointer] == main_value) {
                    main_pointer++;
                }

                continue;
            }
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

## 16. 最接近的三数之和

英文题目名称: 3Sum Closest

标签: 数组, 双指针, 排序

思路:

- 本题可以重用 [15. 三数之和](https://leetcode.cn/problems/3sum/description/)中的思路, 只需要稍微修改一下检查的目标即可:
  - 如果三元组之和等于目标 `target`, 那么答案已经找到, 可以直接 return;
  - 如果三元组之和小于目标 `target`, 那么更新历史最大上有界三元组之和 `maximum_upper_bounded_sum`;
  - 如果三元组之和大于目标 `target`, 那么更新历史最小下有界三元组之和 `minimum_lower_bounded_sum`.

  最终结果要么为 `target` 自身, 要么为 `maximum_upper_bounded_sum` 和 `minimum_lower_bounded_sum` 中距离 `target` 最近的那个.

{{< notice tip >}}

- 由于 `maximum_upper_bounded_sum` 和 `minimum_lower_bounded_sum` 的初值分别为 `int` 类型的两个极值, 最后在计算与 `target` 之间的距离时可能会溢出, 因此需要先强制转换为 `long` 类型.

{{< /notice >}}

代码:

```cpp
class Solution {
public:
    int threeSumClosest(vector<int> &nums, int target) {
        // 升序排序:
        sort(nums.begin(), nums.end());

        int maximum_upper_bounded_sum = numeric_limits<int>::min();
        int minimum_lower_bounded_sum = numeric_limits<int>::max();

        for (int main_pointer = 0; main_pointer < nums.size() - 2;) {
            int main_value = nums[main_pointer];

            int left_pointer = main_pointer + 1;
            int right_pointer = nums.size() - 1;

            while (left_pointer < right_pointer) {
                int left_value = nums[left_pointer];
                int right_value = nums[right_pointer];
                int current_sum = main_value + left_value + right_value;

                // 如果三元组之和等于目标, 则可以直接返回:
                if (current_sum == target) {
                    return target;
                }

                if (current_sum < target) {
                    maximum_upper_bounded_sum =
                        max(maximum_upper_bounded_sum, current_sum);
                    left_pointer++;

                    while (left_pointer < right_pointer
                           && nums[left_pointer] == left_value) {
                        left_pointer++;
                    }
                } else {
                    minimum_lower_bounded_sum =
                        min(minimum_lower_bounded_sum, current_sum);
                    right_pointer--;

                    while (left_pointer < right_pointer
                           && nums[right_pointer] == right_value) {
                        right_pointer--;
                    }
                }
            }

            main_pointer++;

            while (main_pointer < nums.size()
                   && nums[main_pointer] == main_value) {
                main_pointer++;
            }
        }

        // 为避免溢出, 需要强制转换为 `long` 类型:
        int result = abs((long)target - maximum_upper_bounded_sum)
                             < abs((long)target - minimum_lower_bounded_sum)
                         ? maximum_upper_bounded_sum
                         : minimum_lower_bounded_sum;

        return result;
    }
};
```

## 17. 电话号码的字母组合

英文题目名称: Letter Combinations of a Phone Number

标签: 哈希表, 字符串, 回溯

思路:

- 没什么技术含量, 就是一个简单的 DFS.

代码:

```cpp
class Solution {
public:
    unordered_map<char, vector<char>> digit_to_letters{
        {'2', {'a', 'b', 'c'}}, {'3', {'d', 'e', 'f'}},
        {'4', {'g', 'h', 'i'}}, {'5', {'j', 'k', 'l'}},
        {'6', {'m', 'n', 'o'}}, {'7', {'p', 'q', 'r', 's'}},
        {'8', {'t', 'u', 'v'}}, {'9', {'w', 'x', 'y', 'z'}}};
    void DFS_for_letter_combinations(vector<string> &letter_combinations,
                                     char *collected_letters,
                                     int current_digit_position,
                                     const string &digits);
    vector<string> letterCombinations(string &digits);
};

void Solution::DFS_for_letter_combinations(vector<string> &letter_combinations,
                                           char *collected_letters,
                                           int current_digit_position,
                                           const string &digits) {
    if (current_digit_position == digits.length()) {
        letter_combinations.emplace_back(collected_letters);
        return;
    }

    char current_digit = digits[current_digit_position];

    for (char current_letter : digit_to_letters[current_digit]) {
        collected_letters[current_digit_position] = current_letter;
        DFS_for_letter_combinations(letter_combinations, collected_letters,
                                    current_digit_position + 1, digits);
    }

    return;
}

vector<string> Solution::letterCombinations(string &digits) {
    if (digits.empty()) {
        return {};
    }

    vector<string> letter_combinations;
    char *collected_letters = new char[digits.size() + 1];
    collected_letters[digits.size()] = '\0';

    DFS_for_letter_combinations(letter_combinations, collected_letters, 0,
                                digits);

    delete[] collected_letters;

    return letter_combinations;
}
```

## 18. 四数之和

英文题目名称: 4Sum

标签: 数组, 双指针, 排序

思路:

- 和 [15. 三数之和](https://leetcode.cn/problems/3sum/description/)同根同源, 三数之和下固定的是主指针, 拓展到四数之和下只需要同时固定主指针和副指针即可, 余下的部分由双指针解决.
- 本题的优化方式和三数之和中的相同, 但由于增加了一个副指针, 因此主副指针的跳过方式有一点不同, 需要特别处理.

代码:

```cpp
class Solution {
public:
    vector<vector<int>> fourSum(vector<int> &nums, int target) {
        // 合理性检查:
        if (nums.size() < 4) {
            return {};
        }

        sort(nums.begin(), nums.end());

        vector<vector<int>> quadruples;

        for (int main_pointer = 0; main_pointer < nums.size() - 3;) {
            int main_value = nums[main_pointer];

            for (int second_pointer = main_pointer + 1;
                 second_pointer < nums.size() - 2;) {
                int second_value = nums[second_pointer];

                // 剪枝:
                if ((long long)main_value + second_value
                        + nums[second_pointer + 1] + nums[second_pointer + 2]
                    > target) {
                    break;
                }

                // 剪枝:
                if ((long long)main_value + second_value + nums[nums.size() - 1]
                        + nums[nums.size() - 2]
                    < target) {
                    second_pointer++;

                    while (second_pointer < nums.size() - 2
                           && nums[second_pointer] == second_value) {
                        second_pointer++;
                    }

                    continue;
                }

                int left_pointer = second_pointer + 1;
                int right_pointer = nums.size() - 1;

                while (left_pointer < right_pointer) {
                    int left_value = nums[left_pointer];
                    int right_value = nums[right_pointer];
                    long long current_sum = (long long)main_value + second_value
                                            + left_value + right_value;

                    if (current_sum == target) {
                        quadruples.emplace_back(initializer_list<int>{
                            main_value, second_value, left_value, right_value});

                        left_pointer++;
                        right_pointer--;

                        while (left_pointer < right_pointer
                               && nums[left_pointer] == left_value) {
                            left_pointer++;
                        }
                        while (left_pointer < right_pointer
                               && nums[right_pointer] == right_value) {
                            right_pointer--;
                        }
                    } else if (current_sum < target) {
                        left_pointer++;

                        while (left_pointer < right_pointer
                               && nums[left_pointer] == left_value) {
                            left_pointer++;
                        }
                    } else {
                        right_pointer--;

                        while (left_pointer < right_pointer
                               && nums[right_pointer] == right_value) {
                            right_pointer--;
                        }
                    }
                }

                second_pointer++;

                while (second_pointer < nums.size() - 2
                       && nums[second_pointer] == second_value) {
                    second_pointer++;
                }
            }

            main_pointer++;

            while (main_pointer < nums.size() - 3
                   && nums[main_pointer] == main_value) {
                main_pointer++;
            }
        }

        return quadruples;
    }
};
```

## 19. 删除链表的倒数第 N 个结点

英文题目名称: Remove Nth Node From End of List

标签: 链表, 双指针

思路:

- 脑筋急转弯类型的题目, 第一次做的时候被坑过一遍, 后边想忘都忘不了. 由于待删除结点相对于链表末结点的距离已知, 同时一个结点是否为链表末结点是可以检测的, 因此可以使用双指针的方法, 先将左右指针错开与待删除结点和链表末结点之间的距离 (即 `n - 1`) 相同的距离, 然后左右指针再同步前进, 当右指针指向链表末结点时左指针便必然指向待删除结点.
- 由于本题是单链表, 需要将指针指向待删除结点的父结点, 为了处理待删除结点恰好为头结点的情况, 可以使用一个伪头结点 (dummy head) 作为头结点的父结点 (需要注意的是待删除结点的父结点与链表末结点之间的距离为 `n`).

代码:

```cpp
class Solution {
public:
    ListNode *removeNthFromEnd(ListNode *head, int n) {
        ListNode *dummy_head = new ListNode();
        dummy_head->next = head;

        ListNode *delete_position = dummy_head;

        // 首先拉开距离:
        ListNode *list_tail = delete_position;
        for (int i = 0; i < n; i++) {
            list_tail = list_tail->next;
        }

        // 然后固定距离, 同步前进:
        while (list_tail->next) {
            delete_position = delete_position->next;
            list_tail = list_tail->next;
        }

        // 最后停下来的位置即为待删除位置:
        delete_position->next = delete_position->next->next;

        // 还需要考虑删除头结点的情况:
        head = dummy_head->next;
        delete dummy_head;

        return head;
    }
};
```

## 20. 有效的括号

英文题目名称: Valid Parentheses

标签: 栈, 字符串

思路:

- 第一反应是存储左括号, 并在右括号到来时检查栈顶是否为对应的左括号. 但是这题非常细节, 每次提交都会发现一个坑, 归纳如下:
  - 即使通过所有左括号匹配也并不意味着整个模式匹配成功, 因为此时栈中还有可能剩余一些未匹配的左括号.
  - 匹配左括号时需要先检查栈是否为空.
- 看了自己很早之前提交的一次解答, 发现其实完全可以存储右括号, 并在右括号到来时检查栈顶是否为**自身**. 这样做就把匹配左括号的三个 `if` 语句归约为一个 `==` 判断, 时间效率直接爆炸.

代码 (匹配左括号版本):

```cpp
class Solution {
public:
    bool isValid(string s) {
        stack<char> stacked_left_brackets;

        for (char bracket : s) {
            switch (bracket) {
            case '(':
            case '{':
            case '[':
                stacked_left_brackets.push(bracket);
                break;

            case ')':
                if (stacked_left_brackets.empty()
                    || stacked_left_brackets.top() != '(') {
                    return false;
                }

                stacked_left_brackets.pop();
                break;

            case '}':
                if (stacked_left_brackets.empty()
                    || stacked_left_brackets.top() != '{') {
                    return false;
                }

                stacked_left_brackets.pop();
                break;

            case ']':
                if (stacked_left_brackets.empty()
                    || stacked_left_brackets.top() != '[') {
                    return false;
                }

                stacked_left_brackets.pop();
                break;
            }
        }

        return stacked_left_brackets.empty();
    }
};
```

代码 (匹配右括号自身版本):

```cpp
class Solution {
public:
    bool isValid(string s) {
        stack<char> stacked_left_brackets;

        for (char bracket : s) {
            switch (bracket) {
            case '(':
                stacked_left_brackets.push(')');
                break;

            case '{':
                stacked_left_brackets.push('}');
                break;

            case '[':
                stacked_left_brackets.push(']');
                break;

            case ')':
            case '}':
            case ']':
                if (stacked_left_brackets.empty()
                    || stacked_left_brackets.top() != bracket) {
                    return false;
                }

                stacked_left_brackets.pop();
                break;
            }
        }

        return stacked_left_brackets.empty();
    }
};
```

## 21. 合并两个有序链表

英文题目名称: Merge Two Sorted Lists

标签: 递归, 链表

思路:

- 没什么技术含量. 使用 `merged_list_dummy_head` 作为合并链表的伪头结点, 然后老老实实一个一个将 `list1` 和 `list2` 中的结点往合并链表中搬, 搬到最后两个链表必然不能同时为非空, 于是只需要将非空的那个链表中的余下结点拼接至当前合并链表的末尾即可 (易知拼接后的合并链表仍然有序).

代码:

```cpp
class Solution {
public:
    ListNode *mergeTwoLists(ListNode *list1, ListNode *list2) {
        ListNode *merged_list_dummy_head = new ListNode();
        ListNode *merged_list_tail = merged_list_dummy_head;

        while (list1 && list2) {
            if (list1->val < list2->val) {
                merged_list_tail->next = list1;
                list1 = list1->next;
            } else {
                merged_list_tail->next = list2;
                list2 = list2->next;
            }

            merged_list_tail = merged_list_tail->next;
        }

        merged_list_tail->next = list1 ? list1 : list2;

        ListNode *merged_list_head = merged_list_dummy_head->next;
        delete merged_list_dummy_head;

        return merged_list_head;
    }
};
```

## 22. 括号生成

英文题目名称: Generate Parentheses

标签: 字符串, 动态规划, 回溯

思路:

- 第一反应是分治法, 例如将 $n = 3$ 拆成 $n = 2 + 1$, $n = 1 + 1 + 1$ 等等, 但越想越觉得思维难度颇高 (当时还没有想到其实只需要枚举第一个拆分部分的长度并结合递归即可构造出所有有效拆分), 同时还需要像动态规划那样把中间值保存起来, 这令我很不舒服. 于是放弃分治法, 转向回溯法:
- 首先观察到在 $n$ 固定的情况下, 任何一个有效括号组合的长度均为 $2n$, 因此可以从左往右对一个由 $2n$ 个槽位构成的字符串中的每一个槽位枚举该槽位应该放左括号还是右括号来进行生成. 其次也并不是每一个槽位都可以放左括号和右括号:
  - 能够放左括号当且仅当还有未放置的左括号剩余;
  - 能够放右括号当且仅当还有未放置的 (之前放置的左括号所需要匹配的) 右括号剩余.

代码:

```cpp
class Solution {
public:
    void
    DFS_for_generating_parenthesis(vector<string> &valid_bracket_combinations,
                                   int number_of_left_brackets_left,
                                   int number_of_right_brackets_left,
                                   vector<char> &pushed_brackets);
    vector<string> generateParenthesis(int n);
};

void Solution::DFS_for_generating_parenthesis(
    vector<string> &valid_bracket_combinations,
    int number_of_left_brackets_left, int number_of_right_brackets_left,
    vector<char> &pushed_brackets) {
    // 左右括号均已放置完毕:
    if (!number_of_left_brackets_left && !number_of_right_brackets_left) {
        valid_bracket_combinations.emplace_back(pushed_brackets.begin(),
                                                pushed_brackets.end());
        return;
    }

    // 可以放置左括号 (此时左括号剩余数量减少, 同时右括号剩余数量增加):
    if (number_of_left_brackets_left) {
        pushed_brackets.push_back('(');
        DFS_for_generating_parenthesis(
            valid_bracket_combinations, number_of_left_brackets_left - 1,
            number_of_right_brackets_left + 1, pushed_brackets);
        pushed_brackets.pop_back();
    }

    // 可以放置右括号 (此时仅减少右括号剩余数量):
    if (number_of_right_brackets_left) {
        pushed_brackets.push_back(')');
        DFS_for_generating_parenthesis(
            valid_bracket_combinations, number_of_left_brackets_left,
            number_of_right_brackets_left - 1, pushed_brackets);
        pushed_brackets.pop_back();
    }

    return;
}

vector<string> Solution::generateParenthesis(int n) {
    vector<string> valid_bracket_combinations;
    int number_of_left_brackets_left = n;
    int number_of_right_brackets_left = 0;
    vector<char> pushed_brackets;

    DFS_for_generating_parenthesis(
        valid_bracket_combinations, number_of_left_brackets_left,
        number_of_right_brackets_left, pushed_brackets);

    return valid_bracket_combinations;
}
```

## 23. 合并 K 个升序链表

英文题目名称: Merge k Sorted Lists

标签: 链表, 分治, 堆（优先队列）, 归并排序

思路:

- 不知道这题以前做没做过, 但是一看到 k 个有序链表脑子里便自动过拟合到了败者树和堆排序那边去了. 但这里不是硬盘, 因此直接使用堆排序即可. 堆的操作的复杂度为 $O(\log(k))$, 共需要比较 $kn$ 次, 每次都需要至多一次插入和至多一次删除, 因此时间复杂度为 $O(kn\log(k))$.
- 最朴素的做法是顺次合并两个相邻的有序链表, 但是这样做的话每次合并得到的新链表长度均为两个旧链表的长度之和, 累积效应显著 (时间复杂度为 $O(k^2n)$). 优化的话可以考虑 Huffman Tree (或等价的 2 路归并), 每一轮合并的时间复杂度均为 $O(kn)$, 共合并 $\log(k)$ 轮, 因此总的时间复杂度仍为 $O(kn\log(k))$.

代码:

```cpp
struct ListNodeHeapWrapperNode {
    ListNode *list_node;
    int list_node_value;
    int list_index;
};

class ListNodeHeapWrapperNodeLessThanRule {
public:
    bool operator()(ListNodeHeapWrapperNode *const &node_1,
                    const ListNodeHeapWrapperNode *const &node_2) {
        return node_1->list_node_value > node_2->list_node_value;
    }
};

class Solution {
public:
    ListNode *mergeKLists(vector<ListNode *> &lists) {
        priority_queue<ListNodeHeapWrapperNode *,
                       vector<ListNodeHeapWrapperNode *>,
                       ListNodeHeapWrapperNodeLessThanRule>
            k_smallest_heap;

        for (int list_index = 0; list_index < lists.size(); list_index++) {
            if (lists[list_index] == nullptr) {
                continue;
            }

            ListNodeHeapWrapperNode *heap_node = new ListNodeHeapWrapperNode();
            heap_node->list_node = lists[list_index];
            heap_node->list_node_value = lists[list_index]->val;
            heap_node->list_index = list_index;

            k_smallest_heap.push(heap_node);

            lists[list_index] = lists[list_index]->next;
        }

        ListNode *merged_list_dummy_head = new ListNode();
        ListNode *merged_list_tail = merged_list_dummy_head;

        while (!k_smallest_heap.empty()) {
            auto smallest_heap_node = k_smallest_heap.top();
            k_smallest_heap.pop();

            merged_list_tail->next = smallest_heap_node->list_node;
            merged_list_tail = merged_list_tail->next;

            if (lists[smallest_heap_node->list_index] == nullptr) {
                delete smallest_heap_node;
            } else {
                auto &reused_heap_node = smallest_heap_node;
                auto list_index = smallest_heap_node->list_index;

                reused_heap_node->list_node = lists[list_index];
                reused_heap_node->list_node_value = lists[list_index]->val;

                lists[list_index] = lists[list_index]->next;

                k_smallest_heap.push(reused_heap_node);
            }
        }

        ListNode *merged_list_head = merged_list_dummy_head->next;

        delete merged_list_dummy_head;

        return merged_list_head;
    }
};
```
