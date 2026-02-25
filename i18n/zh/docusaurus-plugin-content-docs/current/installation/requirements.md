---
title: 系统需求
---

K3s 尽管非常轻量级，但仍如下所示有一些最低要求：

无论你是在容器中，还是作为原生 Linux 系统服务而配置使用 K3s，每个节点都应满足以下最低条件。这些条件是 K3s 自身及其组件包的（性能）基线，不含工作负载自身消耗的系统资源。

## 前提

任意两个主机的hostname不可相同。

如果多个节点将使用相同的主机名（hostname），或者主机名可能会被自动化部署系统重复使用，请使用 `--with-node-id` 选项为每个节点添加随机后缀；或者，为您添加到集群中的每个节点设计一个唯一的名称，并通过 `--node-name` 选项或 `$K3S_NODE_NAME` 环境变量进行传递。

## 系统架构

K3s 在下述架构下可用：
- x86_64
- armhf（ARM 硬件浮点）
- arm64/aarch64（这两个是不同的名称）

:::warning ARM64 分页大小

在 2023 年 5 月之前的版本（v1.24.14+k3s1、v1.25.10+k3s1、v1.26.5+k3s1、v1.27.2+k3s1）中，aarch64/arm64 系统的内核必须使用 4 KB 分页大小。RHEL9、Ubuntu、Raspberry PI OS 和 SLES 等均满足此要求。

:::

## 操作系统

K3s 应运行于最新 Linux 系统中。

部分操作系统有额外的安装条件：
<Tabs queryString="os">
<TabItem value="suse" label="SUSE Linux Enterprise / openSUSE">


