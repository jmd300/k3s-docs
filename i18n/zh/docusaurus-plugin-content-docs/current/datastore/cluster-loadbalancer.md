---
title: 集群负载均衡器
---
# Cluster Load Balancer

本节介绍了如何在具备高可用性 (HA) 的 K3s 集群服务端节点 (Server Nodes) 前端安装外部负载均衡器。文中提供了两个示例：Nginx 和 HAProxy。

:::tip
外部的 load-balancers 不应该与内置的 ServiceLB 混淆，它是一个内置的控制器，允许在不部署第三方 `load-balancer controller` 的情况下访问k8s的 LoadBalancer Service。
更多信息，看 [Service Load Balancer](../networking/networking-services.md#service-load-balancer). 

外部负载均衡器可用于为节点注册提供固定的注册地址，或用于外部访问 Kubernetes API Server。在暴露 LoadBalancer 类型的服务时，外部负载均衡器可以与 ServiceLB 配合使用或将其取而代之；但在大多数情况下，选择 MetalLB 或 Kube-VIP 等替代型负载均衡控制器会是更好的方案。
:::

## Prerequisites（前置条件）
示例中所有节点都运行在Ubuntu 20.04.
对于所有的例子，假设一个[HA K3s cluster with embedded etcd](../datastore/ha-embedded.md) 已经安装到了3个节点上。

每一个k3s 的 server 都配置了（未理解）：
```yaml
# /etc/rancher/k3s/config.yaml
token: lb-cluster-gd
tls-san: 10.10.10.100
```

节点的主机名和 IP 地址分别为：

* server-1: `10.10.10.50`
* server-2: `10.10.10.51`
* server-3: `10.10.10.52`


另外配置了两个用于负载均衡的节点，其主机名和 IP 地址分别为（非k3s集群的服务）：
* lb-1: `10.10.10.98`
* lb-2: `10.10.10.99`

还有三个额外的节点，主机名和 IP 分别是：
* agent-1: `10.10.10.101`
* agent-2: `10.10.10.102`
* agent-3: `10.10.10.103`

## Setup Load Balancer
<Tabs>
<TabItem value="HAProxy" default>

[HAProxy](http://www.haproxy.org/) 是一个提供 TCP 负载均衡功能的开源选项。它还支持负载均衡器自身的高可用（HA），从而确保集群各个层级都具备冗余性。更多信息请参阅 [HAProxy Documentation](http://docs.haproxy.org/2.8/intro.html) 官方文档。

此外，我们将使用 Keepalived 生成一个虚拟 IP (VIP)，该 IP 将用于访问集群。
查看更多信息 [KeepAlived Documentation](https://www.keepalived.org/manpage.html)。


1) 安装 HAProxy and KeepAlived:

```bash
sudo apt-get install haproxy keepalived
```

2) 增加以下内容到 lb-1 and lb-2 节点的 `/etc/haproxy/haproxy.cfg`:

```
frontend k3s-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend k3s-backend

backend k3s-backend
    mode tcp
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s
    server server-1 10.10.10.50:6443 check
    server server-2 10.10.10.51:6443 check
    server server-3 10.10.10.52:6443 check
```
3) 增加以下内容到 lb-1 and lb-2 节点的 `/etc/keepalived/keepalived.conf`:

```
vrrp_script chk_haproxy {
    script 'killall -0 haproxy' # faster than pidof
    interval 2
}

vrrp_instance haproxy-vip {
   interface eth1
    state <STATE> # MASTER on lb-1, BACKUP on lb-2
    priority <PRIORITY> # 200 on lb-1, 100 on lb-2

    virtual_router_id 51

    virtual_ipaddress {
        10.10.10.100/24
    }

    track_script {
        chk_haproxy
    }
}
```

6) 重启 lb-1 and lb-2 节点上的 HAProxy and KeepAlived:

```bash
systemctl restart haproxy
systemctl restart keepalived
```

5) 在agent节点上执行以下指令安装k3s 并加入集群：

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=lb-cluster-gd sh -s - agent --server https://10.10.10.100:6443
```
你现在可以在 server 上使用 `kubectl`与集群交互。
```bash
root@server-1 $ k3s kubectl get nodes -A
NAME       STATUS   ROLES                       AGE     VERSION
agent-1    Ready    <none>                      32s     v1.27.3+k3s1
agent-2    Ready    <none>                      20s     v1.27.3+k3s1
agent-3    Ready    <none>                      9s      v1.27.3+k3s1
server-1   Ready    control-plane,etcd,master   4m22s   v1.27.3+k3s1
server-2   Ready    control-plane,etcd,master   3m58s   v1.27.3+k3s1
server-3   Ready    control-plane,etcd,master   3m12s   v1.27.3+k3s1
```

</TabItem>

<TabItem value="Nginx">

## Nginx Load Balancer

:::danger
Nginx 本身并不原生支持高可用（HA）配置。在搭建高可用集群时，如果在 K3s 前端只设置一个负载均衡器，将会重新引入单点故障（Single Point of Failure）。
:::

[Nginx Open Source](http://nginx.org/) 提供了一个TCP的负载均衡器。 更多信息，看 [Using nginx as HTTP load balancer](https://nginx.org/en/docs/http/load_balancing.html) .

1) 在 lb-1 创建`nginx.conf` 文件，写入以下内容:

```
events {}

stream {
  upstream k3s_servers {
    server 10.10.10.50:6443;
    server 10.10.10.51:6443;
    server 10.10.10.52:6443;
  }

  server {
    listen 6443;
    proxy_pass k3s_servers;
  }
}
```

2) 在 lb-1 上启动Nginx负载均衡器:

使用 docker:

```bash
docker run -d --restart unless-stopped \
    -v ${PWD}/nginx.conf:/etc/nginx/nginx.conf \
    -p 6443:6443 \
    nginx:stable
```

或 [install nginx](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/) 然后启动:

```bash
cp nginx.conf /etc/nginx/nginx.conf
systemctl start nginx
```

3) 在 agent-1, agent-2, and agent-3, 运行以下指令安装 k3s 并加入集群:

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=lb-cluster-gd sh -s - agent --server https://10.10.10.99:6443
```

你现在可以在 server 节点上使用 `kubectl` 与集群交互.
```bash
root@server1 $ k3s kubectl get nodes -A
NAME       STATUS   ROLES                       AGE     VERSION
agent-1    Ready    <none>                      30s     v1.27.3+k3s1
agent-2    Ready    <none>                      22s     v1.27.3+k3s1
agent-3    Ready    <none>                      13s     v1.27.3+k3s1
server-1   Ready    control-plane,etcd,master   4m49s   v1.27.3+k3s1
server-2   Ready    control-plane,etcd,master   3m58s   v1.27.3+k3s1
server-3   Ready    control-plane,etcd,master   3m16s   v1.27.3+k3s1
```
</TabItem>
</Tabs>
