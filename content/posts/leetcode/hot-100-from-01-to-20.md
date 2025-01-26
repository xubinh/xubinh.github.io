---
title: "力扣 Hot 100 题解归档 (01-20)"
hideSummary: true
date: 2025-01-27T00:18:39+08:00
draft: false
tags: ["leetcode"]
series: ["leetcode"]
author: ["xubinh"]
type: posts
math: true
---

## 1. 两数之和 (Two Sum)

标签: 哈希表

解法:

- 假设数组中存在两个数 `A` 和 `B` 相加等于目标值, 并且 `A` 位于 `B` 的左边, 那么当遍历到 `B` 时 `A` 肯定也已经遍历过了. 反过来说只要我们能够在遍历到 `B` 的时候知道数组中确实存在 `A`, 那么我们就找到了答案. 我们可以用一个哈希表来保存当前已经遍历到的所有数字到它们下标的映射, 对于当前遍历到的数字, 如果它是正确答案的那一对数字的其中一个, 不失一般性我们可以假设它是右边那一个, 那么此时左边那一个数字已经在哈希表中了, 于是我们便可以返回正确答案.

代码:

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int> &nums, int target) {
        std::unordered_map<int, int> visited_number_2_position;

        for (int current_position = 0; current_position < nums.size();
             current_position++) {
            const int current_number = nums[current_position];
            const int complement_number = target - current_number;

            auto it = visited_number_2_position.find(complement_number);

            if (it != visited_number_2_position.end()) {
                const int complement_position = it->second;

                return {current_position, complement_position};
            }

            visited_number_2_position[current_number] = current_position;
        }

        throw std::runtime_error("never reaches here");
    }
};
```

## 2. 两数相加 (Add Two Numbers)

标签: 链表

解法:

- 直接暴力模拟: 同时对两个链表向前迭代, 每一次循环首先抓住两个链表的当前结点, 如果结点为非空那最好, 如果结点为空则将其视为一个值为零的非空结点, 然后将这两个结点的值与此前的进位相加并进一步计算余数和新的进位.
- 由于可能出现两个结点均为空但进位为非零的情况, 并且该情况无法在循环中进行处理, 因此必须在循环结束时单独进行检查.

代码:

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode *addTwoNumbers(ListNode *l1, ListNode *l2) {
        ListNode merged_list_dummy_head;
        ListNode *merged_list_tail = &merged_list_dummy_head;
        int carry = 0;

        while (l1 || l2) {
            const int current_value_1 = l1 ? l1->val : 0;
            const int current_value_2 = l2 ? l2->val : 0;
            const int current_sum = current_value_1 + current_value_2 + carry;

            merged_list_tail->next = new ListNode(current_sum % 10);
            merged_list_tail = merged_list_tail->next;
            carry = current_sum / 10;

            if (l1) {
                l1 = l1->next;
            }

            if (l2) {
                l2 = l2->next;
            }
        }

        if (carry > 0) {
            merged_list_tail->next = new ListNode(carry);
        }

        return merged_list_dummy_head.next;
    }
};
```

## 3. 无重复字符的最长子串 (Longest Substring Without Repeating Characters)

标签: 哈希表, 滑动窗口

解法:

- 由于是子串, 因此直接哈希表 + 滑动窗口模拟即可. 哈希表直接使用一个长为 256 的数组进行模拟.
- 边界条件需要仔细处理, 具体见代码注释.

代码:

```cpp
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        if (s.empty()) {
            return 0;
        }

        int non_duplicate_range_begin = 0;
        int non_duplicate_range_end = 0;
        int max_non_duplicate_range_size = 1;
        int char_2_appear_times[256]{};

        while (true) {
            while (non_duplicate_range_end < s.size()) {
                const char current_char = s[non_duplicate_range_end];

                // 如果加入下一个字符将会使得当前范围中出现重复字符则停止
                if (char_2_appear_times[current_char] > 0) {
                    break;
                }

                char_2_appear_times[current_char] = 1;
                non_duplicate_range_end++;
            }

            // 注: 到这里为止要么出现重复字符,
            // 要么没有重复字符但已经遍历到字符串尾

            max_non_duplicate_range_size =
                std::max(max_non_duplicate_range_size,
                         non_duplicate_range_end - non_duplicate_range_begin);

            // 如果已经遍历到字符串尾则直接返回
            if (non_duplicate_range_end == s.size()) {
                return max_non_duplicate_range_size;
            }

            const char duplecate_char = s[non_duplicate_range_end];

            // 不断排出字符, 直到排出重复的那个字符为止
            while (non_duplicate_range_begin < non_duplicate_range_end) {
                const char current_char = s[non_duplicate_range_begin];
                non_duplicate_range_begin++;

                char_2_appear_times[current_char]--;

                if (current_char == duplecate_char) {
                    break;
                }
            }
        }

        throw std::runtime_error("never reaches here");
    }
};
```

## 4. 寻找两个正序数组的中位数 (Median of Two Sorted Arrays)

标签: 二分查找

解法:

- 考虑将两个数组合并后得到的新数组, 由于所要求的是中位数, 如果新数组的总长度为奇数, 那么中位数就是位于新数组的正中间的那个数, 反之则是位于新数组正中间的两个相邻数的平均数. 不论哪种情况, 我们总是要找到新数组的 "正中间", 而又由于原始两个数组已经有序, 因此可以对原始两个数组进行切割 (cut) 并拼接出 "正中间". 由于 "正中间" 的长度固定, 我们可以使用二分法对数组一的切割位置进行枚举, 一旦数组一的切割位置确定, 数组二的切割位置也随之确定.

代码:

