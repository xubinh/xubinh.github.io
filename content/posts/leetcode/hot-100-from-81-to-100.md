---
title: "力扣 Hot 100 题解归档 (81-100)"
hideSummary: true
date: 2025-01-27T00:26:56+08:00
draft: false
tags: ["leetcode"]
series: ["leetcode"]
author: ["xubinh"]
type: posts
math: true
---

## 337. 打家劫舍 III (House Robber III)

标签: 递归

解法:

- 对于当前遍历到的树, 其最大金额等于 "偷窃根结点所能得到的最大金额" 与 "不偷窃根结点所能得到的最大金额" 之间的较大值, 因此本题使用标准的递归即可解答.

代码:

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
    void do_rob(
        TreeNode *root,
        int &max_profit_when_robbing_root,
        int &max_profit_when_not_robbing_root
    ) {
        int max_profit_when_robbing_left_sub_tree_root = 0;
        int max_profit_when_not_robbing_left_sub_tree_root = 0;

        if (root->left) {
            do_rob(
                root->left,
                max_profit_when_robbing_left_sub_tree_root,
                max_profit_when_not_robbing_left_sub_tree_root
            );
        }

        const int max_profit_of_left_sub_tree = std::max(
            max_profit_when_robbing_left_sub_tree_root,
            max_profit_when_not_robbing_left_sub_tree_root
        );

        int max_profit_when_robbing_right_sub_tree_root = 0;
        int max_profit_when_not_robbing_right_sub_tree_root = 0;

        if (root->right) {
            do_rob(
                root->right,
                max_profit_when_robbing_right_sub_tree_root,
                max_profit_when_not_robbing_right_sub_tree_root
            );
        }

        const int max_profit_of_right_sub_tree = std::max(
            max_profit_when_robbing_right_sub_tree_root,
            max_profit_when_not_robbing_right_sub_tree_root
        );

        max_profit_when_robbing_root =
            root->val + max_profit_when_not_robbing_left_sub_tree_root
            + max_profit_when_not_robbing_right_sub_tree_root;

        max_profit_when_not_robbing_root =
            max_profit_of_left_sub_tree + max_profit_of_right_sub_tree;
    }

    int rob(TreeNode *root) {
        if (!root) {
            return 0;
        }

        int max_profit_when_robbing_root;
        int max_profit_when_not_robbing_root;

        do_rob(
            root, max_profit_when_robbing_root, max_profit_when_not_robbing_root
        );

        const int max_profit = std::max(
            max_profit_when_robbing_root, max_profit_when_not_robbing_root
        );

        return max_profit;
    }
};
```

## 338. 比特位计数 (Counting Bits)

标签: 位运算, 动态规划

解法:

- 第一种解法是使用 Brian Kernighan 算法, 该算法常用于一种名叫 "树状数组" 的数据结构中. 该算法的关键在于其注意到表达式 `x & (x - 1)` 会将 `x` 的 "最低非零比特位" 置零. 于是我们便可通过观察将整个 `x` 置零所需的次数来确定 `x` 的比特位总数.
- 第二种解法是使用关于最高有效位的动态规划方法. 观察到将一个数的最高有效位置零后所得到的数必然小于该数, 并且所得到的数的比特位总数总是等于该数的比特位总数减一, 因此我们可以通过在从小到大遍历的过程中维护当前数的最高有效位来快速得到当前数的比特位总数.
- 第三种解法是使用关于最低有效位的动态规划方法. 观察到将一个数向右进行逻辑移位一位所得到的数必然小于该数, 并且所得到的数的比特位总数总是等于该数的比特位总数减去最低有效位, 因此我们可以直接通过该数自身来快速得到当前数的比特位总数.

代码 (使用 Brian Kernighan 算法的解法):

(略)

代码 (使用关于最高有效位的动态规划的解法):

(略)

代码 (使用关于最低有效位的动态规划的解法):

```cpp
class Solution {
public:
    vector<int> countBits(const int n) {
        std::vector<int> dp(n + 1);

        for (int i = 1; i <= n; i++) {
            dp[i] = dp[i >> 1] + (i & 1);
        }

        return dp;
    }
};
```

## 347. 前 K 个高频元素 (Top K Frequent Elements)

标签: 堆排序, 快速排序

解法:

- 第一种解法是使用堆排序. 首先使用一个哈希表获取每个元素的出现频数, 然后针对哈希表中的频数数组进行堆排序. 由于哈希表无法随机读写, 此处选择维护一个大小为 `k` 的堆并动态添加或删除元素 (正常做法是建堆然后弹出 `k` 个堆顶元素).
- 第二种解法是使用快速排序中的扫描算法. 这类似于 "215. 数组中的第K个最大元素 (Kth Largest Element in an Array)" 中的做法, 这里就不做过多介绍了.

代码 (堆排序的解法):

```cpp
class Solution {
private:
    struct Node {
        int value;
        int appeared_times;
    };

