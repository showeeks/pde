# 图算法

[TOC]

DFS可以用递归或者栈来实现，如果把栈替换为一个队列，那就是BFS。BFS 被用来解决最短路径问题，而 DFS 则要解决连通性问题。

迷宫可以看作是一种图，迷宫的交叉点是图的顶点，而通道则相当于是图的边。于是探索图的过程就是一个探索迷宫的过程。

## DFS

### 用DFS计算生成树

用DFS可以直接计算出一个生成树森林。

### 用DFS计算连通分支的元素数

这个问题本身解决很简单，运用一个小技巧，访问的节点超过V以后终止程序。这样复杂度可以降到 $O(V\times\log V)$

### 用DFS计算两点连通性

有一个问题是，DFS更快还是并查集更快。并查集是一个在线算法，我们可以在任何时候以近似常数时间查询连通性，DFS则需要预处理，但是可以以稳定的常数时间查询连通性。在图的ADT中我们一般用DFS，因为 DFS 利用了现有的基础设施。但是并查集和DFS都不能直接处理有大量边插入、删除、连通性查询的情况，这需要单独的DFS来处理。

### 用 DFS 解决二分图问题

是否可以将所有顶点染成两种颜色，使得所有边的头尾是同一种颜色？

### 给定两个顶点，有两条不同的路径连接它们吗？

这里有两个问题要解决：

1. 删掉某个顶点，两个顶点仍然可以连接
2. 删掉某个边，两个顶点仍然可以连接

桥的概念

如果一个图中有一条边，去掉这条边就会多出一个连通分支，那么这条边称为一个桥。没有桥的边称为边连通的。

如上的表述是定义图的一种保持连通的性质，事实上也可以定义为一种分割图的性质。将边连通图称为边可分图也是可以的。边连通分支即为没有桥的最大子图。

性质：v-w是一个桥当且仅当不存在w的后辈和w的祖先是邻居的情况。

计算桥这个算法我最不能理解的就是为什么 `low[w] = pre[v]`



articulation point指的是图的一个顶点，去掉这个顶点后图分割为两个连通分量。
