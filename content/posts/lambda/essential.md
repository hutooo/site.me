---
title: Essential of Lambda-Calculus
author: ash
tags: ["Lambda", "λ", "λ-演算"]
categories: ["Lambda奥秘"]
date: 2020-11-18T17:10:10+08:00
image: miku.jpg
---

> 法国作家、冒险家、艺术家和航空工程师 **安东尼·德·圣埃克苏佩里** (Antoine de Saint-Exupéry) 在论飞机设计时说：
> “La perfection est atteinte non quand il ne reste rien à ajouter，mais quand il ne reste rien à enlever!”
> **完美之道，不在无可增加，而在无可删减!**

## 0x01 什么是 lambda-calculus

**λ-演算**(lambda/λ-calculus)是一套从数学逻辑中发展，以变量绑定和替换的规则，来研究函数如何抽象化定义、函数如何被应用以及递归的形式系统.

```s
20世纪30年代，一个名叫阿隆佐-邱奇的数学家,首次发表了Lambda演算， 从而解决了可计算理论中的判定性问题.
lambda演算作为一种广泛用途的计算模型，可以清晰地定义什么是一个可计算函数，而任何可计算函数都能以这种形式表达和求值.
𝜆-演算 分为 `类型化𝜆-演算(Typed λ-Calculus)` 和 `无类型𝜆-演算(Type-Free λ-Calculus)` 也称为 `朴素λ-演算(Naive λ-Calculs)`.
```

lambda演算简单易读写，语义强大同时图灵完备, 后续内容主要针对是无类型的λ-演算展开讲解.

## 0x02 一切皆函数

大家都学过不少编程语言, C++、Java、Go、Python等... 它们大部分都有着丰富的语法特性，很多特性可以互相替换。
如果我们遵循安东尼的完美之道，去裁剪语法，那么最小化的语言是什么样的呢~

lambda演算可比拟最根本的编程语言, 它的内核非常小,可以用以下规则来描述:

|语法/L-exp|名称|描述|
|--:|---|---|
|a    |变量/原子     |标识符引用就是一个名字，这个名字用于匹配函数表达式中的某个参数名|
|λx.M |抽象化/抽象规则|函数定义, 变量x 在 M 中被绑定|
|(F N)|应用/应用规则  |将函数 F 应用于参数 N |

上面表格中看着比较抽象，试着写得简明一点..

```s
<identifier> --> a | b | ...   // 标识符
<abstraction> --> λ<identifier>.<λ-exp>  // 抽象规则
<application> --> (<λ-exp>)<identifier>  // 应用规则
<λ-exp> --> <identifier> | <abstraction> | <application>  // 所有这些都是 λ表达式
```

## 0x03 柯里化

观察上面的定义，你会发现一个 Lambda表达式 只接受一个参数, 这似乎是一个很大的局限. (嘛~ 其实我们平时可以写多个参数的λ-exp, 更简洁一点)

比如，怎样才能在只有一个参数的情况下实现加法呢？

当然没有问题~，因为函数也是值嘛. 所以单参数函数可以返回另一个单参数的函数，这样就可以实现两个参数相加的函数了. 本质上二者一致.

```s
这就是所谓的柯里化(Currying)，以伟大的逻辑学家 Haskell Curry 命名.
```

## 0x04 自由与约束

自由变量 是那些在lambda抽象中不受到绑定的变量..

啥意思呢? 继续看..

```s
表达式 [lambda x . x] 中的lambda项没有自由变量. 
表达式 [lambda x . y x] 中的lambda项，有一个自由变量 [y].
```

这就引出了一个 lambda表达式 的重要语法：闭包(closure)或者叫完全绑定(complete-binding). (接触过一些函数式编程技术的小伙伴一定听说过闭包吧~), 在对一个Lambda表达式进行求值的时候，不能引用任何未绑定的标识符.

```s
表达式 [lambda y . (lambda x . plus x y)]

在内层演算 [lambda x . plus x y] 中，[y] 和 [plus] 是自由的，[x]是绑定的, 
而在完整的表达中，[x] 和 [y]是绑定的: [x]受内层绑定，而[y]由剩下的演算绑定. [plus]仍然是自由变量.
```

## 0x05 转换与规约

lambda表达式的求值过程中，需要用到的一些操作.

|操作|名称|描述|
|---|---|---|
|(λx.M[x]) -> (λy.M[y])  |α-转换 |重命名表达式中的绑定(形式)变量, 避免名称冲突|
|((λx.M) E) -> (M[x:=E]) |β-归约 |在抽象化的函数定义体中，以参数表达式代替绑定变量|

举个栗子.

```s
// 先来看看 alpha-转换
lambda x . x  --- alpha-转换 --->  lambda z . z

// 再来..
lambda f . (lambda x . f x)   --- alpha-转换 --->  lambda f . (lambda y . f y)

// 再瞧瞧 beta-规约
(lambda x . f x) 666  --- beta-规约 ---> f 666

// 再来..
(lambda n . (lambda x . n f x)) 666  --- beta-规约 ---> lambda x . 666 f x
```

其实lambda还有一种变换操作, 叫做 `η-变换(Eta-变换)`..

```s
Eta-变换 表达的是外延性的概念，在这里外延性指的是，对于任一给定的参数，当且仅当两个函数得到的结果都一致，则它们将被视同为一个函数.
Eta-变换可以令 [lambda x . f x] 和 [f] 相互转换，只要 [x] 不是 [f] 中的自由变量.
```

## 0x06 能不能通俗点?

那就来点小栗子吧~

```s
lambda x . x  // 这个表达式足够简单吧， 入啥出啥， 那应用一下

(lambda x . x) 1 = 1
(lambda x . x) lambda z . z = lambda z . z
```

看看下一个..

```s
lambda x . (lambda y . plus x y)  // 假定 plus 已经实现了数字相加，具体实现请看下回分解吧~

// 应用一下

(lambda x . (lambda y . plus x y)) 2 3
    = (lambda y . plus 2 y) 3  // Beta规约
    = plus 2 3 // Beta-规约
    = 5 
```

## 0x07 参考

* [wiki - λ演算](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)
* [blog - Good Math/Bad Math](http://goodmath.blogspot.com/)
* [wiki - 阿隆佐·邱奇](https://zh.wikipedia.org/wiki/%E9%98%BF%E9%9A%86%E4%BD%90%C2%B7%E9%82%B1%E5%A5%87)
* [blog - Keep Coding](https://liujiacai.net/blog/2014/10/12/lambda-calculus-introduction/)