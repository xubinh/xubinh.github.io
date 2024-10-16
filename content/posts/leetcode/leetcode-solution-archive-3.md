---
title: "LeetCode 题解归档 - 3"
hideSummary: true
date: 2024-10-14T22:39:56+08:00
draft: false
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
  &\, \, \quad \left(1 + n + n (n - 1) + \dots + \frac{n!}{1!} + n!\right) \times n \\
  &= \left(\frac{n!}{n!} + \frac{n!}{(n - 1)!} + \dots + \frac{n!}{1!} + n!\right) \times n \\
  &\leq \left(\frac{n!}{2^(n - 1)} + \frac{n!}{2^{n - 2}} + \dots + \frac{n!}{2^0} + n!\right) \times n \\
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

## 70. 爬楼梯

英文题目名称: Climbing Stairs

标签: 记忆化搜索, 数学, 动态规划

思路:

- 本题一眼动态规划. 一开始还没发现, 做到后面发现这不就是斐波那契数列么.
- 看了题解居然还有矩阵+快速幂以及直接通项公式两种做法, 这就属于奇技淫巧了, 自己并不大想去单独记这些东西, 能记住并在面试的时候提一嘴最好, 记不住那也认了.

代码:

```cpp
class Solution {
public:
    int climbStairs(int n) {
        if (n == 1) {
            return 1;
        }

        if (n == 2) {
            return 2;
        }

        int number_of_methods_for_n_minus_2;
        int number_of_methods_for_n_minus_1 = 1;
        int number_of_methods_for_n = 2;
        int count = 3;

        while (count <= n) {
            number_of_methods_for_n_minus_2 = number_of_methods_for_n_minus_1;
            number_of_methods_for_n_minus_1 = number_of_methods_for_n;
            number_of_methods_for_n = number_of_methods_for_n_minus_2
                                      + number_of_methods_for_n_minus_1;
            count += 1;
        }

        return number_of_methods_for_n;
    }
};
```

## 78. 子集

英文题目名称: Subsets

标签: 位运算, 数组, 回溯

思路:

- 本题一眼递归, 采用的就是一般的 DFS 的做法, 主函数 `subsets` 仅作为入口, 然后自己另外写一个用于 DFS 的帮手函数 `_subsets` 进行递归并在递归到每个叶路径的时候将当前答案 push back 到总的答案集合中.
- 最后一提交发现运行速度打败了 1% 的用户, 这不优化一下不得直接失眠了. 一开始想到的优化手段是将对一整个 `vector` 的 `push_back` 替换为 `emplace_back`, 但仔细一想好像不论是哪个都调用的拷贝构造函数, 最多最多就是 `std::move` 一下顶天了, 而 `std::move` 放在这里也不合适, 因为将要复制的 `vector` 之后还要用, 得到的推论就是要传入整个 `vector` 就避免不了对这个 `vector` 的整体复制. 最后想到 `vector` 还有个关于迭代器的构造函数, 使用迭代器进行 inplace 构造相当于在两个数组之间进行复制, 这再怎么样也比先复制整个 `vector` 再进行逐元素复制要快, 于是便采用这个方法并顺利优化运行时间至击败 100%.
- 除了运行时间, 内存消耗击败 5% 也得优化一下. 一开始想到的是将递归改写为栈, 但这个想法太复杂了. 看了题解发现可以通过枚举二进制模式并根据每个模式临时构造对应的序列, 这种做法虽然思维难度为零, 但是一点也不优雅, 自己并不想采用. 最后想着把题解的递归做法提交一遍看看它的内存消耗会不会比自己的小, 如果不会那自己也不改了, 结果一提交发现题解的内存消耗居然还不错. 经过二分法对齐自己和题解的代码之后发现很奇妙的一点是只需要将

  ```cpp
  // 不使用当前数字:
  _subsets(nums, range_end + 1, current_used_integers, results);

  // 使用当前数字:
  current_used_integers.push_back(nums[range_end]);
  _subsets(nums, range_end + 1, current_used_integers, results);
  current_used_integers.pop_back();
  ```

  上下替换一下改为

  ```cpp
  // 使用当前数字:
  current_used_integers.push_back(nums[range_end]);
  _subsets(nums, range_end + 1, current_used_integers, results);
  current_used_integers.pop_back();

  // 不使用当前数字:
  _subsets(nums, range_end + 1, current_used_integers, results);
  ```

  内存消耗就下去了, 这应该是涉及到硬件例如缓存命中率之类的东西了, 这就太微妙了. 由于上面的形式直观上是 "先短后长", 下面的形式直观上是 "先分配后缩短", 因此两种代码组织方法在内存消耗上的差异估计和 `malloc` 的底层行为有关.