```cpp
class Solution {
public:
    int get_left_max(const std::vector<int> &nums1, const int cut_1,
                     const std::vector<int> &nums2, const int cut_2) {
        return cut_1 == 0
                   ? nums2[cut_2 - 1]
                   : (cut_2 == 0
                          ? nums1[cut_1 - 1]
                          : (std::max(nums1[cut_1 - 1], nums2[cut_2 - 1])));
    }

    int get_right_min(const std::vector<int> &nums1, const int cut_1,
                      const std::vector<int> &nums2, const int cut_2) {
        return cut_1 == nums1.size()
                   ? nums2[cut_2]
                   : (cut_2 == nums2.size()
                          ? nums1[cut_1]
                          : (std::min(nums1[cut_1], nums2[cut_2])));
    }

    bool check_if_cut_1_smaller_than_cut_2(const std::vector<int> &nums1,
                                           const int cut_1,
                                           const std::vector<int> &nums2,
                                           const int cut_2) {
        return cut_1 == 0
                   ? true
                   : (cut_2 == 0 ? false
                                 : (nums1[cut_1 - 1] < nums2[cut_2 - 1]));
    }

    double findMedianSortedArrays(vector<int> &nums1, vector<int> &nums2) {
        if (nums1.empty() || nums2.empty()) {
            const auto &nums = nums1.empty() ? nums2 : nums1;

            const int nums_size = nums.size();
            const bool is_odd = nums_size % 2 == 1;

            return is_odd
                       ? nums[nums_size / 2]
                       : (nums[nums_size / 2] + nums[nums_size / 2 - 1]) / 2.0;
        }

        if (nums1.size() > nums2.size()) {
            swap(nums1, nums2);
        }

        const int nums1_size = nums1.size();
        const int nums2_size = nums2.size();
        const int total_size = nums1_size + nums2_size;
        const bool is_odd = total_size % 2 == 1;
        const int cut_1_plus_cut_2 = (total_size + 1) / 2;

        int left = std::max(0, cut_1_plus_cut_2 - nums2_size);
        int right = std::min(nums1_size, cut_1_plus_cut_2) + 1;

        while (true) {
            const int cut_1 = left + (right - left) / 2;
            const int cut_2 = cut_1_plus_cut_2 - cut_1;

            const int left_max = get_left_max(nums1, cut_1, nums2, cut_2);
            const int right_min = get_right_min(nums1, cut_1, nums2, cut_2);

            if (left_max <= right_min) {
                return is_odd ? left_max : ((left_max + right_min) / 2.0);
            }

            const bool is_cut_1_smaller_than_cut_2 =
                check_if_cut_1_smaller_than_cut_2(nums1, cut_1, nums2, cut_2);

            if (is_cut_1_smaller_than_cut_2) {
                left = cut_1 + 1;
            }

            else {
                right = cut_1;
            }
        }

        throw std::runtime_error("never reaches here");
    }
};
```

## 5. 最长回文子串 (Longest Palindromic Substring)

标签: 动态规划

解法:

- 直接使用最自然的思路, 即只要一个回文子串的两端的两个字符也相等, 那么该回文子串就能向两边同时拓展一位形成新的更长的回文子串. 因此只需要对原始字符串中的所有位置 (包括奇数位置和偶数位置) 进行枚举并向外拓展即可. 最终答案等于所有这些位置上经过拓展得到的最长回文子串的长度的最大值.
- 本题还有一种称为 Manacher 算法的解法, 其时间复杂度为 $O(n)$, 但其做法过于复杂, 一般不作为面试内容.

代码:

```cpp
class Solution {
public:
    bool check_if_extendable(const std::string &s,
                             const int current_palindromic_sub_string_begin,
                             const int current_palindromic_sub_string_size) {
        const int next_left = current_palindromic_sub_string_begin - 1;
        const int next_right = current_palindromic_sub_string_begin
                               + current_palindromic_sub_string_size;

        return next_left >= 0 && next_right < s.size()
               && s[next_left] == s[next_right];
    }

    string longestPalindrome(string s) {
        int longest_palindromic_sub_string_begin = 0;
        int longest_palindromic_sub_string_size = 1;

        // 枚举奇数位置
        for (int current_position = 0; current_position < s.size();
             current_position++) {

            // 初始长度为 1
            int current_longest_palindromic_sub_string_begin = current_position;
            int current_longest_palindromic_sub_string_size = 1;

            while (check_if_extendable(
                s, current_longest_palindromic_sub_string_begin,
                current_longest_palindromic_sub_string_size)) {

                current_longest_palindromic_sub_string_begin--;
                current_longest_palindromic_sub_string_size += 2;
            }

            if (current_longest_palindromic_sub_string_size
                > longest_palindromic_sub_string_size) {

                longest_palindromic_sub_string_begin =
                    current_longest_palindromic_sub_string_begin;
                longest_palindromic_sub_string_size =
                    current_longest_palindromic_sub_string_size;
            }
        }

        // 枚举偶数位置
        for (int current_position = 0; current_position < s.size() - 1;
             current_position++) {

            // 初始长度为 0
            int current_longest_palindromic_sub_string_begin =
                current_position + 1;
            int current_longest_palindromic_sub_string_size = 0;

            while (check_if_extendable(
                s, current_longest_palindromic_sub_string_begin,
                current_longest_palindromic_sub_string_size)) {

                current_longest_palindromic_sub_string_begin--;
                current_longest_palindromic_sub_string_size += 2;
            }

            if (current_longest_palindromic_sub_string_size
                > longest_palindromic_sub_string_size) {

                longest_palindromic_sub_string_begin =
                    current_longest_palindromic_sub_string_begin;
                longest_palindromic_sub_string_size =
                    current_longest_palindromic_sub_string_size;
            }
        }

        return s.substr(longest_palindromic_sub_string_begin,
                        longest_palindromic_sub_string_size);
    }
};
```

## 10. 正则表达式匹配 (Regular Expression Matching)

标签: 动态规划

解法:

