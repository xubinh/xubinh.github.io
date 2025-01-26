---
title: "力扣 Hot 100 题解归档 (21-40)"
hideSummary: true
date: 2025-01-27T00:23:15+08:00
draft: false
tags: ["leetcode"]
series: ["leetcode"]
author: ["xubinh"]
type: posts
math: true
---

## 46. 全排列 (Permutations)

标签: 回溯

解法:

- 首先注意到题目已经约定数组中不会出现重复的数字, 并且可以按照任意顺序返回所有全排列, 这两点大大降低了解题难度.
- 由于需要返回所有全排列, 因此只能老老实实使用回溯来获取所有全排列.

代码:

```cpp
class Solution {
public:
    void do_permute_by_dfs(std::vector<int> &current_combination,
                           const int current_position,
                           std::vector<std::vector<int>> &valid_permutations) {
        if (current_position == current_combination.size()) {
            valid_permutations.push_back(current_combination);

            return;
        }

        do_permute_by_dfs(current_combination, current_position + 1,
                          valid_permutations);

        for (int i = current_position + 1; i < current_combination.size();
             i++) {
            swap(current_combination[current_position], current_combination[i]);

            do_permute_by_dfs(current_combination, current_position + 1,
                              valid_permutations);

            swap(current_combination[current_position], current_combination[i]);
        }
    }

    vector<vector<int>> permute(vector<int> &nums) {
        std::vector<std::vector<int>> valid_permutations;

        do_permute_by_dfs(nums, 0, valid_permutations);

        return valid_permutations;
    }
};
```

## 48. 旋转图像 (Rotate Image)

标签: 模拟

解法:

- 手动模拟即可. 注意到这是个方阵, 直接使用第一层循环来遍历矩阵中的各个同心方框; 对于每个同心方框, 使用第二层循环以及 $O(1)$ 的额外空间来原地旋转该方框的四条边, 例如第一次循环用于旋转左上角, 右上角, 右下角, 以及左下角这四个元素, 然后第二次循环用于旋转左上角的右边, 右上角的下面, 右下角的左边, 以及左下角的上面这四个元素, 以此类推.

代码:

```cpp
class Solution {
public:
    void rotate(vector<vector<int>> &matrix) {
        int matrix_order = matrix.size();

        for (int circle_track_number = 0;
             circle_track_number < matrix_order / 2;
             circle_track_number++) {
            int top_left_i = circle_track_number;
            int top_left_j = circle_track_number;

            int bottom_left_i = matrix_order - 1 - circle_track_number;
            int bottom_left_j = circle_track_number;

            int bottom_right_i = matrix_order - 1 - circle_track_number;
            int bottom_right_j = matrix_order - 1 - circle_track_number;

            int top_right_i = circle_track_number;
            int top_right_j = matrix_order - 1 - circle_track_number;

            for (int sector_number = 0;
                 sector_number < matrix_order - 1 - circle_track_number * 2;
                 sector_number++) {

                int temporary = matrix[top_left_i][top_left_j];
                matrix[top_left_i][top_left_j] =
                    matrix[bottom_left_i][bottom_left_j];
                matrix[bottom_left_i][bottom_left_j] =
                    matrix[bottom_right_i][bottom_right_j];
                matrix[bottom_right_i][bottom_right_j] =
                    matrix[top_right_i][top_right_j];
                matrix[top_right_i][top_right_j] = temporary;

                top_left_j++;
                bottom_left_i--;
                bottom_right_j--;
                top_right_i++;
            }
        }
    }
};
```

## 49. 字母异位词分组 (Group Anagrams)

标签: 哈希表

解法:

- 由于所有字母异位词所含有的字母的数量均相同, 对所有字母异位词进行排序之后得到的字符串也应该相同, 因此可以使用排序后的字符串的哈希值作为该字母异位词等价类的代表值, 并使用哈希表来维护从排序后的字符串的哈希值到字母异位词等价类的映射.
- 注意到上述从哈希值到字母异位词等价类的映射并不是一个双射 (即有可能存在两个不同的等价类, 其拥有相同的哈希值), 但出现这种情况的概率极低, 完全可以忽略不计.

代码:

