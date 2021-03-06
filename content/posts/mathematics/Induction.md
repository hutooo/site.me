---
title: 数学归纳法
author: ash
tags: ["数学归纳法", "Mathematics", "Induction"]
categories: ["数学", "Mathematics"]
date: 2021-07-01T14:16:05+08:00
cover: "/images/touhou01.jpg"
math: md
---

> 如何推倒一排多米诺骨牌?
>
> 只要排成一列，推倒其中一个，能顺次带倒下一个就行了.
>
> 还不够~ 还需要推倒第一个.

# 0x00 高斯求和的故事

大家一定都听说过高斯求和的故事, 据传在一个宁静的下午，课堂上的老师布置了一道算术题，要求学生计算从1加2一直加到100的结果(看得出来老师想摸鱼)，学生们都开始埋头计算~ (1+2=3, 1+2+3=3+3=6, 1+2+3+4...)

老师本以为可以愉快地摸鱼了，可是小高斯很快就算出了答案(老师的摸鱼计划被打乱了)~

让我们看看小高斯的是怎么做的~

首先 $1+2+3+4+...+100$ 的结果和 $100+99+98+...+2+1$ 的结果应该是相等的，那么可以将两串数字纵向相加

$$
{1+2+3+4+...99+100}\\\\{\underline{100+99+98+97+...+2+1}}\\\\{\underbrace{101+101+101+...+101+101}_{总共100 个 101}}
$$

这样就非常简单了，只需要计算 100 个 101 相加的结果,
$100 \times 101 = 10100$，但是10100是要求结果的2倍，所以还需要除以2, 最终答案是5050.

小高斯的方法绝妙非凡，有了这个方法，我们能计算的不仅仅是1到100的加法，我们可以求和更多的数.

# 0x01 归纳

小高斯运用了如下等式:

$$1+2+3+...+100=\frac{(100+1) \times 100}{2}$$

如果我们使用变量 `n`，将 1+2+...+100，替换为 1+2+...+n.
等式变为如下形式:

$$1+2+3+...+n=\frac{(n+1) \times n}{2}$$

以上等式对于0以上的任意整数都成立嘛? n为100, 100万呢?100亿呢? 如果成立，怎么证明呢~

# 0x02 数学归纳法

数学归纳法 是证明有关整数的断言，对于任意自然数是否成立时所用的方法.

数学归纳法需要经过以下两个步骤进行证明.

1. 基底(base): 证明 P(0) 成立.
2. 归纳(induction): 证明 对于任意自然数K ，若 P(k) 成立， 则 P(k+1) 也成立.

类比多米诺骨牌:

1. 确保让第一个多米诺骨牌倒下
2. 确保只要第K个多米诺骨牌倒下，那么第K+1个也会倒下


# 0x03 用数学归纳法证明高斯断言

断言G(n): 0到n的整数之和 与 $\frac{(n+1) \times n}{2}$ 相等.

## 步骤1: 基底的证明

证明G(0)成立.

G(0) 就是0到0的整数之和，0到0只有0一个整数，所以和也是0.

而 $\frac{(0+1) \times 0}{2}$ 也等于0.

步骤1完成.

## 步骤2: 归纳的证明

证明 K 为 0 以上的任意整数时 ，若 P(k) 成立， 则 P(k+1) 也成立.

假设G(k)成立，则 $P(k) = 0+1+...k = \frac{(k+1) \times k}{2}$ 成立.

要证明的等式 G(k+1) 则是 $0+1+...k+(k+1) = \frac{(k+1+1) \times (k+1)}{2}$

$$
\begin{split}
\underline{0+1+...k}+(k+1) &= p(k) + (k+1)\\\\
&=\frac{(k+1) \times k}{2} + (k+1)\\\\
&=\frac{(k+1) \times k}{2} + \frac{2k+2}{2}\\\\
&=\frac{k^2+k+2k+2}{2}\\\\
&=\frac{k^2+3k+2}{2}\\\\
&=\frac{(k+2) \times (k+1)}{2}\\\\
&=\frac{(k+1+1) \times (k+1)}{2}\\\\
\end{split}
$$

至此，由G(k)到G(k+1)推导成功，步骤2得到证明.

