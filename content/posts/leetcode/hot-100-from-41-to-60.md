---
title: "力扣 Hot 100 题解归档 (41-60)"
hideSummary: true
date: 2025-01-27T00:24:52+08:00
draft: false
tags: ["leetcode"]
series: ["leetcode"]
author: ["xubinh"]
type: posts
math: true
---

## 102. 二叉树的层序遍历 (Binary Tree Level Order Traversal)

标签: 广度优先搜索

解法:

- 直接套用广度优先搜索的样板代码即可.

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
    vector<vector<int>> levelOrder(TreeNode *root) {
        if (!root) {
            return {};
        }

        std::queue<TreeNode *> current_layer_node_queue;
        std::queue<TreeNode *> next_layer_node_queue;
        std::vector<std::vector<int>> layer_order_sequences;
        std::vector<int> current_layer_order_sequence;

        next_layer_node_queue.push(root);

        while (!next_layer_node_queue.empty()) {
            swap(current_layer_node_queue, next_layer_node_queue);

            current_layer_order_sequence.clear();

            while (!current_layer_node_queue.empty()) {
                const auto current_node = current_layer_node_queue.front();
                current_layer_node_queue.pop();

                current_layer_order_sequence.push_back(current_node->val);

                if (current_node->left) {
                    next_layer_node_queue.push(current_node->left);
                }

                if (current_node->right) {
                    next_layer_node_queue.push(current_node->right);
                }
            }

            layer_order_sequences.push_back(current_layer_order_sequence);
        }

        return layer_order_sequences;
    }
};
```

## 104. 二叉树的最大深度 (Maximum Depth of Binary Tree)

标签: 深度优先搜索, 广度优先搜索

解法:

- 第一种解法是使用深度优先搜索. 因为一棵二叉树的最大深度等于其所有子树的最大深度之间的较大者加一, 因此可以使用递归来进行求解.
- 第二种解法是使用广度优先搜索. 每当进入下一层便将深度加一, 返回最终的深度即可.

代码 (深度优先搜索的解法):

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    int maxDepth(TreeNode* root) {
        return root ? std::max(maxDepth(root->left), maxDepth(root->right)) + 1 : 0;
    }
};
```

代码 (广度优先搜索的解法):

(略)

## 105. 从前序与中序遍历序列构造二叉树 (Construct Binary Tree from Preorder and Inorder Traversal)

标签: 哈希表, 分治

解法:

