---
title: WSL[2] 安装 ArchLinux
author: ash
tags: ["Linux", "WSL", "windows"]
categories: ["小工具大道理"]
date: 2020-06-15T17:55:01+08:00
cover: "/images/fate00.png"
---

## 简化安装


## 手动安装

### 0x01 启用 WSL

使用管理员权限 打开 PowerShell> 执行:

```ps1
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

若只想安装 `WSL1`，现在可以重启计算机，然后执行步骤 `0x06` 

### 0x02 WSL 升级 WSL2 的必要条件

* 对于 x64 系统：版本 1903 或更高版本，采用 内部版本 18362 或更高版本
* 对于 ARM64 系统：版本 2004 或更高版本，采用 内部版本 19041 或更高版本
* 低于 18362 的版本不支持 `WSL2`

### 0x03 启用虚拟机功能

安装 `WSL2` 之前，必须启用“虚拟机平台”可选功能. 计算机需要虚拟化功能才能使用此功能.

使用管理员权限 打开 PowerShell> 执行:

```ps1
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

重新启动 计算机，以完成 `WSL` 安装并更新到 `WSL2` .

### 0x04 下载 Linux 内核更新包

[点击下载x64安装包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)
[点击下载arm64安装包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_arm64.msi)

安装运行上一步中下载的更新包.

### 0x05 将 WSL2 设置为默认版本

使用管理员权限 打开 PowerShell> 执行:

```ps1
wsl --set-default-version 2
```

### 0x06 下载 安装 LxRunOffline

[LxRunOffline 下载链接](https://github.com/DDoSolitary/LxRunOffline/releases)

选择最新版下载，解压后将得到 LxRunOffline 文件夹.

### 0x07 下载 ArchLinux

[archlinux 清华源](https://mirrors.tuna.tsinghua.edu.cn/archlinux/iso/latest/)

选择 bootstrap 包下载， 例如：`archlinux-bootstrap-2021.03.01-x86_64.tar.gz`

### 0x08 安装 ArchLinux 到 WSL

执行如下命令:

```ps1
LxRunOffline i -n <自定义名称> -f <Arch镜像位置> -d <安装系统的位置> -r root.x86_64
```

举个栗子!
```ps1
LxRunOffline i -n archlinux -f /mnt/d/iso/archlinux-bootstrap-2021.03.01-x86_64.tar.gz -d /mnt/d/wsl-linux/arch/ -r root.x86_64
```

设置 ArchLinux 的 WSL版本, 执行:

```ps1
wsl --set-version <名称> 2

# wsl --set-version archlinux 2  // 这行是注释 举个例子的 别复制噢~ 
```

名称就是 上个命令中 你自定义的名称啦~


### 0x09 安装系统

执行>

```ps1
wsl -d archlinux
```

现在已经进入 archlinux 内部了, 删除 /etc/resolv.conf 文件>:

```sh
rm /etc/resolv.conf
```

重启 archlinux

```sh
exit
```

执行上述命令后，又回到 powershell 了， 执行：

```ps1
wsl --shutdown archlinux
```

然后再进入 archlinux, 执行:

```ps1
wsl -d archlinux
```

现在 又进入 archlinux 内部了.. 执行:

```sh
cd /etc/
explorer.exe .
```

注意后面的 `.` ，执行这条命令后会用 windows 的 文件管理器 打开 `/etc` 目录.

找到 `pacman.conf` 文件， 编辑文件， 在文件最后加入:

```md
[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

找到目录 `pacman.d` ， 进入 编辑里面的 `mirrolist` 文件，去除 `China` 源的注释. [选择部分即可]

然后回到Arch，执行

```sh
pacman -Syy
pacman-key --init
pacman-key --populate
pacman -S archlinuxcn-keyring
pacman -S base base-devel neovim git wget
```

把基础设施安装好~ 也别忘了修改一下 root 密码:

```sh
passwd
```

再创建一个普通用户.

```sh
useradd -m -G wheel -s /bin/bash <用户名>
passwd <用户名>
```

将文件 `/etc/sudoers` 中的 `wheel ALL=(ALL) ALL` 那一行前面的注释去掉.

注意，这个文件是只读的，所以需要 赋予 写权限

```sh
chmod +w /etc/sudoers
nvim /etc/sudoers
chmod -w /etc/sudoers
```

查看用户id

```sh
id -u <用户名>
```


### 0x10 设置使用 普通用户 登录 Archlinux

紧接上一步，退出Arch:

```sh
exit
```

在powershell中执行:

```ps1
lxrunoffline su -n archlinux -v <0x09中的用户id>
```

### 

Finish!! 开始愉快的 Arch 之旅吧 ~ ~