- 首先观察模式串. 模式串中含有两种模式零件, 分别是长度为 1 的固定匹配单字符的零件和长度为 2 的可匹配零到多个字符的零件. 下面对其分情况讨论:
  - 如果当前遍历到的是长度为 1 的零件, 那么字符串 $(s_0, s_1, \dots, s_{j - 1})$ 能够匹配当且仅当 $s_{j - 1}$ 等于该零件并且 $(s_0, s_1, \dots, s_{j - 2})$ 也能够匹配, 因此状态成功转移到了子串当中. 对于空串, 由于该零件必须匹配一个字符, 因此空串必定失配.
  - 如果当前遍历到的是长度为 2 的零件, 那么字符串 $(s_0, s_1, \dots, s_{j - 1})$ 能够匹配当且仅当后半部分 $(s_k, s_{k + 1}, \dots, s_{j - 1})$ 中的所有字符均等于该零件并且前半部分 $(s_0, s_1, \dots, s_{k - 1})$ 也能够匹配, 其中 $k \in [0, j - 1]$. 对于空串, 由于该零件能够匹配零个字符, 因此空串同样需要转移状态并进行判定.
- 由于在遍历模式零件的过程中, 当前迭代的结果仅由上一轮迭代的结果即可完全确定, 因此可以使用滚动数组的方法将空间复杂度优化至 $O(n)$.
- 在开始时可以预先将模式串分解为模式零件, 这可以通过定义一个帮手类来实现, 这样做不仅能够有效降低思维难度, 同时也能降低程序的出错概率.
- 此外解题过程中需要格外注意边界条件的检查, 包括刚开始时需要约定空模式串能够匹配空字符串等等.

代码:

```cpp
class Solution {
private:
    class PatternVector {
    public:
        PatternVector(const std::string &pattern)
            : _pattern_vector(PatternVector::_get_pattern_vector(pattern)) {
        }

        const size_t size() const {
            return _pattern_vector.size();
        }

        const std::string &operator[](const size_t position) const {
            return _pattern_vector[position];
        }

    private:
        static std::vector<std::string>
        _get_pattern_vector(const std::string &pattern) {
            std::vector<std::string> pattern_vector;
            int current_position = 0;

            while (current_position < pattern.size()) {
                std::string current_pattern;
                current_pattern.push_back(pattern[current_position]);
                current_position++;

                if (current_position < pattern.size()
                    && pattern[current_position] == '*') {
                    current_pattern.push_back('*');
                    current_position++;
                }

                pattern_vector.emplace_back(std::move(current_pattern));
            }

            return pattern_vector;
        }

        const std::vector<std::string> _pattern_vector;
    };

public:
    bool isMatch(string s, string p) {
        // 将模式串拆解为一个个零件
        const PatternVector pattern_vector(p);

        const int I = pattern_vector.size();
        const int J = s.size();

        std::vector<char> dp(J + 1, false);
        dp[0] = true;

        // 遍历每个模式零件
        for (int i = 0; i < I; i++) {
            const std::string &current_pattern = pattern_vector[i];
            const char current_pattern_char = current_pattern[0];

            // 如果是形如 `a*` 这样的零件
            if (current_pattern.size() > 1) {
                for (int j = s.size(); j >= 0; j--) {
                    bool check_result = dp[j];

                    if (check_result) {
                        continue;
                    }

                    for (int k = j - 1; !check_result && k >= 0
                                        && (current_pattern_char == '.'
                                            || s[k] == current_pattern_char);
                         k--) {

                        check_result = dp[k];
                    }

                    dp[j] = check_result;
                }

                // 注意空串也需要状态转移并进行判断,
                // 而该情况已经被归纳进上述循环中 (注意上述循环中 `j`
                // 可以等于零)
            }

            // 如果是形如 `a` 或者 `.` 这样的零件
            else {
                for (int j = s.size(); j > 0; j--) {
                    dp[j] = (current_pattern_char == '.'
                             || s[j - 1] == current_pattern_char)
                                ? dp[j - 1]
                                : false;
                }

                // 空串必然失配
                dp[0] = false;
            }
        }

        return dp[J];
    }
};
```

## 11. 盛最多水的容器 (Container With Most Water)

标签: 双指针

解法:

- 这道题之所以可以使用双指针的解法, 是因为两堵墙在构成一个水箱的过程中对水箱的高度施加了内在的限制, 即水箱的高度只能是两堵墙的高度中的较小值, 这就为不同的两堵墙的组合之间提供了偏序关系, 降低了枚举的自由度. 下面详细解释如何使用双指针的解法解决本题.
- 首先考虑两个指针 `i` 和 `j`, 分别表示两堵墙在数组中的下标, 其中 `0 <= i < j < heights.size()`. 将所有不同的 `(i, j)` 组合按照下标排序形成一个二维矩阵, 其中所有组合构成了矩阵的主对角线以上的部分 (上三角). 初始时 `(i, j)` 位于矩阵的右上角, 表示最左边的那堵墙和最右边的那堵墙形成的水箱. 下面分情况讨论:
  - 如果左墙的高度小于等于右墙, 即 `height[i] <= height[j]`, 此时考虑移动右指针会发生什么:
    - 如果右墙在移动后高度变高或者不变, 即 `height[j - 1] >= height[j]`, 此时仍然有 `height[i] <= height[j - 1]`, 水箱的高度不变, 而宽度却减小, 于是面积必然减小, 因此该组合可以忽略;
    - 如果右墙在移动后高度变低, 即 `height[j - 1] < height[j]`, 此时水箱的高度和宽度均减小, 于是面积必然减小, 因此该组合可以忽略.

    可以看到在左墙的高度小于右墙, 即 `height[i] < height[j]` 的情况下, 所有左指针等于 `i` 的组合均被忽略, 如果放到二维矩阵的角度上看, 这就相当于矩阵第 1 行中的所有组合均被忽略, 因此我们可以将 `i` 指针向下移动, 移动后的 `(i, j)` 位于新的矩阵的右上角, 原问题转化为等价的子问题.
  - 如果左墙的高度大于等于右墙, 同理可知此时可以将 `j` 指针向左移动, 移动后的 `(i, j)` 位于新的矩阵的右上角, 原问题转化为等价的子问题.

  综合上述两点即可证明双指针做法的正确性. 最终的做法总结如下:

  - 更新最大面积;
  - 如果左墙的高度小于右墙, 即 `height[i] < height[j]`, 则 `i++`;
  - 如果左墙的高度大于右墙, 即 `height[i] >= height[j]`, 则 `j--`.

