---
title: Number of Lambda-Calculus
author: ash
tags: ["Lambda", "λ", "λ-演算"]
categories: ["Lambda奥秘"]
date: 2020-11-19T11:40:10+08:00
---

> 万物皆数，数是宇宙的根本，找到数就找到了宇宙的本原. --------- 毕达哥拉斯

## 0x01 函数化的数字

```s
1   //  这是什么数? |  数字 1 呀
2   // 那这个呢？  |  数字 2 呗
1 + 2 // 这个的结果呢  |  数字 3 吧
```

小伙伴(😡): ???当我三岁小孩呢~ 

上面的示例中, 使用了数字和算术运算. 但数字并不真正存在于lambda演算中，我们有的只有函数! 我们得想个法子用函数来表示数字. 那么往下看..

```s
lambda f x . x    // 那么这是啥呢~ , 我们假设它就是 0
lambda f x . f x  // 假定这个就是 1
lambda f x . f (f x)  // 假定这个就是 2 , 
lambda f x . f (f (f x))   // 那么 3 呢，
lambda f x . f (f (f (f (f x))))   // 5呢？ 看出啥规律没鸭?
```

我们简单地用  -> 函数 `f` 运用到 `x` (及其后继) 上的次数  来表示数字.


## 0x02 算术运算

现在我们有了数字，那么我想实现 `2 + 3` 该怎么做呢？ 看看lambda表达式是怎么做加法的..

```s
lambda m n . (lambda f x . m f (n f x))   // 这是什么？ |  两数相加的 lambda表达式.. 

let add = lambda m n . (lambda f x . m f (n f x)) // 来点语法糖， 巴啦啦能量变！！
```

看着有点费解哎~ 我们应用一下试试， (没忘记alpha转换和beta规约吧..)

```s
add 2 3
    = (lambda m n . (lambda f x . m f (n f x))) 2 3 // 展开
    = lambda f x . 2 f (3 f x) // beta规约
    = lambda f x . (lambda s2 z2 . s2 (s2 z2)) f (3 f x) // 看看上面lambda演算中的数字2，alpha转换一下, 回避名称冲突
    = lambda f x . f (f (3 f x)) // beta规约
    = lambda f x . f (f ((lambda s3 z3 . s3 (s3 (s3 z3))) f x)) // 展开 3
    = lambda f x . f (f (f (f (f x)))) // beta规约 仔细看看这是啥. 5?! 
```

嘿!~ 还挺神奇哈!!

```s
// 其它的一些算术运算
succ = lambda n . (lambda f . (lambda x . f (n f x)))
plus = lambda m . (lamnda n . m succ n)
add  = lambda m n . (lambda f x . m f (n f x))
mul  = lambda m n . (lambda f . m (n f))
pow  = lambda b . lambda e . e b
pred = lambda n . (lambda f . (lambda x . n (lambda g . (lambda h . h (g f))) (lambda u . x) (lambda u . u)))
sub  = lambda m . (lambda n .n pred m)
zero? = lambda n . n (TRUE FALSE) TRUE  // TRUE FALSE 的定义 参考下一节
```

