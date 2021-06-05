# 实战kubernetes

kubernetes 1.20+


## 内容
- [为什么是kubernetes](#为什么是kubernetes)
- [cgroup和systemd](#cgroup和systemd)
    - [cgroup](#cgroup)
    - [systemd](#systemd)
- [namespaces](#namespaces)
- [netfilter](#netfilter)
    - [iptables](#iptables)
    - [nftables](#nftables)
    - [ipvs](#ipvs)
- [依赖和配置](#依赖和配置)
    - [容器运行时简介](#容器运行时简介)
    - [runc](#runc)
    - [runc核心代码分析](#runc核心代码分析)
    - [containerd](#containerd)
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
- 云时代的操作系统，跟Linux一样跨时代

## cgroup和systemd

- Linux的一些发布版使用systemd来初始化系统和服务，其中systemd需要依赖cgroups来完成systemd单元（进程）的追踪和资源管理等功能
- 系统初始化的时候会创建cgroups的根控制组、子系统控制组层级树和systemd控制组层级树，并提供对应的控制组虚拟文件系统来提供实际的客户端操作

## cgroup
- cgroups是内核提供的一种机制，可以把任务（进程）及其子任务进行整合或分隔到多个控制组（cgroup）中，并提供一个叫做子系统（subsystem）的模块对控制组进行特定资源的管理和限制。控制组从属于某些子系统，就组成了控制组层级树（cgroup hierarchy tree）的结构
- 实际使用时，需要通过挂载控制组虚拟文件系统（cgroup virtual filesystem）来实现了控制组层级树，控制组层级树就会自动出现在/proc/mounts中:

    ```sh
    cat /proc/mounts | grep cgroup
    ```
- 创建控制组层级树后还需在其上创建新控制组，并指定任务（进程）PID和资源限制，相应的修改就会自动通知给内核，任务的信息也会自动出现在/proc/\<pid>/cgroups里，之后用户就可以追踪和监控相关任务信息。用户还可以进行其他操作，比如创建新控制组、销毁控制组、给任务指定控制组、追踪任务和对任务进行资源限制等，上述过程的命令如下：

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

- Linux的内核被加载，初始化后就会创建各种服务进程和守护进程，在大部分Linux发布版里，系统创建的第一个进程（PID为1）就是[systemd](https://www.freedesktop.org/software/systemd/man/systemd.html#)（因为建立了一个/sbin/init的软连接到systemd程序），可以通过Linux的ps -ef命令显示进程的相关信息，比较特别的是PPID为0的进程，在很多Linux发布版里这样的进程是systemd和kthreadd，前者的PID为1，负责在用户空间对其他服务进行启动和管理，即初始化系统。可以发现很多的进程的PPID都是1，即父进程就为systemd，从内部来看，systemd不是单个守护进程，它重度依赖Linux内核组件并且包含大量的可运行程序、守护进程（比如日志守护进程）和库，它的主要目的是代替传统的init来管理服务和守护进程，消除每个Linux在这块的差异：
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

- 使用systemd初始化的Linux发行版会生成一个根控制组， systemd在cgroups的一个私有目录里，即/sys/fs/cgroup/systemd，systemd利用
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


## namespaces
- namespaces即Linux命名空间，是容器技术的基础，所谓namespace就是把操作系统上的资源抽象封装成一个个互相隔离的资源实例，使得在不同的namespace里的进程虽只拥有资源的一个实例却仍感觉拥有系统资源的全部所有权，对该系统资源的修改也只会对本namespace里的成员进程可见，Linux提供了如下命名空间：
    ```sh
    Cgroup  # cgroup的根目录
    IPC     # 信号量，进程通信等
    Network # 网络设备、网络栈和端口等
    Mount   # 文件系统挂载点
    PID     # 进程id
    User    # user id 和 group id
    UTS     # hostname和域名
    ```
- 可以通过/proc/\<pid>/ns查看进程的全部命名空间，内核3.8之后这些文件都是符号链接（软链接），之前都是硬链接：
    ```sh
    # 查询指定进程的namespace
    cat /proc/[pid]/ns
    ```
- Network命名空间提供了独立的网络设备（物理设备和虚拟设备）、网络协议栈、路由表、防护墙规则、/proc/net目录（/proc/</pid>/net的一个软链接）、/sys/class/net目录、/proc/sys/net里的大量文件、端口和套接字等。其中设备包括物理网络设备和虚拟网络设备，前者在命名空间释放后会迁移回最初的Network命名空间，而后者可以在多个Network命名空间建立通道进行通信，也可以被用来作为网桥同其他命名空间的物理网络进行通信，在命名空间释放后被销毁
- 其他命名空间的内容可以[man namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)

## netfilter
## iptables
## nftables
## ipvs

## 依赖和配置
- kubernetes最主要的任务就是在每个节点管理容器，所以需要在每个 节点安装容器运行时，比如[docker](https://docs.docker.com/)和[containerd](https://containerd.io/)

## 容器运行时简介
- CRI即容器运行时接口，现在一般内置于容器运行时作为CRI插件来提供对外grpc服务，kubelet则作为CRI客户端通过unix套接字来调用该接口，完成镜像管理和容器生命周期管理等操作。
- 容器运行时使用cgroups来完成对容器的资源管理和追踪，使用namespaces来隔离容器间资源，使用cni插件提供容器网络，更详细内容可以参考[这里](https://kubernetes.feisky.xyz/extension/cri)
。现在版本的kubernetes推荐使用systemd来统一托管组件、容器运行时和容器，不同时混用systemd和cgroupfs（cgroup虚拟文件系统），避免导致相互间的资源冲突，更详细的原因可以参考[这里](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/)。可以修改kubelet的配置文件，添加“cgroupDriver: systemd ”来显式指定kubelet使用systemd，如果是使用kubeadm搭建的集群可以修改kubeadm的配置显式给cgroupDriver（1.21版本默认这个字段为systemd）：
    ```yml
    # kubeadm-config.yaml
    
    kind: ClusterConfiguration
    apiVersion: kubeadm.k8s.io/v1beta2
    kubernetesVersion: v1.21.0
    ---
    kind: KubeletConfiguration
    apiVersion: kubelet.config.k8s.io/v1beta1
    cgroupDriver: systemd
    ```
- 这之后执行init命令，kubeadm就会将KubeletConfiguration结构体（实际是k8s里命名空间kube-system里的ConfigMap）的信息写入到节点文件/var/lib/kubelet/config.yaml中，即kubelet的配置文件，详细内容可参考[文档](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)

## runc
- [runC](https://github.com/opencontainers/runc)项目是根据OCI（开放容器倡议）构建的命令行工具，主要依赖dokcer公司开源的[libcontainer](https://github.com/opencontainers/runc/tree/master/libcontainer)项目作为其内部库来创建和管理容器生命周期，该项目能够透明的管理容器的命名空间（namespaces）、控制组（cgroups）、系统调用能力和文件访问，并利用[libseccomp](https://github.com/seccomp/libseccomp)（underlying BPF based syscall filter language）来过滤容器进程发起的系统调用，利用cgroup来对容器进行资源限制，利用namespaces来隔离容器间资源
- runC从v1.0.0-r95开始无论系统的cgroup是v1还是v2都自动忽略kmem.limit（kernel memory limiting），原因是会导致一些低内核版本的发布版内存泄露，比如在RHEL7里这个参数会导致kernel memory无法释放，5.4以上的的内核也废弃了cgroup v1的kernel memory limiting
- 从v1.0.0-rc93自动开启selinux和apparmor，低内核版本建议直接关闭kernel memory limiting，具体说明可以参考[这里](https://github.com/containerd/containerd/blob/master/docs/RUNC.md)：
    ```sh
    make BUILDTAGS='nokmem seccomp' && make install
    ```

## runc核心代码分析
## netfilter
## iptables

## containerd
- containerd是工业级容器运行时，强调简单、可靠和可移植性，可以作为Linux上的容器运行时守护进程，管理完整的容器的生命周期，镜像传输和存储，监控容器的底层存储和网络，官方介绍可以参考[这里](https://containerd.io)，

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
