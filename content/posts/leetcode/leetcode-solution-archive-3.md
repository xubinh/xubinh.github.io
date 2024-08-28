---
title: "LeetCode 题解归档 - 3"
hideSummary: true
date: 2024-07-22T17:08:56+08:00
draft: true
tags: ["LeetCode"]
series: ["LeetCode"]
author: ["xubinh"]
type: posts
math: true
---

## 33. 搜索旋转排序数组

英文题目名称: Search in Rotated Sorted Array

标签: 数组, 二分查找

思路:

- 很早之前做过, 依稀记得当时挣扎了很久, 痛苦的记忆时至今日依然在脑海里留有痕迹. 本题完全可以用二分法做 (同时官方也暗示了使用二分法解决). 本题二分法的思路和普通的有序数组的二分法的思路基本无异, 唯一的区别是需要判断当前数组的形状以及当前所在区间的相对位置以便对向左还是向右二分进行决策. 下面介绍两种不同的思维方式, 其时间复杂度在常系数意义下相等:
  - 最符合直觉的思维方式是对目标 target 大于还是小于中间位置元素进行分情况讨论. 例如平常使用二分法时如果目标 target 小于中间位置的元素, 那么将向左二分, 而在本题的情况下则有可能向右二分, 向右二分当且仅当数组经过旋转 (数组最左端元素大于最右端元素), 当前中间位置元素位于旋转前数组的右半部分 (中间位置元素大于最左端元素), 并且目标 target 小于最左端元素. 其他情况同理.
  - 另一种不太符合直觉的思维方式是对中间位置元素位于旋转前数组的左半部分还是右半部分进行分情况讨论. 如果位于右半部分 (中间位置元素大于最左端元素, 不论是否存在旋转), 那么向右二分当且仅当目标 target 大于中间位置元素或小于最左端元素. 其他情况同理.

代码:

```cpp
class Solution {
public:
    int search(vector<int> &nums, int target) {
        int left_end = 0;
        int right_end = nums.size();

        while (right_end - left_end >= 2) {
            int left_end_number = nums[left_end];
            int right_end_number = nums[right_end - 1];

            int middle_position = left_end + (right_end - left_end) / 2;
            int middle_position_number = nums[middle_position];

            if (target == middle_position_number) {
                return middle_position;
            } else if (middle_position_number > left_end_number) {
                if (target >= left_end_number
                    && target < middle_position_number) {
                    right_end = middle_position;
                } else {
                    left_end = middle_position + 1;
                }
            } else {
                if (target > middle_position_number
                    && target <= right_end_number) {
                    left_end = middle_position + 1;
                } else {
                    right_end = middle_position;
                }
            }
        }

        if (right_end - left_end <= 0) {
            return -1;
        }

        return nums[left_end] == target ? left_end : -1;
    }
};
```

## 46. 全排列

英文题目名称: Permutations

标签: 数组, 回溯

思路:

- 一开始是按照常规的标记位数组的思路来的, 第几个下标的数字用了就将第几个下标的标记位置为 `false`, 然后每当用完所有数字时就将当前的全排列收集起来. 这样做的话每次遍历到非叶结点都需要 $O(n)$ 的遍历标记位数组的时间, 而每次遍历到叶结点也需要 $O(n)$ 的收集时间, 总时间复杂度为

  $$
  \begin{equation*}
  \begin{aligned}
  &\\, \\, \quad \left(1 + n + n (n - 1) + \dots + \frac{n!}{1!} + n!\right) \times n\\
  &= \left(\frac{n!}{n!} + \frac{n!}{(n - 1)!} + \dots + \frac{n!}{1!} + n!\right) \times n\\
  &\leq \left(\frac{n!}{2^(n - 1)} + \frac{n!}{2^{n - 2}} + \dots + \frac{n!}{2^0} + n!\right) \times n\\
  &\leq n! \cdot 3n.
  \end{aligned}
  \end{equation*}
  $$

- 提交之后发现时间复杂度落在第二档, 于是跑去看题解, 发现可以通过原地置换的方法避免标记位数组 (以及中间结果数组) 的使用. 使用原地置换后的时间复杂度为

  $$
  n + n (n - 1) + n (n - 1) (n - 2) + \dots + n! + n! \times n <= n! \cdot (3 + n).
  $$

代码 (常规标记位数组做法):