    struct NodeCompare {
        bool operator()(const Node &node_1, const Node &node_2) {
            return node_1.appeared_times == node_2.appeared_times
                       ? node_1.value < node_2.value
                       : node_1.appeared_times > node_2.appeared_times;
        }
    };

public:
    vector<int> topKFrequent(vector<int> &nums, int k) {
        std::unordered_map<int, int> value_2_appeared_times;

        for (auto number : nums) {
            value_2_appeared_times[number]++;
        }

        auto it = value_2_appeared_times.begin();

        std::priority_queue<Node, std::vector<Node>, NodeCompare> max_k_heap;

        for (int i = 0; i < k; i++, it++) {
            max_k_heap.push(Node{it->first, it->second});
        }

        for (int i = k; it != value_2_appeared_times.end(); i++, it++) {
            if (it->second <= max_k_heap.top().appeared_times) {
                continue;
            }

            max_k_heap.pop();
            max_k_heap.push(Node{it->first, it->second});
        }

        std::vector<int> max_k_elements;

        while (!max_k_heap.empty()) {
            max_k_elements.push_back(max_k_heap.top().value);
            max_k_heap.pop();
        }

        return max_k_elements;
    }
};
```

代码 (快速排序的解法):

(略)

## 394. 字符串解码 (Decode String)

标签: 递归

解法:

- 考虑对原始字符串从左到右进行遍历. 对于当前遍历到的字符:
  - 如果该字符是一般的小写英文字母, 那么直接将其加入当前子串中.
  - 如果该字符是数字, 说明遇到了整数, 那么我们将该整数解码, 定位到紧接着该整数的下一个左括号, 然后使用递归对由该左括号确定的模式进行解析. 返回时当前位置将位于对应的右括号上, 我们将当前位置加一跳过该右括号即可.

  最终所得的子串即为当前模式所对应的原始子串.

代码:

```cpp
class Solution {
public:
    int parse_int(const std::string &s, int &current_position) {
        const char *start_ptr = s.c_str() + current_position;
        char *end_ptr;

        long value = ::strtol(start_ptr, &end_ptr, 10);

        current_position = end_ptr - s.c_str();

        return static_cast<int>(value);
    }

    std::string do_decode_string(const std::string &s, int &current_position) {
        std::string current_string_piece;

        while (current_position < s.size()) {
            const char current_char = s[current_position];

            if (current_char == ']') {
                break;
            }

            else if (std::islower(current_char)) {
                current_string_piece.push_back(current_char);

                current_position++;
            }

            else {
                int repeat_times = parse_int(s, current_position);

                current_position++;

                std::string nested_sub_string =
                    do_decode_string(s, current_position);

                current_position++;

                current_string_piece.reserve(
                    current_string_piece.size()
                    + nested_sub_string.size() * repeat_times
                );

                for (int i = 0; i < repeat_times; i++) {
                    current_string_piece.append(nested_sub_string);
                }
            }
        }

        return current_string_piece;
    }

    string decodeString(string s) {
        int current_position = 0;

        return do_decode_string(s, current_position);
    }
};
```

## 399. 除法求值 (Evaluate Division)

标签: 并查集

解法:

- 由于除法具有传递性, 因而具有类似前缀和的性质. 我们可以使用并查集维护各个结点所位于的集合, 并存储结点与其根结点之间的商, 这样我们就能够快速判别出查询是否合法 (等价于所查询的两个元素是否位于同一个集合), 以及在合法的情况下两个元素的商.

代码:

```cpp
class Solution {
private:
    struct Node {
        int parent_node_index;
        double coefficient;
    };

public:
    int get_root_node_index(
        std::vector<Node> &disjoint_sets, const int current_node_index
    ) {
        int root_node_index;
        int parent_node_index =
            disjoint_sets[current_node_index].parent_node_index;

        if (disjoint_sets[parent_node_index].parent_node_index
            == parent_node_index) {
            root_node_index = parent_node_index;
        }

        else {
            root_node_index =
                get_root_node_index(disjoint_sets, parent_node_index);
            disjoint_sets[current_node_index].parent_node_index =
                root_node_index;
            disjoint_sets[current_node_index].coefficient *=
                disjoint_sets[parent_node_index].coefficient;
        }

        return root_node_index;
    }

    void union_two_disjoint_sets(
        std::vector<Node> &disjoint_sets,
        const int parent_node_index,
        const int child_node_index,
        const double coefficient
    ) {
        const int index_of_root_node_of_parent =
            get_root_node_index(disjoint_sets, parent_node_index);
        const int index_of_root_node_of_child =
            get_root_node_index(disjoint_sets, child_node_index);
        const double coefficient_of_parent_root_to_child_root =
            disjoint_sets[parent_node_index].coefficient * coefficient
            / disjoint_sets[child_node_index].coefficient;

        disjoint_sets[index_of_root_node_of_child].parent_node_index =
            index_of_root_node_of_parent;
        disjoint_sets[index_of_root_node_of_child].coefficient =
            coefficient_of_parent_root_to_child_root;
    }

