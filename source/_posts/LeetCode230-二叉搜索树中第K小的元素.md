---
title: LeetCode230 二叉搜索树中第K小的元素
date: 2021-10-17 14:25:36
tags: [ 算法, 树, LeetCode]
categories: [ 算法]
about:
---
[230 二叉搜索树中第K小的元素](https://leetcode-cn.com/problems/kth-smallest-element-in-a-bst/)

+ ## 思路1：DFS+大根堆/优先队列
``` C++
class Solution {
public:
    int kthSmallest(TreeNode* root, int k) {
        //优先队列走起
        priority_queue<int,vector<int>,less<int>> Queue;
        stack<TreeNode*> NodeStack;
        NodeStack.push(root);
        TreeNode* Now = root;
        while(!NodeStack.empty())
        {
            Now = NodeStack.top();
            NodeStack.pop();
            if(Now->left!=nullptr)NodeStack.push(Now->left);
            if(Now->right!=nullptr)NodeStack.push(Now->right);
            if(Queue.size()<k)
            {
                Queue.push(Now->val);
                continue;
            }
            if(Queue.top()>Now->val)
            {
                Queue.pop();
                Queue.push(Now->val);
            }
        }
        return Queue.top();
    }
};
```
第一种方法如上，使用栈来深度优先搜索，遍历树结构，大根堆查找第K大的值。 

时间复杂度O(nlogk)，空间复杂度O(n+k)。

这种方案可以运行，但是却并不是最好的方案，因为在使用的时候忽略了一个重要的前置条件——二叉搜索树。


+ ## 思路2：中序遍历 

重新复习一下中序遍历

+ 先序为根左右 
+ 中序为左根右 
+ 后序为左右根 

而二叉搜索树具有以下性质
+ 节点的左子树为小于当前节点的数
+ 节点的右子树为大于当前节点的数
+ 所有左子树和右子树也是二叉搜索树

因而，左根右的顺序遍历二叉搜索树，可以得到一个漂亮的递增序列，而第k个值恰巧就是答案。

``` C++
class Solution {
public:
    int kthSmallest(TreeNode* root, int k) {
        stack<TreeNode *> stack;
        while (root != nullptr || stack.size() > 0) {
            while (root != nullptr) {
                stack.push(root);
                root = root->left;
            }
            root = stack.top();
            stack.pop();
            --k;
            if (k == 0) {
                break;
            }
            root = root->right;
        }
        return root->val;
    }
};
```