- 前序序列的第一个元素必然是当前二叉树的树根的值. 由于序列中不存在重复的元素, 因此我们可以使用一个哈希表来存储从元素的值到元素在中序序列中的下标的映射, 同时使用该映射以及已知的根结点的值来确定左子树和右子树的大小并进一步确定它们在前序序列中的范围.

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
    TreeNode *do_build_tree(
        const std::vector<int> &preorder,
        const int preorder_range_begin,
        const int preorder_range_end,
        const std::unordered_map<int, int> &value_2_inorder_position,
        const int inorder_range_begin,
        const int inorder_range_end
    ) {
        if (preorder_range_begin == preorder_range_end) {
            return nullptr;
        }

        const int root_value = preorder[preorder_range_begin];
        const int inorder_cut_position =
            value_2_inorder_position.at(root_value);

        auto root = new TreeNode(root_value);

        const int inorder_range_begin_for_left_sub_tree = inorder_range_begin;
        const int inorder_range_end_for_left_sub_tree = inorder_cut_position;

        const int left_sub_tree_size =
            inorder_cut_position - inorder_range_begin;
        const int preorder_range_begin_for_left_sub_tree =
            preorder_range_begin + 1;
        const int preorder_range_end_for_left_sub_tree =
            preorder_range_begin_for_left_sub_tree + left_sub_tree_size;

        root->left = do_build_tree(
            preorder,
            preorder_range_begin_for_left_sub_tree,
            preorder_range_end_for_left_sub_tree,
            value_2_inorder_position,
            inorder_range_begin_for_left_sub_tree,
            inorder_range_end_for_left_sub_tree
        );

        const int inorder_range_begin_for_right_sub_tree =
            inorder_cut_position + 1;
        const int inorder_range_end_for_right_sub_tree = inorder_range_end;

        const int right_sub_tree_size =
            inorder_range_end - (inorder_cut_position + 1);
        const int preorder_range_begin_for_right_sub_tree =
            preorder_range_end_for_left_sub_tree;
        const int preorder_range_end_for_right_sub_tree =
            preorder_range_begin_for_right_sub_tree + right_sub_tree_size;

        root->right = do_build_tree(
            preorder,
            preorder_range_begin_for_right_sub_tree,
            preorder_range_end_for_right_sub_tree,
            value_2_inorder_position,
            inorder_range_begin_for_right_sub_tree,
            inorder_range_end_for_right_sub_tree
        );

        return root;
    }

    TreeNode *buildTree(vector<int> &preorder, vector<int> &inorder) {
        std::unordered_map<int, int> value_2_inorder_position;

        for (int current_position = 0; current_position < inorder.size();
             current_position++) {
            value_2_inorder_position[inorder[current_position]] =
                current_position;
        }

        return do_build_tree(
            preorder,
            0,
            preorder.size(),
            value_2_inorder_position,
            0,
            inorder.size()
        );
    }
};
```

## 114. 二叉树展开为链表 (Flatten Binary Tree to Linked List)

标签: 前序遍历, 递归, 迭代

解法:

- 第一种方法是使用前序遍历, 即先通过前序遍历获取到前序序列, 然后再一遍构造出链表.
- 第二种方法是使用递归. 要将一棵二叉树展开为链表, 只需要分别将其左右子树展开为链表并将根结点, 左子树链表以及右子树链表这三者串联起来即可. 注意到这种方法需要获取到左子树的最后一个结点以便串联, 同时也需要获取到右子树的最后一个结点用于作为整棵树的最后一个结点并返回, 因此递归函数将需要额外的两个参数.
- 第三种方法是使用迭代模拟前序遍历. 我们使用一个栈来存储 "所有待处理的右子树的根结点". 对于当前遍历到的结点:
  - 如果该结点具有左子树, 那么下一个结点就是左子树的根结点.
  - 如若不然, 如果该结点具有右子树, 那么下一个结点就是右子树的根结点.
  - 如果不然, 如果 "所有待处理的右子树的根结点" 的栈非空, 那么下一个结点就是栈顶结点.
  - 如果不然, 说明整棵树已经遍历完毕, 直接返回即可.

代码 (前序遍历的解法):

(略)

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
    void flatten(TreeNode *root) {
        if (!root) {
            return;
        }

        std::vector<TreeNode *> pending_right_sub_tree_roots;

        TreeNode *current_node = root;

        while (true) {
            TreeNode *next_node;

            if (current_node->left) {
                next_node = current_node->left;

                if (current_node->right) {
                    pending_right_sub_tree_roots.push_back(current_node->right);
                }
            }

            else if (current_node->right) {
                next_node = current_node->right;
            }

            else if (pending_right_sub_tree_roots.empty()) {
                return;
            }

            else {
                next_node = pending_right_sub_tree_roots.back();
                pending_right_sub_tree_roots.pop_back();
            }

            current_node->right = next_node;
            current_node->left = nullptr;
            current_node = next_node;
        }

        throw std::runtime_error("never reaches here");
    }
};
```

## 121. 买卖股票的最佳时机 (Best Time to Buy and Sell Stock)

标签: 动态规划

解法:

- 遍历数组中的每个元素. 对于当前位置, 考虑 "在当前位置卖出能够获取的最大利润". 显然该值等于 "当前位置的价格" 减去 "在该位置之前的价格的最低值" (若为负数则忽略), 因此可以在遍历过程中维护 "在该位置之前的价格的最低值", 并不断更新最终答案.

代码:

```cpp
class Solution {
public:
    int maxProfit(vector<int> &prices) {
        int max_profit = 0;

        int lowest_price_before_current_position = prices[0];

        for (int current_position = 1; current_position < prices.size();
             current_position++) {
            const int current_price = prices[current_position];

            max_profit = std::max(
                max_profit, current_price - lowest_price_before_current_position
            );
            lowest_price_before_current_position =
                min(lowest_price_before_current_position, current_price);
        }

        return max_profit;
    }
};
```

## 124. 二叉树中的最大路径和 (Binary Tree Maximum Path Sum)

标签: 递归

解法:

- 本题中的 "路径" 可以是形如 "从左子树的结点到根结点, 再到右子树的结点" 这样的路径, 因此需要分情况讨论:
  - 对于不经过根结点的那些路径, 它们其实就是左子树中的所有路径加上右子树中的所有路径.
  - 对于经过根结点的那些路径, 它们相当于左子树中那些以左子树的根结点为端点的路径, 加上根结点, 加上右子树中那些以右子树的根结点为端点的路径这三者的串联.

  综上所述, 递归函数应该包含两个返回参数, 分别是 "形状任意的路径的最大路径和" 以及 "以根结点为端点的路径的最大路径和".

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
    void do_max_path_sum(
        TreeNode *root,
        int &max_path_sum_start_from_root,
        int &max_path_sum_arbitrary
    ) {
        if (!root) {
            max_path_sum_start_from_root = std::numeric_limits<int>::min();
            max_path_sum_arbitrary = std::numeric_limits<int>::min();

            return;
        }

        int max_path_sum_start_from_root_of_left_sub_tree;
        int max_path_sum_arbitrary_of_left_sub_tree;

        do_max_path_sum(
            root->left,
            max_path_sum_start_from_root_of_left_sub_tree,
            max_path_sum_arbitrary_of_left_sub_tree
        );

        int max_path_sum_start_from_root_of_right_sub_tree;
        int max_path_sum_arbitrary_of_right_sub_tree;

        do_max_path_sum(
            root->right,
            max_path_sum_start_from_root_of_right_sub_tree,
            max_path_sum_arbitrary_of_right_sub_tree
        );

        max_path_sum_start_from_root =
            root->val
            + std::max(
                0,
                std::max(
                    max_path_sum_start_from_root_of_left_sub_tree,
                    max_path_sum_start_from_root_of_right_sub_tree
                )
            );

        const int max_path_sum_going_through_root =
            root->val
            + std::max(0, max_path_sum_start_from_root_of_left_sub_tree)
            + std::max(0, max_path_sum_start_from_root_of_right_sub_tree);

        max_path_sum_arbitrary = std::max(
            max_path_sum_going_through_root,
            std::max(
                max_path_sum_arbitrary_of_left_sub_tree,
                max_path_sum_arbitrary_of_right_sub_tree
            )
        );
    }

    int maxPathSum(TreeNode *root) {
        int max_path_sum_start_from_root;
        int max_path_sum_arbitrary;

        do_max_path_sum(
            root, max_path_sum_start_from_root, max_path_sum_arbitrary
        );

        return max_path_sum_arbitrary;
    }
};
```

## 128. 最长连续序列 (Longest Consecutive Sequence)

标签: 哈希表

解法:

- 想象原始数组中的所有元素在数轴上构成一个个集合, 每个集合由连续的数字构成. 对于任意一个集合, 我们从该集合中的任意一个数字开始, 不断向两边扩展, 一边扩展一边删除所有遍历到的数字, 直至将整个集合都删除, 此时我们便得到了该集合的长度, 并且同时还准备好进入下一个循环. 不断重复这一过程并更新最终答案即可.

代码:

```cpp
class Solution {
public:
    int longestConsecutive(vector<int> &nums) {
        std::unordered_set<int> unique_numbers;

        unique_numbers.reserve(nums.size());

        unique_numbers.insert(nums.begin(), nums.end());

        int max_interval_length = 0;

        while (!unique_numbers.empty()) {
            const int current_number = *unique_numbers.begin();
            unique_numbers.erase(unique_numbers.begin());

            int current_max_interval_length = 1;

            int left_number = current_number - 1;

            while (true) {
                auto it = unique_numbers.find(left_number);

                if (it == unique_numbers.end()) {
                    break;
                }

                current_max_interval_length++;

                unique_numbers.erase(it);

                left_number--;
            }

            int right_number = current_number + 1;

            while (true) {
                auto it = unique_numbers.find(right_number);

                if (it == unique_numbers.end()) {
                    break;
                }

                current_max_interval_length++;

                unique_numbers.erase(it);

                right_number++;
            }

            max_interval_length =
                std::max(max_interval_length, current_max_interval_length);
        }

        return max_interval_length;
    }
};
```

## 136. 只出现一次的数字 (Single Number)

标签: 位运算

解法:

- 经典老题, 直接使用异或运算将出现两次的数字两两抵消, 剩下的便是只出现一次的数字.

代码:

```cpp
class Solution {
public:
    int singleNumber(vector<int> &nums) {
        return std::accumulate(
            nums.begin(), nums.end(), 0, std::bit_xor<int>()
        );
    }
};
```

## 139. 单词拆分 (Word Break)

标签: 动态规划, 哈希表

解法:

- 首先应当转变过来的思想是本题不应该从 "一个单词是否出现在字符串中" 入手, 而应该从 "字符串的当前子串是否出现在字典中" 入手, 因为后者能够使用哈希表进行优化, 同时还能够使得原问题转化为等价的子问题, 并能进一步使用动态规划进行求解.
- 我们首先使用一个哈希表存储字典, 然后从头开始对原始字符串进行遍历. 对于从原始字符串开头到当前位置为止所形成的子串, 我们将其分割为左右两个部分, 其中右半部分的长度从 1 开始逐渐增大. 对于每个不同的右半部分, 我们使用哈希表检查该右半部分是否恰好为字典中的某个单词, 如果是, 那么当前子串能够拆分当且仅当当前字串的左半部分能够拆分, 因此我们只需要对所有不同的右半部分进行查询即可获知当前子串是否能够拆分.

代码:

```cpp
class Solution {
public:
    bool wordBreak(string s, vector<string> &wordDict) {
        std::unordered_set<std::string> hash_table_of_words(
            wordDict.begin(), wordDict.end()
        );

        const int max_word_size = std::accumulate(
            wordDict.begin(),
            wordDict.end(),
            0,
            [](const size_t acc, const std::string &word) {
                return std::max(acc, word.size());
            }
        );

        std::vector<char> dp(s.size() + 1, false);

        dp[0] = true;

        for (int current_position = 1; current_position <= s.size();
             current_position++) {
            const int temporary_sub_string_end = current_position;

            int current_max_word_size =
                std::min(max_word_size, current_position);

            for (int temporary_sub_string_size = 1;
                 temporary_sub_string_size <= current_max_word_size;
                 temporary_sub_string_size++) {
                const int temporary_sub_string_begin =
                    temporary_sub_string_end - temporary_sub_string_size;
                const auto &temporary_sub_string = s.substr(
                    temporary_sub_string_begin, temporary_sub_string_size
                );

                if (hash_table_of_words.count(temporary_sub_string)
                    && dp[temporary_sub_string_begin]) {
                    dp[current_position] = true;

                    break;
                }
            }
        }

        return dp[s.size()];
    }
};
```

## 141. 环形链表 (Linked List Cycle)

标签: 快慢指针

解法:

- 经典环形链表, 经典快慢指针的解法. 首先确保链表不为空. 一开始设置两个快慢指针指向头结点, 然后开始循环, 在每一轮循环中快指针向前走两步, 慢指针向前走一步, 如果快指针无法向前走两步说明遇到链表尾, 于是进一步说明链表无环, 否则链表有环, 并且快慢指针最终将会相遇, 于是我们只需在快慢指针相遇时返回即可.

代码:

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    bool hasCycle(ListNode *head) {
        if (!head) {
            return false;
        }

        auto slow_ptr = head;
        auto fast_ptr = head;

        while (fast_ptr->next && fast_ptr->next->next) {
            fast_ptr = fast_ptr->next->next;
            slow_ptr = slow_ptr->next;

            if (fast_ptr == slow_ptr) {
                return true;
            }
        }

        return false;
    }
};
```