```cpp
class Solution {
public:
    vector<vector<int>> permute(vector<int> &nums) {
        vector<vector<int>> permutations;
        vector<int> used_numbers;
        int number_of_used_numbers = 0;
        vector<bool> flags_is_number_used(nums.size(), false);
        DFS_for_permute(permutations, nums, used_numbers,
                        number_of_used_numbers, flags_is_number_used);

        return permutations;
    }
    void DFS_for_permute(vector<vector<int>> &permutations, vector<int> &nums,
                         vector<int> &used_numbers, int number_of_used_numbers,
                         vector<bool> &flags_is_number_used) {
        if (number_of_used_numbers == nums.size()) {
            permutations.emplace_back(used_numbers);
            return;
        }

        for (int number_index = 0; number_index < nums.size(); number_index++) {
            int current_selected_number = nums[number_index];

            if (flags_is_number_used[number_index]) {
                continue;
            }

            flags_is_number_used[number_index] = true;
            used_numbers.push_back(current_selected_number);
            DFS_for_permute(permutations, nums, used_numbers,
                            number_of_used_numbers + 1, flags_is_number_used);
            used_numbers.pop_back();
            flags_is_number_used[number_index] = false;
        }
    }
};
```

代码 (原地置换做法):

```cpp
class Solution {
public:
    vector<vector<int>> permute(vector<int> &nums) {
        vector<vector<int>> permutations;
        DFS_for_permute(permutations, nums, 0);

        return permutations;
    }
    void DFS_for_permute(vector<vector<int>> &permutations, vector<int> &nums,
                         int number_of_used_numbers) {
        if (number_of_used_numbers == nums.size()) {
            permutations.emplace_back(nums);
            return;
        }

        for (int number_index = number_of_used_numbers;
             number_index < nums.size(); number_index++) {
            swap(nums[number_of_used_numbers], nums[number_index]);
            DFS_for_permute(permutations, nums, number_of_used_numbers + 1);
            swap(nums[number_of_used_numbers], nums[number_index]);
        }
    }
};
```

## 53. 最大子数组和

英文题目名称: Maximum Subarray

标签: 数组, 分治, 动态规划

思路:

- 很经典的一道动态规划. 如果不使用动态规划, 暴力做法就是枚举每个子数组并计算子数组和, 如果在枚举过程中维护一个前缀和, 时间复杂度虽然可以降到 $O(n^2)$, 但也没法再少了. 动态规划的做法是对 "以位于下标 $i$ 处的数字作为右端的所有可能子数组之和的最大值 $dp[i]$" 进行枚举, 最终结果为所有 $dp[i]$ 的最大值, 状态转移方程为 $dp[i] = \max(nums[i], \\ \\ nums[i] + dp[i - 1])$, 时间复杂度为 $O(n)$.
- 从状态转移方程中可以看出当前状态仅依赖于前一个状态, 因此动态规划解法完全可以使用滚动数组进行优化. 优化后的空间复杂度为 $O(1)$.

代码:

```cpp
class Solution {
public:
    int maxSubArray(vector<int> &nums) {
        int max_sum_of_sub_array_ends_at_current_position = nums[0];
        int max_sub_array_sum = nums[0];

        for (int current_position = 1; current_position < nums.size();
             current_position++) {
            int current_number = nums[current_position];
            max_sum_of_sub_array_ends_at_current_position = max(
                current_number,
                current_number + max_sum_of_sub_array_ends_at_current_position);
            max_sub_array_sum =
                max(max_sub_array_sum,
                    max_sum_of_sub_array_ends_at_current_position);
        }

        return max_sub_array_sum;
    }
};
```

## 62. 不同路径

英文题目名称: Unique Paths

标签: 数学, 动态规划, 组合数学

思路:

- 这题也是很经典的一道动态规划. 易知状态转移方程为 "当前位置的总路径数等于左边位置的总路径数与上面位置的总路径数之和". 虽然可以使用二维数组进行动态规划, 但这么做空间复杂度显然太高. 由于只需要左边和上面的两个状态, 本题和背包问题一样可以使用滚动数组进行优化, 空间复杂度为 $O(n)$.
- 但是自己并不是用的动态规划的做法, 因为上述状态转移方程实际上构建出了一个杨辉三角, 因此直接计算组合数即可.
- 计算组合数时需要注意溢出问题, 本题即使采用了先乘分子再除分母的方法也无法避免溢出, 因此最后还是使用了 `long long` 类型.

代码 (组合数方法):

```cpp
class Solution {
public:
    int uniquePaths(int m, int n) {
        int combinatorial_n = m + n - 2;
        int combinatorial_k = m - 1;
        int number_of_paths = get_combinatorial_number(
            combinatorial_n,
            min(combinatorial_k, combinatorial_n - combinatorial_k));

        return number_of_paths;
    }

    int get_combinatorial_number(int n, int k) {
        long long result = 1;
        for (int i = 1; i <= k; i++) {
            result *= (n + 1 - i);
            result /= i;
        }
        return (int)result;
    }
};
```
