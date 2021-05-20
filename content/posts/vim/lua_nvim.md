---
title: Neovim + Lua = init.lua
author: ash
tags: ["editor", "neovim", "lua"]
categories: ["文本编辑"]
date: 2021-05-05T11:58:12+08:00
---

## 0x00 Why Lua?

`NeoVim` 的目标之一，是使用 Lua 来替代陈旧的 VimL. 

这样做的原因之一是因为 VimL 是一种执行非常缓慢的解释型语言，并且几乎没有进行优化. 同时由于 VimL 的陈旧，当代码基础增长时，VimL 可能会变得有点混乱.

因此，在 vim 的启动和可能阻塞编辑器主循环的插件操作中，大部分时间都花在解析和执行 vimscript 上. 

所以在新版本的 `NeoVim` 中，作为更好的语言选择，Lua被集成进来. 通过内嵌 Lua 运行时来创建和提供更快更强的扩展.

万事开头难(，然后中间难，最后结果难)~  那就一起来看看它的真面目吧~ 

> 本文以 `NeoVim v0.5.0-dev+1297-g63d8a8f4e` 版本为基础 进行编写.

## 0x01 NeoVim Lua Basics