## 142. 环形链表 II (Linked List Cycle II)

标签: 脑筋急转弯, 快慢指针

解法:

- 本题比 "141. 环形链表 (Linked List Cycle)" 要难上一个数量级, 原因在于不仅需要确定是否有环, 还需要求出环的入口.
- 要求解本题首先需要具备一些前置知识. 规定链表中两个结点之间的距离为由这两个结点所构成的路径的边数. 现在假设链表有环, 并且快慢指针已经在环中相遇. 考虑将时间倒回至慢指针刚刚到达入环口的时刻, 此时慢指针所走过的距离为头结点与入环口结点之间的距离, 记为 `a`, 而快指针所走的距离为该距离的两倍, 记环的长度为 `c`, 那么此时快指针应当位于距离入环口 `b = a % c` 的位置, 并且只需要再走 `d = c - b` 步便能再次到达入环口, 而由于慢指针已经位于入环口, 可以想象当慢指针从入环口开始走 `d` 步之后, 快指针将走 `2d` 步, 其中 `d` 步用于到达入环口, 另外的 `d` 步用于追上慢指针, 两者相遇. 由前述推导我们可以得知两件事情:
  - `(a + d) % c = 0`.
  - 快慢指针相遇时, 两者恰好位于从入环口出发走 `d` 步的位置, 并且只需要再走 `b = a % c` 步即可再次到达入环口.

  那么问题就转化为了如何能够使快指针或慢指针再走 `b = a % c` 步的问题. 而这恰好可以通过令快指针重新从链表头结点开始与慢指针一同出发 (此时快指针不再一次走两步, 而是和慢指针一样一次走一步), 并判断两者是否相遇来做到, 因为如果快指针走了 `a` 步, 那么慢指针便恰好走 `b = a % c` 步 (注意到慢指针位于环中), 于是可知此时两者均位于入环口.

