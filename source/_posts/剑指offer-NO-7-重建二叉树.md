---
title: 剑指offer NO.7 重建二叉树
permalink: 剑指offer NO.7 重建二叉树
date: 2019-12-28 20:30:02
tags:
categories:
---

# 重建二叉树

[牛客网-重建二叉树]("https://www.nowcoder.com/practice/8a19cbe657394eeaac2f6ea9b0f6fcf6?tpId=13&tqId=11157&tPage=1&rp=1&ru=/ta/coding-interviews&qru=/ta/coding-interviews/question-ranking")

### 题目描述

根据二叉数的前序和中序遍历构建二叉树，二叉树的前序遍历和中序遍历结果，以int数组的形式输入

### 解题思路

根据树的前序和中序遍历规律，前序遍历的第一个结点是二叉树的根节点，在中序遍历中以根节点划分，左边就是二叉树的左子树，右边是右子树，按这个规律进行递归，代码如下：

```java
/**
 * Definition for binary tree
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
import java.util.*;
public class Solution {
    private Map<Integer,Integer> map = new HashMap<>();
    public TreeNode reConstructBinaryTree(int [] pre,int [] in) {
        for(int i = 0;i < in.length;i++){
            map.put(in[i],i);
        }
        return construct(pre,0,pre.length - 1,0);
    }
    
    private TreeNode construct(int[] pre,int l,int r,int inl){
        if(l > r){
            return null;
        }
        TreeNode root = new TreeNode(pre[l]);
        int index = map.get(root.val);
        int lsize = index - inl;
        root.left = construct(pre,l + 1,l + lsize,inl);
        root.right = construct(pre,l + lsize + 1,r,inl + lsize + 1);
        return root;
    }
}
```



