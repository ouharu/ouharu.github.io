---
title: 从paiza-A034到动态规划中的0-1背包问题
tags:
  - paiza
  - Python
  - algorithm
categories:
  - Work
category_bar: true
math: true
date: 2024-07-05 23:55:49
---

> https://paiza.jp/career/challenges/335/retry

# 题目

有N种商品，每种价格x_i，总预算X，要求：

・お菓子の総額を指定範囲内：商品总金额在X内
・種類の数を最大にする：种类最多
・同じお菓子は複数買わない：同类商品只能买一次
・お釣りを最小：找零最小
・お釣りを最小にすることよりも種類の数を最大にすることを優先する：种类最多大于找零最小

# 问题

这道题联想到：动态规划，0-1背包问题 https://www.hello-algo.com/chapter_dynamic_programming/knapsack_problem/#2

## 动态规划特征

- 动态规划是一种“从底至顶”的方法：从最小子问题的解（初始条件）开始，迭代地构建更大子问题的解，直至得到原问题的解。
- 动态规划的常用术语。
  - 将数组 dp 称为 dp 表，dp[i]表示状态i对应子问题的解。
  - 将最小子问题对应的状态称为**初始状态**。
  - 在动态规划问题中，当前状态往往仅与前面有限个状态有关,从前面状态得到当前状态的过程称为递推，将递推公式称为**状态转移方程**。

## 判断动态规划问题

问题是否适合使用回溯（穷举）解决，本题可以通过穷举所有商品价格的组合实现：
排列组合公式：

### 排列（Permutation）

排列是指从n个不同的元素中取出r个元素进行排列，排列的顺序是重要的。排列的公式是：
$$
P(n, r) = \frac{n!}{(n-r)!}
$$
其中，n! 表示n的阶乘，即从1乘到n的所有整数的积。

### 组合（Combination） 对应 0-1背包

组合是指从n个不同的元素中取出r个元素进行选择，组合的顺序是不重要的。组合的公式是：
$$
C(n, r) = \frac{n!}{r!(n-r)!}
$$

### 可重复抽取的组合（Combination with Repetition）对应 完全背包

在某些情况下，我们可以从n个元素中进行有重复的抽取，即每个元素可以被选择多次。可重复抽取的组合公式是：
$$
C(n+r-1, r) = \frac{(n+r-1)!}{r!(n-1)!}
$$


证明：把n种元素当成n个顺序摆放的盒子，r是r个完全相同的球，这样从n种元素中有重复取r个元素的方法就转化成，把r个同质球放入n个盒子的方法
✅为什么可以这样呢，想想，把一个球放到第i个盒子就相当于从n种元素中我们取的第i种元素，如果有多个球放在第i个盒子中，相当于从n个元素中重复了取了第i种元素。（https://blog.csdn.net/yishuihanq/article/details/106644983）
使用星号 (*) 来表示物品，使用竖线 (|) 来表示盒子的分隔。
组合方法：总共有 r 个星号和 n−1 个竖线。一共有r+n-1个位置，从中选出r个位置放星号（跟顺序无关）进行组合

```python
from itertools import combinations

def solve(N, X, prices):
    # 最大种类
    max_count = 0
    # 最小找零
    min_change = X
		# N 轮决策，每次决策用到的商品种类为i
    for i in range(1, N + 1):
        # 从价格列表中抽出i个价格，comb一共有C(len(prices),i)个情况
        for comb in combinations(prices, i):
            total = sum(comb)
            if total <= X:
                if i > max_count or (i == max_count and X - total < min_change):
                    max_count = i
                    min_change = X - total

    return min_change

# 读取输入
N, X = map(int, input().split())
prices = [int(input()) for _ in range(N)]

# 计算并输出结果
result = solve(N, X, prices)
print(result)
```



1. 这种问题可以使用树形结构来描述，其中每一个节点代表一个决策，每一条路径代表一个决策序列。

本题为例，
第一个决策，是否买编号1的商品，不买，种类0；买，种类1。
第二个决策，是否买编号2的商品，不买，种类0/1；买，种类1/2。
第三个决策，是否买编号3的商品，不买，种类0/1/1/2；买，种类1/2/2/3。

2. 问题为找出最优解

满足最大种类找零最小

2. 种类状态用列表表示，状态间存在递推关系

本题符合递推关系

# 思路

## 1.建立dp表