```cpp
class Solution {
public:
    vector<vector<string>> groupAnagrams(vector<string> &strs) {
        std::unordered_map<long, std::vector<std::string>>
            hash_value_2_equivalent_groups;

        auto get_hash = [](std::string s) -> long {
            std::sort(s.begin(), s.end());

            return std::hash<std::string>()(s);
        };

        for (auto &current_string : strs) {
            hash_value_2_equivalent_groups[get_hash(current_string)].push_back(
                current_string
            );
        }

        std::vector<std::vector<std::string>> equivalent_groups;

        for (auto &pair : hash_value_2_equivalent_groups) {
            const auto &equivalent_group = pair.second;
            equivalent_groups.emplace_back(std::move(equivalent_group));
        }

        return equivalent_groups;
    }
};
```

## 53. 最大子数组和 (Maximum Subarray)

标签: 动态规划

解法:

- 考虑从数组最左端开始到当前位置为止这一范围中的所有子数组和, 根据这些子数组和的右端点是否位于当前位置可以将其分为不相交的两部分:
  - 对于右端点位于当前位置的所有子数组和, 其最大值等于当前元素加上 "以位于当前元素左边的元素为右端点的最大子数组和" 与零之间的较大值 (因为所要求的是最大值, 因此如果是负数就得忽略, 只有当其为非负的时候才可以加上).
  - 对于右端点不位于当前位置的所有子数组和, 其最大值就等于 "从数组最左端开始到当前位置的**左边位置**为止这一范围中的所有子数组和" 的最大值.

  综上所属即可得到状态转移方程.

代码:

```cpp
class Solution {
public:
    int maxSubArray(vector<int> &nums) {
        int current_max_sum_arbitrary = nums[0];
        int current_max_sum_fix_right_end = nums[0];

        for (int current_position = 1; current_position < nums.size();
             current_position++) {
            const int current_number = nums[current_position];

            current_max_sum_fix_right_end =
                current_number + std::max(current_max_sum_fix_right_end, 0);

            current_max_sum_arbitrary = std::max(
                current_max_sum_arbitrary, current_max_sum_fix_right_end
            );
        }

        return current_max_sum_arbitrary;
    }
};
```

## 55. 跳跃游戏 (Jump Game)

标签: 贪心

解法:

- 本题解法比较简单, 直接进行模拟即可. 我们可以维护一个表示 "当前最远可以跳跃到的位置" 的变量, 在跳跃的过程中不断更新该变量, 并在该变量到达数组尾部的时候返回 `true`, 否则说明数组尾部不可到达, 返回 `false`.

代码:

```cpp
class Solution {
public:
    bool canJump(vector<int> &nums) {
        const int final_max_position = nums.size() - 1;

        int current_position = 0;
        int current_max_position = 0;

        while (current_position <= current_max_position) {
            current_max_position = std::max(
                current_max_position, current_position + nums[current_position]
            );

            if (current_max_position >= final_max_position) {
                return true;
            }

            current_position++;
        }

        return false;
    }
};
```

## 56. 合并区间 (Merge Intervals)

标签: 模拟

解法:

- 本题直接根据在草稿纸上手动合并区间的过程进行模拟即可. 首先将所有区间按照左端点从小到大进行排序, 左端点相同的区间按照右端点从大到小进行排序, 然后对所有左端点相同的区间进行去重 (留下右端点最大的那个区间作为代表) (去重步骤为可选). 之后开始遍历每个区间, 在遍历过程中维护 "目前为止所合并的最大区间", 对于当前区间:
  - 如果 "目前为止所合并的最大区间" 的右端点严格小于当前区间的左端点, 说明这两个区间不重叠, 因此可以将 "目前为止所合并的最大区间" 收集至答案集合中, 并将 "目前为止所合并的最大区间" 更新为当前区间;
  - 如果 "目前为止所合并的最大区间" 的右端点大于等于当前区间的左端点, 说明两个区间发生重叠, "目前为止所合并的最大区间" 的右端点更新为其原值与当前区间的右端点之间的较大值.

  遍历结束之后还需要记得将 "目前为止所合并的最大区间" 中留有的最后一个合并区间收集至答案集合中.

代码:

```cpp
class Solution {
public:
    vector<vector<int>> merge(vector<vector<int>> &intervals) {
        const auto &compare_two_intervals_strong =
            [](const std::vector<int> &interval_1,
               const std::vector<int> &interval_2) -> bool {
            return interval_1[0] == interval_2[0]
                       ? interval_1[1] > interval_2[1]
                       : interval_1[0] < interval_2[0];
        };

        std::sort(
            intervals.begin(), intervals.end(), compare_two_intervals_strong
        );

        const auto &compare_two_intervals_weak =
            [](const std::vector<int> &interval_1,
               const std::vector<int> &interval_2) -> bool {
            return interval_1[0] == interval_2[0];
        };

        const auto &it_erase_begin = std::unique(
            intervals.begin(), intervals.end(), compare_two_intervals_weak
        );
        const auto &it_erase_end = intervals.end();

        intervals.erase(it_erase_begin, it_erase_end);

        int previous_interval_left_end = intervals[0][0];
        int previous_interval_right_end = intervals[0][1];

        std::vector<std::vector<int>> merged_intervals;

        for (int current_position = 1; current_position < intervals.size();
             current_position++) {

            const int current_interval_left_end =
                intervals[current_position][0];
            const int current_interval_right_end =
                intervals[current_position][1];

            if (previous_interval_right_end >= current_interval_left_end) {
                previous_interval_right_end = std::max(
                    previous_interval_right_end, current_interval_right_end
                );
            }

            else {
                std::vector<int> new_merged_interval = {
                    previous_interval_left_end, previous_interval_right_end
                };

                merged_intervals.push_back(new_merged_interval);

                previous_interval_left_end = current_interval_left_end;
                previous_interval_right_end = current_interval_right_end;
            }
        }

        {
            std::vector<int> new_merged_interval = {
                previous_interval_left_end, previous_interval_right_end
            };

            merged_intervals.push_back(new_merged_interval);
        }

        return merged_intervals;
    }
};
```

## 62. 不同路径 (Unique Paths)

标签: 动态规划, 数学

解法:

- 第一种解法是使用动态规划, 状态转移方程也十分简单, 即 "当前位置的路径总数" = "上方位置的路径总数" + "左边位置的路径总数", 使用一个滚动数组进行动态规划即可.
- 第二种解法是使用数学中的组合数学. 观察到动态规划解法中的状态转移方程实际上就是杨辉三角的计算公式, 而第 m 行, 第 n 列位置对应于杨辉三角中的第 (m + n - 1) 行的第 n 个元素, 其计算公式为 $C_{m + n - 2}^{n - 1}$ (或等价的 $C_{m + n - 2}^{m - 1}$).

代码 (动态规划的解法):

```cpp
class Solution {
public:
    int uniquePaths(int m, int n) {
        if (m > n) {
            swap(m, n);
        }

        std::vector<int> dp(m, 1);

        for (int i = 1; i < n; i++) {
            dp[0] = 1;

            for (int j = 1; j < m; j++) {
                dp[j] += dp[j - 1];
            }
        }

        return dp[m - 1];
    }
};
```

代码 (数学的解法):

(略)

## 64. 最小路径和 (Minimum Path Sum)

标签: 动态规划

解法:

- 本题虽然属于动态规划, 其解法却非常简单, 类似于 "62. 不同路径 (Unique Paths)". 状态转移方程为 "当前位置的最小路径和" = "上面位置的最小路径和" 与 "左边位置的最小路径和" 之间的较小值 + 当前位置的权重.

代码:

```cpp
class Solution {
public:
    int minPathSum(vector<vector<int>> &grid) {
        for (int i = 1; i < grid.size(); i++) {
            grid[i][0] += grid[i - 1][0];
        }

        for (int j = 1; j < grid[0].size(); j++) {
            grid[0][j] += grid[0][j - 1];
        }

        for (int i = 1; i < grid.size(); i++) {
            for (int j = 1; j < grid[0].size(); j++) {
                grid[i][j] += std::min(grid[i - 1][j], grid[i][j - 1]);
            }
        }

        return grid.back().back();
    }
};
```

## 70. 爬楼梯 (Climbing Stairs)

标签: 动态规划

解法:

- 本题属于一维动态规划, 只需使用 $O(1)$ 的额外空间即可, 其解法也非常简单, 状态转移方程为 "当前位置的方法总数" = "前一个位置的方法总数" + "前两个位置的方法总数".

代码:

```cpp
class Solution {
public:
    int climbStairs(int n) {
        if (n <= 2) {
            if (n == 1) {
                return 1;
            }

            if (n == 2) {
                return 2;
            }
        }

        int number_of_methods_for_current_position_minus_2;
        int number_of_methods_for_current_position_minus_1 = 1;
        int number_of_methods_for_current_position = 2;

        int current_position = 3;

        while (current_position <= n) {
            number_of_methods_for_current_position_minus_2 =
                number_of_methods_for_current_position_minus_1;
            number_of_methods_for_current_position_minus_1 =
                number_of_methods_for_current_position;

            number_of_methods_for_current_position =
                number_of_methods_for_current_position_minus_2
                + number_of_methods_for_current_position_minus_1;

            current_position++;
        }

        return number_of_methods_for_current_position;
    }
};
```

