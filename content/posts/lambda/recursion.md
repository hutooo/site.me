---
title: Recursion of Lambda-Calculus
author: ash
tags: ["Lambda", "λ", "λ-演算"]
categories: ["Lambda奥秘"]
date: 2020-11-23T15:57:10+08:00
image: mhr.png
---

> 从前有座山，山里有座庙，庙里有个老和尚，正在给小和尚讲故事呢！故事是什么呢？“从前有座山，山里有座庙，庙里有个老和尚，正在给小和尚讲故事呢！故事是什么呢？‘从前有座山，山里有座庙，庙里有个老和尚，正在给小和尚讲故事呢！故事是什么呢？……’”

## 0x01 不动点

定义函数 𝑓:ℝ↦ℝ，如果 ∃𝑥∈𝐷(𝑓)，使得 𝑥=𝑓(𝑥)，则称点 𝑥 是函数 𝑓(𝑥) 的不动点.

```md
// 等式 x = f(x) 可以进行如下的无穷变换

x = f(x)
  = f(f(x))
  = f(f(f(x)))
  = ...
```

## 0x02 递归

> 递归的强大之处在于它允许用户用有限的语句描述无限的对象! 瑞士的计算机科学家 `尼克劳斯·维尔特` 如是说..

考虑一下 阶乘函数 `fact` 的定义:

```md
fact n = 1 if n == 0 else (n * fact (n-1))

// 尝试进行 lambda 抽象 [忽略表达式的细节]
let zero? = lambda n x y . if-then-else (n == 0) x y
let pred = lambda x . x - 1

fact = lambda n . zero? n 1 (n * fact (pred n))

// 右侧还是有fact，怎么办，再次使用lambda, 把它拽出来试试..

fact = (lambda f . (lambda n . zero? n 1 (n * f (pred n)))) fact

// 看着好乱，抽象一下吧..
F = E(F) // F是fact, E是中间的lambda函数
  = E(E(F)) // 可以无穷替换哎!!!
  = E(E(E(F)))
  = ...
```

也就是说, `F` 是函数 `E` 的不动点，但是这个函数看着好复杂鸭，我该如何求解呢(这里先不讨论高阶函数的求解方法)~ 

## 0x03 传说中的 Y组合子

召唤!! `Y组合子` !! 一起来瞅瞅这是个啥东东~ 

`Y组合子` 作为高级函数不动点的 `"通用求根公式"`，只要函数存在不动点，我们就可以通过 `Y组合子` 结合函数，进行求解

```md
Y(E) = F
     = E(F)      // F = E(F) 
     = E(Y(E))
     = E(E(Y(E)))
```

利用 `Y组合子` 可以求解得到 `F` 

```md
F = Y(E)
  = Y(lambda f . (lambda n . zero? n 1 f(n * (pred n))))
```

那么这个 `Y` 到底长啥样呢~ (让我们通过一个有趣的小栗子, 一起推导一下吧~)

## 0x04 推导 Y 组合子[编程]

这节我们将从一个常见的函数出发，编程推导出 `Y组合子` ，不熟悉编程的小伙伴，可以直接跳过，不过我采用的Lua语言简单灵活，很方便阅读理解的，那么开始吧~