建立二维dp[i] [total]列表， i 表示商品编号为i，一共N种商品需要考虑；total 表示累计金额，在X范围内；
dp[i] [total]表示前i个商品决策后累计购买为total时的最大种类。

## 2.确定转移状态方程

0-1背包时：种类由前i-1个商品购买状态（矩阵中上一行）决定

第i个商品没买，dp[i] [total]状态与前i-1次dp[i-1] [total]相同；
第i个商品买，dp[i] [total]状态等于dp[i-1] [total-pi]+1相同，pi为商品i价格；
转移方程：dp[i] [total] = max(dp[i-1] [total],dp[i-1] [total-pi]+1)。注意pi大于total时dp[i] [total] = dp[i-1] [total]

-------
✅完全背包时：不买商品i时种类由前i-1个商品购买状态（矩阵中上一行）或者买商品i时状态（矩阵中当前行左边元素） 决定

第i个商品没买，dp[i] [total]状态与前i-1次dp[i-1] [total]相同；
第i个商品买，dp[i] [total]状态等于dp[i] [total-pi]+1相同，pi为商品i价格；
转移方程：dp[i] [total] = max(dp[i-1] [total],dp[i] [total-pi]+1)。注意pi大于total时dp[i] [total] = dp[i-1] [total]

状态转移时参考的矩阵元素不同，也造成了dp降维时的遍历方向不同。参考同行元素，完全背包正向遍历；参考上行元素，01背包反向遍历。

## 3.确定边界条件

~~dp[0] [total] 为0，表示0个商品，种类为0；dp[i] [0] 为0，表示累计total为0，种类为0。~~   未选择商品而有金额这一点是非正常状态，应该设置为-1，同样选择商品后金额为0（除非所有商品为0）也是非正常状态，设置为-1

初始条件仅设置0,0 避免提前赋值造成dp流向改变

一共N商品，X限额，dp大小为N+1，X+1。

# 原因

组合式穷举，时间复杂度为O(2^N)；动态规划2层遍历O(N*X)。
再结合动态规划的条件：决策树的形式，求最优化值，确定用动态规划方法求解。

# 解决

## 2维 

```python
def solve(N, X, prices):
    # 创建二维DP表，dp[i][j]表示考虑前i个物品，总价值为j时的最大物品数
    # -1 代表一个"不可能"或"未达到"的状态。
    dp = [[-1] * (X + 1) for _ in range(N + 1)]
    # dp[0][0] 表示在没有选择任何物品（0个物品）且总价值为0的情况下物品数为0。
    # 初始化错误将导致错误的状态转移
    dp[0][0] = 0

    for i in range(1, N + 1):
        for j in range(X + 1):
            # 选第i个物品
            if j >= prices[i - 1] and dp[i - 1][j - prices[i - 1]] != -1:
                dp[i][j] = max(dp[i-1][j], dp[i - 1][j - prices[i - 1]] + 1)
            # 不选第i个物品
            else:
            		dp[i][j] = dp[i - 1][j]

    # 找到最大物品数
    max_items = max(dp[N])

    # 在最大物品数的情况下，找到最大总价值
    max_value = 0
    for j in range(X, -1, -1):
        if dp[N][j] == max_items:
            max_value = j
            break

    return X - max_value


# 读取输入
N, X = map(int, input().split())
prices = [int(input()) for _ in range(N)]

# 计算并输出结果
result = solve(N, X, prices)
print(result)
```



## 1维

```python
def solve(N, X, prices):
    # 初始化一维DP数组
    dp = [-1] * (X + 1)
    dp[0] = 0  # 基础情况：不选择任何物品，总价值为0

    for i in range(N):
        # 从后向前更新，避免覆盖还需要使用的数据
        for j in range(X, prices[i] - 1, -1):
          	# 保证只有price及其和被赋值
            if dp[j - prices[i]] != -1:
                dp[j] = max(dp[j], dp[j - prices[i]] + 1)

    # 找到最大物品数
    max_items = max(dp)

    # 在最大物品数的情况下，找到最大总价值
    max_value = 0
    for j in range(X, -1, -1):
        if dp[j] == max_items:
            max_value = j
            break

    return X - max_value

# 读取输入
N, X = map(int, input().split())
prices = [int(input()) for _ in range(N)]

# 计算并输出结果
result = solve(N, X, prices)
print(result)
```



---

完
