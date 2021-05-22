# 实战kubernetes

kubernetes 1.20+


## 内容
- [为什么是kubernetes](#为什么是kubernetes)
- [cgroup](#cgroup)
- [systemd](#systemd)
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

## cgroup
- cgroup是内核提供的一种机制，可以把任务（进程或线程）及其子任务进行整合或分隔到多个控制组（cgroup）中，并提供一个叫做子系统（subsystem）的模块对控制组进行特定资源的管理和限制。控制组从属于某些子系统，就组成了控制组层级树（cgroup hierarchy tree）的结构
- 实际使用时，需要通过挂载控制组虚拟文件系统（cgroup virtual filesystem）来实现了控制组层级树，控制组层级树就会自动出现在/proc/mounts中，如果在其上创建新控制组，在其中指定任务的PID和资源限制，相应的修改就会自动通知给内核，任务的信息也会自动出现在/proc/\<pid>/cgroups里。用户可以通过这个虚拟文件系统进行其他操作，比如创建新控制组、销毁控制组、给任务指定控制组、追踪任务和对任务进行资源限制等，上述过程的命令如下：
    ```sh
    # 拥有全部的子系统
    mount -t cgroup xxx /sys/fs/cgroup 

    # or

    # 指定的子系统
    mount -t tmpfs cgroup_root /sys/fs/cgroup
    # 创建资源组
    mkdir /sys/fs/cgroup/rg1 
    mount -t cgroup -o cpuset,memory hier1 /sys/fs/cgroup/rg1
    
    # 创建控制组
    cd /sys/fs/cgroup/rg1
    mkdir Charlie 
    ```
- 详细内容请参考[文档](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)

## systemd

- Linux的内核被加载，初始化后就会创建各种服务进程和守护进程，在大部分Linux发布版里，系统创建的第一个进程（PID为1）就是[systemd](https://www.freedesktop.org/software/systemd/man/systemd.html#)（因为建立了一个/sbin/init的软连接到systemd程序），可以通过Linux的ps -ef命令显示进程的相关信息，比较特别的是PPID为0的进程，在很多Linux发布版里这样的进程是systemd和kthreadd，前者的PID为1，负责在用户空间对其他服务进行启动和管理。可以发现很多的进程的PPID都是1，即父进程就为systemd，从内部来看，systemd不是单个守护进程，它重度依赖Linux内核组件并且包含大量的可运行程序、守护进程（比如日志守护进程）和库，它的主要目的是代替传统的init来管理服务和守护进程，消除每个Linux在这块的差异：
    ```sh
    # 能力简介

    1. 并行运行服务
    2. 自动处理服务依赖
    3. systemd相关服务常驻内存
    4. systemctl命令来处理任务
    5. 可以按功能对服务进行组织（叫做unit）
    6. 监控每个服务的状态和资源使用情况
    7. 整合cgroup对服务进行资源管理
    ```

- 可以在systemd里监控和改变unit的状态，因为有一个job队列专门处理unit的状态变化请求，上面有说过，systemd会按需加载符合条件的unit常驻内存，但这个过程对用户是不透明，[文档](https://www.freedesktop.org/software/systemd/man/systemd.html#)里提到需要至少满足如下一个条件才会被在内存常驻：
    ```
    1. 状态为active、activating、deactivating或failed 
    2. job队列还有该unit的消息
    3. 被内存里的unit作为依赖
    4. 还有资源分配，未销毁
    5. D-Bus调用捆绑
    ```
- systemd提供了一套以unit作为基础的依赖系统，其中unit抽象了应用在实际系统里的进程运作细节，目前有11个unit类型，详细内容可以[man systemd.unit](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#)，常用的unit如下：
    ```sh
    service # 常规服务，一般直接封装进程，比如本地服务或者网络服务
    socket # 套接字服务，多用于进程间通信
    target # 执行环境，一堆服务的捆绑
    timer # 循环执行的服务
    scope # 定义了外部通过fork创建的并注册到systemd运行时的进程
    slice # 组织service和scop类型的服务到层级树（用于systemd的资源管理）
    path # 检测文件服务
    mount # 文件系统挂载服务
    automount # 文件系统挂载服务

    ```

- 存在一个默认target，在系统启动时候自动被激活，实际上是启动了与之关联的一堆捆绑服务，通过指定不同的target可以进入不同的环境
- 绝大部分的unit可以通过配置文件来创建，被创建的unit一般是按照配置文件名来命名的（一般在/usr/lib/systemd/system），但也有一部分比如上面的[scope](https://www.freedesktop.org/software/systemd/man/systemd.scope.html#)类型的unit是无法通过配置文件创建，是systemd运行时创建的
- systemd在cgroups的一个私有目录里，即/sys/fs/cgroup/systemd，systemd利用
- systemd在cgroups的文件系统目录里创建专属于systemd的控制组层级结构，对应的虚拟文件系统目录为/sys/fs/cgroup/systemd，并创建和service、scope和slice等unit对应的控制组层级结构，每个服务就是一个控制组以便systemd能利用cgroup对unit进行追踪和资源管理，可以使用systemd-cgls来展示整个systemd cgroup层级结构，systemd和cgroup的相关内容还可以参考[文档](https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/)

- systemd的常用命令：
    ```sh 
    # 例子

    systemctl status sshd.service # 状态
    # 等价
    systemctl status sshd # 确定服务状态
    systemctl start sshd # 开启服务
    systemctl stop sshd # 关闭服务
    systemctl enable sshd # 开机启动服务
    systemctl disable sshd # 开机不启动服务
    systemctl reload ssh # 重载服务配置
    systemctl restart ssh # 重启服务
    systemctl # 启动状态的服务
    systemctl list-unit-files # 全部服务状态
    systemctl list-units # 启动状态的服务
    systemctl list-unit-files --type=service # 某类型的服务状态
    systemctl list-units --type=target # 某类型的启动状态的服务
    systemctl get-default # 当前默认运行的target
    systemctl set-default multi-user.target # 设置默认运行的target为命令行
    systemctl isolate graphical.target # 图形界面
    ```
其他内容可以参考文档




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