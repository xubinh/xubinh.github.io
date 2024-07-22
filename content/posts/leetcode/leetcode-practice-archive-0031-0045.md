---
title: "LeetCode 题解归档 (0031~0045)"
hideSummary: true
date: 2024-07-22T17:08:56+08:00
draft: false
tags: ["leetcode"]
series: ["leetcode"]
author: ["xubinh"]
type: posts
math: false
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
