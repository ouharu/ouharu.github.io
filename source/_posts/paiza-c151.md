---
title:  paiza C151:四舍五入引发的问题
tags:
  - 编程
  - paiza
  - Python
categories:
  - Work
date: 2024-06-28 23:16:33
---

> 题目：https://paiza.jp/career/challenges/745/retry

# 题目
原题要求对给出的正整数和位数，进行四舍五入计算：

```text
入力例1
583271 // 正整数
3      // 位数
出力例1
583000 
```

本例中因为从右往左数第3位为2，则四舍五入后结果后3位舍去。

## 💭思路

由此想到可以对正整数除以10^(位数)的商进行四舍五入后再乘回去得到结果

```python
N = int(input())
R = int(input())

Rouding = N / 10**R             # 取商
Rouding2 = round(Rouding)       # 四舍五入
res = Rouding2*10**R            # 回到原来位数

print(res)
```

## 🚫问题

```text
コード実行結果：☓不正解
提出前動作確認の入力値
123450000
5

期待する出力
123500000

あなたのコードの出力
123400000
```

莫名奇妙地遇到5没有进位？

## 🛠原因

经过查阅资料发现Python等其他编程语言中存在浮点数表示不准确问题`1234.5`实际存储为`1234.4999999`，在进行`round()`处理时就没有进位。

浮点数在计算机中的存储问题，分析详见：[https://www.cnblogs.com/ddzj01/p/17319315.html](https://www.cnblogs.com/ddzj01/p/17319315.html)


## ✅解决

四舍五入在编程语言中实现时可采用`floor(num+0.5)`对小数点后1位浮点数num实现四舍五入的取整。
分析：`floor()`为向下取整，假设`num=4.7`,`floor(4.7+0.5)=5`符合四舍五入结果。

进行拓展，正整数N，位数R，N/10^R代表num，则N对于位数R的四舍五入表示为`floor(N/10^R+0.5)*10^R`...1。继续简化，`floor(a/b)==a//b`，所以`floor(N/10^R+0.5)==(N+10^R//2)//10^R`(⚠注意为了保证最终结果为整数,//运算被除数必须为整数，0.5*10^R表示为10^R//2)，最终可简化为`(N+10^R//2)//10^R*10^R`...1

代码如下：

```python
# 读取输入
N = int(input())
R = int(input())

# 计算四舍五入的因子
rounding_factor = 10 ** R

# 进行四舍五入
# 通过加上0.5 * rounding_factor来实现四舍五入
res = (N + rounding_factor // 2) // rounding_factor * rounding_factor

# 输出结果
print(res)
```
---
完