## 72. 编辑距离 (Edit Distance)

标签: 动态规划

解法:

- 本题为最经典的编辑距离问题, 其解法也是最经典的二维动态规划. 我们维护当前在二维矩阵中的位置 `(i, j)`, 其中 `i` 为字符串 `word1` 的下标, 表示 `word1` 的 `[0, i)` 这一范围的子串 (注意是左闭右开); `j` 为字符串 `word2` 的下标, 其意义类似 `i`; 而二维矩阵中当前位置 `(i, j)` 的值则表示 `word1` 的 `[0, i)` 这一范围的子串与 `word2` 的 `[0, j)` 这一范围的子串之间的编辑距离. 位置 `(i, j)` 处的值与三种情况有关:
  - 如果删除 `word1` 的第 `i - 1` 个字符 (等价于在 `word2` 的第 `j` 个位置处增加一个相同的字符), 则状态转移至 `(i - 1, j)`.
  - 如果删除 `word2` 的第 `j - 1` 个字符 (等价于在 `word1` 的第 `i` 个位置处增加一个相同的字符), 则状态转移至 `(i, j - 1)`.
  - 如果替换 `word1` 的第 `i - 1` 个字符 (等价于替换 `word2` 的第 `j - 1` 个字符), 则状态转移至 `(i - 1, j - 1)`.

  可以看到增加字符的操作实际上等价于删除字符的操作, 同时上述状态转移需要用到 dp 数组中当前位置上方, 左边, 以及左上方这三个位置的数据, 因此在使用滚动数组的时候必须使用两个滚动数组.

代码:

```cpp
class Solution {
public:
    int minDistance(string word1, string word2) {
        if (word1.size() < word2.size()) {
            swap(word1, word2);
        }

        const int I = word1.size();
        const int J = word2.size();

        std::vector<int> previous_dp(J + 1);
        std::vector<int> dp(J + 1);

        for (int j = 0; j <= J; j++) {
            dp[j] = j;
        }

        for (int i = 1; i <= I; i++) {
            swap(previous_dp, dp);

            dp[0] = i;

            for (int j = 1; j <= J; j++) {
                const int operation_number_if_delete_word1_char =
                    previous_dp[j] + 1;
                const int operation_number_if_delete_word2_char = dp[j - 1] + 1;
                const int operation_number_if_replace =
                    previous_dp[j - 1] + (word1[i - 1] == word2[j - 1] ? 0 : 1);

                dp[j] = std::min(
                    std::min(
                        operation_number_if_delete_word1_char,
                        operation_number_if_delete_word2_char
                    ),
                    operation_number_if_replace
                );
            }
        }

        return dp.back();
    }
};
```

## 75. 颜色分类 (Sort Colors)

标签: 快速排序, 扫描, 主元

解法:

- 直接使用快速排序中的扫描算法对数组执行一趟扫描即可.

代码:

```cpp
class Solution {
public:
    void sortColors(vector<int> &nums) {
        int left_ptr = 0;
        int current_ptr = 0;
        int right_ptr = nums.size();

        while (current_ptr < right_ptr) {
            const int &current_number = nums[current_ptr];

            if (current_number == 0) {
                swap(nums[left_ptr], nums[current_ptr]);

                left_ptr++;
                current_ptr++;
            }

            else if (current_number == 1) {
                current_ptr++;
            }

            else {
                right_ptr--;

                swap(nums[current_ptr], nums[right_ptr]);
            }
        }
    }
};
```

## 76. 最小覆盖子串 (Minimum Window Substring)

标签: 哈希表, 滑动窗口

解法:

- 本题的类型为滑动窗口. 初始时两个指针 `i` 和 `j` 处在原始字符串的最左端, 然后 `j` 开始向右走并不断吸纳字符, 直至所吸纳的字符构成的子串形成了目标字符串的一个覆盖, 由于在吸纳的过程中有可能吸纳进一些无用的字符, 为了确定最小覆盖, 还需要将 `i` 指针向右走并排出无用的字符, 直至排出这样一个字符, 该字符的排出使得子串失去覆盖性质, 那么排出该字符前的子串大小即为最小覆盖子串大小的一个候选, 可以用于更新最终的最小覆盖子串大小. 不断重复上述过程直至循环结束为止即可.