```lua
--[[
# Y魔法
# Y Y = Y (Y Y)
]]

local arr1 = {1,2,3,4,5} -- Lua中定义的数组(其实是表，不用在意这些细节)

--[[
    这里先定义两个函数,
    isEmpty 用于判断数组是否为空;
    shift   用于将数组中第一项元素移除, JS玩家应该很熟悉这个命名吧~
]]

local function isEmpty(arr)
    if next(arr) == nil then   -- 蹩脚的 空表判断 arr == {}, 将就一下吧
        return true
    end
    return false
end

local function shift(arr)
    table.remove(arr, 1)   -- Lua下标从1开始
    return arr
end

-- 好的，开始我们的旅程吧~

--[[
* 现在问题来了，我想求解 数组的长度 该怎么做呢?
]]

-- [务实的小伙伴肯定第一时间想到，Lua有没有内建函数或者语法糖，有的话，那就很方便了鸭~]
-- 一番检索之后，发现 Lua 还真有
-- print(#arr1) -- 哈哈，成功得到了 数组arr 的长度

-- 看来大家都能很轻松地应对嘛~  [就这!??]


-- 那如果，不允许使用内置语法糖呢??
-- [聪明的小伙伴表示: 这也难不倒我~ 看我递归显神通...]
local function rlength(arr)
    if isEmpty(arr) then
        return 0
    end
    return 1 + rlength(shift(arr))  -- 使用了本身的函数名
end
-- print(rlength(arr1))  

-- (啪-啪-啪~ 鼓掌声)

--[[ 
    但是!! 你们以为这就完了嘛~?~ too young too naive~

    [我知道！尾调用优化，改成尾递归形式~]

    不不不，不要误会，我不是在一步步刁难各位~
    我只是发现，函数体内使用着函数本身的名字，想把它去掉而已

    [纳尼！？我看，你就是在刁难我胖虎！！]
]]

local lenX = function(arr)
    if isEmpty(arr) then
        return 0
    end
    -- return 1 + ???(shift(arr))
end


--[[
    先去看点别的
]]

-- 这个函数做啥鸭，好像不断调用自身，永远不会停止!
local function eternity(x)
    return eternity(x)
end

-- 来看看 这个函数是啥
local len0 = function(arr)
    if isEmpty(arr) then
        return 0
    end
    return 1 + eternity(shift(arr))
end
-- 我们先跨出最小的一步，求解 长度为0 的数组长度
-- print(len0({}))


-- 那么长度为1 的数组呢
local len1 = function(arr)
    if isEmpty(arr) then
        return 0
    end
    return 1 + len0(shift(arr))
end
-- print(len1({1}))


-- 函数展开一下
len1 = function (arr)
    if isEmpty(arr) then
        return 0
    end
    return 1 + (function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + eternity(shift(arr))
    end)(shift(arr))
end
-- print(len1({1}))


-- 好像 很难没有太多启发嘛，那就继续，看看 长度为2 的数组
local len2 = function (arr)
    if isEmpty(arr) then
        return 0
    end
    return 1 + len1(shift(arr))
end
-- print(len2({1,2}))

-- 继续展开
len2 = function (arr)
    if isEmpty(arr) then
        return 0
    end
    return 1 + (function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + (function (arr)
            if isEmpty(arr) then
                return 0
            end
            return 1 + eternity(shift(arr))
        end)(shift(arr))
    end)(shift(arr))
end
-- print(len2({1,2}))


-- 开始看着有点绕了呢，但是还是没有看出太多端倪呢~
-- 前进!!  长度为3 的数组
local len3 = function (arr)
    if isEmpty(arr) then 
        return 0
    end
    return 1 + (function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + (function (arr)
            if isEmpty(arr) then
                return 0
            end
            return 1 + (function (arr)
                if isEmpty(arr) then
                    return 0
                end
                return 1 + eternity(shift(arr))
            end)(shift(arr))
        end)(shift(arr))
    end)(shift(arr))
end
-- print(len3({1,2,3}))


-- 害，重复代码挺多的，看看能不能提取出来
-- 简单一点就从 长度为0 的数组开始吧
len0 = (function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)(eternity)
-- print(len0({}))

-- 那么 长度为1，2，3的数组 也就明了了
len1 = (function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)(len0)
-- print(len1({1}))

len2 = (function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)(len1)
-- print(len2({1,2}))

len3 = (function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)(len2)
-- print(len3({1,2,3}))


-- 感觉看不出啥端倪，继续展开看看
len1 = (function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)((function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)(eternity))


len2 = (function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)((function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)((function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)(eternity)))


len3 = (function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)((function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)((function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)((function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)(eternity))))

-- print(len0({}))
-- print(len1({1}))
-- print(len2({1,2}))
-- print(len3({1,2,3}))


-- 自己观察一下，好像是 f(f(f(ete))) 这样的模式呢，emmmm... 那么再提取试试
len0 = (function (f)
    return f(eternity)
end)(function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)
-- print(len0({}))


-- 学废了！！ 长度为1，2，3的数组  依样画葫芦~
len1 = (function (f)
    return f(f(eternity))
end)(function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)
-- print(len1({1}))

len2 = (function (f)
    return f(f(f(eternity)))
end)(function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)
-- print(len2({1,2}))

len3 = (function (f)
    return f(f(f(f(eternity))))
end)(function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)
-- print(len3({1,2,3}))


--[[
   感觉悟出了一些啥，但是还是没有进一步的思路哎~
   害，既然如此，就搞事情！！
   eternity函数既然不会执行到, 那就可以随意替换, 比如这样-> [替换自身]
]]
len0 = (function (f)
    return f(f)
end)(function (flen)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + flen(shift(arr))
    end
end)
-- print(len0({}))

-- emmm~ 心满意足!  再仔细看看，好像flen这个函数名字也可以换一下~ 一家人就是要整整齐齐!~
len0 = (function (f)
    return f(f)
end)(function (f)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + f(shift(arr))
    end
end)
-- print(len0({}))

--  规约一下看看是啥样~
len0 = (function (f)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + f(shift(arr))
    end
end)(function (f)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + f(shift(arr))
    end
end)
-- print(len0({}))

-- 天呐!!! 函数把自己传给了自己~!!!!


-- 进一步规约！
len0 = function (arr)
    if isEmpty(arr) then
        return 0
    end
    return 1 + (function (f)
        return function (arr)
            if isEmpty(arr) then
                return 0
            end
            return 1 + f(shift(arr))
        end
    end)(shift(arr))
end
-- print(len0({}))

--[[
    细细揣摩一下, F(F) 构造出的函数Z
    1. 当传入长度为0的空数组时，返回0   Z({}) => 0
    2. Z({1}) => 1 + 函数F(...)  如果此时这个F 还是 F(F) 呢 => F(F)(...)
  
  想象一下，会发生什么~

]]
--

len0 = (function (f)
    return f(f)
end)(function (f)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        -- return 1 + f(shift(arr))
        return 1 + f(f)(shift(arr))  -- 注意此处的变化
    end
end)
-- print(len0({}))
-- print(len0({1,2}))

-- 通过不断产生的 F(F)， 不断推迟计算，直至 数组长度为0. 

-- 这里是不是蕴涵着某种更通用的神秘力量呢~
-- 进一步提取
lenX = (function (f)
    return f(f)
end)(function (f)
    return (function (length)
        return function (arr)
            if isEmpty(arr) then
                return 0
            end
            return 1 + length(shift(arr))
        end
    end)
    -- (f(f))          -- call-by-value的求值策略下，避免参数无限求值，导致栈溢出
    (function (x)
        return f(f)(x)
    end)
end)
-- print(lenX({1,2,3,4,5}))


-- lenX = (function (f)
--     return f(f)
-- end)((function (flen)
--     return function (f)
--         return flen(function (x)
--             return f(f)(x)
--         end)
--     end
-- end)(function (length)
--     return function (arr)
--         if isEmpty(arr) then
--             return 0
--         end
--         return 1 + length(shift(arr))
--     end
-- end))

-- 再提取
lenX = (function (flen)
    return (function (f)
        return f(f)
    end)(function (f)
        return flen(function (x)
            return f(f)(x)
        end)
    end)
end)(function (length)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + length(shift(arr))
    end
end)
-- print(lenX({1,2,3,4,5}))

-- OK, 到了这里就差不多了，观察一下函数形式  F = Y(E)
-- 提取Y函数

Y = function (f)
    return (function (u)
        return u(u)
    end)(function (u)
        return f(function (x)
            return u(u)(x)
        end)
    end)
end

local final_len = Y(function (length)
    return function (arr)
        if isEmpty(arr) then
            return 0
        end
        return 1 + length(shift(arr))
    end
end)

-- print(final_len({1,2,3,4,5}))

--[[
    有了Y函数，我们就能很方便地处理匿名递归
]]

-- 阶乘函数[递归形式]
local function fact(n)
    if n == 0 then
        return 1
    end
    return n * fact(n - 1)
end
-- print(fact(4))


-- 阶乘函数[匿名递归]
local anon_fact = Y(function (fact)
    return function (n)
        if n == 0 then
            return 1
        end
        return n * fact(n - 1)
    end
end)
-- print(anon_fact(5))


--[[
    [所以这个Y函数到底是啥呢??] 
    在这里，我们可以称为 -> [应用序]Y组合子
]]
```