代码:

```cpp
class Solution {
public:
    int maxArea(vector<int> &height) {
        int left_ptr = 0;
        int right_ptr = height.size() - 1;
        int max_area = std::numeric_limits<int>::min();

        while (left_ptr < right_ptr) {
            const int left_wall_height = height[left_ptr];
            const int right_wall_height = height[right_ptr];

            const int rectangle_height =
                std::min(left_wall_height, right_wall_height);
            const int rectangle_width = right_ptr - left_ptr;

            const int current_volume = rectangle_height * rectangle_width;

            max_area = std::max(max_area, current_volume);

            if (left_wall_height >= right_wall_height) {
                right_ptr--;
            }

            else {
                left_ptr++;
            }
        }

        return max_area;
    }
};
```

## 15. 三数之和 (3Sum)

标签: 排序, 双指针

解法:

- 这题和 "1. 两数之和 (Two Sum)" 不同, 两数之和中因为时间复杂度必须控制在 $O(n)$ 的缘故, 只能使用线性时间的哈希表而不是排序 + 双指针的解法来求解. 而在三数之和的情况下其时间复杂度最低也至少为 $O(n^2)$, 我们完全可以对数组进行排序然后采用双指针的方法获取所有合法三元组.

代码:

```cpp
class Solution {
public:
    vector<vector<int>> threeSum(vector<int>& nums) {
        auto nums_sorted = nums;

        std::sort(nums_sorted.begin(), nums_sorted.end());

        std::vector<std::vector<int>> valid_triples;

        int first_ptr = 0;

        while (first_ptr < nums_sorted.size() - 2) {
            const int first_number = nums_sorted[first_ptr];
            const int target = 0 - first_number;

            int second_ptr = first_ptr + 1;
            int third_ptr = nums_sorted.size() - 1;

            while (second_ptr < third_ptr) {
                const int second_number = nums_sorted[second_ptr];
                const int third_number = nums_sorted[third_ptr];

                if (second_number + third_number >= target) {
                    if (second_number + third_number == target) {
                        valid_triples.push_back(
                            {first_number, second_number, third_number});
                    }

                    third_ptr--;

                    while (second_ptr < third_ptr &&
                           nums_sorted[third_ptr] == third_number) {
                        third_ptr--;
                    }
                }

                else {
                    second_ptr++;

                    while (second_ptr < third_ptr &&
                           nums_sorted[second_ptr] == second_number) {
                        second_ptr++;
                    }
                }
            }

            first_ptr++;

            while (first_ptr < nums_sorted.size() - 2 &&
                   nums_sorted[first_ptr] == first_number) {
                first_ptr++;
            }
        }

        return valid_triples;
    }
};
```

## 17. 电话号码的字母组合 (Letter Combinations of a Phone Number)

标签: 哈希表, 回溯

解法:

- 直接创建哈希表然后使用一般的回溯方法即可.

代码:

```cpp
class Solution {
public:
    void do_letter_combinations_by_dfs(
        const std::string &digits, int current_position,
        std::string &current_combination,
        std::vector<std::string> &valid_combinations) {
        if (current_position == digits.size()) {
            valid_combinations.push_back(current_combination);

            return;
        }

        int index_of_current_digit = digits[current_position] - '2';

        for (auto current_char : _number_2_chars[index_of_current_digit]) {
            current_combination.push_back(current_char);

            do_letter_combinations_by_dfs(digits, current_position + 1,
                                          current_combination,
                                          valid_combinations);

            current_combination.pop_back();
        }
    }

    vector<string> letterCombinations(string digits) {
        if (digits.empty()) {
            return {};
        }

        std::vector<std::string> valid_combinations;
        std::string current_combination;

        do_letter_combinations_by_dfs(digits, 0, current_combination,
                                      valid_combinations);

        return valid_combinations;
    }

private:
    static std::vector<std::string> _number_2_chars;
};

std::vector<std::string> Solution::_number_2_chars = {
    "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};
```

## 19. 删除链表的倒数第 N 个结点 (Remove Nth Node From End of List)

标签: 双指针

解法:

- 因为可能将链表的原始头结点给删除掉, 因此一开始需要使用一个哨兵结点 `dummy_head` 挂住原始头结点 `head`, 这样方便在原始头结点被删除之后获取新的头结点.
- 因为待删除结点与链表尾结点之间的距离为已知, 我们可以设置快慢指针, 从哨兵结点开始, 让快指针先行出发, 并在快指针和慢指针之间的距离等于待删除结点与链表尾结点之间的距离加一的时候停止, 然后再让快慢指针一同出发, 这样当快指针到达链表尾时, 慢指针所指的就是待删除结点的父结点.