代码:

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode *detectCycle(ListNode *head) {
        if (!head) {
            return nullptr;
        }

        auto slow_ptr = head;
        auto fast_ptr = head;

        int total_length = 0;

        while (fast_ptr->next && fast_ptr->next->next) {
            slow_ptr = slow_ptr->next;
            fast_ptr = fast_ptr->next->next;

            total_length++;

            if (slow_ptr == fast_ptr) {
                break;
            }
        }

        if (total_length == 0 || slow_ptr != fast_ptr) {
            return nullptr;
        }

        while (head != slow_ptr) {
            head = head->next;
            slow_ptr = slow_ptr->next;
        }

        return head;
    }
};
```

## 146. LRU 缓存 (LRU Cache)

标签: 模拟, 双向链表, 哈希表

解法:

- 由于需要以 $O(1)$ 的时间进行插入和删除, 并且需要按最近使用的顺序对键值对进行排序, 这就决定了必须使用双向链表来存储键值对. 此外为了支持 $O(1)$ 的查找, 还需要使用一个哈希表来存储从键到链表中对应结点的地址的映射. 剩下的便是如何实现一个双向链表以及如何维护最大缓存容量的问题了.

代码:

```cpp
class LRUCache {
private:
    struct Node {
        Node()
            : previous(nullptr)
            , next(nullptr)
            , key(0)
            , value(0) {
        }

        Node(const int key, const int value)
            : previous(nullptr)
            , next(nullptr)
            , key(key)
            , value(value) {
        }

        int key;
        int value;
        Node *previous;
        Node *next;
    };

public:
    LRUCache(int capacity)
        : _capacity(capacity) {
        _dummy_head.next = &_dummy_head;
        _dummy_head.previous = &_dummy_head;
    }

    ~LRUCache() {
        const auto list_end = &_dummy_head;

        while (_dummy_head.next != list_end) {
            const auto node_to_be_deleted = _dummy_head.next;
            _dummy_head.next = node_to_be_deleted->next;
            delete node_to_be_deleted;
        }
    }

    int get(int key) {
        auto it = _key_2_node_ptr.find(key);

        if (it == _key_2_node_ptr.end()) {
            return -1;
        }

        const auto node_ptr = it->second;

        _cut_off_from_list(node_ptr);

        _insert_to_list_front(node_ptr);

        return node_ptr->value;
    }

    void put(int key, int value) {
        auto it = _key_2_node_ptr.find(key);

        Node *node_ptr;

        if (it == _key_2_node_ptr.end()) {
            node_ptr = new Node(key, value);
            _key_2_node_ptr[key] = node_ptr;

            if (_key_2_node_ptr.size() > _capacity) {
                const auto tail_node_ptr = _dummy_head.previous;

                _cut_off_from_list(tail_node_ptr);
                _key_2_node_ptr.erase(tail_node_ptr->key);
                delete tail_node_ptr;
            }
        }

        else {
            node_ptr = it->second;
            node_ptr->value = value;
            _cut_off_from_list(node_ptr);
        }

        _insert_to_list_front(node_ptr);
    }

private:
    void _cut_off_from_list(Node *node_ptr) {
        node_ptr->previous->next = node_ptr->next;
        node_ptr->next->previous = node_ptr->previous;
    }

    void _insert_to_list_front(Node *node_ptr) {
        node_ptr->next = _dummy_head.next;
        _dummy_head.next->previous = node_ptr;
        _dummy_head.next = node_ptr;
        node_ptr->previous = &_dummy_head;
    }

