---
title: Bool and Condition of Lambda-Calculus
author: ash
tags: ["Lambda", "λ", "λ-演算"]
categories: ["Lambda奥秘"]
date: 2020-11-20T12:12:10+08:00
image: chino01.jpg
---

> 我想寻求这样一张特殊的字母表，其元素表示的不是声音而是概念。有了这样一个符号系统，我们就可以发展出一种语言，
> 我们仅凭符号演算，就可以确定用这种语言写成的哪些句子为真，以及它们之间存在着什么样的逻辑关系.  --------- 莱布尼茨的愿望

## 0x01 函数化的布尔值

在计算机编程语言中，为了达到条件判断，大都会引入 `if-then-else` 这类条件分支语言，像这样~

```lua
if n == 1 then
    ...
else
    ...
end
```

为了实现条件真假判断，先得看看lambda演算中的布尔值和逻辑运算.(带上点语法糖，看起来更有味儿)

```md
let TRUE  = lambda x y . x
let FALSE = lambda x y . y
```

嘛~ 现在貌似还看不出 `TRUE` 和 `FALSE` 的lambda表达式的用意.. 再看看逻辑运算符吧

```md
let And = lambda x y . x y FALSE
let Or  = lambda x y . x TRUE y
let Not = lambda x . x FALSE TRUE
```

咦~ 什么乱起八糟的，果然只有动动手，才能加深理解了.. 冲！

```md
And TRUE FALSE =  (lambda x y . x y FALSE) TRUE FALSE // 展开
    = TRUE FALSE FALSE // beta规约
    = (lambda x y . x) FALSE FALSE // 展开
    = FALSE // beta规约

And FALSE TRUE = (lambda x y . x y FALSE) FALSE TRUE // 展开
    = FALSE TRUE FALSE // beta规约
    = (lambda x y. y) TRUE FALSE // 展开
    = FALSE // beta 规约

// emmm... 貌似懂了，那再看看别的?

Or FALSE TRUE = (lambda x y . x TRUE y) FALSE TRUE // 展开
    = FALSE TRUE TRUE // beta规约
    = (lambda x y . y) TRUE TRUE // 展开
    = TRUE // beta规约

Not FALSE = (lambda x . x FALSE TRUE) FALSE // 展开
    = FALSE FALSE TRUE // beta规约
    = (lambda x y . y) FALSE TRUE // 展开
    = TRUE // beta 规约
```

厉害了~ 还蛮有意思的..

## 0x02 If-Then-Else

有了上面的知识，我们来完成一个 If-Then-Else 的条件选择

```md
let If-Then-Else = lambda cond T-exp F-exp . cond T-exp F-exp

// if (true or false) then T-exp else F-exp => T-exp
If-Then-Else (Or TRUE FALSE) T-exp F-exp
    = (lambda cond T-exp F-exp . cond T-exp F-exp) (Or TRUE FALSE) T-exp F-exp // 展开
    = (Or TRUE FALSE) T-exp F-exp // beta规约
    = ((lambda x y . x TRUE y) TRUE FALSE) T-exp F-exp // 展开
    = (TRUE TRUE FALSE) T-exp F-exp // beta规约
    = ((lambda x y . x) TRUE FALSE) T-exp F-exp // 展开
    = TRUE T-exp F-exp // beta规约
    = (lambda x y . x) T-exp F-exp // 展开
    = T-exp // beta 规约
```

呼... 练练手吧

## 0x03 编程练习

```lua
-- let TRUE = lambda x y . x
TRUE = function (x)
    return function (y)
        return x
    end
end
-- let FALSE = lambda x y . y
-- print(TRUE(7)(8))
FALSE = function (x)
    return function (y)
        return y
    end
end

-- let And = lambda x y . x y FALSE
And = function (x, y)
    return x(y)(FALSE)
end

-- let Or = lambda x y. x TRUE y
Or = function (x, y)
    return x(TRUE)(y)
end

-- let Not = lambda x . x FALSE TRUE
Not = function (x)
    return x(FALSE)(TRUE)
end

print(And(TRUE,TRUE)(1)(0))
print(And(TRUE,FALSE)(1)(0))
print(Or(FALSE,TRUE)(1)(0))
print(Not(FALSE)(1)(0))

-- let ifThenElse = lambda cond T-exp F-exp . cond T-exp F-exp
If_Then_Else = function (cond)
    return function (Texp)
        return function (Fexp)
            return cond(Texp)(Fexp)
        end
    end
end
print(If_Then_Else(Or(TRUE,FALSE))(7)(8))
If_Then_Else(And(FALSE,TRUE))(function ()
    print("ttt")
end)(function ()
    print("fff")
end)()
```

## 0x04 参考

* [wiki - 布尔代数](https://zh.wikipedia.org/wiki/%E5%B8%83%E5%B0%94%E4%BB%A3%E6%95%B0)
* [wiki - 乔治·布尔](https://zh.wikipedia.org/zh-hans/%E4%B9%94%E6%B2%BB%C2%B7%E5%B8%83%E5%B0%94)
* [wiki - λ演算](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)
* [blog - Good Math/Bad Math](http://goodmath.blogspot.com/)
* [WechatOA - 符号, 计算抽象](https://mp.weixin.qq.com/s/n4DDPDyQRbDjNFstkv3VwQ)