代码:

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode *removeNthFromEnd(ListNode *head, int n) {
        ListNode dummy_head;
        dummy_head.next = head;

        ListNode *slow_ptr = &dummy_head;
        ListNode *fast_ptr = &dummy_head;

        for (int i = 0; i < n; i++) {
            fast_ptr = fast_ptr->next;
        }

        while (fast_ptr->next) {
            fast_ptr = fast_ptr->next;
            slow_ptr = slow_ptr->next;
        }

        auto node_to_be_deleted = slow_ptr->next;
        slow_ptr->next = node_to_be_deleted->next;
        delete node_to_be_deleted;

        return dummy_head.next;
    }
};
```

## 20. 有效的括号 (Valid Parentheses)

标签: 栈, 哈希表

解法:

- 直接使用一个哈希表存储从右括号到左括号的映射 (左括号不用管, 因为只要遇到左括号就入栈, 但要想匹配左括号就必须知道所要匹配的左括号是什么), 然后使用栈进行一般的括号匹配过程即可.

代码:

```cpp
class Solution {
public:
    bool isValid(string s) {
        if (s.size() % 2 == 1) {
            return false;
        }

        std::stack<char> brackets;

        std::unordered_map<char, char> right_bracket_2_left_bracket;

        right_bracket_2_left_bracket[')'] = '(';
        right_bracket_2_left_bracket['}'] = '{';
        right_bracket_2_left_bracket[']'] = '[';

        for (char c : s) {
            // 左括号
            if (!right_bracket_2_left_bracket.count(c)) {
                brackets.push(c);
            }

            // 右括号
            else {
                if (brackets.empty()
                    || brackets.top() != right_bracket_2_left_bracket[c]) {
                    return false;
                }

                brackets.pop();
            }
        }

        return brackets.empty();
    }
};
```

## 21. 合并两个有序链表 (Merge Two Sorted Lists)

标签: 迭代

解法:

- 直接使用一般的哨兵结点 + 迭代的做法进行合并即可, 类似于归并排序.

代码:

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode *mergeTwoLists(ListNode *list1, ListNode *list2) {
        ListNode dummy_node;
        ListNode *merged_list_tail = &dummy_node;

        while (list1 && list2) {
            const bool is_list1_smaller_than_list2 = list1->val < list2->val;

            auto current_node = is_list1_smaller_than_list2 ? list1 : list2;
            list1 = is_list1_smaller_than_list2 ? list1->next : list1;
            list2 = is_list1_smaller_than_list2 ? list2 : list2->next;

            merged_list_tail->next = current_node;
            merged_list_tail = merged_list_tail->next;
        }

        if (!list1) {
            merged_list_tail->next = list2;
        }

        else {
            merged_list_tail->next = list1;
        }

        return dummy_node.next;
    }
};
```

## 22. 括号生成 (Generate Parentheses)

标签: 回溯

解法:

- 因为要获取所有合法的组合, 因此只能通过回溯进行枚举. 已知合法的组合所构成的字符串的长度固定 (等于 `2n`), 对于字符串中的每一个位置, 要么可以放置一个左括号, 要么可以放置一个右括号, 因此我们只需要遍历每一个位置, 通过维护一些必要信息对在当前位置放置左括号或右括号是否合法进行判断, 并结合回溯对所有合法组合进行枚举即可.

代码:

```cpp
class Solution {
public:
    void do_generate_parenthesis_by_dfs(
        const int number_of_available_left_brackets,
        const int number_of_available_right_brackets,
        std::string &current_combination,
        std::vector<std::string> &valid_combinations) {
        if (number_of_available_left_brackets == 0
            && number_of_available_right_brackets == 0) {
            valid_combinations.push_back(current_combination);

            return;
        }

        if (number_of_available_left_brackets > 0) {
            current_combination.push_back('(');

            do_generate_parenthesis_by_dfs(
                number_of_available_left_brackets - 1,
                number_of_available_right_brackets + 1, current_combination,
                valid_combinations);

            current_combination.pop_back();
        }

        if (number_of_available_right_brackets > 0) {
            current_combination.push_back(')');

            do_generate_parenthesis_by_dfs(
                number_of_available_left_brackets,
                number_of_available_right_brackets - 1, current_combination,
                valid_combinations);

            current_combination.pop_back();
        }
    }

    vector<string> generateParenthesis(int n) {
        std::string current_combination;
        std::vector<std::string> valid_combinations;

        do_generate_parenthesis_by_dfs(n, 0, current_combination,
                                       valid_combinations);

        return valid_combinations;
    }
};
```

## 23. 合并 K 个升序链表 (Merge k Sorted Lists)

标签：堆排序, 归并排序

解法:

- 一种方法是为所有链表的头结点维护一个小顶堆, 每次弹出头结点最小的那个链表的头结点并同时加入该头结点的下一个结点 (若有), 时间复杂度为 $O(kn\log{k})$.
- 另一种方法是使用归并排序, 每一轮迭代中通过分治的方法两两合并链表, 例如第一轮合并链表 1 和链表 2 得到链表 a, 同时合并链表 3 和链表 4 得到链表 b, 那么第二轮将合并链表 a 和链表 b 得到最终的链表.
  - 注意归并排序中的分治合并链表和顺序合并链表不一样, 在顺序合并链表的情况下, 每次合并的两个链表中左边的链表的长度为 $in$, 右边的链表的长度为 $n$, 合并需要的比较次数为 $(i + 1)n$, 因此最终的时间复杂度将为 $O(k^2n)$.

代码 (小顶堆的解法):

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
private:
    struct NodeCompare {
        bool operator()(ListNode *const node_1, ListNode *const node_2) {
            return node_1->val > node_2->val;
        }
    };