    std::unordered_map<int, Node *> _key_2_node_ptr;
    Node _dummy_head;
    const int _capacity;
};

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache* obj = new LRUCache(capacity);
 * int param_1 = obj->get(key);
 * obj->put(key,value);
 */
```

## 148. 排序链表 (Sort List)

标签: 快慢指针, 分治, 归并排序

解法:

- 要想仅使用 $O(1)$ 的额外空间对链表进行排序, 就只能使用分治 + 归并排序的方法. 对于一条链表, 首先使用快慢指针将其切割为等长的两部分, 然后递归地对这两条子链表进行排序, 最后使用归并排序将这两条子链表合并为完整的一条链表.

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
    ListNode *cut_list_into_half(ListNode *head) {
        if (!head) {
            return nullptr;
        }

        auto slow_ptr = head;
        auto fast_ptr = head;

        while (fast_ptr->next && fast_ptr->next->next) {
            fast_ptr = fast_ptr->next->next;
            slow_ptr = slow_ptr->next;
        }

        auto right_half_list_head = slow_ptr->next;
        slow_ptr->next = nullptr;

        return right_half_list_head;
    }

    ListNode *merge_two_lists(ListNode *head_1, ListNode *head_2) {
        ListNode dummy_head;
        ListNode *merged_list_tail = &dummy_head;

        while (head_1 && head_2) {
            const bool is_head_1_smaller_than_head_2 =
                head_1->val < head_2->val;

            const auto next_node =
                is_head_1_smaller_than_head_2 ? head_1 : head_2;
            head_1 = is_head_1_smaller_than_head_2 ? head_1->next : head_1;
            head_2 = is_head_1_smaller_than_head_2 ? head_2 : head_2->next;

            merged_list_tail->next = next_node;
            merged_list_tail = next_node;
        }

        if (!head_1) {
            merged_list_tail->next = head_2;
        }

        else {
            merged_list_tail->next = head_1;
        }

        return dummy_head.next;
    }

    ListNode *do_sort_list(ListNode *head) {
        if (!head || !head->next) {
            return head;
        }

        auto left_half_list_head = head;
        auto right_half_list_head = cut_list_into_half(head);

        left_half_list_head = do_sort_list(left_half_list_head);
        right_half_list_head = do_sort_list(right_half_list_head);

        auto merged_list_head =
            merge_two_lists(left_half_list_head, right_half_list_head);

        return merged_list_head;
    }

    ListNode *sortList(ListNode *head) {
        return do_sort_list(head);
    }
};
```

## 152. 乘积最大子数组 (Maximum Product Subarray)

标签: 动态规划

解法:

- 因为是求最大乘积, 并且数组中的元素的符号不定 (有正有负), 当前最大的正数乘积乘上当前元素 (假设为负数) 之后就可能变成一个负数了, 因此我们不能只盯着一个乘积的当前值来看, 而应该着眼于乘积的**绝对值**. 考虑同时维护 "以当前元素结尾的非空连续子数组的乘积的最大值" 和 "以当前元素结尾的非空连续子数组的乘积的最小值", 不论这两个变量的符号如何变化, 新的值总是能够在这两个值当中产生, 也能够很容易地写出状态转移方程, 这里就不详细展开了.

代码:

```cpp
class Solution {
public:
    int maxProduct(vector<int> &nums) {
        int current_max_product = nums[0];
        int current_min_product = nums[0];

        int max_product = nums[0];

        for (int current_position = 1; current_position < nums.size();
             current_position++) {
            const int current_number = nums[current_position];

            int temporary_product_1 = current_max_product * current_number;
            int temporary_product_2 = current_min_product * current_number;

            current_max_product = std::max(
                current_number,
                std::max(temporary_product_1, temporary_product_2)
            );
            current_min_product = std::min(
                current_number,
                std::min(temporary_product_1, temporary_product_2)
            );

            max_product = std::max(max_product, current_max_product);
        }

        return max_product;
    }
};
```

## 155. 最小栈 (Min Stack)

标签: 动态规划

解法:

- 由于栈的先进后出的特殊性质, 对于当前栈中的序列, 其最小值等于将栈顶弹出后的较短序列的最小值与栈顶元素之间的较小值, 反过来向栈中压入一个元素后形成的更长序列的最小值等于当前栈中序列的最小值与所压入元素之间的较小值. 因此我们考虑使用另一个辅助栈来存储 "当前主栈中的序列的最小值", 这样就能方便快捷地达到 $O(1)$ 时间的查询效率.

代码:

```cpp
class MinStack {
public:
    MinStack() {
    }

    void push(int value) {
        _number_stack.push(value);

        const int previous_smallest_number =
            _smallest_number_stack.empty() ? std::numeric_limits<int>::max()
                                           : _smallest_number_stack.top();
        const int current_smallest_number =
            std::min(previous_smallest_number, value);

        _smallest_number_stack.push(current_smallest_number);
    }

    void pop() {
        _number_stack.pop();
        _smallest_number_stack.pop();
    }

    int top() {
        return _number_stack.top();
    }

    int getMin() {
        return _smallest_number_stack.top();
    }

private:
    std::stack<int> _number_stack;
    std::stack<int> _smallest_number_stack;
};

/**
 * Your MinStack object will be instantiated and called as such:
 * MinStack* obj = new MinStack();
 * obj->push(val);
 * obj->pop();
 * int param_3 = obj->top();
 * int param_4 = obj->getMin();
 */
```

## 160. 相交链表 (Intersection of Two Linked Lists)

标签: 脑筋急转弯, 双指针

解法:

- 首先假设两个链表存在相交结点. 设从链表 1 到达相交结点的距离为 a, 从链表 2 到达相交结点的距离为 b, 从相交结点到达链表尾的距离为 c, 由加法交换律, $(a + c) + b = (b + c) + a$, 如果我们令指针 1 从链表 1 开始, 指针 2 从链表 2 开始, 并令指针 1 在到达链表尾部时转而从链表 2 开始, 令指针 2 在到达链表尾部时转而从链表 1 开始, 那么最终两者将恰好在相交结点处相遇 (在 $a = b$ 的情况下甚至还会提前相遇). 如果两个链表不存在相交结点, 那么这样做将会令两个指针同时到达链表尾部. 因此我们可以根据两个指针的值的不同情况对链表中是否存在环做出判断.