    vector<double> calcEquation(
        vector<vector<string>> &equations,
        vector<double> &values,
        vector<vector<string>> &queries
    ) {
        std::vector<Node> disjoint_sets;
        std::unordered_map<std::string, int> node_name_2_node_index;

        for (int i = 0; i < equations.size(); i++) {
            const std::string &parent_node_name = equations[i][0];
            const std::string &child_node_name = equations[i][1];
            const double coefficient = values[i];

            int parent_node_index;

            if (node_name_2_node_index.count(parent_node_name) == 0) {
                parent_node_index = disjoint_sets.size();
                node_name_2_node_index[parent_node_name] = parent_node_index;
                disjoint_sets.push_back(
                    Node{static_cast<int>(disjoint_sets.size()), 1.0}
                );
            }

            else {
                parent_node_index = node_name_2_node_index[parent_node_name];
            }

            int child_node_index;

            if (node_name_2_node_index.count(child_node_name) == 0) {
                child_node_index = disjoint_sets.size();
                node_name_2_node_index[child_node_name] = child_node_index;
                disjoint_sets.push_back(
                    Node{static_cast<int>(disjoint_sets.size()), 1.0}
                );
            }

            else {
                child_node_index = node_name_2_node_index[child_node_name];
            }

            union_two_disjoint_sets(
                disjoint_sets, parent_node_index, child_node_index, coefficient
            );
        }

        std::vector<double> results;

        for (auto &query : queries) {
            const std::string &parent_node_name = query[0];

            if (node_name_2_node_index.count(parent_node_name) == 0) {
                results.push_back(-1.0);

                continue;
            }

            const std::string &child_node_name = query[1];

            if (node_name_2_node_index.count(child_node_name) == 0) {
                results.push_back(-1.0);

                continue;
            }

            const int parent_node_index =
                node_name_2_node_index[parent_node_name];
            const int child_node_index =
                node_name_2_node_index[child_node_name];

            const int index_of_root_node_of_parent =
                get_root_node_index(disjoint_sets, parent_node_index);
            const int index_of_root_node_of_child =
                get_root_node_index(disjoint_sets, child_node_index);

            if (index_of_root_node_of_parent != index_of_root_node_of_child) {
                results.push_back(-1.0);

                continue;
            }

            const double coefficient =
                disjoint_sets[child_node_index].coefficient
                / disjoint_sets[parent_node_index].coefficient;

            results.push_back(coefficient);
        }

        return results;
    }
};
```

## 406. 根据身高重建队列 (Queue Reconstruction by Height)

标签: 模拟

- 第一种解法是从矮到高进行模拟. 对于当前遍历到的人, 假设身高小于这个人的所有人都已经被安排好位置, 那么我们只需要将当前这个人安排在一个位置使其前面恰好具有足够多的空位能够容纳接下来身高大于等于他的人即可.
- 第二种解法是从高到矮进行模拟. 对于当前遍历到的人, 假设身高大于等于这个人**并且排位也比这个人靠前**的所有人都已经被安排好位置, 那么我们只需要将其插入至一个位置使其前面恰好有足够多身高大于等于他的人即可. 注意到与从矮到高进行模拟不同, 在从高到矮进行模拟的过程中我们无法维护空位, 因此对于排位靠后的人来说我们无法预先为其预留空位来容纳那些身高等于他但排位比他靠前的人, 因此在遍历时我们还需同时按照排位从前到后进行排序.

代码 (从矮到高进行模拟的解法):

```cpp
class Solution {
public:
    vector<vector<int>> reconstructQueue(vector<vector<int>> &people) {
        const auto &short_people_first = [](const std::vector<int> &person_1,
                                            const std::vector<int> &person_2) {
            return person_1[0] < person_2[0];
        };

        std::sort(people.begin(), people.end(), short_people_first);

        std::vector<std::vector<int>> reconstructed_queue(
            people.size(), {-1, 0}
        );

        for (auto &person : people) {
            const int current_person_height = person[0];
            int number_of_slots_needed_for_people_taller_than_current_person =
                person[1];

            for (int i = 0; i < reconstructed_queue.size(); i++) {
                if (reconstructed_queue[i][0] != -1
                    && reconstructed_queue[i][0] != current_person_height) {
                    continue;
                }

                else if (number_of_slots_needed_for_people_taller_than_current_person
                         == 0) {
                    reconstructed_queue[i] = person;

                    break;
                }

                else {
                    number_of_slots_needed_for_people_taller_than_current_person--;

                    continue;
                }
            }
        }

        return reconstructed_queue;
    }
};
```

代码 (从高到矮进行模拟的解法):

(略)

## 416. 分割等和子集 (Partition Equal Subset Sum)

标签: 动态规划

解法:

- 如果要分割集合为两个和相等的子集, 那么和已知, 集合已知, 我们便能够套用 0-1 背包问题的解法来求解. 这是一道典型的动态规划的题目.

代码:

```cpp
class Solution {
public:
    bool do_composition(const std::vector<int> &nums, const int target) {
        std::vector<char> dp(target + 1, false);
        dp[0] = true;

        for (auto current_component : nums) {
            if (current_component > target) {
                return false;
            }

            for (int temporary_target = target;
                 temporary_target >= current_component + 1;
                 temporary_target--) {
                dp[temporary_target] |=
                    dp[temporary_target - current_component];
            }

            dp[current_component] = true;
        }

        return dp[target];
    }