public:
    ListNode *mergeKLists(vector<ListNode *> &lists) {
        ListNode dummy_head;
        ListNode *merged_list_tail = &dummy_head;

        std::priority_queue<ListNode *, std::vector<ListNode *>, NodeCompare>
            node_queue;

        for (auto list_head : lists) {
            if (list_head) {
                node_queue.push(list_head);
            }
        }

        while (!node_queue.empty()) {
            auto chosen_node = node_queue.top();
            node_queue.pop();

            if (chosen_node->next) {
                node_queue.push(chosen_node->next);
            }

            merged_list_tail->next = chosen_node;
            merged_list_tail = merged_list_tail->next;
        }

        return dummy_head.next;
    }
};
```

代码 (归并排序的解法):

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode *merge_two_lists(ListNode *head_1, ListNode *head_2) {
        ListNode dummy_head;
        ListNode *merged_list_tail = &dummy_head;

        while (head_1 && head_2) {
            const int value_1 = head_1->val;
            const int value_2 = head_2->val;
            const bool is_value_1_smaller_than_value_2 = value_1 < value_2;

            merged_list_tail->next =
                is_value_1_smaller_than_value_2 ? head_1 : head_2;
            merged_list_tail = merged_list_tail->next;

            head_1 = is_value_1_smaller_than_value_2 ? head_1->next : head_1;
            head_2 = is_value_1_smaller_than_value_2 ? head_2 : head_2->next;
        }

        if (!head_1) {
            merged_list_tail->next = head_2;
        }

        else {
            merged_list_tail->next = head_1;
        }

        return dummy_head.next;
    }

    ListNode *mergeKLists(vector<ListNode *> &lists) {
        if (lists.empty()) {
            return nullptr;
        }

        std::queue<ListNode *> list_head_queue(lists.begin(), lists.end());

        while (list_head_queue.size() > 1) {
            auto head_1 = list_head_queue.front();
            list_head_queue.pop();

            auto head_2 = list_head_queue.front();
            list_head_queue.pop();

            auto merged_list_head = merge_two_lists(head_1, head_2);

            list_head_queue.push(merged_list_head);
        }

        return list_head_queue.front();
    }
};
```

## 31. 下一个排列 (Next Permutation)

标签: 模拟

解法:

- 本题更像是一道脑筋急转弯.
- 考虑一个已经位于字典序中最末尾的排列, 这个排列中的所有元素必然是按照降序排序的, 例如 `[3, 1, 1]`, 其下一个排列显然是将其反转后所得到的序列, 即 `[1, 1, 3]`.
- 进一步考虑一般的序列, 例如 `[2, 0, 3, 1, 1]`, 此时该序列的后半部分 `[3, 1, 1]` 已经降序有序, 那么其下一个排列需要找到位于该部分的前一个元素 (即 `0`), 保持位于该元素之前的所有元素 (即 `2`) 不变, 并将该元素 (即 `0`) 切换至当前可用的元素中按字典序排序的下一个元素 (即 `1`, 其中 "当前可用的元素" 就是该序列的后半部分 `[3, 1, 1]` 中的所有元素). 最终所得到的下一个排列为 `[2, 1, 0, 1, 3]` (注意到我们只需要将 `0` 与 `1` 进行互换并对后半部分进行反转即可得到下一个排列).
- 需要注意的是数组中的元素可能重复, 例如序列 `[1, 1, 1]` 仅有一种全排列 (即其自身).

代码:

```cpp
class Solution {
public:
    void nextPermutation(vector<int> &nums) {
        int decreasing_range_begin = nums.size() - 1;

        // 确定序列后半部分降序有序的最大范围
        while (decreasing_range_begin > 0
               && nums[decreasing_range_begin - 1]
                      >= nums[decreasing_range_begin]) {
            decreasing_range_begin--;
        }

        if (decreasing_range_begin > 0) {
            const int target_position = decreasing_range_begin - 1;
            const int target = nums[target_position];
            const int target_lower_bound_position =
                std::lower_bound(nums.begin() + decreasing_range_begin,
                                 nums.end(), target, std::greater<int>())
                - nums.begin();

            swap(nums[target_position], nums[target_lower_bound_position - 1]);
        }

        std::reverse(nums.begin() + decreasing_range_begin, nums.end());
    }
};
```

## 32. 最长有效括号 (Longest Valid Parentheses)

标签: 动态规划, 栈

解法:

- 第一种解法是动态规划. 考虑维护一个保存有 "以当前元素结尾的最长有效括号子串的长度" 的数组 `dp`:
  - 如果当前元素为左括号, 则 `dp[i]` 必然为零.
  - 如果当前元素为右括号, 要想向左扩展有效括号子串, 我们就应该进一步观察前一个位置结尾的最长有效括号子串 (其长度为 `dp[i - 1]`), 如果该子串的首元素的前一个元素恰好是一个左括号, 那么这个左括号就能和当前的右括号以及该子串共同形成一个新的更长的有效括号子串. 除此之外我们还应当观察新的子串的左边是否还存在有效括号子串, 因为如果还存在其他有效括号子串, 这些旧子串就能和当前的新子串串联形成更长的子串.

  最终答案即为 `dp` 数组中的元素的最大值.
- 第二种解法是使用栈. 考虑维护一个保存有元素下标的栈. 对于当前遍历到的元素:
  - 如果该元素为左括号 `(`, 那么我们立即将其**下标**入栈.
  - 如果该元素为右括号 `)`, 我们观察栈顶:
    - 如果栈为非空并且栈顶下标所对应的元素为左括号 `(`, 那么两者形成匹配, 左括号出栈, 两个下标之差即为两者所形成的有效括号子串的长度, 而最终的有效括号子串还需要继续向左与旧有的有效括号子串进行串联, 最左端能够到达的范围实际上就是左括号出栈后的新栈顶的下标 (这里栈可能为空, 这可以通过预先插入一个哨兵下标来消除, 下面会提到), 因此最终的有效括号子串的长度即为新栈顶的下标与当前元素下标之差.
    - 如果栈顶下标所对应的元素为右括号 `)` (或者栈为空, 这可以通过预先插入一个哨兵下标来消除, 下面会提到), 那么将栈顶右括号出栈, 并将当前右括号的下标入栈.
  - 不失一般性, 我们可以将整个原始字符串视作位于一个下标为 `-1` 的假想右括号的右边的子串, 因此可以在一开始的时候向栈中添加一个下标为 `-1` 的哨兵, 简化代码逻辑.
  
  最终答案即为所有有效括号子串的长度的最大值.

代码 (动态规划的解法):

