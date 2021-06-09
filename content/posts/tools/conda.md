---
title: Anaconda / Miniconda
author: ash
tags: ["conda"]
categories: ["小工具大道理"]
date: 2020-06-11T18:35:01+08:00
---

## 配置文件

1. conda配置文件
   - windows: `c:/Users/xxx/.condarc`
   - linux: `~/.condarc`

```c
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

1. pip配置文件
    - windows: `C:/Users/xxx/pip/pip.ini`
    - linux: `~/.pip/pip.conf`

```sh
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
```

## 更换源

```sh
# 添加 conda 清华源
$ conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free
$ conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
$ conda config --set show_channel_urls yes
```