代码:

```cpp
class Solution {
public:
    vector<vector<int>> subsets(vector<int> &nums) {
        vector<vector<int>> results;
        vector<int> current_used_integers;

        _subsets(nums, 0, current_used_integers, results);

        return results;
    }

    void _subsets(const vector<int> &nums, int range_end,
                  vector<int> &current_used_integers,
                  vector<vector<int>> &results) {
        if (range_end >= nums.size()) {
            results.emplace_back(current_used_integers.begin(),
                                 current_used_integers.end());
            return;
        }

        // 使用当前数字:
        current_used_integers.push_back(nums[range_end]);
        _subsets(nums, range_end + 1, current_used_integers, results);
        current_used_integers.pop_back();

        // 不使用当前数字:
        _subsets(nums, range_end + 1, current_used_integers, results);

        return;
    }
};
```

## 104. 二叉树的最大深度

英文题目名称: Maximum Depth of Binary Tree

标签: 树, 深度优先搜索, 广度优先搜索, 二叉树

思路:

- 一开始自己是从上往下的思路, 即将根节点 `root` 的深度定义为 `1`, 然后在往下层递归的过程中通过参数为当前层打上一个深度标签. 这种做法能直接秒, 但是看了题解之后发现普遍的切入口是子问题分解那种, 也就是 "当前树的深度等于左右两个子树的深度的最大值加 1", 相当于从下往上聚合. 两种思路都是 DFS, 但是后一种更加 "面试" 一些, 因此最后还是选择重写为后一种做法. 使用后一种做法还有一个比较爽的点是思维难度极低, 写出来的代码只需要一行.
- 除了 DFS, 自然还有 BFS 的做法. BFS 的做法本质上也是从上往下传播, 不过用的是队列那一套, 层与层之间通过在进入对队列的遍历时所记录的队列长度来分隔.

代码 (DFS 版本):

```cpp
class Solution {
public:
    int maxDepth(TreeNode* root) {
        return root ? max(maxDepth(root->left), maxDepth(root->right)) + 1 : 0;
    }
};
```

## 121. 买卖股票的最佳时机

英文题目名称: Best Time to Buy and Sell Stock

标签: 数组, 动态规划

思路:

- 做这种最终答案等于两个变量的计算结果的题总是会第一时间往维护两个历史数据的方向上去想, 但实际上本题只需要维护一个历史数据, 即 $[0, i)$ 这一个范围内的最小价格, 因为买卖肯定是右边 (时间晚) 价格高的减去左边 (时间早) 价格低的. 最终结果等于所有下标上的结果的最大值, 所以只需要在遍历的过程中顺便 max 一下就 OK.
- 另外就是看了题解发现它的做法除了在细节上有些不同以外和自己的代码几乎相同, 这个感觉挺爽的.

代码:

```cpp
class Solution {
public:
    int maxProfit(vector<int> &prices) {
        int max_profit = 0;
        int min_price_before_current_position = prices[0];
        for (int current_position = 1; current_position < prices.size();
             ++current_position) {
            if (prices[current_position] > min_price_before_current_position) {
                max_profit =
                    max(max_profit, prices[current_position]
                                        - min_price_before_current_position);
            } else {
                min_price_before_current_position =
                    min(min_price_before_current_position,
                        prices[current_position]);
            }
        }

        return max_profit;
    }
};
```

## 160. 相交链表

英文题目名称: Intersection of Two Linked Lists

标签: 哈希表, 链表, 双指针

思路:

- 很早之前做过这道题, 印象中依稀记得这是一道脑筋急转弯, 做下来发现确实想不出来, 因此一开始的做法是直接使用哈希表暴力求解. 哈希表的做法速度还行, 就是比较占内存.
- 看了题解之后理解了脑筋急转弯的做法. 设链表 A 的非公共部分为 a, 链表 B 的非公共部分为 b, 两个链表的公共部分为 c, 脑筋急转弯的做法实际上就是让两个指针都走 a + b + c 步. 如果 c 等于零, 即两个链表不相交, 那么两个指针走着走着会同时走到互相的链表尾部, 反之如果两个链表确实存在相交, 那么两个指针走着走着必然会碰到一起. 这种做法没有什么思路的切入点, 只能靠背诵强行给它记住.

代码:

```cpp
class Solution {
public:
    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        auto headA_temp = headA;
        auto headB_temp = headB;

        while (1) {
            if (!headA && !headB) {
                return nullptr;
            }

            if (headA == headB) {
                return headA;
            }

            if (!headA) {
                headA = headB_temp;
            } else {
                headA = headA->next;
            }

            if (!headB) {
                headB = headA_temp;
            } else {
                headB = headB->next;
            }
        }

        return nullptr;
    }
};
```

## 206. 反转链表

英文题目名称: Reverse Linked List

标签: 递归, 链表

思路:

- 这道题用递归的方法非常简单, 但也正因为如此非常容易就这样结束了, 说实在的如果没有题目末尾特意提一下这道题还有个迭代的解法我是真想不到. 估计面试的时候面试官很大概率会提一嘴迭代的方法.
- 迭代方法的关键点就是找到循环不变量, 并且设计好代码的逻辑顺序确保进入和退出循环不变量的时候不会出错. 关于循环不变量, 每轮迭代手里都有一前一后两个结点, 前面结点已经处理好了, 要做的就是在把后面结点的 next 指针挂到前面结点的同时保存好后面结点的 next 指针的原来的值, 以便顺利进入下一轮迭代. 进入循环的准备工作是将首个结点的 next 指针设置为 null, 同时保存好首个结点的 next 指针的原始值, 然后令首个结点为前面结点, next 指针的原始值为后面结点, 并进入循环. 退出循环的时机是后面结点为 null 的时候. 此外上述方法同样能够正确处理链表只有一个结点的情况.

代码:

```cpp
class Solution {
public:
    ListNode *reverseList(ListNode *head) {
        if (!head) {
            return nullptr;
        }

        // 准备工作:
        auto current_node = head;
        auto next_node = head->next;
        head->next = nullptr;

        // 进入循环:
        while (next_node) {
            // 保存好原来的 next 指针:
            auto temp_node = next_node->next;

            // 后面的结点反过来挂到前面的结点:
            next_node->next = current_node;

            // 移动窗口:
            current_node = next_node;
            next_node = temp_node;
        }

        return current_node;
    }
};
```

## 236. 二叉树的最近公共祖先

英文题目名称: Lowest Common Ancestor of a Binary Tree

标签: 树, 深度优先搜索, 二叉树

思路:

- 这题一开始以为很简单, 思考一会之后才发现有蹊跷. 一般这种二叉树的题都是采用 "获取左子树的返回值 -> 获取右子树的返回值 -> 结合两个子树的返回值得到当前树的结果并返回" 的思路, 而本题蹊跷就蹊跷在**一个返回值根本不够用**, 因为有太多种可能结果:
  1. 当前树中啥也没有.
  1. 当前树的根结点为 `p` (或 `q`), 并且 `q` (或 `p`) 并不位于当前树中.
  1. 当前树的根结点为 `p` (或 `q`), 并且 `q` (或 `p`) 位于当前树的某个子树中.
  1. 当前树的左子树同时包含 `p` 和 `q`.
  1. 当前树的右子树同时包含 `p` 和 `q`.
  1. 当前树的左右子树分别包含 `p` 和 `q`.
- 为了解决单一返回值表达能力不足的问题, 最终采用的办法是将单一返回值**结合上下文进行多种解释**. 设主函数 `lowestCommonAncestor(root, p, q)` 的逻辑为:

  1. 如果根结点恰为 `p`, `q` 之一则直接返回根结点. **此时根结点并不一定是最近公共祖先, 需要调用者根据上下文进行确定**.
  1. 否则递归向左子树和右子树进行调用:
     1. 如果左子树和右子树均返回了非空指针, 说明当前树就是最近公共祖先, 因此返回根结点.
     1. 如果左子树和右子树均返回空指针, 那说明什么也没有, 因此当前树也返回空指针.
     1. 否则仅有左子树和右子树其中之一返回了非空指针, 那么返回该指针. **此时该指针并不一定是最近公共祖先, 需要调用者根据上下文进行确定**.

  "上下文" 除了包含 "左右子树的结果的是否非空" 的信息以外, 实际上还包含 "当前树是否为顶层树" 的信息:

  1. 如果当前树为顶层树, 那么由 "根结点恰为 `p`, `q` 之一" 可立即推出最近公共祖先为根结点, 因此 `lowestCommonAncestor` 应该返回根结点, 而在上述逻辑中 `lowestCommonAncestor` 返回的也确实是根结点.
  1. 如果当前树为顶层树, 那么由 "仅有左子树和右子树其中之一返回了非空指针" 可立即推出最近公共祖先为所返回的指针. 而上述逻辑中所返回的也确实是该指针.