```cpp
class Solution {
public:
    int longestValidParentheses(string s) {
        std::vector<int> dp(s.size(), 0);

        int max_range_size = 0;

        for (int current_position = 1; current_position < s.size();
             current_position++) {
            const char current_char = s[current_position];

            if (current_char == '(') {
                continue;
            }

            const int range_size_of_immediate_valid_sub_string =
                dp[current_position - 1];
            int range_begin =
                current_position - range_size_of_immediate_valid_sub_string - 1;

            if (range_begin < 0 || s[range_begin] != '(') {
                continue;
            }

            if (range_begin > 0) {
                range_begin -= dp[range_begin - 1];
            }

            const int range_end = current_position + 1;
            const int range_size = range_end - range_begin;

            dp[current_position] = range_size;

            max_range_size = std::max(max_range_size, range_size);
        }

        return max_range_size;
    }
};
```

代码 (栈的解法):

```cpp
class Solution {
public:
    int longestValidParentheses(string s) {
        std::stack<int> position_stack;

        position_stack.push(-1);

        int max_range_size = 0;

        for (int current_position = 0; current_position < s.size();
             current_position++) {

            const char c = s[current_position];

            if (c == '(') {
                position_stack.push(current_position);
            }

            else {
                const int might_matched_position = position_stack.top();
                position_stack.pop();

                if (might_matched_position == -1
                    || s[might_matched_position] == ')') {

                    position_stack.push(current_position);
                }

                else {
                    const int range_size =
                        current_position - position_stack.top();

                    max_range_size = std::max(max_range_size, range_size);
                }
            }
        }

        return max_range_size;
    }
};
```

## 33. 搜索旋转排序数组 (Search in Rotated Sorted Array)

标签: 二分查找

解法:

- 注意到本题约定了数组中的元素各不相同, 这实际上降低了解题的难度. 根据当前搜索范围中左右两端的数的大小关系可以判断出当前范围中是否发生了旋转:
  - 如果最左端的数小于最右端的数, 说明数组未发生旋转, 此时直接使用一般的二分法即可.
  - 如果最左端的数大于最右端的数, 说明数组发生了旋转. 根据位于当前搜索范围的中点处的数的大小关系可以判断出中点位于前半部分 (即旋转前的后半部分) 还是后半部分 (即旋转前的前半部分):
    - 如果中点处的数大于最左端的数, 说明中点位于前半部分, 那么从中点到最左端的序列严格有序, 根据这一点可以判断目标数位于前半部分还是后半部分.
    - 如果中点处的数小于最左端的数, 说明中点位于后半部分, 那么从中点到最右端的序列严格有序, 根据这一点同样可以判断目标数位于前半部分还是后半部分.
- 为了确保执行二分法时最左端, 中点处, 以及最右端这三个数互不相同, 我们需要保证搜索范围的大小至少为 3. 对于小于 3 的搜索范围, 我们直接在范围内进行顺序遍历即可.

代码:

```cpp
class Solution {
public:
    int search(vector<int> &nums, int target) {
        int left = 0;
        int right = nums.size();

        while (right - left >= 3) {
            const int mid = left + (right - left) / 2;

            const int mid_number = nums[mid];

            if (target == mid_number) {
                return mid;
            }

            const int left_number = nums[left];
            const int right_number = nums[right - 1];

            const bool is_rotated = left_number > right_number;

            if (!is_rotated) {
                if (target < mid_number) {
                    right = mid;
                }

                else {
                    left = mid + 1;
                }
            }

            else {
                const bool is_in_left_part = mid_number > left_number;

                if (is_in_left_part) {
                    if (target < mid_number && target >= left_number) {
                        right = mid;
                    }

                    else {
                        left = mid + 1;
                    }
                }

                else {
                    if (target > mid_number && target <= right_number) {
                        left = mid + 1;
                    }

                    else {
                        right = mid;
                    }
                }
            }
        }

        for (int i = left; i < right; i++) {
            if (nums[i] == target) {
                return i;
            }
        }

        return -1;
    }
};
```

## 34. 在排序数组中查找元素的第一个和最后一个位置 (Find First and Last Position of Element in Sorted Array)

标签: 二分查找

解法:

- 本题的大致解法就是使用二分法实现两个帮手函数 `get_lower_bound` 和 `get_upper_bound`, 然后利用这两个帮手函数来方便快捷地获取正确答案.
- 和一般的二分法不同, 在一般的二分法中, 如果当前中点处的值恰好等于目标值, 那么程序便可立即返回 `true` (表示 "已经找到"), 而在本题中——以 `get_lower_bound` 为例, 即使当前中点处的值恰好等于目标值, 程序也无法立即返回, 因为 "lower bound" 的含义是 "**第一个**大于等于目标值的下标", 即使当前中点处的值恰好等于目标值也并不意味着它就是第一个恰好等于目标值的下标. 这就意味着我们无法通过观察当前中点处的值与目标值之间的大小关系来判断是否应该退出循环, 而只能通过观察**当前搜索范围的大小**来判断是否应该退出循环. 同时在退出循环之后还需要通过顺序遍历来判断数组中是否根本就不存在目标值.
- `get_upper_bound` 的情况同理.
- 此外本题的解法还遵循了通用算法的约定, 即仅使用一种关系运算符 `<`, 而不使用 `<=` 或是 `>` 等运算符.

代码:

