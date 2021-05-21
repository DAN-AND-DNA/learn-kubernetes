# 实战kubernetes

kubernetes 1.20+


## 内容
- [为什么是kubernetes](#为什么是kubernetes)
- [systemd](#systemd)
- [cgroup](#cgroup)
- [netfilter](#netfilter)
    - [iptables](#iptables)
    - [nftables](#nftables)
    - [ipvs](#ipvs)
- [依赖和配置](#依赖和配置)
    - [containerd](#containerd)
    - [runc](#runc)
    - [crictl](#crictl)
    - [kubernetes](#kubernetes)
    - [ca](#ca)
    - [cfssl](#cfssl)
- [基础镜像](#基础镜像)
    - [打包](#打包)
    - [私有仓库](#私有仓库)
- [安装](#安装)
    - [centos7](#centos7)
    - [archlinux](#archlinux)
    - [快速搭建](#快速搭建)
    - [硬核手工高可用](#硬核手工高可用)
- [kube核心组件和配置](#kube核心组件和配置)
    - [kubelet](#kubelet)
    - [etcd](#etcd)
    - [kube-apiserver](#kube-apiserver)
    - [kube-proxy](#kube-proxy)
    - [kube-controller-manager](#kube-controller-manager)
    - [kube-scheduler](#kube-scheduler)
    - [coredns](#coredns)
- [可选组件和工具](#可选)
    - [日志](#日志)
    - [监控](#监控)
    - [调试](#调试)
    - [metrics-server](#metrics-server)
    - [k9s](#k9s)
    - [calico](#calico)
    - [prometheus](#prometheus)
- [运行流程](#运行流程)
- [工作负载](#工作负载)
    - [pod](#pod)
    - [deployment](#deployment)
    - [service](#service)
    - [ ... ]
- [存储](#存储)
- [简单运维](#简单运维)
- [扩展和开发](#扩展和开发)
    - [golang](#golang)
    - [golangci-lint](#golangci-lint)
    - [go-fuzz](#go-fuzz)
    - [CRD](#CRD)
    - [traefik](#traefik)
    - [operator](#operator)
    - [kubebuilder](#kubebuilder)
- [核心代码分析](#核心代码分析)
- [内核优化](#内核优化)
    - [节点](#节点)
    - [容器内](#容器内)
- [测试](#测试)
    - [一致性测试](#一致性测试)

## 为什么是kubernetes

- 因为简单

## systemd

- Linux的内核被加载初始化后就会创建各种服务进程和守护进程，其中在大部分Linux发布版里，系统创建的第一个进程（PID为1）就是[systemd](https://www.freedesktop.org/software/systemd/man/systemd.html#)（因为建立了一个/sbin/init的软连接到systemd程序），可以通过Linux的ps -ef命令显示进程的相关信息，比较特别的是PPID为0的进程，在很多Linux发布版里这样的进程是systemd和kthreadd，前者的PID为1，是系统和服务管理守护进程，负责在用户空间对服务进行启动和管理，可以发现很多的进程的PPID都是1，即父进程就为systemd。从内部来看，systemd不是单个守护进程，它重度依赖Linux内核组件并且包含大量的可运行程序、守护进程（比如日志守护进程）和库，它的主要目的是代替传统的init来管理服务和守护进程，消除每个Linux在这块的差异，实现现代系统的标准化：
    ```sh
    # 能力简介

    1. 并行运行服务
    2. 自动处理服务依赖
    3. systemd相关服务常驻内存
    4. systemctl命令来处理任务
    5. 可以按功能对服务进行组织（叫做unit）
    6. 监控每个服务的状态和资源使用情况
    7. 整合CGroup对服务进行资源管理
    ```

- 可以在systemd里监控和改变unit的状态，因为有一个job队列专门处理unit的状态变化请求，上面有说过，systemd会按需加载符合条件的unit常驻内存，但这个过程对用户是不透明，[文档](https://www.freedesktop.org/software/systemd/man/systemd.html#)里提到需要至少满足如下一个条件：
    ```
    1. 状态为active、activating、deactivating或failed 
    2. job队列还有该unit的消息
    3. 被内存里的unit作为依赖
    4. 还有资源分配，未销毁
    5. D-Bus调用捆绑
    ```
- systemd提供了一套以unit为作为基础的依赖系统，其中unit抽象了应用在实际系统里的运作细节，有11个unit类型，详细内容可以[man systemd.unit](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#)：
    ```sh
    service # 常规服务，一般直接封装进程，比如本地服务或者网络服务
    socket # 进程间通信的套接字服务，多用于IPC
    target # 执行环境，一堆服务的捆绑
    timer # 循环执行的服务
    scope # 定义了通过fork创建的并注册到systemd运行时的进程
    slice # 组织service和scop类型的服务到层级树（用于systemd的资源管理）
    ```

- 上面unit类型中的target是在系统启动时候被激活的，通过指定不同的target来进入不同的环境，也就启动了不同的一堆捆绑的服务，可以通过systemctl相关命令获得系统启动的的默认执行环境default.target，当然也可以通过systemctl相关命令来修改默认环境
- 绝大部分的unit可以通过配置文件来创建，被创建的unit一般是按照配置文件的名字来命名的，但也有一部分比如上面的[scope](https://www.freedesktop.org/software/systemd/man/systemd.scope.html#)类型的unit是无法通过配置文件创建，主要用于systemd进行资源管理，是通过系统状态或者systemd运行时创建的
- systemd利用cgroup在内存中创建目录/sys/fs/cgroup/systemd，并利用cgroup创建和service、scope和slice等unit对应层级结构，以便systemd利用cgroup对unit进行进程追踪和资源管理，可以使用systemd-cgls来展示整个cgroup层级结构，systemd和cgroup的相关内容还可以参考[文档](https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/)

- systemctl的常用命令：
    ```sh
    # TODO
    ```


## cgroup

## 安装
## centos7
```sh
# TODO
```

## archlinux
```sh
# TODO
```

## 内核优化