代码:

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        auto headA_backup = headA;
        auto headB_backup = headB;

        while (true) {
            if (headA == nullptr && headB == nullptr) {
                return nullptr;
            }

            else if (headA == headB) {
                return headA;
            }

            headA = headA ? headA->next : headB_backup;
            headB = headB ? headB->next : headA_backup;
        }

        throw std::runtime_error("never reaches here");
    }
};
```

## 169. 多数元素 (Majority Element)

标签: 脑筋急转弯, Boyer-Moore 投票算法

解法:

- 假设存在多数元素, 现将数组中的互不相同的元素两两打包并抵消, 那么剩下的数就必然是多数元素. 如果不存在多数元素, 那么最后剩下的元素有可能是 "少数元素" 中无法被抵消的 "幸存者", 因此还需要从头对数组进行遍历以检查这些剩下的元素是否确实是多数元素. 但由于本题中已经约定了必然存在多数元素, 因此最后的遍历可以省去.

代码:

```cpp
class Solution {
public:
    int majorityElement(vector<int> &nums) {
        int current_vote_number = 1;
        int current_most_voted_number = nums.back();
        nums.pop_back();

        for (auto current_number : nums) {
            if (current_number == current_most_voted_number) {
                current_vote_number++;
            }

            else if (current_vote_number == 0) {
                current_vote_number = 1;
                current_most_voted_number = current_number;
            }

            else {
                current_vote_number--;
            }
        }

        return current_most_voted_number;
    }
};
```

## 198. 打家劫舍 (House Robber)

标签: 动态规划

解法:

- 本题属于较为简单的一类一维动态规划的题目. 考虑 "从数组开头到当前位置为止的范围内能够偷窃到的最高金额", 如果偷窃当前位置的房屋, 那么由于报警系统的制约, 能够偷窃到的最高金额等于当前房屋的价值加上 "从数组开头到当前位置的**前两个位置**为止的范围内能够偷窃到的最高金额"; 反之如果不偷窃当前位置的房屋, 那么能够偷窃到的金额就等于 "从数组开头到当前位置的**前一个位置**为止的范围内能够偷窃到的最高金额". 于是我们可以很容易地写出状态转移方程.

代码:

```cpp
class Solution {
public:
    int rob(vector<int> &nums) {
        if (nums.size() <= 2) {
            if (nums.size() == 1) {
                return nums[0];
            }

            else {
                return std::max(nums[0], nums[1]);
            }
        }

        int max_value_ends_at_current_position_minus_2;
        int max_value_ends_at_current_position_minus_1 = nums[0];
        int max_value_ends_at_current_position = std::max(nums[0], nums[1]);

        for (int i = 2; i < nums.size(); i++) {
            max_value_ends_at_current_position_minus_2 =
                max_value_ends_at_current_position_minus_1;
            max_value_ends_at_current_position_minus_1 =
                max_value_ends_at_current_position;

            max_value_ends_at_current_position = std::max(
                nums[i] + max_value_ends_at_current_position_minus_2,
                max_value_ends_at_current_position_minus_1
            );
        }

        return max_value_ends_at_current_position;
    }
};
```

## 200. 岛屿数量 (Number of Islands)

标签: 深度优先搜索, 广度优先搜索, 并查集

解法:

- 第一种解法是使用深度优先搜索. 最外层循环对矩阵进行逐行逐列的遍历, 每当遇到为 `1` 的元素便使用深度优先搜索将该元素所处的岛屿填充为 `0`, 这样最外层循环所遍历到的为 `1` 的元素的总数就是岛屿的数量.
- 第二种解法是使用广度优先搜索. 其中最外层循环的逻辑和深度优先搜索中的相同, 不同的是对岛屿的填充方式由深度优先搜索更改为广度优先搜索.
- 第三种解法是使用并查集, 即将当前元素与上下左右四个元素所处的岛屿进行合并.

代码 (深度优先搜索的解法):

```cpp
class Solution {
public:
    void fill_in(vector<vector<char>> &grid, int i, int j) {
        grid[i][j] = '0';

        if (i - 1 >= 0 && grid[i - 1][j] == '1') {
            fill_in(grid, i - 1, j);
        }

        if (j - 1 >= 0 && grid[i][j - 1] == '1') {
            fill_in(grid, i, j - 1);
        }

        if (i + 1 < grid.size() && grid[i + 1][j] == '1') {
            fill_in(grid, i + 1, j);
        }

        if (j + 1 < grid[0].size() && grid[i][j + 1] == '1') {
            fill_in(grid, i, j + 1);
        }
    }

    int numIslands(vector<vector<char>> &grid) {
        int number_of_islands = 0;

        for (int i = 0; i < grid.size(); i++) {
            for (int j = 0; j < grid[0].size(); j++) {
                if (grid[i][j] == '1') {
                    fill_in(grid, i, j);

                    number_of_islands++;
                }
            }
        }

        return number_of_islands;
    }
};
```

代码 (广度优先搜索的解法):

(略)

代码 (并查集的解法):

(略)

## 206. 反转链表 (Reverse Linked List)

标签: 迭代

解法:

- 第一种解法是使用迭代. 直接一板一眼地使用迭代对链表进行反转即可.
- 第二种解法是使用递归. 递归的解法稍微有些复杂, 对于当前链表, 我们将从 "链表的头结点的下一结点" 开始的子链表送入递归中进行反转并得到子链表的反转后的链表的头结点, 此时原子链表的头结点变为新子链表的尾结点, 于是我们可以将原始链表的头结点插入新子链表的尾结点之后作为新链表的尾结点.

代码 (迭代的解法):

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
    ListNode *reverseList(ListNode *head) {
        ListNode *new_list_head = nullptr;

        while (head) {
            const auto next_node = head->next;
            head->next = new_list_head;
            new_list_head = head;
            head = next_node;
        }

        return new_list_head;
    }
};
```

代码 (递归的解法):

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
    ListNode *do_reverse_list(ListNode *head) {
        if (!head->next) {
            return head;
        }

        const auto reverted_list_head = do_reverse_list(head->next);
        const auto reverted_list_before_tail = head->next;
        const auto &reverted_list_tail = head;

        reverted_list_before_tail->next = reverted_list_tail;
        reverted_list_tail->next = nullptr;

        return reverted_list_head;
    }

    ListNode *reverseList(ListNode *head) {
        if (!head) {
            return nullptr;
        }

        return do_reverse_list(head);
    }
};
```
