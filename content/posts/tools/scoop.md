---
title: SCOOP - windows系统 包管理工具
author: ash
tags: ["windows", "scoop", "package management tool"]
categories: ["小工具大道理"]
date: 2020-06-12T18:30:01+08:00
cover: "/images/re0-02.png"
---

## UPDATE

* win10推出官方包管理器 `winget` ，所以关于 `scoop` 和 `chocolate` 就到此为止吧...
    > 虽然其实根本就没写 `choco`，小声BB

## 准备

1. PowerShell版本 >=3 [最好高于5]

    ```sh
    > $psversiontable.psversion.major
    # $psversiontable
    ```

2. 允许PowerShell执行本地脚本

    ```ps
    > set-executionpolicy remotesigned -scope currentuser
    ```

3. 更改默认安装位置

    ```sh
    > [environment]::setEnvironmentVariable('SCOOP','D:\scoop','User')
    > $env:SCOOP='D:\scoop'
    ```

## 安装scoop

```sh
> iwr -useb get.scoop.sh | iex
# 或者
> iex (new-object net.webclient).downloadstring('https://get.scoop.sh')
```

## 添加仓库[必须有git]

```sh
> scoop bucket add extras
# scoop bucket add ...
# 添加第三方仓库[example]
> scoop bucket add scoopbucket https://github.com/yuanying1199/scoopbucket
```

## 查找、安装、卸载、更新

```sh
> scoop update # 更新scoop
> scoop search [software name]
> scoop install [software name]
> scoop uninstall [software name]
> scoop update [software name]
```

## 利用`aria2`加速下载

```sh
> scoop install aria2
# 设置线程数
> scoop config aria2-max-connection-per-server 16
> scoop config aria2-split 16
> scoop config aria2-min-split-size 1M
# 启停aria2
> scoop config aria2-enabled false/true
```
