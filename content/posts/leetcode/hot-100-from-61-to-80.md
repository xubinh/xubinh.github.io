---
title: "力扣 Hot 100 题解归档 (61-80)"
hideSummary: true
date: 2025-01-27T00:25:39+08:00
draft: false
tags: ["leetcode"]
series: ["leetcode"]
author: ["xubinh"]
type: posts
math: true
---

## 207. 课程表 (Course Schedule)

标签: 拓扑排序, 广度优先搜索, 深度优先搜索

解法:

- 第一种解法是使用拓扑排序, 或者说从入度为零的结点开始的广度优先搜索. 我们使用一个队列来存储所有当前遍历到的入读为零的结点, 每次我们从队列中出队一个结点并将其加入当前拓扑序列的末尾, 然后遍历该结点的所有邻居, 由于该结点已经被遍历, 因此所有这些邻居的入度都应该减一, 我们将那些入度降为零的邻居入队, 然后重复前述过程直到队列为空为止, 此时的拓扑序列即为最终的合法的拓扑序列.
- 第二种解法是使用深度优先搜索. 第一次我们从任意一个结点出发, 使用深度优先搜索收集所有遍历到的结点, 如果路上遇到了**当前正在被遍历的**结点说明当前深度优先搜索路径中存在环, 于是可以直接返回 `false`, 否则遍历返回的过程中我们将路径中的每个结点标记为**已经被遍历过的**. 此后的每一次我们均从任意没有被遍历到的结点出发, 如果路上遇到了已经被遍历过的结点则忽略, 如果遇到了当前正在被遍历的结点则仍然直接返回 `false`, 以此类推, 直到所有结点均被遍历过为止. 可以看到我们需要维护三个状态, 分别是初始状态 (`NOT_VISITED`), 当前正在被遍历 (`VISITING`), 以及已经被遍历过 (`VISITED`).
- 如果说广度优先搜索是按照 "从左到右" 的顺序 "一个一个" 复原出拓扑序列, 那么深度优先搜索就是按照 "从右到左" 的顺序 "一段一段" 复原出拓扑序列.

代码 (拓扑排序的解法):

```cpp
class Solution {
public:
    bool canFinish(int numCourses, vector<vector<int>> &prerequisites) {
        std::vector<std::vector<int>> adjacency_list(numCourses);
        std::vector<int> in_degrees_of_nodes(numCourses, 0);

        for (const auto &v_and_u : prerequisites) {
            int u = v_and_u[1];
            int v = v_and_u[0];

            adjacency_list[u].push_back(v);
            in_degrees_of_nodes[v]++;
        }

        std::queue<int> bfs_queue;
        int number_of_dequeued_nodes = 0;

        for (int u = 0; u < numCourses; u++) {
            if (in_degrees_of_nodes[u] == 0) {
                bfs_queue.push(u);
            }
        }

        while (!bfs_queue.empty()) {
            const int u = bfs_queue.front();
            bfs_queue.pop();

            number_of_dequeued_nodes++;

            for (const int v : adjacency_list[u]) {
                in_degrees_of_nodes[v]--;

                if (in_degrees_of_nodes[v] < 0) {
                    return false;
                }

                else if (in_degrees_of_nodes[v] == 0) {
                    bfs_queue.push(v);
                }
            }
        }

        return number_of_dequeued_nodes == numCourses;
    }
};
```

代码 (深度优先搜索的解法):

```cpp
class Solution {
public:
    enum VisitStatus { NOT_VISITED, VISITING, VISITED };

    bool
    dfs(std::vector<std::vector<int>> &adjacency_list,
        int u,
        std::vector<VisitStatus> &visit_statuses) {
        visit_statuses[u] = VISITING;

        for (const int v : adjacency_list[u]) {
            if (visit_statuses[v] == VISITING) {
                return false;
            }

            else if (visit_statuses[v] == NOT_VISITED) {
                if (!dfs(adjacency_list, v, visit_statuses)) {
                    return false;
                }
            }
        }

        visit_statuses[u] = VISITED;

        return true;
    }

    bool canFinish(int numCourses, vector<vector<int>> &prerequisites) {
        std::vector<std::vector<int>> adjacency_list(numCourses);
        std::vector<VisitStatus> visit_statuses(numCourses, NOT_VISITED);

        for (auto &v_and_u : prerequisites) {
            const int u = v_and_u[1];
            const int v = v_and_u[0];

            adjacency_list[u].push_back(v);
        }

        for (int u = 0; u < numCourses; u++) {
            if (visit_statuses[u] == NOT_VISITED) {
                if (!dfs(adjacency_list, u, visit_statuses)) {
                    return false;
                }
            }
        }

        return true;
    }
};
```

## 208. 实现 Trie (前缀树) (Implement Trie (Prefix Tree))

标签: 字典树

解法:

- 本题实际上是在实现一个度为 26 的树.
  - 对于插入操作 (`insert`), 我们可以从根结点开始遍历, 在遍历过程中一边遍历一边创建此前未出现的字母所对应的结点, 并在到达最后一个字母所对应的结点时将其标记为 "有单词以此结点结尾", 以便为之后的查找提供信息.
  - 对于查找操作 (`search`), 我们同样可以从根结点开始遍历, 如果中途发现当前字母所对应的结点还未创建则说明查找失败, 如果所有结点均已创建但最后一个字母所对应的结点并未被标记为 "有单词以此结点结尾" 则同样说明查找失败, 否则说明查找成功.
  - 对于查找前缀操作 (`startsWith`), 我们使用类似查找操作中的过程进行查找, 但区别是我们无需确保最后一个字母所对应的结点必须被标记为 "有单词以此结点结尾", 而是只要确定所有结点均已创建即可返回查找成功.

代码:

```cpp
class Trie {
private:
    struct Letter {
        bool ends_here = false;
        Letter *next_letters[26]{};
    };

    struct MemoryManager {
        ~MemoryManager() {
            for (auto allocated_letter : allocated_nletter) {
                delete allocated_letter;
            }
        }

        void push(Letter *allocated_letter) {
            allocated_nletter.push_back(allocated_letter);
        }

        std::vector<Letter *> allocated_nletter;
    };

public:
    Trie()
        : _root(new Letter) {
        _memory_manager.push(_root);
    }

    void insert(const string &word) {
        auto current_node = _root;

        for (const char letter : word) {
            const int next_letter_index = letter - 'a';

            if (!current_node->next_letters[next_letter_index]) {
                const auto next_letter_ptr = new Letter;

                _memory_manager.push(next_letter_ptr);

                current_node->next_letters[next_letter_index] = next_letter_ptr;
            }

            current_node = current_node->next_letters[next_letter_index];
        }

        current_node->ends_here = true;
    }

    bool search(const string &word) {
        auto current_node = _root;

        for (auto letter : word) {
            const int next_letter_index = letter - 'a';

            if (!current_node->next_letters[next_letter_index]) {
                return false;
            }

            current_node = current_node->next_letters[next_letter_index];
        }

        return current_node->ends_here;
    }

    bool startsWith(const string &prefix) {
        auto current_node = _root;

        for (auto letter : prefix) {
            const int next_letter_index = letter - 'a';

            if (!current_node->next_letters[next_letter_index]) {
                return false;
            }

            current_node = current_node->next_letters[next_letter_index];
        }

        return true;
    }

private:
    Letter *_root;
    MemoryManager _memory_manager;
};

/**
 * Your Trie object will be instantiated and called as such:
 * Trie* obj = new Trie();
 * obj->insert(word);
 * bool param_2 = obj->search(word);
 * bool param_3 = obj->startsWith(prefix);
 */
```

## 215. 数组中的第K个最大元素 (Kth Largest Element in an Array)

标签: 快速排序, 堆排序

解法:

- 第一种解法是使用快速排序中的扫描算法, 每一趟扫描都能够将数组分为三个部分, 分别是小于当前主元的部分, 等于当前主元的部分, 以及大于当前主元的部分, 而第 k 大的元素必然出现在这三个部分中的其中一个部分中, 于是我们便能够通过不断循环并缩小搜索范围来确定第 k 大的数.
- 第二种解法是使用堆排序. 堆排序的具体算法可以参考《算法导论》. 假设我们要维护一个最小堆, 那么堆排序将涉及到如下基本操作:
  - 向下调整 (`adjust_down`): 假设当前堆的左右两个子堆符合最小堆的性质, 而根结点却大于两个子堆的堆顶结点 (即最小结点), 我们便需要将根结点与两个子堆的堆顶结点中较小的那个进行互换 (此时两个子堆的堆顶结点中较小的那个结点的值实际上就是整个堆的最小值). 互换后的那个子堆将不再符合最小堆的性质, 因此我们需要继续对互换后的子堆进行向下调整. 重复上述过程直到到达堆底为止.
  - 建堆 (`build`): 建堆的过程实际上就是从堆的最后一个元素开始按相反的顺序进行遍历, 对于每个遍历到的元素, 我们对以该元素为根结点的堆执行一次向下调整操作, 直到遍历完第一个结点为止. 可以证明建堆过程总体的时间复杂度不会超过 $O(n)$, 其中 $n$ 为堆的总大小.
  - 出堆 (`pop`): 出堆实际上就是将堆的最后一个元素放置到堆顶以模仿弹出堆顶元素的操作, 然后对其进行向下调整操作. 之所以需要执行向下调整操作是因为堆的最后一个元素可能大于原来的堆顶元素, 而将其放置到堆顶有可能破坏堆的性质.

代码 (快速排序的解法):

```cpp
class Solution {
public:
    int findKthLargest(vector<int> &nums, int k) {
        k--;

        int range_begin = 0;
        int range_end = nums.size();

        while (true) {
            const int pivot = nums[range_begin];

            int left_ptr = range_begin;
            int current_ptr = range_begin;
            int right_ptr = range_end;

            while (current_ptr < right_ptr) {
                const int current_number = nums[current_ptr];

                if (current_number > pivot) {
                    swap(nums[current_ptr], nums[left_ptr]);

                    left_ptr++;
                    current_ptr++;
                }

                else if (current_number == pivot) {
                    current_ptr++;
                }

                else {
                    right_ptr--;

                    swap(nums[current_ptr], nums[right_ptr]);
                }
            }

            if (k < left_ptr) {
                range_end = left_ptr;
            }

            else if (k < current_ptr) {
                return nums[left_ptr];
            }

            else {
                range_begin = current_ptr;
            }
        }

        throw std::runtime_error("never reaches here");
    }
};
```

代码 (堆排序的解法):

```cpp
class Solution {
public:
    void max_heap_adjust_down(std::vector<int> &nums, const int root_index) {
        const int nums_size = nums.size();

        int current_node_index = root_index;

        while (true) {
            const int left_child_node_index = current_node_index * 2 + 1;
            const int right_child_node_index = left_child_node_index + 1;

            int largest_node_index = current_node_index;

            if (left_child_node_index < nums_size
                && nums[left_child_node_index] > nums[largest_node_index]) {
                largest_node_index = left_child_node_index;
            }

            if (right_child_node_index < nums_size
                && nums[right_child_node_index] > nums[largest_node_index]) {
                largest_node_index = right_child_node_index;
            }

            if (largest_node_index != current_node_index) {
                swap(nums[current_node_index], nums[largest_node_index]);

                current_node_index = largest_node_index;
            }

            else {
                return;
            }
        }

        throw std::runtime_error("never reaches here");
    }

    void max_heap_build(std::vector<int> &nums) {
        for (int current_node_index = ((nums.size() - 1) - 1) / 2;
             current_node_index >= 0;
             current_node_index--) {
            max_heap_adjust_down(nums, current_node_index);
        }
    }

    int max_heap_pop(std::vector<int> &nums) {
        const int popped_value = nums[0];
        nums[0] = nums.back();
        nums.pop_back();

        max_heap_adjust_down(nums, 0);

        return popped_value;
    }

    int findKthLargest(vector<int> &nums, int k) {
        max_heap_build(nums);

        for (int i = 0; i < k - 1; i++) {
            max_heap_pop(nums);
        }

        return max_heap_pop(nums);
    }
};
```

## 221. 最大正方形 (Maximal Square)

标签: 动态规划

解法:

- 本题与 "85. 最大矩形 (Maximal Rectangle)" 不同. 本题中所要求的是**正方形**, 相比矩形而言要额外多了一层约束, 因此状态转移方程要相对简单许多. 对于当前遍历到的元素, 我们考虑 "以该元素为右下角的最大正方形", 易知该正方形的边长由 "以该元素的左边元素为右下角的最大正方形" 的边长和 "以该元素的上方元素为右下角的最大正方形" 的边长中的较小值决定, 如果这两个正方形的重叠的矩形区域的左上角元素的左上方元素同样为 `1`, 那么 "以该元素为右下角的最大正方形" 的最大边长还能够再加一, 于是我们能够很容易地写出状态转移方程. 由于状态转移过程中只用到了左边元素和上方元素的信息, 因此我们可以使用滚动数组将空间复杂度优化至 $O(n)$.

代码:

```cpp
class Solution {
public:
    int maximalSquare(vector<vector<char>> &matrix) {
        int I = matrix.size();
        int J = matrix[0].size();

        std::vector<int> dp;

        int max_side_length = 0;

        std::transform(
            matrix[0].begin(),
            matrix[0].end(),
            std::back_inserter(dp),
            [&](char c) -> int {
                int value = c - '0';
                max_side_length = std::max(max_side_length, value);

                return value;
            }
        );

        for (int i = 1; i < I; i++) {
            dp[0] = matrix[i][0] == '0' ? 0 : 1;
            max_side_length = std::max(max_side_length, dp[0]);

            for (int j = 1; j < J; j++) {
                if (matrix[i][j] == '0') {
                    dp[j] = 0;

                    continue;
                }

                int current_max_side_length = std::min(dp[j - 1], dp[j]);

                if (matrix[i - current_max_side_length]
                          [j - current_max_side_length]
                    == '1') {
                    current_max_side_length += 1;
                }

                dp[j] = current_max_side_length;

                max_side_length =
                    std::max(max_side_length, current_max_side_length);
            }
        }

        return max_side_length * max_side_length;
    }
};
```

## 226. 翻转二叉树 (Invert Binary Tree)

标签: 递归, 迭代

解法:

- 第一种解法是使用递归. 我们可以递归地先对两棵子树进行翻转, 然后再交换左右子树的根结点, 这等价于对整棵树进行翻转. 递归的解法实际上类似于深度优先搜索.
- 第二种解法是使用迭代, 类似于广度优先搜索.

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
    void do_invert_tree(TreeNode *root) {
        if (!root) {
            return;
        }

        do_invert_tree(root->left);
        do_invert_tree(root->right);
        std::swap(root->left, root->right);
    }

    TreeNode *invertTree(TreeNode *root) {
        do_invert_tree(root);

        return root;
    }
};
```

代码 (迭代的解法):

(略)

## 234. 回文链表 (Palindrome Linked List)

标签: 双指针, 迭代, 递归

解法:

- 第一种解法是使用迭代. 如果要判断一个链表是否为回文链表, 我们实际上只需要对链表进行二分, 并对右半部分进行反转, 然后逐个匹配左右两个子链表即可. 由于最后需要对链表进行复原, 这里我们选择对右半部分而不是左半部分进行反转.
- 第二种解法是使用递归. 在递归的解法中我们实际上是在使用函数的调用栈来正向遍历并存储整个链表的所有结点, 并在递归返回时自然地对链表进行逆向遍历, 在逆向遍历的过程中我们可以额外维护一个独立地进行正向遍历的指针, 在每层递归返回时, 逆向指针自然地向左移动, 而正向指针将被我们手动向右移动, 于是我们便能够通过比较正向指针和逆向指针所指向地一对结点的值是否相等来判断该链表是否为回文链表.

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
    ListNode *cut_list_in_half(ListNode *head) {
        if (!head || !head->next) {
            return nullptr;
        }

        auto slow_ptr = head;
        auto fast_ptr = head;

        while (fast_ptr->next && fast_ptr->next->next) {
            fast_ptr = fast_ptr->next->next;
            slow_ptr = slow_ptr->next;
        }

        return slow_ptr->next;
    }

    ListNode *revert_list(ListNode *head) {
        ListNode *reverted_list_head = nullptr;
        ListNode *current_node = head;

        while (current_node) {
            auto next_node = current_node->next;
            current_node->next = reverted_list_head;
            reverted_list_head = current_node;
            current_node = next_node;
        }

        return reverted_list_head;
    }

    bool isPalindrome(ListNode *head) {
        if (!head || !head->next) {
            return true;
        }

        auto left_half_list = head;
        auto right_half_list = cut_list_in_half(head);
        auto reverted_right_half_list = revert_list(right_half_list);
        auto reverted_right_half_list_backup = reverted_right_half_list;

        bool is_palindrome = true;

        while (left_half_list && reverted_right_half_list) {
            if (left_half_list->val != reverted_right_half_list->val) {
                is_palindrome = false;

                break;
            }

            left_half_list = left_half_list->next;
            reverted_right_half_list = reverted_right_half_list->next;
        }

        revert_list(reverted_right_half_list_backup);

        return is_palindrome;
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
    bool do_is_palindrome(
        ListNode *right_to_left_ptr, ListNode *&left_to_right_ptr
    ) {
        if (!right_to_left_ptr) {
            return true;
        }

        if (!do_is_palindrome(right_to_left_ptr->next, left_to_right_ptr)) {
            return false;
        }

        if (left_to_right_ptr->val != right_to_left_ptr->val) {
            return false;
        }

        left_to_right_ptr = left_to_right_ptr->next;

        return true;
    }

    bool isPalindrome(ListNode *head) {
        ListNode *left_to_right_ptr = head;

        return do_is_palindrome(head, left_to_right_ptr);
    }
};
```

## 236. 二叉树的最近公共祖先 (Lowest Common Ancestor of a Binary Tree)

标签: 递归

解法:

- 我们递归地对树进行遍历. 对于当前遍历到的树:
  - 如果根结点为空, 那么返回空指针.
  - 如若不然, 如果根结点恰为 `p` 或 `q` 中的一个, 那么返回值为根结点.
  - 如若不然, 如果左子树为空, 那么我们递归地遍历右子树并返回其结果.
  - 如若不然, 如果右子树为空, 那么我们递归地遍历左子树并返回其结果.
  - 如若不然, 我们同时对左右子树进行遍历. 对于左右子树所返回的结果:
    - 如果左子树和右子树均返回了非空结果, 说明 `p` 和 `q` 分别位于左右子树中, 当前根结点就是二者的最近公共祖先, 于是我们返回根结点.
    - 如若不然, 如果左子树返回了空结果, 那么我们返回右子树的结果.
    - 如若不然, 我们返回左子树的结果.

  由上述过程可知, 如果遍历一棵树所得到的结果为空, 那么说明这棵树中不存在 `p` 和 `q`; 如果所得到的结果为非空, 那么说明这棵树中至少存在 `p` 和 `q` 中的其中一个. 由于题目保证 `p` 和 `q` 必然存在于顶层树中, 因此当顶层树的左子树返回空结果时说明右子树的根结点就是二者的最近公共祖先, 反之亦然. 算法的正确性得证.

代码:

```cpp
class Solution {
public:
    TreeNode *lowestCommonAncestor(TreeNode *root, TreeNode *p, TreeNode *q) {
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

## 238. 除自身以外数组的乘积 (Product of Array Except Self)

标签: 前缀和

解法:

- 对于当前元素而言, "除自身以外数组的乘积" 实际上就等于 "自身左边所有元素的乘积" 乘上 "自身右边所有元素的乘积". 由于这里涉及到两组前缀和 (或前缀乘积 (名称不重要, 重要的是思想)), 因此我们必须额外使用一个数组来记住任意一个前缀和, 然后在反向遍历的过程中实时计算出另一组前缀和并生成最终的答案数组.

代码:

```cpp
class Solution {
public:
    vector<int> productExceptSelf(vector<int> &nums) {
        std::vector<int> products(nums.size());

        products[0] = 1;

        for (int i = 1; i < nums.size(); i++) {
            products[i] = products[i - 1] * nums[i - 1];
        }

        int suffix_product = 1;

        for (int i = nums.size() - 2; i >= 0; i--) {
            suffix_product *= nums[i + 1];

            products[i] *= suffix_product;
        }

        return products;
    }
};
```

## 239. 滑动窗口最大值 (Sliding Window Maximum)

标签: 滑动窗口, 单调队列

解法:

- 对于每个滑动窗口, 我们需要**实时**维护该窗口中的最大值. 观察到滑动窗口总是从左向右移动, 对于当前窗口中的最大值, 我们总是可以确定**从当前窗口的最左端开始直到最大值所处的位置为止** 这中间所有的元素 (除最大值元素本身以外) 均已不可能成为接下来任何滑动窗口的最大值, 因此我们可以放心地将其忽略. 同理, 从最大值到次大值也是如此, 以此类推, 最终滑动窗口中将只剩下一个由严格递减的元素构成的单调队列. 而每当滑动窗口向右移动时, 原窗口中的最右端的元素的下一个元素将加入单调队列, 而原窗口中的最左端的元素 (若未被忽略) 则将从单调队列中出队. 对于每个滑动窗口, 其对应的单调队列中最左端的值即为该窗口中的最大值, 我们只需要如实记录即可.

代码:

```cpp
class Solution {
public:
    vector<int> maxSlidingWindow(vector<int> &nums, int k) {
        std::vector<int> biggest_number_of_each_window;

        std::deque<int> current_window_decreasing_queue;

        for (int current_position = 0; current_position < nums.size();
             current_position++) {
            auto current_number = nums[current_position];

            while (!current_window_decreasing_queue.empty()
                   && nums[current_window_decreasing_queue.back()]
                          <= current_number) {
                current_window_decreasing_queue.pop_back();
            }

            current_window_decreasing_queue.push_back(current_position);

            if (current_position < k - 1) {
                continue;
            }

            if (current_window_decreasing_queue.front()
                <= current_position - k) {
                current_window_decreasing_queue.pop_front();
            }

            biggest_number_of_each_window.push_back(
                nums[current_window_decreasing_queue.front()]
            );
        }

        return biggest_number_of_each_window;
    }
};
```

## 240. 搜索二维矩阵 II (Search a 2D Matrix II)

标签: 双指针

解法:

- 由于矩阵具有从左到右非递减, 从上到下非递减的性质, 我们考虑矩阵右上角的元素:
  - 如果该元素大于目标值, 那么该元素下方的所有元素 (由于从上到下非递减的性质) 均将会大于目标值, 因此可将其忽略. 这等价于矩阵的最后一列被删去, 原矩阵化为列数减一的新子矩阵, 原问题转化为等价的子问题.
  - 如果该元素小于目标值, 那么该元素左边的所有元素 (由于从左到右非递减的性质) 均将会小于目标值, 因此可将其忽略. 这等价于矩阵的第一行被删去, 原矩阵化为行数减一的新子矩阵, 原问题转化为等价的子问题.

  由此可见如果我们重复上述步骤, 原问题将最终被转化为一个平凡的边界子问题.

代码:

```cpp
class Solution {
public:
    bool searchMatrix(vector<vector<int>> &matrix, int target) {
        int i = 0;
        int j = matrix[0].size() - 1;

        while (i < matrix.size() && j >= 0) {
            if (matrix[i][j] == target) {
                return true;
            }

            else if (matrix[i][j] > target) {
                j--;
            }

            else {
                i++;
            }
        }

        return false;
    }
};
```

## 253. 会议室 II

(本题为力扣 VIP 会员专属题目)

## 279. 完全平方数 (Perfect Squares)

标签: 动态规划

解法:

- 本题类似于背包问题, 但由于完全平方数的性质, 本题实际上可以采用更为简单粗暴的解法. 对于当前遍历到的数 $n$ 而言, 其第一个完全平方数零件可以是 $1^2, 2^2, \dots, \lfloor \sqrt{n} \rfloor^2$ 中的任意一个, 此后原问题转化为求 $n - i^2$ 的组合个数的子问题, 其中 $i = 1, 2, \dots, \lfloor \sqrt{n} \rfloor$. 注意到我们在此处是对 $n$ 进行遍历, 而不是对可能的完全平方数零件 $i^2$ 进行遍历, 而在一般的 0-1 背包问题中由于每个零件的大小关系不确定, 我们在主循环中是对各个零件进行遍历, 这里存在较大的区别.

代码:

```cpp
class Solution {
public:
    int get_sqrt_floor(int target) {
        return static_cast<int>(std::sqrt(static_cast<double>(target)) + 1e-8);
    }