    bool canPartition(vector<int> &nums) {
        auto sum_of_all_numbers = std::reduce(nums.begin(), nums.end(), 0);

        if (sum_of_all_numbers % 2 == 1) {
            return false;
        }

        else {
            return do_composition(nums, sum_of_all_numbers / 2);
        }
    }
};
```

## 437. 路径总和 III (Path Sum III)

标签: 深度优先搜索

解法:

- 由于本题中的路径仅位于同一个树中, 深度优先搜索经过的路径实际上形成了前缀和, 因此我们可以通过哈希表来快速计算出 "以当前根结点为结尾的长度等于目标和的路径" 的数量.

代码:

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
    using BigIntegerType = uint64_t;

public:
    void preorder_traversal(
        TreeNode *root,
        const BigIntegerType target_sum,
        std::unordered_map<BigIntegerType, int> &prefix_sum_2_appeared_times,
        const BigIntegerType previous_prefix_sum,
        int &number_of_valid_paths
    ) {

        const BigIntegerType current_prefix_sum =
            previous_prefix_sum + root->val;

        const BigIntegerType required_prefix_sum =
            current_prefix_sum - target_sum;

        number_of_valid_paths +=
            prefix_sum_2_appeared_times[required_prefix_sum];

        prefix_sum_2_appeared_times[current_prefix_sum]++;

        if (root->left) {
            preorder_traversal(
                root->left,
                target_sum,
                prefix_sum_2_appeared_times,
                current_prefix_sum,
                number_of_valid_paths
            );
        }

        if (root->right) {
            preorder_traversal(
                root->right,
                target_sum,
                prefix_sum_2_appeared_times,
                current_prefix_sum,
                number_of_valid_paths
            );
        }

        prefix_sum_2_appeared_times[current_prefix_sum]--;
    }

    int pathSum(TreeNode *root, int targetSum) {
        if (!root) {
            return 0;
        }

        std::unordered_map<BigIntegerType, int> prefix_sum_2_appeared_times;
        int number_of_valid_paths = 0;
        prefix_sum_2_appeared_times[0] = 1;

        preorder_traversal(
            root,
            targetSum,
            prefix_sum_2_appeared_times,
            0,
            number_of_valid_paths
        );

        return number_of_valid_paths;
    }
};
```

## 438. 找到字符串中所有字母异位词 (Find All Anagrams in a String)

标签: 哈希表, 滑动窗口

解法:

- 直接使用哈希表维护当前滑动窗口中的所有不同字符的个数, 并在遍历过程中维护一个用于表示当前滑动窗口是否合法的状态信息. 每当添加一个字符或删除一个字符的时候便更新该状态, 并在状态变为合法时将当前窗口的起始位置添加进答案集合中, 重复上述过程即可.

代码:

```cpp
class Solution {
public:
    vector<int> findAnagrams(string s, string p) {
        if (s.size() < p.size()) {
            return {};
        }

        int reference_appear_times[26]{};
        int reference_total_appear_times = 0;

        for (auto c : p) {
            int index = c - 'a';

            reference_appear_times[index]++;
            reference_total_appear_times++;
        }

        const int k = p.size();

        int current_appear_times[26]{};
        int current_total_appear_times = 0;

        for (int i = 0; i < k - 1; i++) {
            const int newest_char_index = s[i] - 'a';

            current_appear_times[newest_char_index]++;

            if (current_appear_times[newest_char_index]
                <= reference_appear_times[newest_char_index]) {
                current_total_appear_times++;
            }
        }

        std::vector<int> index_of_valid_sub_strings;

        for (int i = k - 1; i < s.size(); i++) {
            const int newest_char_index = s[i] - 'a';

            current_appear_times[newest_char_index]++;

            if (current_appear_times[newest_char_index]
                <= reference_appear_times[newest_char_index]) {
                current_total_appear_times++;

                if (current_total_appear_times
                    == reference_total_appear_times) {
                    index_of_valid_sub_strings.push_back(i - k + 1);
                }
            }

            const int oldest_char_index = s[i - k + 1] - 'a';

            current_appear_times[oldest_char_index]--;

            if (current_appear_times[oldest_char_index]
                < reference_appear_times[oldest_char_index]) {
                current_total_appear_times--;
            }
        }

        return index_of_valid_sub_strings;
    }
};
```