## 0x05 终于见到 Y

经过上面的编程推演, 最终得到了 `Y组合子`.

```md
Y = λf.(λx.(f (x x)) λx.(f (x x)))  // Call-By-Name
Y = λf.((λu.(u u)) λx.(f λv.(x x v))) // Call-By-Value
```

最后用一个例子函数g来展开它，可以看到这个函数是如何成为一个不动点组合子的.

```md
(Y g)
    = (λf.(λx.(f (x x)) λx.(f (x x))) g) // 展开
    = (λx.(g (x x)) λx.(g (x x))) // λf的β-归约 - 应用主函数于g
    = (λy.(g (y y)) λx.(g (x x)))// α-转换 - 重命名约束变量
    = (g (λx.(g (x x)) λx.(g (x x)))) // λy的β-归约 - 应用左侧函数于右侧函数
    = (g (Y g)) // Y的定义
```

## 0x06 参考

* [wiki - 哈斯凯尔·柯里](https://zh.wikipedia.org/wiki/%E5%93%88%E6%96%AF%E5%87%AF%E5%B0%94%C2%B7%E6%9F%AF%E9%87%8C)
* [book - TheLittleSchemer](https://mitpress.mit.edu/books/little-schemer-fourth-edition)
* [paper - Programming Languages and Lambda Calculi](...)
* [wiki - 递归](https://zh.wikipedia.org/wiki/%E9%80%92%E5%BD%92)
* [wiki - λ演算](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)
* [blog - Good Math/Bad Math](http://goodmath.blogspot.com/)
* [share - 王垠](https://www.slideshare.net/yinwang0/reinventing-the-ycombinator)
* [paper - The Theory of Recursive Functions, Approaching its Centennial](...)
* [paper - 谈谈 lambda](...)
* [paper - History of Lambda-calculus and Combinatory Logic](...)