    int numSquares(int n) {
        std::vector<int> dp(n + 1, std::numeric_limits<int>::max());

        dp[0] = 0;
        dp[1] = 1;

        for (int current_target = 2; current_target <= n; current_target++) {
            int sqrt_floor = get_sqrt_floor(current_target);

            for (int base = 1; base <= sqrt_floor; base++) {
                int square = base * base;

                dp[current_target] = std::min(
                    dp[current_target], 1 + dp[current_target - square]
                );
            }
        }

        return dp[n];
    }
};
```

## 283. 移动零 (Move Zeroes)

标签: 快速排序, 双指针

解法:

- 直接套用快速排序中的扫描算法即可, 其中 `0` 作为主元被扫到第二部分, 其余不为零的元素均视作 "小于" 零并被扫到第一部分, 第三部分保持为空.

代码:

```cpp
class Solution {
public:
    void moveZeroes(vector<int> &nums) {
        int left_ptr = 0;
        int current_ptr = 0;
        int nums_size = nums.size();

        while (current_ptr < nums_size) {
            if (nums[current_ptr] == 0) {
                current_ptr++;
            }

            else {
                std::swap(nums[left_ptr], nums[current_ptr]);

                left_ptr++;
                current_ptr++;
            }
        }
    }
};
```

## 287. 寻找重复数 (Find the Duplicate Number)

标签: 二分查找, 位运算, 快慢指针

解法:

- 首先题目给出了较为严格的约束: 整个数组中只有一个重复的整数.
- 其次题目提出了较为严格的要求: 必须不修改数组, 且只用常量级 $O(1)$ 的额外空间.
- 第一种解法是使用二分查找. 注意到对于任何 "小于重复的数字" 的数字而言, "整个数组中小于等于该数字的数字的个数" 必然小于等于 "该数字的值"; 反过来对于任何 "大于等于重复的数字" 的数字而言, "整个数组中小于等于该数字的数字的个数" 必然大于 "该数字的值", 这就在两者之间建立起了顺序关系, 我们便可以通过二分查找来找到这一分界线, 而分界线的值恰好就是我们要求的目标值.
- 第二种解法是使用位运算. 以最低有效位为例, 由于仅存在一个重复的数字, 如果该数字的最低有效位为一, 由于数组的长度为 $n + 1$, 整个数组中的所有数字的最低有效位之和必然大于 $1$ 到 $n$ 这 $n$ 个互不相同的数字的最低有效位之和; 反之整个数组中的所有数字的最低有效位之和必然小于等于 $1$ 到 $n$ 这 $n$ 个互不相同的数字的最低有效位之和. 根据这一点我们便能够一位一位地推出所重复的数字.
- 第三种解法是使用快慢指针. 注意到由于数组中的元素的值总是位于 $[1, n]$ 这个范围, 整个数组实际上形成了一个静态链表, 数组元素的值即为该元素所指向的下一个元素在数组中的索引. 由于每个结点总是指向非空结点, 无论我们从哪个结点出发, 我们最终都将会进入一个环中, 同时题目保证了入环口的存在性 (由数组长度大于 $n$ 推出, 否则数组中的元素有可能形成首尾相连的无重复元素的环) 和唯一性 (整个数组中只有一个重复的元素), 因此我们只需要从数组的最后一个元素出发 (为了确保从入环口外开始, 而非一开始就位于环的内部), 使用快慢指针确定入环口, 那么该入环口便是所重复的数字.

代码 (二分查找的解法):

(略)

代码 (位运算):

(略)

代码 (快慢指针):

```cpp
class Solution {
public:
    int findDuplicate(vector<int> &nums) {
        int head = nums.back();

        int slow_ptr = head;
        int fast_ptr = head;

        do {
            fast_ptr = nums[fast_ptr - 1];
            fast_ptr = nums[fast_ptr - 1];

            slow_ptr = nums[slow_ptr - 1];
        } while (slow_ptr != fast_ptr);

        int detached_ptr = head;

        while (detached_ptr != slow_ptr) {
            detached_ptr = nums[detached_ptr - 1];
            slow_ptr = nums[slow_ptr - 1];
        }

        return detached_ptr;
    }
};
```

## 297. 二叉树的序列化与反序列化 (Serialize and Deserialize Binary Tree)

标签: 前序遍历, 中序遍历, 层序遍历

解法:

- 第一种解法是使用前序遍历. 在序列化的时候, 我们首先序列化当前树的根结点, 然后递归地序列化左子树和右子树. 如果一棵树为空, 那么我们使用某个特殊的字符串 (例如 `N`) 对其进行标记. 在反序列化时, 我们首先提取出根结点, 然后同样递归地反序列化左子树和右子树. 由于我们使用了特殊的字符串标记了空子树, 我们实际上等价地标记了左子树和右子树的范围, 因此递归过程最终将能够成功返回.
- 第二种解法是使用中序遍历. 由于中序序列中根结点的位置从一开始就是不确定的, 因此我们必须使用额外的标记来确定根结点的位置, 例如使用类似于巴科斯范式 (BNF) 的记号 `(<left-sub-tree>) <root-value> (right-sub-tree)`, 其中使用括号表示子树的范围, 同时我们仍然使用特殊的字符串来标记空子树, 这样便能够保证递归过程的正确性.
- 第三种解法是使用层序遍历. 在层序遍历中每一层的结点个数由上一层的非空结点个数决定, 而第一层的结点个数总是确定的 (因为第一层仅包含根结点), 因此能够确保解析过程的正确性.

代码 (前序遍历的解法):

(略)

代码 (中序遍历的解法):

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Codec {
public:
    // Encodes a tree to a single string.
    string serialize(TreeNode *root) {
        if (!root) {
            return "N";
        }

        std::string left_sub_tree = serialize(root->left);
        std::string right_sub_tree = serialize(root->right);
        std::string value_string = std::to_string(root->val);

        size_t tree_size = left_sub_tree.size() + right_sub_tree.size()
                           + value_string.size() + 4;

        std::string tree;

        tree.reserve(tree_size);

        tree += "(";
        tree += left_sub_tree;
        tree += ")";
        tree += value_string;
        tree += "(";
        tree += right_sub_tree;
        tree += ")";

        return tree;
    }

    int parse_int(std::string &data, size_t &current_position) {
        char *next_left_bracket_position;

        long value = ::strtol(
            data.c_str() + current_position, &next_left_bracket_position, 10
        );

        current_position = next_left_bracket_position - data.c_str();

        return static_cast<int>(value);
    }

    TreeNode *do_deserialize(std::string &data, size_t &current_position) {
        if (data[current_position] == 'N') {
            current_position++;

            return nullptr;
        }

        current_position++;

        auto left_sub_tree = do_deserialize(data, current_position);

        current_position++;

        auto root = new TreeNode(parse_int(data, current_position));

        current_position++;

        auto right_sub_tree = do_deserialize(data, current_position);

        current_position++;

        root->left = left_sub_tree;
        root->right = right_sub_tree;

        return root;
    }

    // Decodes your encoded data to tree.
    TreeNode *deserialize(string data) {
        size_t current_position = 0;

        return do_deserialize(data, current_position);
    }
};

// Your Codec object will be instantiated and called as such:
// Codec ser, deser;
// TreeNode* ans = deser.deserialize(ser.serialize(root));
```