## 448. 找到所有数组中消失的数字 (Find All Numbers Disappeared in an Array)

标签: 哈希表

解法:

- 由于本题除原始数组以外不允许使用额外空间, 因此我们可以考虑使用**原地哈希**的做法. 我们从左到右遍历数组, 对于当前遍历到的元素, 该元素原本应当位于的位置等于该元素的值, 因此我们可以观察该元素原本应当位于的位置上的元素, 若该元素原本应当位于的位置上的元素并不等于该元素, 说明该元素应当被复位, 于是我们可以将这两个元素进行交换; 否则该元素已经复位, 并且当前元素是多余的, 于是我们直接忽略该元素. 在上述过程结束之后, 整个数组中的元素要么已经复位, 要么由于重复而位于下标不等于自身的位置上, 因此我们可以通过再次遍历数组来获取所有未能复位的元素, 此即最终答案.

代码:

```cpp
class Solution {
public:
    vector<int> findDisappearedNumbers(vector<int> &nums) {
        int current_position = 0;

        while (current_position < nums.size()) {
            while (true) {
                const int number_supposed_to_be = nums[current_position];
                const int number_actually_is = nums[number_supposed_to_be - 1];

                if (number_supposed_to_be == number_actually_is) {
                    break;
                }

                std::swap(
                    nums[current_position], nums[number_supposed_to_be - 1]
                );
            }

            current_position++;
        }

        std::vector<int> numbers_not_appeared;

        for (int current_position = 0; current_position < nums.size();
             current_position++) {

            const int number_supposed_to_be = current_position + 1;
            const int number_actually_is = nums[current_position];

            if (number_supposed_to_be != number_actually_is) {
                numbers_not_appeared.push_back(number_supposed_to_be);
            }
        }

        return numbers_not_appeared;
    }
};
```

## 461. 汉明距离 (Hamming Distance)

标签: 位运算

解法:

- 根据汉明距离的定义, 两个数之间的汉明距离实际上就是两者的异或所得的值中非零比特位的个数. 直接使用各类编程语言内置的获取整数非零比特位个数的函数即可.

代码:

```cpp
class Solution {
public:
    int hammingDistance(int x, int y) {
        return __builtin_popcount(x ^ y);
    }
};
```

## 494. 目标和 (Target Sum)

标签: 动态规划

解法:

- 由于涉及到减法, 我们可以将减法看成是 "不加", 而将加法看成是 "加两倍", 那么原问题就转化为了等价的 0-1 背包问题.
- 由于操作变为了 "加两倍", 这里实际上额外提出了一个剪枝的角度, 即与原问题等价的 0-1 背包问题的目标值必须是个偶数.

代码:

```cpp
class Solution {
public:
    int findTargetSumWays(vector<int> &nums, int target) {
        int sum_of_all_numbers = std::reduce(nums.begin(), nums.end(), 0);

        if (target < -sum_of_all_numbers || sum_of_all_numbers < target
            || (target + sum_of_all_numbers) % 2 == 1) {
            return 0;
        }

        target = (target + sum_of_all_numbers) / 2;

        std::vector<int> dp(target + 1, 0);
        dp[0] = 1;

        for (const int current_component : nums) {
            for (int temporary_target = target;
                 temporary_target >= current_component;
                 temporary_target--) {
                dp[temporary_target] +=
                    dp[temporary_target - current_component];
            }
        }

        return dp[target];
    }
};
```

## 538. 把二叉搜索树转换为累加树 (Convert BST to Greater Tree)

标签: 中序遍历, Morris 遍历

解法:

- 第一种解法是使用中序遍历. 由于二叉搜索树的性质, 值大于当前结点的值的结点均位于当前结点的右子树中, 因此我们完全可以通过 (反向) 中序遍历来维护 "当前结点的右子树中所有结点的和".
- 第二种解法与第一种解法类似, 都是使用中序遍历, 只不过第二种解法采用称为 "Morris 遍历" 的算法以时间换空间来进行二叉树的中序遍历, 下面以**正向**中序遍历为例说明其思想:
  - 如果当前树的左子树为非空, 那么首先找到根结点位于左子树中的前驱结点:
    - 如果根结点的前驱结点的右子树为空, 那么将其右子树**暂时**设为当前树 (即将根结点作为该前驱结点的右孩子), 并递归地进入左子树 (注意此处是中序遍历, 因此首先应当进入左子树).
    - 如果根结点的前驱结点的右子树已经是根结点, 说明我们已经从左子树中递归返回, 于是可以将前驱结点的右孩子重新置空, 然后正常遍历根结点并递归地进入右子树.
  - 如果当前树的左子树为空, 那么正常遍历根结点并直接进入右子树. 由于

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
    void do_convert_bst(TreeNode *root, int &current_sum_of_node_values) {
        if (root->right) {
            do_convert_bst(root->right, current_sum_of_node_values);
        }

        current_sum_of_node_values += root->val;
        root->val = current_sum_of_node_values;

        if (root->left) {
            do_convert_bst(root->left, current_sum_of_node_values);
        }
    }

    TreeNode *convertBST(TreeNode *root) {
        if (root) {
            int current_sum_of_node_values = 0;

            do_convert_bst(root, current_sum_of_node_values);
        }

        return root;
    }
};
```

代码 (Morris 遍历的解法):

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
    TreeNode *convertBST(TreeNode *root) {
        int current_sum_of_node_values = 0;
        const auto root_backup = root;

        while (root) {
            if (root->right) {
                TreeNode *temporary_node = root->right;

                while (true) {
                    if (!temporary_node->left) {
                        temporary_node->left = root;

                        root = root->right;

                        break;
                    }

                    else if (temporary_node->left == root) {
                        temporary_node->left = nullptr;

                        current_sum_of_node_values += root->val;
                        root->val = current_sum_of_node_values;
                        root = root->left;

                        break;
                    }

                    temporary_node = temporary_node->left;
                }
            }

            else {
                current_sum_of_node_values += root->val;
                root->val = current_sum_of_node_values;
                root = root->left;
            }
        }

        return root_backup;
    }
};
```

## 543. 二叉树的直径 (Diameter of Binary Tree)

标签: 递归

解法:

- 根据定义, 二叉树的直径等于其左子树的直径, 其右子树的直径, 以及经过根结点的最长路径的长度这三者之间的最大值, 于是我们自然而然可以使用递归来求解这三者并得到当前树的直径.

