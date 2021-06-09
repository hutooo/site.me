---
title: Docker Swarm
date: 2017-07-24T18:36:00+02:00
description: >
    Swarm 容器编排引擎(COE)
categories: [Container]
keywords: []
slug: "docker swarm"
aliases: []
author: ash
images:
- images/chino.png
---

## 0x00 what

* **`Docker Swarm`** 是 `Docker` 官方推出的 **容器编排引擎(COE)** ，提供了容器集群服务，是 `Docker` 官方对容器云生态进行支持的核心方案。
* **`Docker Swarm`** 可以将多个 `Docker` 引擎组装为一个更大 `Docker` 服务，从而快速构造一个容器云平台。从 `Docker` v1.12版本开始，提供原生支持。
* **`Docker Swarm`** 采用去中心化设计， 提供服务发现，负载均衡 路由网格，动态伸缩，安全传输，滚动更新等诸多特性。

## 0x01 why

## 0x02 how

* `Swarm` 集群初始化

    ```sh
    # Swarm 初始化管理节点
    $ docker swarm init
    # docker swarm init --advertise-addr 192.168.99.100

    # Swarm 信息
    $ docker info
    $ docker node ls

    # Swarm 工作节点
    # docker swarm join-token worker
    # docker swarm join --token ...
    ```

* 初始化 `Swarm` 后，在本机 `Docker` 引擎中会添加两个新的网络 `ingress` 和 `docker_gwbridge`

    ```sh
    # 查看 docker 网络
    $ docker network ls
    ```

    |NETWORK ID| NAME | DRIVER | SCOPE |
    |-|-:|-:|-:|
    dd9801f4d4e8 |  bridge           |  bridge  | local
    63528dad4c85 |  docker_gwbridge  |  bridge  | local
    99e54b9a7444 |  host             |  host    | local
    6eh2eml2h6l4 |  ingress          |  overlay | swarm
    3838f72c8317 |  none             |  null    | local

* 你可以创建自定义的 `overlay` 网络

    ```sh
    # docker network create --help
    $ docker network create -d overlay my-overlay-network

    # 想要swarm或者独立容器 与 其它docker主机上的容器通信
    $ docker network create -d overlay --attachable my-attachable-network

    # 指定子网段
    $ docker network create  --driver overlay --subnet 172.70.1.0/24  --opt encrypted  my-swarm-network
    ```

* `Swarm` 中的 管理数据(management traffic) 都通过AES算法加密，如果想加密 应用程序数据(application data), 使用 `--opt encrypted`

    ```sh
    # 加密应用数据 --opt encrypted
    $ docker network create --opt encrypted --driver overlay --attachable my-attachable-multi-host-network
    ```

* 部署service

    ```sh
    # docker service --help
    $ docker service create --network my-swarm-network --replicas 3 --name helloworld alpine ping docker.com

    # 查看服务列表
    $ docker service ls

    # 检查特定服务
    $ docker service inspect --pretty helloworld
    $ docker service ps helloworld

    # 扩展服务节点
    $ docker service scale <SERVICE-ID>=<NUMBER-OF-TASKS>

    # 删除服务
    $ docker service rm helloworld
    ```

## 0x03 Reference

1. [Swarm-Tutorial](https://docs.docker.com/engine/swarm/swarm-tutorial/)