代码:

```cpp
class Solution {
public:
    string minWindow(string s, string t) {
        std::unordered_map<char, int> fixed_original_record;
        std::unordered_map<char, int> current_record;

        for (auto c : t) {
            fixed_original_record[c]++;
        }

        int shortest_sub_string_length = std::numeric_limits<int>::max();
        std::string shortest_sub_string = "";

        int slide_window_begin = 0;
        int slide_window_end = 0;

        const auto &is_in_original_record = [&](const char current_char
                                            ) -> bool {
            return fixed_original_record[current_char] > 0;
        };

        const auto &check_if_covered =
            [&](std::unordered_map<char, int> &current_record) -> bool {
            for (auto &pair : fixed_original_record) {
                if (current_record[pair.first] < pair.second) {
                    return false;
                }
            }

            return true;
        };

        while (true) {
            while (true) {
                const char current_char = s[slide_window_end];
                slide_window_end++;

                if (is_in_original_record(current_char)) {
                    current_record[current_char]++;

                    if (check_if_covered(current_record)) {
                        break;
                    }
                }

                if (slide_window_end == s.size()) {
                    return shortest_sub_string;
                }
            }

            while (true) {
                char current_char = s[slide_window_begin];
                slide_window_begin++;

                if (is_in_original_record(current_char)) {
                    current_record[current_char]--;

                    if (!check_if_covered(current_record)) {
                        break;
                    }
                }
            }

            const int current_shortest_sub_string_length =
                slide_window_end - (slide_window_begin - 1);

            if (current_shortest_sub_string_length
                < shortest_sub_string_length) {
                shortest_sub_string_length = current_shortest_sub_string_length;
                shortest_sub_string = s.substr(
                    slide_window_begin - 1, current_shortest_sub_string_length
                );
            }

            if (slide_window_end == s.size()) {
                return shortest_sub_string;
            }
        }

        throw std::runtime_error("never reaches here");
    }
};
```

## 78. 子集 (Subsets)

标签: 回溯, 迭代, 位运算

解法:

- 第一种解法是使用回溯, 直接使用一般的 DFS 样板代码套进去即可.
- 第二种解法是使用迭代模拟回溯中的调用栈, 避免函数调用开销.
- 第三种解法是使用位运算, 直接使用整数的二进制模式来表示要选取原始集合中的哪些元素. 例如对于一个大小为 $2^5 = 32$ 的原始集合, 整数 `3` 的二进制模式是 `0x00011`, 表示选取第 4 和第 5 个数 (因为第 4 位和第 5 位被置一).

代码 (回溯的解法):

```cpp
class Solution {
public:
    void do_subsets_by_dfs(
        const std::vector<int> &nums,
        const int current_position,
        std::vector<int> &current_subset,
        std::vector<std::vector<int>> &all_possible_subsets
    ) {
        if (current_position == nums.size()) {
            all_possible_subsets.push_back(current_subset);

            return;
        }

        do_subsets_by_dfs(
            nums, current_position + 1, current_subset, all_possible_subsets
        );

        current_subset.push_back(nums[current_position]);
        do_subsets_by_dfs(
            nums, current_position + 1, current_subset, all_possible_subsets
        );
        current_subset.pop_back();
    }

    vector<vector<int>> subsets(vector<int> &nums) {
        std::vector<int> current_subset;
        std::vector<std::vector<int>> all_possible_subsets;

        do_subsets_by_dfs(nums, 0, current_subset, all_possible_subsets);

        return all_possible_subsets;
    }
};
```

代码 (迭代的解法):

```cpp
class Solution {
private:
    enum CallStatus { START, HALF_WAY, FINISH };

    struct Node {
        int current_position;
        CallStatus call_status;
    };

public:
    vector<vector<int>> subsets(vector<int> &nums) {
        std::stack<Node> call_stack;
        std::vector<int> current_subset;
        std::vector<std::vector<int>> valid_subsets;

        call_stack.push({0, START});

        while (!call_stack.empty()) {
            if (call_stack.top().current_position == nums.size()) {
                valid_subsets.push_back(current_subset);

                call_stack.pop();

                continue;
            }

            const int current_position = call_stack.top().current_position;
            const auto call_status = call_stack.top().call_status;

            switch (call_status) {
            case START:
                call_stack.top().call_status = HALF_WAY;
                call_stack.push({current_position + 1, START});

                break;

            case HALF_WAY:
                current_subset.push_back(nums[current_position]);
                call_stack.top().call_status = FINISH;
                call_stack.push({current_position + 1, START});

                break;

            case FINISH:
                current_subset.pop_back();
                call_stack.pop();

                break;
            }
        }

        return valid_subsets;
    }
};
```