代码:

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
    void
    do_diameter_of_binary_tree(TreeNode *root, int &max_height, int &diameter) {
        int max_height_of_left_sub_tree = 0;

        if (root->left) {
            do_diameter_of_binary_tree(
                root->left, max_height_of_left_sub_tree, diameter
            );
        }

        int max_height_of_right_sub_tree = 0;

        if (root->right) {
            do_diameter_of_binary_tree(
                root->right, max_height_of_right_sub_tree, diameter
            );
        }

        max_height =
            std::max(max_height_of_left_sub_tree, max_height_of_right_sub_tree)
            + 1;

        const int diameter_when_passing_root =
            max_height_of_left_sub_tree + max_height_of_right_sub_tree;
        diameter = std::max(diameter, diameter_when_passing_root);
    }

    int diameterOfBinaryTree(TreeNode *root) {
        if (!root) {
            return 0;
        }

        int max_height;
        int diameter = 0;

        do_diameter_of_binary_tree(root, max_height, diameter);

        return diameter;
    }
};
```

## 560. 和为 K 的子数组 (Subarray Sum Equals K)

标签: 前缀和, 哈希表

解法:

- 由于题目求的是**子数组的和** (类似于字符串中的子串, 为连续的区间而非离散的序列), 因此我们自然而然可以想到使用前缀和来进行求解. 首先我们从左到右对数组进行遍历, 遍历过程中我们使用一个哈希表来存储从数组的最左端开始到各个位置为止的子串的元素之和 (即前缀和). 对于当前位置, 如果存在和为 `k` (即目标值) 的子串以当前元素结尾, 那么我们可以断定哈希表中存在有前缀和等于当前子串前缀和减去 `k` 的子串, 因此我们只需对哈希表进行查询即可.

代码:

```cpp
class Solution {
public:
    int subarraySum(vector<int> &nums, int k) {
        std::unordered_map<int, int> prefix_sum_2_number_of_sub_arrays;

        prefix_sum_2_number_of_sub_arrays[0] = 1;

        int current_prefix_sum = 0;

        int number_of_valid_sub_arrays = 0;

        for (int current_position = 0; current_position < nums.size();
             current_position++) {
            current_prefix_sum += nums[current_position];

            auto it =
                prefix_sum_2_number_of_sub_arrays.find(current_prefix_sum - k);

            number_of_valid_sub_arrays +=
                (it == prefix_sum_2_number_of_sub_arrays.end() ? 0 : it->second
                );

            prefix_sum_2_number_of_sub_arrays[current_prefix_sum]++;
        }

        return number_of_valid_sub_arrays;
    }
};
```

## 581. 最短无序连续子数组 (Shortest Unsorted Continuous Subarray)

标签: 贪心

解法:

- 根据定义, 如果我们对数组进行排序, 那么最短无序连续子数组的左右两个端点必然会被移动 (因为如果它们不移动, 说明它们已经有序, 那么删去它们将能够形成更短的最短无序连续子数组, 形成矛盾), 这说明 "在最短无序连续子数组中必然存在小于左端点的数以及大于右端点的数", 而对于那些位于最短无序连续子数组左边的元素而言, 任何位于它们右边的元素均大于等于它们, 同时对于那些位于最短无序连续子数组右边的元素而言, 任何位于它们左边的元素均小于等于它们. 因此我们可以通过从右向左遍历找到尽可能位于左端的不满足 "任何位于它们右边的元素均大于等于它们" 的元素作为最短无序连续子数组的左端点, 然后同理找到右端点, 最终确定最短无序连续子数组的范围.

代码:

```cpp
class Solution {
public:
    int findUnsortedSubarray(vector<int> &nums) {
        const int nums_size = nums.size();

        int left_most_position = std::numeric_limits<int>::max();

        int current_smallest_number = std::numeric_limits<int>::max();

        for (int current_position = nums_size - 1; current_position >= 0;
             current_position--) {
            const int current_number = nums[current_position];

            if (current_number > current_smallest_number) {
                left_most_position = current_position;
            }

            else {
                current_smallest_number =
                    std::min(current_smallest_number, current_number);
            }
        }

        if (left_most_position == std::numeric_limits<int>::max()) {
            return 0;
        }

        int right_most_position = std::numeric_limits<int>::min();

        int current_biggest_number = std::numeric_limits<int>::min();

        for (int current_position = 0; current_position < nums_size;
             current_position++) {
            const int current_number = nums[current_position];

            if (current_number < current_biggest_number) {
                right_most_position = current_position;
            }

            else {
                current_biggest_number =
                    std::max(current_biggest_number, current_number);
            }
        }

        return (right_most_position + 1) - left_most_position;
    }
};
```

## 617. 合并二叉树 (Merge Two Binary Trees)

标签: 递归, 迭代

解法:

- 第一种解法是使用递归. 对于当前遍历到的两个根结点:
  - 如果两个根结点均为空, 则直接返回空指针.
  - 如若不然, 说明两个根结点中存在非空结点. 如果其中一个根结点为空, 那么返回另一个根结点作为已经合并的树.
  - 如若不然, 说明两个结点均为非空结点. 于是我们递归地对两棵树的左子树和右子树进行合并.
- 第二种解法是使用迭代来模拟递归.

代码 (递归的解法):

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
    TreeNode *mergeTrees(TreeNode *root1, TreeNode *root2) {
        if (!root1 && !root2) {
            return nullptr;
        }

        else if (!root1) {
            return root2;
        }

        else if (!root2) {
            return root1;
        }

        root1->val += root2->val;

        root1->left = mergeTrees(root1->left, root2->left);
        root1->right = mergeTrees(root1->right, root2->right);

        delete root2;

        return root1;
    }
};
```

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
    TreeNode *mergeTrees(TreeNode *root1, TreeNode *root2) {
        if (!root1 && !root2) {
            return nullptr;
        }

        else if (!root1) {
            return root2;
        }

        else if (!root2) {
            return root1;
        }

        std::queue<TreeNode *> node_queue_1;
        std::queue<TreeNode *> node_queue_2;

        node_queue_1.push(root1);
        node_queue_2.push(root2);

        while (!node_queue_1.empty()) {
            auto current_node_1 = node_queue_1.front();
            node_queue_1.pop();

            auto current_node_2 = node_queue_2.front();
            node_queue_2.pop();

            current_node_1->val += current_node_2->val;

            if (current_node_1->left || current_node_2->left) {
                if (!current_node_1->left) {
                    current_node_1->left = current_node_2->left;
                }

                else if (current_node_2->left) {
                    node_queue_1.push(current_node_1->left);
                    node_queue_2.push(current_node_2->left);
                }
            }

            if (current_node_1->right || current_node_2->right) {
                if (!current_node_1->right) {
                    current_node_1->right = current_node_2->right;
                }

                else if (current_node_2->right) {
                    node_queue_1.push(current_node_1->right);
                    node_queue_2.push(current_node_2->right);
                }
            }
        }

        return root1;
    }
};
```

## 621. 任务调度器 (Task Scheduler)

标签: 模拟

解法:

- 由于两个相同的任务之间必须间隔 `n` 个距离, 考虑向一个列数为 `n + 1` 的矩阵中填入所有任务. 我们从出现次数最多的任务 `A` 开始, 首先将 `A` 填入矩阵的第一列, 此时矩阵的最大行数由任务 `A` 的出现次数决定. 由于矩阵的列数为 `n + 1`, 第一列中上下任意两个相邻的任务之间必然间隔 `n`, 因此由第一列构成的任务序列是合法的. 紧接着我们考虑将出现次数第二多的任务 `B` 以 "从左到右, 从下到上" 的顺序填入矩阵, 如果任务 `B` 的出现次数等于任务 `A` 的出现次数, 那么两者将分别填充第一列和第二列; 否则我们从**第二列的倒数第二行**开始向上填充, 并且有可能会由于任务 `B` 无法填充第二列而在第二列的头几个位置留下几个空槽. 接着将出现次数第三多的任务 `C` 填入矩阵, 任务 `C` 将首先从第二列中任务 `B` 未能填充的部分开始, 并在填充好第二列之后开始自下而上填充第三列, 由于任务 `C` 的出现次数仍然小于等于任务 `B` 的出现次数, 因此和任务 `B` 一样, 任务 `C` 要么单独占据一列从而满足合法序列, 要么由于无法占据满一列而同样满足合法序列, 以此类推. 如果将所有任务填入矩阵后仍然无法填满矩阵, 那么由于所有长度等于任务 `A` 的任务 (包括任务 `A` 在内) 存在, 整个序列无法提前退出, 而必须等待所有这些任务完成才能退出, 因此最短合法序列的长度等于将矩阵填满后得到的序列的长度; 反之如果将所有任务填入矩阵后能够填满矩阵或是发生溢出, 那么最短合法序列的长度就是所有任务的总数.

代码:

```cpp
class Solution {
public:
    int leastInterval(vector<char> &tasks, int n) {
        std::unordered_map<char, int> task_2_repetition_number;

        for (const char task : tasks) {
            task_2_repetition_number[task]++;
        }

        using ElementType = decltype(task_2_repetition_number)::value_type;

        int max_repetition_number =
            std::max_element(
                task_2_repetition_number.begin(),
                task_2_repetition_number.end(),
                [](const ElementType &pair_1, const ElementType &pair_2) {
                    return pair_1.second < pair_2.second;
                }
            )->second;

        int number_tasks_with_max_repetition_number = std::accumulate(
            task_2_repetition_number.begin(),
            task_2_repetition_number.end(),
            0,
            [=](const int acc, const ElementType &pair) {
                return acc + (pair.second == max_repetition_number);
            }
        );

        return std::max(
            (max_repetition_number - 1) * (n + 1)
                + number_tasks_with_max_repetition_number,
            static_cast<int>(tasks.size())
        );
    }
};
```

## 647. 回文子串 (Palindromic Substrings)

标签: 动态规划

解法:

- 本题解法与 "5. 最长回文子串 (Longest Palindromic Substring)" 大同小异, 后者无非是在更新最大回文子串大小, 我们只需要对其稍加改动, 在确认当前子串为回文子串时将回文子串总数加一 (而非更新最长回文子串长度) 即可.

代码:

```cpp
class Solution {
public:
    int countSubstrings(string s) {
        int total_number_of_valid_sub_strings = s.size();

        for (int current_position = 0; current_position <= s.size();
             current_position++) {
            int sub_string_begin = current_position;
            int sub_string_end = current_position;

            while (true) {
                sub_string_begin--;
                sub_string_end++;

                if (sub_string_begin < 0 || sub_string_end > s.size()
                    || s[sub_string_begin] != s[sub_string_end - 1]) {
                    break;
                }

                else {
                    total_number_of_valid_sub_strings++;
                }
            }
        }

        for (int current_position = 0; current_position < s.size();
             current_position++) {
            int sub_string_begin = current_position;
            int sub_string_end = current_position + 1;

            while (true) {
                sub_string_begin--;
                sub_string_end++;

                if (sub_string_begin < 0 || sub_string_end > s.size()
                    || s[sub_string_begin] != s[sub_string_end - 1]) {
                    break;
                }

                else {
                    total_number_of_valid_sub_strings++;
                }
            }
        }

        return total_number_of_valid_sub_strings;
    }
};
```

## 739. 每日温度 (Daily Temperatures)

标签: 单调栈

解法:

- 根据题意, 从当前位置开始到下一个更高温度的位置之间的这段范围内出现的温度均小于等于当前位置的温度, 因此所有这些温度都无法更新当前位置的温度, 而当下一个更高温度出现之后, 当前位置所对应的信息就确定不变了. 我们可以使用单调栈来模拟这一现象. 对于当前遍历到的温度, 我们不断弹出单调栈中温度小于当前温度的日期, 将当前温度作为所弹出的温度的下一个更高温度, 而使用 "弹出" 这一操作来表示该位置所对应的信息已经确定不变了.

代码:

```cpp
class Solution {
public:
    vector<int> dailyTemperatures(vector<int> &temperatures) {
        std::vector<int> intervals_till_next_day_with_higher_temperature(
            temperatures.size(), 0
        );

        std::vector<int> decreasing_temperature_stack;

        for (int current_position = 0; current_position < temperatures.size();
             current_position++) {
            const int current_temperature = temperatures[current_position];

            while (!decreasing_temperature_stack.empty()
                   && temperatures[decreasing_temperature_stack.back()]
                          < current_temperature) {
                const int previous_position =
                    decreasing_temperature_stack.back();
                decreasing_temperature_stack.pop_back();

                intervals_till_next_day_with_higher_temperature
                    [previous_position] = current_position - previous_position;
            }

            decreasing_temperature_stack.push_back(current_position);
        }

        return intervals_till_next_day_with_higher_temperature;
    }
};
```