代码 (层序遍历的解法):

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Codec {
public:
    // Encodes a tree to a single string.
    string serialize(TreeNode *root) {
        if (!root) {
            return "N,";
        }

        std::queue<TreeNode *> bfs_node_queue;
        bfs_node_queue.push(root);

        std::string tree_string;

        while (!bfs_node_queue.empty()) {
            auto current_node = bfs_node_queue.front();
            bfs_node_queue.pop();

            if (!current_node) {
                tree_string += "N,";

                continue;
            }

            tree_string += std::to_string(current_node->val);
            tree_string += ",";

            bfs_node_queue.push(current_node->left);
            bfs_node_queue.push(current_node->right);
        }

        return tree_string;
    }

    int parse_int(std::string &data, size_t &current_position) {
        char *next_comma_address;

        long value =
            ::strtol(data.c_str() + current_position, &next_comma_address, 10);

        current_position = next_comma_address - data.c_str() + 1;

        return static_cast<int>(value);
    }

    // Decodes your encoded data to tree.
    TreeNode *deserialize(string data) {
        if (data == "N,") {
            return nullptr;
        }

        size_t current_position = 0;
        auto root = new TreeNode(parse_int(data, current_position));

        std::queue<TreeNode *> bfs_node_queue;
        bfs_node_queue.push(root);

        while (!bfs_node_queue.empty()) {
            auto current_node = bfs_node_queue.front();
            bfs_node_queue.pop();

            if (data[current_position] == 'N') {
                current_node->left = nullptr;
                current_position += 2;
            }

            else {
                auto new_node = new TreeNode(parse_int(data, current_position));
                current_node->left = new_node;
                bfs_node_queue.push(new_node);
            }

            if (data[current_position] == 'N') {
                current_node->right = nullptr;
                current_position += 2;
            }

            else {
                auto new_node = new TreeNode(parse_int(data, current_position));
                current_node->right = new_node;
                bfs_node_queue.push(new_node);
            }
        }

        return root;
    }
};

