---
title: B152:選べる日替わり弁当
tags:
  - paiza
  - Python
categories:
  - Diary
category_bar: true
math: true
date: 2024-09-27 17:25:56
---

# 问题描述

输入内容：

N X
v_1 v_2 ... v_N
a_1 a_2 ... a_N
・ N 整数，表示菜品种类
・ X 整数，目标总价
・ v_i 整数，数组price[i]表示菜品i的价格
・ a_i 整数，数组satisfication[i]表示菜品i的満足度
・ 入力は合計で 3 行となり、末尾に改行が 1 つ入ります

输出内容：在N种菜品中选择1-3种，总价为X时的最大满意度

# 解题思路

## ❌双指针、哈希表

无法应对本题出现的重复price对应不同满足度的情况。

## 动态规划

```python

def max_satisfaction(N, X, prices, satisfactions):
    # dp[i][j][k] 表示从前i个菜品中选择k个，总价为j时的最大满意度
    dp = [[[-float('inf')] * 4 for _ in range(X + 1)] for _ in range(N + 1)]
    
    # 初始化：不选择任何菜品时，满意度为0
    dp[0][0][0] = 0

    for i in range(1, N + 1):
        for j in range(X + 1):
            for k in range(4):  # k表示已选择的菜品数量，范围是0到3
                # 不选择当前菜品
                dp[i][j][k] = dp[i-1][j][k]
                
                # 选择当前菜品（如果价格允许且已选菜品数量小于3）
                if k > 0 and j >= prices[i-1]:
                    dp[i][j][k] = max(dp[i][j][k], dp[i-1][j-prices[i-1]][k-1] + satisfactions[i-1])

    # 返回选择1到3个菜品，总价恰好为X时的最大满意度
    max_satisfaction = max(dp[N][X][1], dp[N][X][2], dp[N][X][3])
    return max_satisfaction if max_satisfaction > -float('inf') else None

# 读取输入
N, X = map(int, input().split())
prices = list(map(int, input().split()))
satisfactions = list(map(int, input().split()))

# 计算并输出结果
result = max_satisfaction(N, X, prices, satisfactions)
print(result if result is not None else "No solution")
```
