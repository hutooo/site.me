---
title: 瑞士军刀 - NetCat
author: ash
tags: ["linux", "netcat", "unix"]
categories: ["UNIX哲学"]
date: 2021-05-05T11:58:12+08:00
cover: "/images/vr.jpg"
---

# 0x00 

# 0x01 文件传输

接收端: nc -l -p {port} > {filename}

```sh
# ip => 10.151.116.62

nc -l -p 12222 > testfile
mv testfile NT-V1.0.0-S.box
```

发送端: nc {remote_addr} {remote_port} < {file_to_send}

```sh
nc 10.151.116.62 12222 < NT-V1.0.0-S.box
```

# 0x02 端口扫描

nc -w {timeout} {scan_addr} {scan_port}

```sh
nc -w 1 193.169.10.101 9001 < /dev/null && echo "tcp port ok"
```