// Your Codec object will be instantiated and called as such:
// Codec ser, deser;
// TreeNode* ans = deser.deserialize(ser.serialize(root));
```

## 300. 最长递增子序列 (Longest Increasing Subsequence)

标签: 动态规划, 二分查找

解法:

- 考虑从左到右遍历数组. 对于每一个遍历到的位置, 我们维护一个 dp 数组表示 "从数组的最左端到当前位置为止这一范围内对应于每个可能的递增子序列长度的所有子序列的末尾元素的最小值". 例如对于序列 `[2, 1, 3]`, 其对应的 dp 数组为 `[1, 3]`, 其中 `1` 是长度为 1 的递增子序列的末尾元素的最小值, 而 `3` 是长度为 2 的递增子序列的末尾元素的最小值, 该序列中不存在长度为 3 以上的递增子序列. 易证 dp 数组中的元素同样满足从左到右严格递增的顺序. 每次向右遍历一个元素时, 我们考虑添加所遍历到的当前元素:
  - 对于 dp 数组中大于等于当前元素的元素而言, 由于这些元素已经大于等于当前元素, 也就无法形成更长的递增序列, 因此可以忽略.
  - 对于 dp 数组中小于当前元素的元素而言, 将当前元素添加进这些元素中的任何一个的末尾都将能够形成更长的递增序列, 但问题是如何添加以便维护 dp 数组的性质不变? 答案是添加至这些元素中最大的那个元素所代表的递增序列的末尾并尝试更新 dp 数组. 这是因为当前元素已经大于已经大于其他更小的元素, 将其添加进任何其他更小的元素的末尾所形成的新的递增序列都无法更新 dp 数组.

  为了确保长度为 1 的递增序列也能够通过上述过程进行更新, 我们可以预先在 dp 数组中添加一个值为 `INT_MIN` 的数来模拟一个长度为 0 的假想数组.

代码:

```cpp
class Solution {
public:
    int lengthOfLIS(vector<int> &nums) {
        std::vector<int> smallest_heads;

        const int nums_size = nums.size();

        smallest_heads.reserve(nums_size + 1);

        smallest_heads.push_back(std::numeric_limits<int>::min());
        smallest_heads.push_back(nums[0]);

        for (int i = 1; i < nums_size; i++) {
            int current_number = nums[i];

            auto it = std::lower_bound(
                smallest_heads.begin(), smallest_heads.end(), current_number
            );

            if (it == smallest_heads.end()) {
                smallest_heads.push_back(current_number);
            }

            else if ((*it) == current_number) {
                continue;
            }

            else {
                (*it) = current_number;
            }
        }

        return smallest_heads.size() - 1;
    }
};
```

## 301. 删除无效的括号 (Remove Invalid Parentheses)

标签: 回溯

解法:

- 本题直接使用一般的回溯即可解答, 但如果不仔细考虑剪枝的话十分容易便会超出时间限制. 本题的剪枝方法主要有以下三点:
  - 所要删除的左右括号的数量是固定的.
  - 右括号能够添加当且仅当目前仍然存在左括号能够用于匹配.
  - 对于连续都是左括号或者连续都是右括号的区间, 我们需要确保不重复且不遗漏地遍历其中的组合, 其中重点是不重复.
- 对于第一点, 要删除的左括号的数量就是整个序列中未能匹配的左括号的数量, 而要删除的右括号的数量等于遍历过程中未能匹配的右括号的数量.
- 对于第三点, 如果我们按照枚举子集的方法来进行回溯, 那么很容易便会超出时间限制. 因此我们必须按照枚举组合的方法, 先通过遍历得到当前连续都是左括号 (或者连续都是右括号) 的区间的长度, 然后从零开始顺序枚举长度 (而非对应于各个长度的所有子集), 这样便能够使时间复杂度最优.

代码:

```cpp
class Solution {
public:
    void do_remove_invalid_parentheses(
        const std::string &s,
        const int current_position,
        const int total_number_of_deletions_needed_for_left_bracket,
        const int total_number_of_deletions_needed_for_right_bracket,
        const int current_number_of_left_brackets_not_offset,
        std::string &current_sub_string,
        std::unordered_set<std::string> &result_set
    ) {

        if (current_position == s.size()) {
            if (total_number_of_deletions_needed_for_left_bracket == 0
                && total_number_of_deletions_needed_for_right_bracket == 0) {
                result_set.insert(current_sub_string);
            }

            return;
        }

        const char current_char = s[current_position];

        const int size_of_current_sub_string = current_sub_string.size();

        if (current_char == '(') {
            int number_of_consecutive_left_brackets = 1;
            int next_starting_position = current_position + 1;

            while (next_starting_position < s.size()
                   && s[next_starting_position] == '(') {

                number_of_consecutive_left_brackets++;
                next_starting_position++;
            }

            for (int current_number_of_deletions_of_left_brackets =
                         number_of_consecutive_left_brackets,
                     current_number_of_left_brackets_kept = 0;
                 current_number_of_deletions_of_left_brackets >= 0;
                 current_number_of_deletions_of_left_brackets--,
                     current_number_of_left_brackets_kept++) {

                if (current_number_of_deletions_of_left_brackets
                    <= total_number_of_deletions_needed_for_left_bracket) {

                    do_remove_invalid_parentheses(
                        s,
                        next_starting_position,
                        total_number_of_deletions_needed_for_left_bracket
                            - current_number_of_deletions_of_left_brackets,
                        total_number_of_deletions_needed_for_right_bracket,
                        current_number_of_left_brackets_not_offset
                            + current_number_of_left_brackets_kept,
                        current_sub_string,
                        result_set
                    );
                }

                current_sub_string.push_back('(');
            }
        }

        else if (current_char == ')') {
            int number_of_consecutive_right_brackets = 1;
            int next_starting_position = current_position + 1;

            while (next_starting_position < s.size()
                   && s[next_starting_position] == ')') {

                number_of_consecutive_right_brackets++;
                next_starting_position++;
            }

            for (int current_number_of_deletions_of_right_brackets =
                         number_of_consecutive_right_brackets,
                     current_number_of_right_brackets_kept = 0;
                 current_number_of_deletions_of_right_brackets >= 0;
                 current_number_of_deletions_of_right_brackets--,
                     current_number_of_right_brackets_kept++) {

                if (current_number_of_deletions_of_right_brackets
                        <= total_number_of_deletions_needed_for_right_bracket
                    && current_number_of_right_brackets_kept
                           <= current_number_of_left_brackets_not_offset) {

                    do_remove_invalid_parentheses(
                        s,
                        next_starting_position,
                        total_number_of_deletions_needed_for_left_bracket,
                        total_number_of_deletions_needed_for_right_bracket
                            - current_number_of_deletions_of_right_brackets,
                        current_number_of_left_brackets_not_offset
                            - current_number_of_right_brackets_kept,
                        current_sub_string,
                        result_set
                    );
                }

                current_sub_string.push_back(')');
            }
        }

        else {
            current_sub_string.push_back(current_char);

            int next_starting_position = current_position + 1;

            while (next_starting_position < s.size()
                   && s[next_starting_position] != '('
                   && s[next_starting_position] != ')') {

                current_sub_string.push_back(s[next_starting_position]);
                next_starting_position++;
            }

            do_remove_invalid_parentheses(
                s,
                next_starting_position,
                total_number_of_deletions_needed_for_left_bracket,
                total_number_of_deletions_needed_for_right_bracket,
                current_number_of_left_brackets_not_offset,
                current_sub_string,
                result_set
            );
        }

        current_sub_string.resize(size_of_current_sub_string);
    }

