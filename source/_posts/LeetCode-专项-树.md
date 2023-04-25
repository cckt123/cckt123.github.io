---
title: LeetCode-专项 树
date: 2022-02-15 17:05:52
tags: [ 算法, 树, LeetCode]
categories: [ 算法]
about:
description: "看书看吐了，刷刷简单题缓解一下。"
---

####   [94. 二叉树的中序遍历](https://leetcode-cn.com/problems/binary-tree-inorder-traversal/)

~~//这简单题居然有三种方法...算了，回头再说吧..~~

这里是第一种方法，递归法。

题目要求的中序遍历，即左根右的顺序检索二叉树，每次InorderTraversal的过程就是强行模拟了这个判断方式。

``` csharp
public class Solution {
    private List<int> Result = new List<int>();
    public IList<int> InorderTraversal(TreeNode root) {
        if(root==null)return Result;
        InorderTraversal(root.left);
        Result.Add(root.val);
        InorderTraversal(root.right);
        return Result;
    }
}
```

第二种方法，迭代法。

迭代法的本质和上一种递归法其实并无太大差别。上面的递归法在每次递归的同时实质上在将原先的函数压到操作系统提供的栈当中，而迭代法在这里将这个栈显示的声明了出来，进行了模拟。两者并无明显的优劣划分。

```csharp
public class Solution {
    public IList<int> InorderTraversal(TreeNode root) {
        List<int> list = new List<int>();
        Stack<TreeNode> stack = new Stack<TreeNode>();
        while(root!=null || stack.Count != 0)
        {
            while(root!=null)
            {
                stack.Push(root);
                root = root.left;
            }
            root = stack.Pop();
            list.Add(root.val);
            root = root.right;
        }
        return list;
    }
}
```

第三种 Morris 中序遍历

相较于上述两种方式，这种方式的优点在于其空间复杂度为o(1)。将二叉树视为链表处理，利用右节点模拟链表结构，但问题在于他会破坏树本身的结构，我感觉除了面试阶段之外其他情况不宜使用，参考学习的意义大于实际价值。

``` csharp
using System;

public class Solution {
    	public List<int> Result = new List<int>();
        private TreeNode predecessor;
        public IList<int> InorderTraversal(TreeNode root) 
        {
        while(root!=null)
        {
            if(root.left!=null)
            {
                predecessor = root.left;
                while(predecessor.right!=null&&predecessor.right!=root)
                {
                    predecessor = predecessor.right;
                }
                
                if(predecessor.right == null)
                {
                    predecessor.right = root;
                    root = root.left;
                }
                else
                {
                    Result.Add(root.val);
                    predecessor.right = null;
                    root = root.right;
                }
            }
            else
            {
                Result.Add(root.val);
                root = root.right;
            }
        }
        return Result;
        }
}
```

参考资料

+ [二叉树中序遍历](https://leetcode-cn.com/problems/binary-tree-inorder-traversal/solution/er-cha-shu-zhong-xu-bian-li-by-jin-li-er-qnac/)
+ [二叉树中序遍历](https://leetcode-cn.com/problems/binary-tree-inorder-traversal/solution/er-cha-shu-de-zhong-xu-bian-li-by-leetcode-solutio/)

#### [606. 根据二叉树创建字符串](https://leetcode-cn.com/problems/construct-string-from-binary-tree/)

``` csharp
public class Solution {
    public string Tree2str(TreeNode root) {
        if(root==null)return null;
        StringBuilder Result = new StringBuilder();
        Result.Append(root.val.ToString());
        if(root.left!=null)Result.Append("("+Tree2str(root.left)+")");
        if(root.left==null&&root.right!=null)Result.Append("()");
        if(root.right!=null)Result.Append("("+Tree2str(root.right)+")");
        return Result.ToString();
    }
}
```

递归存储中间结果，相较于94的中序遍历题目有相似之处。这种方式是通过递归来模拟栈的结构，实际上也会造成和上述一样的问题。

#### [95. 不同的二叉搜索树 II](https://leetcode-cn.com/problems/unique-binary-search-trees-ii/)