代码:

```cpp
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if (root == p || root == q) {
            return root;
        }

        if (!root->left && !root->right) {
            return nullptr;
        }

        if (!root->left) {
            return lowestCommonAncestor(root->right, p, q);
        }

        if (!root->right) {
            return lowestCommonAncestor(root->left, p, q);
        }

        auto left_sub_tree_result = lowestCommonAncestor(root->left, p, q);
        auto right_sub_tree_result = lowestCommonAncestor(root->right, p, q);

        if (left_sub_tree_result && right_sub_tree_result) {
            return root;
        }

        return left_sub_tree_result ? left_sub_tree_result
                                    : right_sub_tree_result;
    }
};
```

## 141. 环形链表

英文题目名称: Linked List Cycle

标签: 哈希表, 链表, 双指针

思路:

- 快慢指针绕圈追尾, 没啥好说的. 这题才是快慢指针, 前面那题 "160. 相交链表" 之前做的时候一直往快慢指针上面靠结果最后发现并不是, 浪费了许多时间.

代码:

```cpp
class Solution {
public:
    bool hasCycle(ListNode* head) {
        if (!head) {
            return false;
        }

        auto fast_pointer = head;
        auto slow_pointer = head;

        while (1) {
            fast_pointer = fast_pointer->next;

            if (!fast_pointer) {
                return false;
            }

            fast_pointer = fast_pointer->next;

            if (!fast_pointer) {
                return false;
            }

            slow_pointer = slow_pointer->next;

            if (slow_pointer == fast_pointer) {
                return true;
            }
        }

        return false;
    }
};
```

## 136. 只出现一次的数字

英文题目名称: Single Number

标签: 位运算, 数组

思路:

- 很早之前做过的一道脑筋急转弯, 做法是天杀的异或运算. 因为形象地讲异或运算具有一种 "你可以拿着信息往里加, 也可以在之后再拿着信息从里面减掉 (抵消掉)" 的性质. 同一个二进制模式连续异或两次会 (实际上只要是偶数次均可) 将自己抵消掉, 而不会对原始信息造成任何影响. 因此只要将数组 `nums` 中的所有元素全都异或一遍, 出现两次的那些元素全都抵消掉, 剩下的就是只出现一次的那个数字了.
- 这里初始值为 `0`. 从信息论的角度来讲 `0` 代表信息量为零, 添加多少信息就会显示多少信息, 而从数学的角度来讲就是 "任何数与 `0` 进行异或都等于它本身".

代码:

```cpp
class Solution {
public:
    int singleNumber(vector<int>& nums) {
        return std::accumulate(nums.cbegin(), nums.cend(), 0,
                               std::bit_xor<int>());
    }
};
```

## 146. LRU 缓存

英文题目名称: LRU Cache

标签: 设计, 哈希表, 链表, 双向链表

思路:

- 很好的一道题, 很早之前做过但是忘光了, 这次相当于复习了一遍, 学到很多东西.
- 因为要实现 LRU, 所以肯定使用双向链表. 同时需要使用一个哈希表存储从键到对应的链表结点指针的映射.
- **双向链表一定要使用假头结点和假尾结点**, 自己第一遍做的时候没有这个意识, 写出来的代码这里检查那里检查的, 可读性基本为零.
- 实现链表的基本操作的时候不要同步修改除链表以外的其他成员, 例如哈希表等等. 因为很多时候将一个结点从链表中移出很快又会把它移入, 这种动作只会影响链表, 并不会影响其他成员. 修改其他成员的动作放到调用者中去做.
- 插入和删除元素的时候最好按照成员的定义顺序一个一个检查是否造成改变, 以避免遗漏, 并且删除元素对成员的修改动作最好和插入元素的逆着来.

代码:

```cpp
struct KeyValuePairNode {
    const int key;
    int value;
    KeyValuePairNode *previous;
    KeyValuePairNode *next;
    KeyValuePairNode(const int &key_input, const int &value_input)
        : key(key_input), value(value_input), previous(nullptr), next(nullptr) {
    }
};

class LRUCache {
private:
    const int capacity;
    KeyValuePairNode *phony_head = new KeyValuePairNode(0, 0);
    KeyValuePairNode *phony_tail = new KeyValuePairNode(0, 0);
    int size = 0;
    unordered_map<int, KeyValuePairNode *> key2address;

public:
    LRUCache(int capacity_input) : capacity(capacity_input) {
        phony_head->next = phony_tail;
        phony_tail->previous = phony_head;
    }

    void _pop(KeyValuePairNode *const &target_ptr) {
        target_ptr->next->previous = target_ptr->previous;
        target_ptr->previous->next = target_ptr->next;
    }

    KeyValuePairNode *_pop_back() {
        auto back = phony_tail->previous;

        _pop(back);

        return back;
    }

    void _insert_to_front(KeyValuePairNode *const &target_ptr) {
        target_ptr->next = phony_head->next;
        phony_head->next->previous = target_ptr;
        phony_head->next = target_ptr;
        target_ptr->previous = phony_head;
    }

    void _move_to_front(KeyValuePairNode *const &target_ptr) {
        _pop(target_ptr);
        _insert_to_front(target_ptr);
    }

    int get(int key) {
        if (!key2address.count(key)) {
            return -1;
        }

        auto target_ptr = key2address[key];

        _move_to_front(target_ptr);

        return target_ptr->value;
    }

    void put(int key, int value) {
        if (key2address.count(key)) {
            auto target_ptr = key2address[key];

            target_ptr->value = value;

            _move_to_front(target_ptr);
        } else {
            auto target_ptr = new KeyValuePairNode(key, value);

            _insert_to_front(target_ptr);

            ++size;

            key2address[key] = target_ptr;

            if (size > capacity) {
                auto back = _pop_back();

                key2address.erase(back->key);

                --size;

                delete back;
            }
        }
    }
};
```

## 148. 排序链表

英文题目名称: Sort List

标签: 链表, 双指针, 分治, 排序, 归并排序

思路:

- 因为是链表, 无法随机读写, 所以只能归并排序.
  - 力扣题解还提到一个自底向上的**迭代**的做法, 那个做法太过于依赖微操, 面试遇到直接开摆.
- 可以将主函数分解为 "对链表进行二分" -> "递归对两个子链表进行排序" -> "归并两个有序链表", 其中将 "二分" 实现为一个函数, 将 "归并" 实现为另一个函数.
- 二分函数直接用快慢指针的话需要特别注意 "off-by-one" 细节, 因为需要将左子链表和右子链表断开, 而需要注意的点就是判定什么时候到达了左子链表的尾结点. 归并函数没什么好说的, 就正常实现即可.
- 另外提一嘴不要使用哨兵结点什么的, 因为这里存在递归, 只要使用哨兵结点就是至少 $O(\log{n})$ (实际上是 $O(n)$) 次的 `malloc` 消耗.

代码:

```cpp
class Solution {
public:
    ListNode *sortList(ListNode *head) {
        if (!head || !head->next) {
            return head;
        }

        head = _sort_list(head);

        return head;
    }

    // 对链表进行实际的递归的排序, 假设链表长度大于等于 2:
    ListNode *_sort_list(ListNode *list) {
        auto new_list = _split_list(list);

        if (list->next) {
            list = _sort_list(list);
        }

        if (new_list->next) {
            new_list = _sort_list(new_list);
        }

        list = _merge_two_ordered_lists(list, new_list);

        return list;
    }

    // 二分链表, 假设链表长度大于等于 2, 返回新链表的头结点:
    ListNode *_split_list(ListNode *list) {
        auto fast_ptr = list;
        auto slow_ptr = list;

        // 这里需要注意 off-by-one 的细节, 比较微妙, 最好在脑子里复现一下:
        while (fast_ptr->next && fast_ptr->next->next) {
            fast_ptr = fast_ptr->next->next;
            slow_ptr = slow_ptr->next;
        }

        auto new_list_head = slow_ptr->next;
        slow_ptr->next = nullptr;

        return new_list_head;
    }

    // 合并两个有序非空链表:
    ListNode *_merge_two_ordered_lists(ListNode *list_1, ListNode *list_2) {
        ListNode *new_list_head = list_1->val < list_2->val ? list_1 : list_2;
        list_1 = new_list_head == list_1 ? list_1->next : list_1;
        list_2 = new_list_head == list_2 ? list_2->next : list_2;
        auto new_list_tail = new_list_head;

        while (list_1 && list_2) {
            if (list_1->val < list_2->val) {
                new_list_tail->next = list_1;
                list_1 = list_1->next;
            } else {
                new_list_tail->next = list_2;
                list_2 = list_2->next;
            }

            new_list_tail = new_list_tail->next;
        }

        new_list_tail->next = list_1 ? list_1 : list_2;

        return new_list_head;
    }
};
```
