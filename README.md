# 实战kubernetes

kubernetes 1.20+


## 内容
- [从容器开始](#从容器开始)
    - [docker](#docker)
    - [docker-compose](#docker-compose)
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

## 从容器开始

## docker

## docker-compose

## containerd

## 为什么是kubernetes

- 因为简单

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