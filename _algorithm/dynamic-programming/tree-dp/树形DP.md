---
title: "树形DP（Tree DP）"
date: 2023-11-06
categories: [算法, 动态规划]
tags: [dp, 动态规划, 树形dp, tree-dp, 树, 图论]
---
# 树型DP

## 题二十（依赖背包问题）

![题二十-1](/algorithm/dynamic-programming/tree-dp/pic/Question1-1.png)   
![题二十-2](/algorithm/dynamic-programming/tree-dp/pic/Question1-2.png)

y总分析法：

一、状态表示

1. 集合：dp[i][j] 表示考虑以i 为根节点的子树中选，且体积不超过j 的方案
2. 属性：Max

二、状态计算

![20-1](/algorithm/dynamic-programming/tree-dp/pic/1-1.png)