    vector<string> removeInvalidParentheses(string s) {
        int total_number_of_deletions_needed_for_left_bracket = 0;
        int total_number_of_deletions_needed_for_right_bracket = 0;

        for (auto c : s) {
            if (c == '(') {
                total_number_of_deletions_needed_for_left_bracket++;
            }

            else if (c == ')') {
                if (total_number_of_deletions_needed_for_left_bracket > 0) {
                    total_number_of_deletions_needed_for_left_bracket--;
                }

                else {
                    total_number_of_deletions_needed_for_right_bracket++;
                }
            }
        }

        if (total_number_of_deletions_needed_for_left_bracket == 0
            && total_number_of_deletions_needed_for_right_bracket == 0) {
            return std::vector<std::string>{s};
        }

        std::string current_sub_string;
        std::unordered_set<std::string> result_set;

        do_remove_invalid_parentheses(
            s,
            0,
            total_number_of_deletions_needed_for_left_bracket,
            total_number_of_deletions_needed_for_right_bracket,
            0,
            current_sub_string,
            result_set
        );

        std::vector<std::string> result_vector;

        result_vector.reserve(result_set.size());

        result_vector.insert(
            result_vector.end(),
            std::make_move_iterator(result_set.begin()),
            std::make_move_iterator(result_set.end())
        );

        return result_vector;
    }
};
```

## 309. 买卖股票的最佳时机含冷冻期 (Best Time to Buy and Sell Stock with Cooldown)

标签: 动态规划

解法:

- 本题属于经典的动态规划的题目. 我们考虑维护三个状态, 分别是 "在从数组的最左端到当前位置为止之间的任意位置买入并在此后不做任何操作所能得到的最大收益" (这是因为买入和卖出之间可以间隔任意长的区间), "在当前位置卖出所能得到的最大收益", 以及 "在当前位置冻结所能得到的最大收益". 我们能够很容易地写出状态转移方程:
  - "在从数组的最左端到当前位置为止之间的任意位置买入并在此后不做任何操作所能得到的最大收益" = "在从数组的最左端到当前位置的前一个位置为止之间的任意位置买入并在此后不做任何操作所能得到的最大收益" (即不在当前位置买入所能得到的最大收益) 与 "在当前位置买入所能得到的最大收益" 之间的较大值.
  - "在当前位置卖出所能得到的最大收益" = "在从数组的最左端到当前位置的前一个位置为止之间的任意位置买入并在此后不做任何操作所能得到的最大收益" + "当前位置的价格".
  - "在当前位置冻结所能得到的最大收益" = "在当前位置的前一个位置冻结所能得到的最大收益" 与 "在当前位置的前一个位置卖出所能得到的最大收益" 之间的较大值.
- 由于当前位置的状态仅由前一个位置的状态决定, 因此我们可以使用 $O(1)$ 长度的滚动数组来优化空间复杂度.

代码:

```cpp
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int dp_buy = -prices[0];
        int dp_sold = 0;
        int dp_freeze = 0;

        for (int i = 1; i < prices.size(); i++) {
            const int current_price = prices[i];

            int dp_buy_temp = std::max(dp_buy, dp_freeze - current_price);
            int dp_sold_temp = dp_buy + current_price;
            int dp_freeze_temp = std::max(dp_sold, dp_freeze);

            dp_buy = dp_buy_temp;
            dp_sold = dp_sold_temp;
            dp_freeze = dp_freeze_temp;
        }

        int max_profit = std::max(dp_sold, dp_freeze);

        return max_profit;
    }
};
```

## 312. 戳气球 (Burst Balloons)

标签: 动态规划

解法:

- 我们考虑戳破范围 `[i, j)` 内的气球. 假设我们最后戳破的是第 `k` 个气球, 其中 `i <= k < j`, 那么无论我们以何种顺序戳破范围内的气球, 这一戳破气球的序列总是能够被分为 "戳破 `[i, k)` 范围内的气球", "戳破 `[k + 1, j)` 范围内的气球", 以及 "戳破第 `k` 个气球" 这三个部分, 由此可知 "戳破 `[i, j)` 范围内的气球所能得到的最大收益" 实际上恰好等于 "戳破 `[i, k)` 范围内的气球所能得到的最大收益" 加上 "戳破 `[k + 1, j)` 范围内的气球所能得到的最大收益" 再加上 "戳破第 k 个气球所能得到的收益". 于是我们便能够通过动态规划求出所有范围的最大收益并得到最终的答案.

代码:

```cpp
class Solution {
public:
    int maxCoins(const vector<int> &nums) {
        std::vector<std::vector<int>> dp(nums.size() + 1);

        for (auto &row : dp) {
            row.resize(nums.size() + 1);
        }

        for (int range_size = 1; range_size <= nums.size(); range_size++) {
            for (int range_begin = 0, range_end = range_begin + range_size;
                 range_end <= nums.size();
                 range_begin++, range_end++) {
                const int left_end_balloon =
                    range_begin == 0 ? 1 : nums[range_begin - 1];
                const int right_end_ballooon =
                    range_end == nums.size() ? 1 : nums[range_end];

                int current_max_number_of_coins = 0;

                for (int chosen_balloon_position = range_begin;
                     chosen_balloon_position < range_end;
                     chosen_balloon_position++) {
                    const int chosen_balloon = nums[chosen_balloon_position];
                    const int coins_got_by_popping_chosen_balloon_at_last =
                        left_end_balloon * chosen_balloon * right_end_ballooon;
                    const int total_coins_got =
                        dp[range_begin][chosen_balloon_position]
                        + dp[chosen_balloon_position + 1][range_end]
                        + coins_got_by_popping_chosen_balloon_at_last;

                    current_max_number_of_coins =
                        std::max(current_max_number_of_coins, total_coins_got);
                }

                dp[range_begin][range_end] = current_max_number_of_coins;
            }
        }

        return dp[0][nums.size()];
    }
};
```

## 322. 零钱兑换 (Coin Change)

标签: 动态规划

解法:

- 本题类似于 "279. 完全平方数 (Perfect Squares)", 由于零件的个数和数量级远小于背包的容量, 因此我们应当在主循环中遍历背包容量而不是遍历零件集合.

代码:

```cpp
class Solution {
public:
    int coinChange(vector<int> &coins, int amount) {
        std::sort(coins.begin(), coins.end());
        coins.erase(std::unique(coins.begin(), coins.end()), coins.end());

        std::vector<int> dp(amount + 1, std::numeric_limits<int>::max());
        dp[0] = 0;

        for (int current_amount = 1; current_amount <= amount;
             current_amount++) {
            int min_number_of_coins_needed_for_current_amount =
                std::numeric_limits<int>::max();

            for (auto current_coin : coins) {
                if (current_coin > current_amount) {
                    break;
                }

                const int number_of_coins_needed_for_reduced_amount =
                    dp[current_amount - current_coin];

                if (number_of_coins_needed_for_reduced_amount
                    < std::numeric_limits<int>::max()) {

                    min_number_of_coins_needed_for_current_amount = std::min(
                        min_number_of_coins_needed_for_current_amount,
                        1 + number_of_coins_needed_for_reduced_amount
                    );
                }
            }

            dp[current_amount] = min_number_of_coins_needed_for_current_amount;
        }

        return dp[amount] == std::numeric_limits<int>::max() ? -1 : dp[amount];
    }
};
```
