---
title: 数据结构 -- 树
categories: 数据结构与算法
comments: true
tags: [数据结构]
description: 介绍树的基础知识
date: 2014-7-5 10:00:00
---

https://www.imooc.com/article/17707

https://www.cnblogs.com/songwenjie/p/8878851.html

https://blog.csdn.net/qq_28171461/article/details/78855987

二叉平衡树和红黑树

https://www.sohu.com/a/201923614_466939

## 什么是树

树是我们在数据结构中经常遇到到，而且是面试必备知识。通常我们为树做如下定义：

 - 是有 n (n>=0) 的有限集，n 为 0 时称为空树。
 - 非空树有且仅有一个称为根节点的节点。
 - 非空树可以分为 m 个互不相交的有限集，其中每个集合本首又是一颗树，并且称为根节点的子树。

下图符合树的定义：

![效果图](/images/data-structure-basic-tree/tree.png)

下图不符合树的定义，违背了子树互不相交的原则。

![效果图](/images/data-structure-basic-tree/not-tree.png)

## 相关术语

 - 节点的度：一个节点含有的子树的个数称为该节点的度
 - 树的度：一棵树中，所有节点中的度的最大值称为树的度
 - 叶节点或终端节点：度为0的节点称为叶节点
 - 节点的层次：从根开始定义起，根为第1层，根的子节点为第2层，以此类推
 - 树的高度或深度：树中节点的最大层次

## 二叉树

二叉树（Binary Tree）的特点：

 - 每个节点最多有两棵子树，也即是二叉树中不存在度大于2的节点
 - 两棵子树是有顺序的，为左子树和右子树。即使有一棵子树，也要区分是左子树还是右子树

### 完全二叉树

完全二叉树特点：

 - 叶子节点只能出现在最下面两层。
 - 最下层叶子一定集中在左部连续区域
 - 如果节点度为1，那么该节点只有左子树，不存在只有右子树的情况
 - 将完全二叉树进行从上到下，从左到右编号。我们发现如果完全二叉树的一个父结点编号为k，那么它左儿子的编号就是2*k（如果存在），右儿子的编号就是2*k+1（如果存在）。如果已知儿子（左儿子或右儿子）的编号是x，那么它父结点的编号就是x/2。知道了这个规律，我们完全可以用一个一维数组来存储完全二叉树。

![效果图](/images/data-structure-basic-tree/complete-binary-tree.png)


https://blog.csdn.net/qq_41117236/article/details/81029618

### 满二叉树

满二叉树可以看成是一种特殊的完全二叉树。
满二叉树特点：

 - 所有的分支节点都存在左子树和右子树
 - 叶子节点只能出现在最下面一层
 - 非叶子节点的度一定是2。即要么为叶子节点，要么同时具有左右孩子节点。
 - 同深度的二叉树中，满二叉树节点数量最多，叶子节点数最多

![效果图](/images/data-structure-basic-tree/full-binary-tree.png)

### 二叉排序树

二叉排序树（Binary Sort Tree），又称二叉查找树（Binary Search Tree），亦称二叉搜索树。
二叉排序树或者是一棵空树，或者是具有下列性质的二叉树：

 - 若左子树不空，则左子树上所有结点的值均小于它的根结点的值
 - 若右子树不空，则右子树上所有结点的值均大于它的根结点的值
 - 左、右子树也分别为二叉排序树
 - 没有键值相等的节点

![效果图](/images/data-structure-basic-tree/binary-sort-tree.png)

### 平衡二叉树

平衡二叉树是二叉排序树的一种，又叫平衡二叉排序树，或AVL树，它具有下面特点：

 - 它是一棵空树或它的左右两个子树的高度差的绝对值不超过1
 - 左、右两个子树都是一棵平衡二叉树

![效果图](/images/data-structure-basic-tree/banlanced-binary-tree.png)

### 二叉堆



## 二叉树的遍历

二叉树最基本的操作是遍历，一般我们约定遍历时左节点优先于右节点，这样二叉树的遍历一般我们分为三种：

 - 前序遍历：先遍历根节点，再处理左右节点
 - 中序遍历：先遍历左节点，然后处理根节点，最后处理右节点
 - 后序遍历：先遍历左右节点，最后处理根节点

![效果图](https://img-blog.csdn.net/20170803221343901)

针对上图做遍历：
先序遍历的结果为：0  1  3  7  4  2  5  6
中序遍历的结果为：7  3  1  4  0  5  2  6
后序遍历的结果为：7  3  4  1  5  6  2  0

还可以分为：

 - 广度优先遍历：
 - 深度优先遍历：

代码实现：

```
    {
        TreeNode node = createTree();
        preOrderTraverse(node);
    }

    public class TreeNode{
        int data;
        TreeNode leftNode;
        TreeNode rightNode;
        public TreeNode(){
        }
        public TreeNode(int data){
            this.data = data;
            this.leftNode = null;
            this.rightNode = null;
        }
        public TreeNode(int data,TreeNode leftNode,TreeNode rightNode){
            this.data = data;
            this.leftNode = leftNode;
            this.rightNode = rightNode;
        }
    }

    //建立一个完全二叉树
    public TreeNode createTree(){
        int data[] = {1,2,0,0,3,4,5,0,6,7,8,0,0,9};
        ArrayList<TreeNode> list = new ArrayList();
        for(int i = 0; i < data.length; i++) {
            list.add(new TreeNode(data[i]));
        }

        TreeNode root = list.get(0);
        for(int i = 0; i<data.length/2; i++){
            if (2*i+1 < data.length)
                list.get(i).leftNode = list.get(2*i+1);
            if (2*i+2 < data.length)
                list.get(i).rightNode = list.get(2*i+2);
        }
        return root;
    }

    // 前序遍历
    public void preOrderTraverse(TreeNode node) {
        if (node == null) {
            return;
        }

        Log.e("Test", ""+node.data);
        preOrderTraverse(node.leftNode);
        preOrderTraverse(node.rightNode);
    }

    // 中序遍历
    public void inOrderTraverse(TreeNode node) {
        if (node == null) {
            return;
        }

        inOrderTraverse(node.leftNode);
        Log.e("Test", ""+node.data);
        inOrderTraverse(node.rightNode);
    }
    
    // 后序遍历
    public void postOrderTraverse(TreeNode node) {
        if (node == null) {
            return;
        }

        postOrderTraverse(node.leftNode);
        postOrderTraverse(node.rightNode);
        Log.e("Test", ""+node.data);
    }
```
