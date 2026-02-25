---
slug: /
title: "K3s - 轻量级 Kubernetes"
---

K3s 是轻量级的 Kubernetes。K3s 易于安装，仅需要 Kubernetes 内存的一半，所有组件都在一个小于 100 MB 的二进制文件中。

它适用于：

* 边缘环境
* 家庭实验室
* 物联网
* 持续集成
* 开发环境
* 单板计算机（ARM）
* 集成式 K8s（将k8s的分散的组件做成集成式的了）
* 在某些情况下，像 Kubernetes 集群学这样的学科博士学位是不现实或无法实现的

# 什么是 K3s？

K3s 是一个完全兼容的 Kubernetes 发行版，具有以下增强功能：

* 以单个二进制文件或精简容器镜像的形式分发。
* 基于 sqlite3 的轻量级数据存储，作为默认存储后端。也支持 etcd3、MySQL 和 Postgres。
* 封装在简单的启动程序中，处理了许多 TLS 和选项的复杂性。
* 默认安全，对轻量级环境提供合理的默认配置。
* 所有 Kubernetes 控制平面组件的操作都被封装在一个单一的二进制文件和进程中，使得 K3s 能够自动化和管理复杂的集群操作，如证书分发。
* 外部依赖已最小化；唯一的要求是现代内核和 cgroup 挂载。

* 打包所需的依赖，便于轻松创建“开箱即用”的集群
   * containerd
   * Flannel (Container Network Interface)
   * CoreDNS Cluster DNS
   * Traefik Ingress controller
   * ServiceLB Load-Balancer controller
   * Kube-router Network Policy controller
   * Local-path-provisioner Persistent Volume controller
   * Spegel distributed container image registry mirror
   * Host utilities (iptables, socat, etc)


# 为什么叫 K3s?

我们希望安装的 Kubernetes 只占用一半的内存。Kubernetes 是一个 10 个字母的单词，简写为 K8s。Kubernetes 的一半就是一个 5 个字母的单词，因此简写为 K3s。K3s 没有全称，也没有官方的发音。