#### 防火墙
防火墙关闭为好:
```bash
systemctl disable firewalld --now
# 或
systemctl disable ufw --now
```
如果你想保持防火墙运行，默认情况喜爱，需要以下的规则：
```bash
firewall-cmd --permanent --add-port=6443/tcp #apiserver
firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16 #pods
firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16 #services
firewall-cmd --reload

# K3s 中 Pods     的默认 IP 地址范围 10.42.0.0/16
# K3s 中 Services 的默认 IP 地址范围 10.43.0.0/16
```
根据您的设置，可能需要打开其他端口。有关更多信息，请参阅 [入站规则](#inbound-rules-for-k3s-nodes)￼。如果您更改了 pods 或 services 的默认 CIDR，则需要相应地更新防火墙规则。

:::warning 旧的 RHEL/CentOS 发行版
RHEL 8.4 之前的操作系统版本包含一个已知的 NetworkManager bug，可能会干扰 K3s 的网络功能。如果使用的是较旧的版本，则需要禁用 nm-cloud-setup 并重启节点：
```bash
systemctl disable nm-cloud-setup.service nm-cloud-setup.timer
reboot
```

:::

</TabItem>
<TabItem value="debian" label="Ubuntu / Debian">
旧的 Debian 发行版可能有已知的iptables bug. 参见 [Known Issues](../known-issues.md#iptables).

#### UFW
建议关闭 ufw (uncomplicated firewall):
```bash
ufw disable
```
如果你想ufw运行，默认情况下，需要以下规则：
```bash
ufw allow 6443/tcp #apiserver
ufw allow from 10.42.0.0/16 to any #pods
ufw allow from 10.43.0.0/16 to any #services
```
根据您的设置，可能需要打开其他端口。有关更多信息，请参阅 [入站规则](#inbound-rules-for-k3s-nodes)￼。如果您更改了 pods 或 services 的默认 CIDR，则需要相应地更新防火墙规则。

</TabItem>
<TabItem value="pi" label="Raspberry Pi">

Raspberry Pi OS 基于 Debian, 可能有 iptables bug. 参见 [Known Issues](../known-issues.md#iptables).

#### Cgroups
Standard Raspberry Pi OS installations do not start with `cgroups` enabled. **K3S** needs `cgroups` to start the systemd service. `cgroups`can be enabled by appending `cgroup_memory=1 cgroup_enable=memory` to `/boot/cmdline.txt`.

标准的树莓派操作系统（Raspberry Pi OS）安装后默认不会启用 cgroups。而 K3s 需要 cgroups 才能启动其 systemd 服务。

启用方法： 在 `/boot/firmware/cmdline.txt` 文件的末尾追加以下参数： `cgroup_memory=1 cgroup_enable=memory`

cmdline.txt 示例:
```
console=serial0,115200 console=tty1 root=PARTUUID=58b06195-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait cgroup_memory=1 cgroup_enable=memory
```

#### Ubuntu Vxlan Module
在 Ubuntu 21.10 至 Ubuntu 23.10 版本中，树莓派上的 VXLAN 支持被移到了一个独立的内核模块中。如果您使用的是 Ubuntu 24.04 及更高版本，则不需要执行此步骤。

```bash
sudo apt install linux-modules-extra-raspi
```
</TabItem>
</Tabs>

有关哪些操作系统经过了 Rancher 托管的 K3s 集群测试的更多信息，请参阅 。
[Rancher support and maintenance terms.](https://rancher.com/support-maintenance-terms/)

## Hardware
硬件需求会根据您部署规模的大小而调整。最低配置要求如下：

|  Node  |   CPU   |  RAM   |
|--------|---------|--------|
| Server | 2 cores | 2 GB   |
| Agent  | 1 core  | 512 MB |


[Resource Profiling](../reference/resource-profiling.md) 
记录了测试与分析的结果，用以确定以下场景的最低资源需求：K3s Agent（工作节点）、带有工作负载的 K3s Server（控制平面），以及带有一个 Agent 的 K3s Server。

### 磁盘
K3s 的性能依赖于数据库的性能。为了确保最佳速度，我们建议尽可能使用 SSD。

如果在 Raspberry Pi 或其他 ARM 设备上部署 K3s，建议使用外部 SSD。因为 etcd 是写密集型的，SD 卡和 eMMC 无法承受 I/O 负载。

### 服务器配置指南

当服务器（控制平面 + etcd）节点的 CPU 和 RAM 资源有限时，加入的代理节点数量会受到标准工作负载条件的限制。

| Server CPU | Server RAM | Number of Agents |
| ---------- | ---------- | ---------------- |
| 2          | 4 GB       | 0-350            |
| 4          | 8 GB       | 351-900          |
| 8          | 16 GB      | 901-1800         |
| 16+        | 32 GB      | 1800+            |

:::tip 高可用性规划
当使用 3 个服务器节点的高可用性架构时，agent的数量可以比上表的数量大约增加 50%。
例如：3 个服务器，每个服务器配置 4 vCPU/8 GB 内存，可以支持大约 1200 个agent。
:::

建议将代理节点分批加入，每批不超过 50 个，以便让 CPU 有足够的空间释放，因为节点加入时会出现瞬时峰值。如果希望节点数量超过 255 个，请记得修改默认的 `cluster-cidr`！

[Resource Profiling](../reference/resource-profiling.md#server-sizing-requirements-for-k3s) 包含了更多关于这些建议是如何得出的信息。


## Networking
K3s 服务器需要所有节点都能访问 6443 端口。
当使用 Flannel VXLAN 后端时，节点需要能够通过 UDP 端口 8472 访问其他节点；当使用 Flannel WireGuard 后端时，节点需要能够通过 UDP 端口 51820（如果使用 IPv6，还需要 51821）访问其他节点。节点不应监听任何其他端口。K3s 使用反向隧道，节点会向服务器发起外部连接，所有的 kubelet 流量都会通过该隧道传输。然而，如果不使用 Flannel 并且提供自定义的 CNI，那么 K3s 不需要 Flannel 所需的端口。

如果你希望使用 metrics server，所有节点必须能够相互访问端口 10250。

如果计划通过嵌入式 etcd 实现高可用性，服务器节点必须能够相互访问端口 2379 和 2380。


:::tip Important  
The VXLAN port on nodes should not be exposed to the world as it opens up your cluster network to be accessed by anyone. Run your nodes behind a firewall/security group that disables access to port 8472.
:::

:::danger
Flannel relies on the [Bridge CNI plugin](https://www.cni.dev/plugins/current/main/bridge/) to create a L2 network that switches traffic. Rogue pods with `NET_RAW` capabilities can abuse that L2 network to launch attacks such as ARP spoofing. Therefore, as documented in the [Kubernetes docs](https://kubernetes.io/docs/concepts/security/pod-security-standards/), please set a restricted profile that disables `NET_RAW` on non-trustable pods.
:::

### Inbound Rules for K3s Nodes

| Protocol | Port      | Source    | Destination | Description
|----------|-----------|-----------|-------------|------------
| TCP      | 2379-2380 | Servers   | Servers     | Required only for HA with embedded etcd
| TCP      | 6443      | Agents    | Servers     | K3s supervisor and Kubernetes API Server
| UDP      | 8472      | All nodes | All nodes   | Required only for Flannel VXLAN
| TCP      | 10250     | All nodes | All nodes   | Kubelet metrics
| UDP      | 51820     | All nodes | All nodes   | Required only for Flannel Wireguard with IPv4
| UDP      | 51821     | All nodes | All nodes   | Required only for Flannel Wireguard with IPv6
| TCP      | 5001      | All nodes | All nodes   | Required only for embedded distributed registry (Spegel)
| TCP      | 6443      | All nodes | All nodes   | Required only for embedded distributed registry (Spegel)

Typically, all outbound traffic is allowed.

Additional changes to the firewall may be required depending on the OS used.

## Large Clusters

Hardware requirements are based on the size of your K3s cluster. For production and large clusters, we recommend using a high-availability setup with an external database. The following options are recommended for the external database in production:

- MySQL
- PostgreSQL
- etcd

### CPU and Memory

The following are the minimum CPU and memory requirements for nodes in a high-availability K3s server:

| Deployment Size |   Nodes   | vCPUs |  RAM  |
|:---------------:|:---------:|:-----:|:-----:|
|      Small      |  Up to 10 |   2   |  4 GB |
|      Medium     | Up to 100 |   4   |  8 GB |
|      Large      | Up to 250 |   8   | 16 GB |
|     X-Large     | Up to 500 |   16  | 32 GB |
|     XX-Large    |   500+    |   32  | 64 GB |

### Disks

The cluster performance depends on database performance. To ensure optimal speed, we recommend always using SSD disks to back your K3s cluster. On cloud providers, you will also want to use the minimum size that allows the maximum IOPS.

### Network

You should consider increasing the subnet size for the cluster CIDR so that you don't run out of IPs for the pods. You can do that by passing the `--cluster-cidr` option to K3s server upon starting.

### Database

K3s supports different databases including MySQL, PostgreSQL, MariaDB, and etcd.  See [Cluster Datastore](../datastore/datastore.md) for more info.

The following is a sizing guide for the database resources you need to run large clusters:

| Deployment Size |   Nodes   | vCPUs |  RAM  |
|:---------------:|:---------:|:-----:|:-----:|
|      Small      |  Up to 10 |   1   |  2 GB |
|      Medium     | Up to 100 |   2   |  8 GB |
|      Large      | Up to 250 |   4   | 16 GB |
|     X-Large     | Up to 500 |   8   | 32 GB |
|     XX-Large    |   500+    |   16  | 64 GB |