代码 (位运算的解法):

(略)

## 79. 单词搜索 (Word Search)

标签: 回溯

解法:

- 直接使用标准的回溯解法即可. 维护一个 `visited` 二维布尔数组用于存储当前已经遍历到的所有元素的位置信息, 即若 `visited[i][j]` 等于 `true` 表示元素 `(i, j)` 已经遍历到 (或者说已被使用过). 对于二维字符数组 `board` 的当前位置的元素, 如果该元素等于字符串 `word` 的当前位置的元素, 那么可以将 `visited` 中的对应元素置为 `true`, 然后使用回溯法对 `board` 中的该位置的上下左右四个位置进行进一步探索; 否则直接返回 `false`.

代码:

```cpp
class Solution {
public:
    bool do_exist(
        std::vector<std::vector<char>> &board,
        std::string &word,
        int current_position,
        int i,
        int j,
        std::vector<std::vector<char>> &visited
    ) {
        if (current_position == word.size()) {
            return true;
        }

        if (board[i][j] != word[current_position] || visited[i][j]) {
            return false;
        }

        visited[i][j] = true;

        char result = false;

        if (current_position + 1 >= word.size()) {
            result = true;
        }

        else {
            if (i - 1 >= 0
                && do_exist(
                    board, word, current_position + 1, i - 1, j, visited
                )) {

                result = true;
            }

            if (j - 1 >= 0
                && do_exist(
                    board, word, current_position + 1, i, j - 1, visited
                )) {

                result = true;
            }

            if (i + 1 < board.size()
                && do_exist(
                    board, word, current_position + 1, i + 1, j, visited
                )) {

                result = true;
            }

            if (j + 1 < board[0].size()
                && do_exist(
                    board, word, current_position + 1, i, j + 1, visited
                )) {

                result = true;
            }
        }

        visited[i][j] = false;

        return result;
    }

    bool exist(vector<vector<char>> &board, string word) {
        std::vector<std::vector<char>> visited(
            board.size(), std::vector<char>(board[0].size(), false)
        );

        for (int i = 0; i < board.size(); i++) {
            for (int j = 0; j < board[0].size(); j++) {
                if (do_exist(board, word, 0, i, j, visited)) {
                    return true;
                }
            }
        }

        return false;
    }
};
```

## 84. 柱状图中最大的矩形 (Largest Rectangle in Histogram)

标签: 单调栈

解法:

- 遍历数组中的所有墙. 对于当前遍历到的墙, 将其暂时作为右墙, 并枚举目前单调栈中所有比该右墙高 (同时由于单调栈的性质也必然比左墙高, 于是能够形成合法的矩形) 的中间墙, 易知所形成的矩形的高度等于中间墙的高度, 而矩形宽度等于右墙位置与左墙位置之差. 于是我们便可通过遍历所有右墙来更新最终答案.
- 此外为了处理遍历结束时留存在单调栈中的未处理的左墙, 我们可以在原始数组中额外 push 进去一个高度为零的墙作为哨兵.

代码:

```cpp
class Solution {
public:
    int largestRectangleArea(vector<int> &heights) {
        heights.push_back(0);

        std::stack<int> increasing_wall_position_stack;

        int max_rectangle_area = 0;

        for (int current_position = 0; current_position < heights.size();
             current_position++) {
            while (!increasing_wall_position_stack.empty()
                   && heights[increasing_wall_position_stack.top()]
                          >= heights[current_position]) {

                const int rectangle_height =
                    heights[increasing_wall_position_stack.top()];
                increasing_wall_position_stack.pop();

                const int rectangle_width_end = current_position;
                const int rectangle_width_begin =
                    (increasing_wall_position_stack.empty()
                         ? -1
                         : increasing_wall_position_stack.top())
                    + 1;
                const int rectangle_width =
                    rectangle_width_end - rectangle_width_begin;

                const int rectangle_area = rectangle_height * rectangle_width;

                max_rectangle_area =
                    std::max(max_rectangle_area, rectangle_area);
            }

            increasing_wall_position_stack.push(current_position);
        }

        return max_rectangle_area;
    }
};
```

## 85. 最大矩形 (Maximal Rectangle)

标签: 单调栈

解法:

- 本题和 "84. 柱状图中最大的矩形 (Largest Rectangle in Histogram)" 有着千丝万缕的联系, 可以合理怀疑第 84 题其实就是为了求解本题而提出的一个子问题.
- 考虑一个矩形, 我们实际上可以将其视作 "从该矩形的底边开始向上生长的柱状图" 中的一个矩形, 于是求解 "矩阵中的最大矩形" 的问题就转化为了求解 "所有柱状图中的最大矩形的最大值" 的问题, 而后者已经在第 84 题中被解决, 直接套用即可.

代码:

```cpp
class Solution {
public:
    int get_max_area_from_heights(std::vector<int> &heights) {
        heights.push_back(0);

        std::stack<int> increasing_wall_position_stack;

        int max_rectangle_area = 0;

        for (int current_position = 0; current_position < heights.size();
             current_position++) {
            while (!increasing_wall_position_stack.empty()
                   && heights[increasing_wall_position_stack.top()]
                          >= heights[current_position]) {

                const int rectangle_height =
                    heights[increasing_wall_position_stack.top()];
                increasing_wall_position_stack.pop();

                const int rectangle_width_end = current_position;
                const int rectangle_width_begin =
                    (increasing_wall_position_stack.empty()
                         ? -1
                         : increasing_wall_position_stack.top())
                    + 1;
                const int rectangle_width =
                    rectangle_width_end - rectangle_width_begin;

                const int rectangle_area = rectangle_height * rectangle_width;

                max_rectangle_area =
                    std::max(max_rectangle_area, rectangle_area);
            }

            increasing_wall_position_stack.push(current_position);
        }

        return max_rectangle_area;
    }

    void update_heights(
        const std::vector<char> &current_matrix_row, std::vector<int> &heights
    ) {
        for (int i = 0; i < current_matrix_row.size(); i++) {
            heights[i] = current_matrix_row[i] == '1' ? (heights[i] + 1) : 0;
        }
    }

    int maximalRectangle(vector<vector<char>> &matrix) {
        std::vector<int> heights;

        heights.reserve(matrix[0].size() + 1);

        std::transform(
            matrix[0].begin(),
            matrix[0].end(),
            std::back_inserter(heights),
            [](char c) {
                return c - '0';
            }
        );

        int max_area = get_max_area_from_heights(heights);

        for (int i = 1; i < matrix.size(); i++) {
            const auto &current_matrix_row = matrix[i];

            update_heights(current_matrix_row, heights);

            const int current_max_area = get_max_area_from_heights(heights);

            max_area = std::max(max_area, current_max_area);
        }

        return max_area;
    }
};
```

## 94. 二叉树的中序遍历 (Binary Tree Inorder Traversal)

标签: 递归, 迭代

解法:

- 第一种解法是使用经典的递归.
- 第二种解法是使用迭代来模拟递归.

代码 (递归的解法):

(略)

代码 (迭代的解法):

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left),
 * right(right) {}
 * };
 */
class Solution {
private:
    enum CallStatus { START, INTO_LEFT_SUB_TREE, INTO_RIGHT_SUB_TREE };