```cpp
class Solution {
public:
    int get_lower_bound(const std::vector<int> &nums, const int target) {
        int left = 0;
        int right = nums.size();

        const auto is_in_left_half_part = [=](const int mid_number) {
            return mid_number < target;
        };

        while (right - left >= 3) {
            const int mid = left + (right - left) / 2;
            const int mid_number = nums[mid];

            if (is_in_left_half_part(mid_number)) {
                left = mid + 1;
            }

            else {
                right = mid;
            }
        }

        for (int i = left; i < right; i++) {
            if (is_in_left_half_part(nums[i])) {
                continue;
            }

            return i;
        }

        return right;
    }

    int get_upper_bound(const std::vector<int> &nums, const int target) {
        int left = 0;
        int right = nums.size();

        const auto is_in_left_half_part = [=](const int mid_number) {
            return !(target < mid_number);
        };

        while (right - left >= 3) {
            const int mid = left + (right - left) / 2;
            const int mid_number = nums[mid];

            if (is_in_left_half_part(mid_number)) {
                left = mid + 1;
            }

            else {
                right = mid;
            }
        }

        for (int i = left; i < right; i++) {
            if (is_in_left_half_part(nums[i])) {
                continue;
            }

            return i;
        }

        return right;
    }

    vector<int> searchRange(vector<int> &nums, int target) {
        const int lower_bound = get_lower_bound(nums, target);

        if (lower_bound == nums.size() || nums[lower_bound] != target) {
            return {-1, -1};
        }

        else {
            const int upper_bound = get_upper_bound(nums, target);

            return {lower_bound, upper_bound - 1};
        }
    }
};
```

## 39. 组合总和 (Combination Sum)

标签: 回溯

解法:

- 由于本题需要返回所有可行解, 无法使用动态规划, 因此直接使用标准的回溯法进行求解即可.

代码:

```cpp
class Solution {
public:
    void do_combination_sum(const std::vector<int> &candidates,
                            const int target, const int current_position,
                            std::vector<int> &current_combination,
                            std::vector<std::vector<int>> &valid_combinations) {
        if (current_position == candidates.size()) {
            if (target == 0) {
                valid_combinations.push_back(current_combination);
            }

            return;
        }

        const int current_combination_size_backup = current_combination.size();

        const int current_candidate = candidates[current_position];

        int current_target = target;

        while (current_target >= 0) {
            do_combination_sum(candidates, current_target, current_position + 1,
                               current_combination, valid_combinations);

            current_target -= current_candidate;
            current_combination.push_back(current_candidate);
        }

        current_combination.resize(current_combination_size_backup);
    }

    vector<vector<int>> combinationSum(vector<int> &candidates, int target) {
        std::vector<std::vector<int>> valid_combinations;
        std::vector<int> current_combination;

        do_combination_sum(candidates, target, 0, current_combination,
                           valid_combinations);

        return valid_combinations;
    }
};
```

## 42. 接雨水 (Trapping Rain Water)

标签: 单调栈, 双指针

解法:

- 第一种解法是使用单调栈. 在单调栈的解法中, 每个水坑被一条条水平线切割为上下多层的长条, 通过单调栈维护每一个长条的左墙的高度和位置, 并在右墙到来时实时计算长条的面积.
- 第二种解法是使用双指针 (动态规划). 在双指针 (动态规划) 的解法中, 每个水坑由坑中每个底墙上方的宽为 1 的水柱构成, 其中每条水柱的高度等于 "位于该底墙左端的所有墙的高度的最大值与位于该底墙右端的所有墙的高度的最大值中的较小值" 与 "该底墙的高度" 之差, 而 "位于该底墙左端 (或右端) 的所有墙的高度的最大值" 能够通过双指针 (动态规划) 的方法求得, 类似于 "11. 盛最多水的容器 (Container With Most Water)" 中的思想.

代码 (单调栈的解法):

```cpp
class Solution {
public:
    int trap(vector<int> &height) {
        std::stack<int> stack_of_positions_of_decreasing_walls;

        int total_volume = 0;

        for (int current_position = 0; current_position < height.size();
             current_position++) {

            const int height_of_current_right_wall = height[current_position];

            while (!stack_of_positions_of_decreasing_walls.empty()) {
                const int position_of_previous_wall =
                    stack_of_positions_of_decreasing_walls.top();
                const int height_of_previous_wall =
                    height[position_of_previous_wall];

                if (height_of_previous_wall > height_of_current_right_wall) {
                    break;
                }

                stack_of_positions_of_decreasing_walls.pop();

                if (height_of_previous_wall < height_of_current_right_wall
                    && !stack_of_positions_of_decreasing_walls.empty()) {

                    const int &height_of_current_bottom_wall =
                        height_of_previous_wall;

                    const int position_of_current_left_wall =
                        stack_of_positions_of_decreasing_walls.top();
                    const int height_of_current_left_wall =
                        height[position_of_current_left_wall];

                    const int volume_height =
                        std::min(height_of_current_left_wall,
                                 height_of_current_right_wall)
                        - height_of_current_bottom_wall;
                    const int volume_width =
                        current_position - (position_of_current_left_wall + 1);

                    total_volume += volume_height * volume_width;
                }
            }

            stack_of_positions_of_decreasing_walls.push(current_position);
        }

        return total_volume;
    }
};
```

代码 (双指针的解法):

```cpp
class Solution {
public:
    int trap(vector<int> &height) {
        int left_ptr = 0;
        int max_height_before_left_ptr = 0;

        int right_ptr = height.size() - 1;
        int max_height_after_right_ptr = 0;

        int total_volume = 0;

        while (left_ptr <= right_ptr) {
            if (max_height_before_left_ptr < max_height_after_right_ptr) {
                const int height_of_bottom_wall = height[left_ptr];
                const int volume_height =
                    std::max(0, std::min(max_height_before_left_ptr,
                                         max_height_after_right_ptr)
                                    - height_of_bottom_wall);
                const int volume_width = 1;

                total_volume += volume_height * volume_width;

                max_height_before_left_ptr =
                    std::max(max_height_before_left_ptr, height_of_bottom_wall);
                left_ptr++;
            }

            else {
                const int height_of_bottom_wall = height[right_ptr];
                const int volume_height =
                    std::max(0, std::min(max_height_before_left_ptr,
                                         max_height_after_right_ptr)
                                    - height_of_bottom_wall);
                const int volume_width = 1;

                total_volume += volume_height * volume_width;

                max_height_after_right_ptr =
                    std::max(max_height_after_right_ptr, height_of_bottom_wall);
                right_ptr--;
            }
        }

        return total_volume;
    }
};
```
