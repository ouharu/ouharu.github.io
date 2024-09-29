---
title: paiza-b141
tags:
  - python
  - paiza
categories:
  - Work
category_bar: true
math: true
date: 2024-09-30 03:39:11
---

# 问题描述

输入内容：

N
C_1
C_2
...
C_{2N}

・ N 整数，表示花的颜色种类
・ C_i为1-N之间的数且各有两个 (1 ≦ i ≦ 2N)

输出内容：排序C_i为降序数组*2所需的最小交换次数（交换发生在相邻两个元素之间）

# 解题思路

## 贪心

算法思路：
a. 创建目标序列：
我们首先创建目标序列 [1,2,3,...,N,1,2,3,...,N]。
这代表了我们想要达到的最终排列。
b. 遍历目标序列：
我们从左到右遍历目标序列。
对于每个位置，我们检查当前的花是否已经是正确的颜色。
c. 处理不正确的位置：
如果当前位置的花不是正确的颜色，我们需要找到正确的花并将其移动到这个位置。
我们在当前位置之后的序列中寻找正确颜色的花。
d. 移动花到正确位置：
一旦找到正确颜色的花，我们通过一系列相邻交换将其移动到当前位置。
每次交换都会增加交换次数计数器。
e. 重复过程：
我们继续这个过程，直到处理完所有位置。

```python
def min_operations(N, colors):
    # 构建目标序列
    target = [i for i in range(1, N + 1)] * 2
    swaps = 0

    # 遍历目标序列，进行相邻交换
    for i in range(2 * N):
        if colors[i] != target[i]:
            # 找到目标颜色在当前序列中的下一个正确位置
            for j in range(i + 1, 2 * N):
                if colors[j] == target[i]:
                    # 执行相邻交换直到目标颜色到达正确位置
                    for k in range(j, i, -1):
                        colors[k], colors[k - 1] = colors[k - 1], colors[k]
                        swaps += 1
                    break
    return swaps

# 读取输入
N = int(input())
colors = [int(input()) for _ in range(2*N)]

# 计算并输出结果
print(min_operations(N, colors))
```