    struct Node {
        TreeNode *root;
        CallStatus call_status;
    };

public:
    vector<int> inorderTraversal(TreeNode *root) {
        std::stack<Node> call_stack;
        std::vector<int> in_order_sequence;

        call_stack.push({root, START});

        while (!call_stack.empty()) {
            const auto root = call_stack.top().root;

            if (!root) {
                call_stack.pop();

                continue;
            }

            const auto call_status = call_stack.top().call_status;

            switch (call_status) {
            case START:
                call_stack.top().call_status = INTO_LEFT_SUB_TREE;
                call_stack.push({root->left, START});

                break;

            case INTO_LEFT_SUB_TREE:
                in_order_sequence.push_back(root->val);
                call_stack.top().call_status = INTO_RIGHT_SUB_TREE;
                call_stack.push({root->right, START});

                break;

            case INTO_RIGHT_SUB_TREE:
                call_stack.pop();

                break;
            }
        }

        return in_order_sequence;
    }
};
```

## 96. 不同的二叉搜索树 (Unique Binary Search Trees)

标签: 动态规划

解法:

- 由于二叉搜索树的左子树和右子树也是二叉搜索树, 原问题被转化为等价的子问题, 状态转移方程为 "当前大小的二叉搜索树的不同形状的总数" = "从零开始到当前大小减一的所有可能的左子树以及对应的右子树的不同形状的总数的乘积之和". 由于二叉搜索树区分左右, 因此左子树的大小应该从从零开始一直遍历到 "当前大小减一".

代码:

```cpp
class Solution {
public:
    int numTrees(int n) {
        std::vector<int> dp(n + 1, 0);

        dp[0] = 1;
        dp[1] = 1;

        for (int current_tree_size = 2; current_tree_size <= n;
             current_tree_size++) {

            for (int left_sub_tree_size = 0;
                 left_sub_tree_size <= current_tree_size - 1;
                 left_sub_tree_size++) {

                const int right_sub_tree_size =
                    current_tree_size - 1 - left_sub_tree_size;

                dp[current_tree_size] +=
                    dp[left_sub_tree_size] * dp[right_sub_tree_size];
            }
        }

        return dp[n];
    }
};
```

## 98. 验证二叉搜索树 (Validate Binary Search Tree)

标签: 递归, 中序遍历

解法:

- 第一种解法是使用递归. 因为二叉搜索树的左子树和右子树也必然是二叉搜索树, 因此我们可以通过递归验证左右子树来确定当前树是否为合法的二叉搜索树. 此外要想当前树为合法的二叉搜索树, 我们还需要确保左子树的最大值不超过当前树的树根的值, 同时右子树的最小值也不低于当前树的树根的值, 这可以通过在递归进入下一层时传入恰当的信息来做到.
- 第二种解法是使用中序遍历. 因为一棵树是二叉搜索树当且仅当对其进行中序遍历所得到的序列有序, 因此我们可以对树进行中序遍历, 在遍历过程中维护 "当前已遍历到的序列中的最大值" (因为我们并不需要整个序列, 而仅仅需要判断加入当前树的树根的值之后该序列是否仍然有序), 并使用该值检查当前树是否为合法的二叉搜索树.

代码 (递归的解法):

(略)

代码 (中序遍历的解法):

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left),
 * right(right) {}
 * };
 */
class Solution {
public:
    bool do_is_valid_bst(TreeNode *root, int64_t &previous_value) {
        if (root->left) {
            if (!do_is_valid_bst(root->left, previous_value)) {
                return false;
            }
        }

        if (previous_value >= root->val) {
            return false;
        }

        previous_value = root->val;

        if (root->right) {
            return do_is_valid_bst(root->right, previous_value);
        }

        else {
            return true;
        }
    }

    bool isValidBST(TreeNode *root) {
        int64_t previous_value = std::numeric_limits<int64_t>::min();

        return do_is_valid_bst(root, previous_value);
    }
};
```

## 101. 对称二叉树 (Symmetric Tree)

标签: 递归, 迭代

解法:

- 第一种解法是使用递归. 因为一棵二叉树是对称的当且仅当其左右子树为镜像对称的, 而两棵二叉树为镜像对称的当且仅当两棵二叉树的树根的值相同并且第一棵树的左子树与第二棵树的右子树镜像对称, 第一棵树的右子树也与第二棵树的左子树镜像对称. 于是原问题便被分解为等价的两个子问题, 可以使用递归进行求解.
- 第二种解法是使用迭代. 观察到在递归解法中, 所检查的实际上是每一对根结点的值是否相等, 因此我们完全可以通过迭代以任意顺序遍历所有根结点对, 只要确保不重复且不遗漏即可. 具体迭代方法包括但不限于广度优先搜索, 深度优先搜索等等.

代码 (递归的解法):

(略)

代码 (迭代的解法):

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left),
 * right(right) {}
 * };
 */
class Solution {
public:
    bool isSymmetric(TreeNode *root) {
        std::queue<TreeNode *> left_sub_tree_nodes;
        std::queue<TreeNode *> right_sub_tree_nodes;

        left_sub_tree_nodes.push(root->left);
        right_sub_tree_nodes.push(root->right);

        while (!left_sub_tree_nodes.empty()) {
            auto left_root = left_sub_tree_nodes.front();
            left_sub_tree_nodes.pop();

            auto right_root = right_sub_tree_nodes.front();
            right_sub_tree_nodes.pop();

            if (!left_root && !right_root) {
                continue;
            }

            if (!left_root || !right_root) {
                return false;
            }

            if (left_root->val != right_root->val) {
                return false;
            }

            left_sub_tree_nodes.push(left_root->left);
            left_sub_tree_nodes.push(left_root->right);

            right_sub_tree_nodes.push(right_root->right);
            right_sub_tree_nodes.push(right_root->left);
        }

        return true;
    }
};
```