```s
mul 2 3
    = (lambda m n . (lambda f . m (n f))) 2 3
    = lambda f . 2 (3 f)
    = lambda f . 2 (lambda s z . s (s (s z))) f
    = lambda f . 2 (lambda s . (lambda z . s (s (s z)))) f
    = lambda f . 2 (lambda z . f (f (f z)))
    = lambda f . (lambda s2 . (lambda z2 . (s2 (s2 z2)))) (lambda z . f (f (f z)))
    = lambda f . (lambda z2 . (lambda z . f (f (f z))) ((lambda z . f (f (f z))) z2))
    = lambda f . (lambda z2 . (lambda z . f (f (f z))) (f (f (f z2))))
    = lambda f . (lambda z2 . f (f (f (f (f (f z2)))))
    = lambda f . (lambda x . f (f (f (f (f (f x))))))

zero? 0
    = (lambda n . n (TRUE FALSE) TRUE) 0
    = 0 (TRUE FALSE) TRUE
    = (lambda f x . x) (TRUE FALSE) TRUE
    = TRUE

zero? 1
    = (lambda n . n (TRUE FALSE) TRUE) 1
    = 1 (TRUE FALSE) TRUE
    = (lambda f x . f x) (TRUE FALSE) TRUE
    = (TRUE FALSE) TRUE 
    = (lambda x y . x) FALSE TRUE  // TRUE FALSE 的定义和展开 参考下一节
    = FALSE

zero? 2
    = (lambda n . n (TRUE FALSE) TRUE) 2
    = 2 (TREU FALSE) TRUE
    = (lambda f x . f (f x)) (TRUE FALSE) TRUE
    = (TRUE FALSE) ((TRUE FALSE) TRUE)
    = (lambda x y .  x) FALSE ((TRUE FALSE) TRUE)
    = FALSE

pred 2
    = [lambda n . (lambda f x . n (lambda g h . h (g f)) (lambda u . x) (lambda u . u))] 2
    = lambda f x . 2 (lambda g h . h (g f)) (lambda u . x) (lambda u . u)
    = lambda f x . (lambda s z . s (s z)) (lambda g h . h (g f)) (lambda u . x) (lambda u . u)
    = lambda f x . (lambda g h . h (g f)) ((lambda g h . h (g f)) (lambda u . x)) (lambda u . u)
    = lambda f x . (lambda u . u) (((lambda g h . h (g f)) (lambda u . x)) f)
    = lambda f x . (lambda g h . h (g f)) (lambda u . x) f
    = lambda f x . f ((lambda u . x) f)
    = lambda f x . f x
    = 1

sub 3 2
    = (lambda m . (lambda n .n pred m)) 3 2
    = (lambda m n . n pred m) 3 2
    = 2 pred 3
    = (lambda f x . f (f x)) pred 3
    = pred (pred 3)
    = 1

mul 3 2
    = (lambda m n . (lambda f . m (n f))) 3 2
    = lambda f . 3 (2 f)
    = lambda f . 3 ((lambda s . (lambda z . s (s z))) f)
    = lambda f . 3 (lambda z . f (f z))
    = lambda f . (lambda w . (lambda x . w (w (w x)))) (lambda z . f (f z))
    = lambda f . (lambda x . (lambda z . f (f z)) ((lambda z . f (f z)) ((lambda z . f (f z)) x))
    = lamdda f x . (lambda z . f (f z)) ((lambda z . f (f z)) ((lambda z . f (f z)) x))
    = lamdda f x . (lambda z . f (f z)) ((lambda z . f (f z)) (f (f x)))
    = lambda f x . (lambda z . f (f z)) (f (f (f (f x))))
    = lambda f x . f (f (f (f (f (f x)))))
    = 6

pow 2 2
    = (lambda b . lambda e . e b) 2 2
    = (lambda b e . e b) 2 2
    = 2 2
    = (lambda f . (lambda x . f (f x))) 2
    = lambda w . 2 (2 w)
    = lambda w . 2 ((lambda f . (lambda x . f (f x))) w)
    = lambda w . 2 (lambda x . w (w x))
    = lambda f . (lambda s . (lambda z . s (s z))) (lambda x . f (f x))
    = lambda f . (lambda z . (lambda x . f (f x)) ((lambda x . f (f x)) z))
    = lambda f z . (lambda x . f (f x)) ((lambda x . f (f x)) z)
    = lambda f z . (lambda x . f (f x)) (f (f z))
    = lambda f z . f (f (f (f z))) 
    = 4
```

## 0x03 它还能指导编程语言?

```lua
-- 0 = lambda f x. x
zero = function (f)
    return function (x)
        return x
    end
end

-- 1 = lambda f x . f x
one = function (f)
    return function (x)
        return f(x)
    end
end

-- add = lambda m n . (lambda f x . m f (n f x))
add = function (m, n)
    return function (f)
        return function (x)
            return m(f)(n(f)(x))
        end
    end
end

two = add(one, one)
three = add(two, one)
five = add(two, three)
```

emmm...对上了, 函数化的数字!

```lua
three(function ()
    print("给爷打印3次")
end)()

five(function ()
    print("再来5次康康")
end)()
```

## 0x04 参考

* [wiki - λ演算](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)
* [blog - Good Math/Bad Math](http://goodmath.blogspot.com/)
* [blog - Keep Coding](https://liujiacai.net/blog/2014/10/12/lambda-calculus-introduction/)
* [site - 廖雪峰的官方网站](https://www.liaoxuefeng.com/wiki/1022910821149312)