---
title: 高级选项 / 配置
---
# Advanced Options / Configuration

本章节包含高级进阶信息，详细描述了运行和管理 K3s 的不同方式，以及为 K3s 准备宿主机操作系统所需的必要步骤。

## 证书管理

### Certificate Authority Certificates
k3s 在启动首个服务器节点（Server Node）期间，会生成自签名的证书颁发机构（CA）证书。这些 CA 证书的有效期为 10 年，且不会自动续期。

有关使用自定义 CA 证书或续订自签名 CA 证书的信息，请参阅 [`k3s certificate rotate-ca` command documentation](./cli/certificate.md#certificate-authority-ca-certificates).

### 客户端和服务器证书
K3s 的客户端和服务端证书自颁发之日起 365 天内有效。每当 K3s 启动时，任何已过期或距离过期不足 90 天的证书都会被自动续订。

有关手动轮换客户端和服务端证书的信息，请参阅 [`k3s certificate rotate` command documentation](./cli/certificate.md#client-and-server-certificates).

CA (Certificate Authority) 证书是“根证书”。在 K3s 集群里，它相当于这个集群内部的最高认证机构。

## Token Management

默认情况下，K3s 为 Server 和 Agent 统一使用一个静态 Token。在集群创建后，可以通过谨慎的操作对该 Token 进行轮换。
此外，还可以启用第二个静态 Token（仅限 Agent 加入时使用），或者创建像 kubeadm 那样会自动过期的临时加入 Token。

有关更多信息，参阅：[`k3s token` command documentation](./cli/token.md#k3s-token-1).

## Configuring DNS Resolution

### Nameserver 有效性检查

在启动时，每个节点都会检查 `/etc/resolv.conf` 和 `/run/systemd/resolve/resolv.conf` 文件，查看其中是否包含 loopback（回环地址）、multicast（组播地址）或 link-local（链路本地地址）的 nameservers。
如果存在任何此类条目，该配置文件将不会被使用。因为这些条目在那些从其节点  的 pods 中无法正常工作， [inherit name resolution configuration](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy)。

如果未找到可用的 resolv.conf，K3s 将在日志中打印一条警告消息，并生成一个 stub resolv.conf（存根配置文件），其使用的  nameservers 为 `8.8.8.8` 和 `2001:4860:4860::8888`。

如果你希望为 K3s 提供替代的解析器配置，而不修改系统配置文件，可以使用 --resolv-conf 选项来指定一个合适文件的路径。
手动指定的解析器配置文件不受“可用性检查”的限制。

### CoreDNS 自定义配置导入

为了自定义 CoreDNS 配置，你可以在 `kube-system` 命名空间下创建一个名为 `coredns-custom` 的 ConfigMap。
文件名（键名）符合 `*.override` 模式的内容，将被导入到 `:.53` 服务器块（Server Block）中。
额外的“服务器块（Server Blocks）”可以存放在匹配 *.server 的键（keys）中。
其他补充内容（如区域文件/Zone files 等）也可以存在该配置中，它们将被挂载到 CoreDNS Pod 内的 /etc/coredns/custom 目录下。

Here is an example ConfigMap that forwards lookups to `example.com` to a nameserver at 10.0.0.1, and serves `example.net` from an [RFC 1035](https://datatracker.ietf.org/doc/html/rfc1035#section-5) compliant text file:

这是一个 ConfigMap 示例，它将针对 example.com 的查询请求转发至位于 10.0.0.1 的域名服务器，并基于一个符合 [RFC 1035] (https://datatracker.ietf.org/doc/html/rfc1035#section-5) 标准的文本文件为 example.net 提供解析服务。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  example-com.override: |
    forward example.com 10.0.0.1
  example-net.server: |
    example.net:53 {
      log
      errors
      file /etc/coredns/custom/db.example.net
    }
  db.example.net: |
    $ORIGIN example.net.
    @       3600 IN SOA    sns.dns.icann.org. noc.dns.icann.org. 2017042745 7200 3600 1209600 3600
            3600 IN NS     a.iana-servers.net.
            3600 IN NS     b.iana-servers.net.
    www          IN A      127.0.0.1
                 IN AAAA   ::1
```

## 配置一个HTTP代理
如果你在只能通过 HTTP 代理连接外网的环境中运行 K3s，你可以在 K3s 的 systemd 服务中配置代理设置。这些代理设置将被 K3s 采用，并向下传递给嵌入式的 containerd 和 kubelet。注意：宿主机的代理配置和其他环境变量并不会自动传递给 Pod。

K3s 安装脚本会自动从当前 Shell 环境中获取 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY`，以及 `CONTAINERD_HTTP_PROXY`、`CONTAINERD_HTTPS_PROXY` 和 `CONTAINERD_NO_PROXY` 变量（如果它们存在的话），并将它们写入到 systemd 服务的环境配置文件中，该文件通常位于：

- `/etc/systemd/system/k3s.service.env`
- `/etc/systemd/system/k3s-agent.service.env`

当然，你也可以通过编辑这些文件来配置代理。

K3s 会自动将集群内部的 Pod 和 Service IP 段以及集群 DNS 域名添加到 NO_PROXY 列表项中。你应该确保 Kubernetes 节点自身所使用的 IP 地址范围（即节点的公网和私网 IP）也被包含在 NO_PROXY 列表中，或者确保这些节点能够通过代理访问。

```
HTTP_PROXY=http://your-proxy.example.com:8888
HTTPS_PROXY=http://your-proxy.example.com:8888
NO_PROXY=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
```
如果你想在不影响 K3s 和 Kubelet 的情况下为 containerd 配置代理设置，可以给变量加上 `CONTAINERD_` 前缀。

```
CONTAINERD_HTTP_PROXY=http://your-proxy.example.com:8888
CONTAINERD_HTTPS_PROXY=http://your-proxy.example.com:8888
CONTAINERD_NO_PROXY=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
```

## 使用 Docker 作为容器运行时
K3s 包含并默认使用 [containerd](https://containerd.io/)，这是一种行业标准的容器运行时。  
从 Kubernetes 1.24 版本开始，Kubelet 不再包含 dockershim —— 这是一个允许 Kubelet 与 dockerd（Docker 守护进程）进行通信的组件。  
K3s 1.24 及更高版本集成了 [cri-dockerd](https://github.com/Mirantis/cri-dockerd)，这使得用户在继续使用 Docker 容器运行时的同时，能够从旧版本的 K3s 实现无缝升级。  

若要使用 Docker 而非 containerd：

1. 在k3s节点上安装Docker. Rancher的 [Docker installation scripts](https://github.com/rancher/install-docker) 可以用来安装docker:

    ```bash
    curl https://releases.rancher.com/install-docker/20.10.sh | sh
    ```

2. 使用 `--docker` 选项安装 k3s:

    ```bash
    curl -sfL https://get.k3s.io | sh -s - --docker
    ```

3. 确认集群是可用的

    ```bash
    $ sudo k3s kubectl get pods --all-namespaces
    NAMESPACE     NAME                                     READY   STATUS      RESTARTS   AGE
    kube-system   local-path-provisioner-6d59f47c7-lncxn   1/1     Running     0          51s
    kube-system   metrics-server-7566d596c8-9tnck          1/1     Running     0          51s
    kube-system   helm-install-traefik-mbkn9               0/1     Completed   1          51s
    kube-system   coredns-8655855d6-rtbnb                  1/1     Running     0          51s
    kube-system   svclb-traefik-jbmvl                      2/2     Running     0          43s
    kube-system   traefik-758cd5fc85-2wz97                 1/1     Running     0          43s
    ```

4. 确认docker的容器是运行的:

    ```bash
    $ sudo docker ps
    CONTAINER ID        IMAGE                     COMMAND                  CREATED              STATUS              PORTS               NAMES
    3e4d34729602        897ce3c5fc8f              "entry"                  About a minute ago   Up About a minute                       k8s_lb-port-443_svclb-traefik-jbmvl_kube-system_d46f10c6-073f-4c7e-8d7a-8e7ac18f9cb0_0
    bffdc9d7a65f        rancher/klipper-lb        "entry"                  About a minute ago   Up About a minute                       k8s_lb-port-80_svclb-traefik-jbmvl_kube-system_d46f10c6-073f-4c7e-8d7a-8e7ac18f9cb0_0
    436b85c5e38d        rancher/library-traefik   "/traefik --configfi…"   About a minute ago   Up About a minute                       k8s_traefik_traefik-758cd5fc85-2wz97_kube-system_07abe831-ffd6-4206-bfa1-7c9ca4fb39e7_0
    de8fded06188        rancher/pause:3.1         "/pause"                 About a minute ago   Up About a minute                       k8s_POD_svclb-traefik-jbmvl_kube-system_d46f10c6-073f-4c7e-8d7a-8e7ac18f9cb0_0
    7c6a30aeeb2f        rancher/pause:3.1         "/pause"                 About a minute ago   Up About a minute                       k8s_POD_traefik-758cd5fc85-2wz97_kube-system_07abe831-ffd6-4206-bfa1-7c9ca4fb39e7_0
    ae6c58cab4a7        9d12f9848b99              "local-path-provisio…"   About a minute ago   Up About a minute                       k8s_local-path-provisioner_local-path-provisioner-6d59f47c7-lncxn_kube-system_2dbd22bf-6ad9-4bea-a73d-620c90a6c1c1_0
    be1450e1a11e        9dd718864ce6              "/metrics-server"        About a minute ago   Up About a minute                       k8s_metrics-server_metrics-server-7566d596c8-9tnck_kube-system_031e74b5-e9ef-47ef-a88d-fbf3f726cbc6_0
    4454d14e4d3f        c4d3d16fe508              "/coredns -conf /etc…"   About a minute ago   Up About a minute                       k8s_coredns_coredns-8655855d6-rtbnb_kube-system_d05725df-4fb1-410a-8e82-2b1c8278a6a1_0
    c3675b87f96c        rancher/pause:3.1         "/pause"                 About a minute ago   Up About a minute                       k8s_POD_coredns-8655855d6-rtbnb_kube-system_d05725df-4fb1-410a-8e82-2b1c8278a6a1_0
    4b1fddbe6ca6        rancher/pause:3.1         "/pause"                 About a minute ago   Up About a minute                       k8s_POD_local-path-provisioner-6d59f47c7-lncxn_kube-system_2dbd22bf-6ad9-4bea-a73d-620c90a6c1c1_0
    64d3517d4a95        rancher/pause:3.1         "/pause"
    ```

## Using etcdctl
etcdctl 提供了与 etcd 服务器交互的命令行界面（CLI）。K3s 并没有捆绑（内置）etcdctl。
如果你想使用 etcdctl 与k3s 内置的 etcd 交互，根据[official documentation](https://etcd.io/docs/latest/install/) 安装 etcdctl。

```bash
ETCD_VERSION="v3.5.5"
ETCD_URL="https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz"
curl -sL ${ETCD_URL} | sudo tar -zxv --strip-components=1 -C /usr/local/bin
```

You may then use etcdctl by configuring it to use the K3s-managed certificates and keys for authentication:

```bash
sudo etcdctl version \
  --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/k3s/server/tls/etcd/client.key
```

## Configuring containerd

:::info Version Gate
K3s includes containerd 2.0 as of the February 2025 releases: v1.31.6+k3s1 and v1.32.2+k3s1.  
Be aware that containerd 2.0 prefers config version 3, while containerd 1.7 prefers config version 2.
:::

K3s will generate a configuration file for containerd at `/var/lib/rancher/k3s/agent/etc/containerd/config.toml`, using values specific to the current cluster and node configuration.

For advanced customization, you can create a containerd config template in the same directory:
* For containerd 2.0, place a version 3 configuration template in `config-v3.toml.tmpl`  
  See the [containerd 2.0 documentation](https://github.com/containerd/containerd/blob/release/2.0/docs/cri/config.md) for more information.
* For containerd 1.7 and earlier, place a version 2 configuration template in `config.toml.tmpl`  
  See the [containerd 1.7 documentation](https://github.com/containerd/containerd/blob/release/1.7/docs/cri/config.md) for more information.

Containerd 2.0 is backwards compatible with prior config versions, and k3s will continue to render legacy version 2 configuration from `config.toml.tmpl` if `config-v3.toml.tmpl` is not found.

The template file is rendered into the containerd config using the [`text/template`](https://pkg.go.dev/text/template) library.
See `ContainerdConfigTemplateV3` and `ContainerdConfigTemplate` in [`templates.go`](https://github.com/k3s-io/k3s/blob/main/pkg/agent/templates/templates.go) for the default template content.
The template is executed with a [`ContainerdConfig`](https://github.com/k3s-io/k3s/blob/main/pkg/agent/templates/templates.go#L22-L33) struct as its dot value (data argument).

### Base template

You can extend the K3s base template instead of copy-pasting the complete stock template out of the K3s source code. This is useful if you only need to build on the existing configuration by adding a few extra lines before or after the defaults.

```toml title="/var/lib/rancher/k3s/agent/etc/containerd/config-v3.toml.tmpl"
{{ template "base" . }}

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.'custom']
  runtime_type = "io.containerd.runc.v2"
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.'custom'.options]
  BinaryName = "/usr/bin/custom-container-runtime"
  SystemdCgroup = true
```

:::warning
For best results, do NOT simply copy a prerendered `config.toml` into the template and make your desired changes. Use the base template, or provide a full template based on the k3s defaults linked above.
:::

## Alternative Container Runtime Support

K3s will automatically detect alternative container runtimes if they are present when K3s starts. Supported container runtimes are:
```
crun, lunatic, nvidia, nvidia-cdi, nvidia-experimental, slight, spin, wasmedge, wasmer, wasmtime, wws
```

K3s uses the service's `PATH` environment variable to search for container runtime executables.
If an installed container runtime is not detected by K3s, ensure it is present in a system path, which generally includes:  
`/usr/local/sbin /usr/local/bin /usr/sbin /usr/bin /sbin /bin`

### NVIDIA Container Runtime

NVIDIA GPUs require installation of the NVIDIA Container Runtime in order to schedule and run accelerated workloads in Pods. To use NVIDIA GPUs with K3s, perform the following steps:

1. Install the nvidia-container package repository on the node by following the instructions at:  
    https://nvidia.github.io/libnvidia-container/
1. Install the nvidia container runtime packages. For example:  
   `apt install -y nvidia-container-runtime cuda-drivers-fabricmanager-515 nvidia-headless-515-server`
1. [Install K3s](./installation), or restart it if already installed.
1. Confirm that the nvidia container runtime has been found by k3s:  
    `grep nvidia /var/lib/rancher/k3s/agent/etc/containerd/config.toml`

If these steps are followed properly, K3s will automatically add NVIDIA runtimes to the containerd configuration, depending on what runtime executables are found.

K3s includes Kubernetes RuntimeClass definitions for all supported alternative runtimes. You can select one of these to replace `runc` as the default runtime on a node by setting the `--default-runtime` value via the k3s CLI or config file.

If you have not changed the default runtime on your GPU nodes, you must explicitly request the NVIDIA runtime by setting `runtimeClassName: nvidia` in the Pod spec:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nbody-gpu-benchmark
  namespace: default
spec:
  restartPolicy: OnFailure
  runtimeClassName: nvidia
  containers:
  - name: cuda-container
    image: nvcr.io/nvidia/k8s/cuda-sample:nbody
    args: ["nbody", "-gpu", "-benchmark"]
    resources:
      limits:
        nvidia.com/gpu: 1
    env:
    - name: NVIDIA_VISIBLE_DEVICES
      value: all
    - name: NVIDIA_DRIVER_CAPABILITIES
      value: all
```

Note that the NVIDIA Container Runtime is also frequently used with [NVIDIA Device Plugin](https://github.com/NVIDIA/k8s-device-plugin/), with modifications to ensure that pod specs include `runtimeClassName: nvidia`, as mentioned above.

## Running Agentless Servers (Experimental)
> **Warning:** This feature is experimental.

When started with the `--disable-agent` flag, servers do not run the kubelet, container runtime, or CNI. They do not register a Node resource in the cluster, and will not appear in `kubectl get nodes` output.
Because they do not host a kubelet, they cannot run pods or be managed by operators that rely on enumerating cluster nodes, including the embedded etcd controller and the system upgrade controller.

Running agentless servers may be advantageous if you want to obscure your control-plane nodes from discovery by agents and workloads, at the cost of increased administrative overhead caused by lack of cluster operator support.

By default, the apiserver on agentless servers will not be able to make outgoing connections to admission webhooks or aggregated apiservices running within the cluster. To remedy this, set the `--egress-selector-mode` server flag to either `pod` or `cluster`. If you are changing this flag on an existing cluster, you'll need to restart all nodes in the cluster for the option to take effect.

## Running Rootless Servers (Experimental)
> **Warning:** This feature is experimental.

Rootless mode allows running K3s servers as an unprivileged user, so as to protect the real root on the host from potential container-breakout attacks.

See https://rootlesscontaine.rs/ to learn more about Rootless Kubernetes.

### Known Issues with Rootless mode

* **Ports**

  When running rootless a new network namespace is created.  This means that K3s instance is running with networking fairly detached from the host.
  The only way to access Services run in K3s from the host is to set up port forwards to the K3s network namespace.
  Rootless K3s includes controller that will automatically bind 6443 and service ports below 1024 to the host with an offset of 10000.

  For example, a Service on port 80 will become 10080 on the host, but 8080 will become 8080 without any offset. Currently, only LoadBalancer Services are automatically bound.

* **Cgroups**

  Cgroup v1 and Hybrid v1/v2 are not supported; only pure Cgroup v2 is supported. If K3s fails to start due to missing cgroups when running rootless, it is likely that your node is in Hybrid mode, and the "missing" cgroups are still bound to a v1 controller.

* **Multi-node/multi-process cluster**

  Multi-node rootless clusters, or multiple rootless k3s processes on the same node, are not currently supported. See [#6488](https://github.com/k3s-io/k3s/issues/6488) for more details.

### Starting Rootless Servers
* Enable cgroup v2 delegation, see https://rootlesscontaine.rs/getting-started/common/cgroup2/ .
  This step is required; the rootless kubelet will fail to start without the proper cgroups delegated.

* On Ubuntu or other distributions with AppArmor support, you must allow the K3s binary to run unconfined:
  ```bash
  cat <<EOF | sudo tee "/etc/apparmor.d/usr.local.bin.k3s"
  abi <abi/4.0>,
  include <tunables/global>

  /usr/local/bin/k3s flags=(unconfined) {
    userns,

    include if exists <local/usr.local.bin.k3s>
  }
  EOF

  sudo systemctl restart apparmor.service
  ```

* Download `k3s-rootless.service` from [`https://github.com/k3s-io/k3s/blob/main/k3s-rootless.service`](https://github.com/k3s-io/k3s/blob/main/k3s-rootless.service).

* Install `k3s-rootless.service` to `~/.config/systemd/user/k3s-rootless.service`.
  Installing this file as a system-wide service (`/etc/systemd/...`) is not supported.
  Depending on the path to the `k3s` binary, you might need to modify the `ExecStart=/usr/local/bin/k3s ...` line of the file.

* Run `systemctl --user daemon-reload`

* Run `systemctl --user enable --now k3s-rootless`

* Run `KUBECONFIG=~/.kube/k3s.yaml kubectl get pods -A`, and make sure the pods are running.

> **Note:** Don't try to run `k3s server --rootless` on a terminal, as terminal sessions do not allow cgroup v2 delegation.
> If you really need to try it on a terminal, use `systemd-run --user -p Delegate=yes --tty k3s server --rootless` to wrap it in a systemd scope.

### Advanced Rootless Configuration

Rootless K3s uses [rootlesskit](https://github.com/rootless-containers/rootlesskit) and [slirp4netns](https://github.com/rootless-containers/slirp4netns) to communicate between host and user network namespaces.
Some of the configuration used by rootlesskit and slirp4nets can be set by environment variables. The best way to set these is to add them to the `Environment` field of the k3s-rootless systemd unit.

| Variable                             | Default      | Description
|--------------------------------------|--------------|------------
| `K3S_ROOTLESS_MTU`                   | 1500         | Sets the MTU for the slirp4netns virtual interfaces.
| `K3S_ROOTLESS_CIDR`                  | 10.41.0.0/16 | Sets the CIDR used by slirp4netns virtual interfaces.
| `K3S_ROOTLESS_ENABLE_IPV6`           | autotedected | Enables slirp4netns IPv6 support. If not specified, it is automatically enabled if K3s is configured for dual-stack operation.
| `K3S_ROOTLESS_PORT_DRIVER`           | builtin      | Selects the rootless port driver; either `builtin` or `slirp4netns`. Builtin is faster, but masquerades the original source address of inbound packets.
| `K3S_ROOTLESS_DISABLE_HOST_LOOPBACK` | true         | Controls whether or not access to the hosts's loopback address via the gateway interface is enabled. It is recommended that this not be changed, for security reasons.

### Troubleshooting Rootless

* Run `systemctl --user status k3s-rootless` to check the daemon status
* Run `journalctl --user -f -u k3s-rootless` to see the daemon log
* See also https://rootlesscontaine.rs/

## Node Labels and Taints

K3s agents can be configured with the options `--node-label` and `--node-taint` which adds a label and taint to the kubelet. The two options only add labels and/or taints [at registration time](./cli/agent.md#node-labels-and-taints-for-agents), so they can only be set when the node is first joined to the cluster.

All current versions of Kubernetes restrict nodes from registering with most labels with `kubernetes.io` and `k8s.io` prefixes, specifically including the `kubernetes.io/role` label. If you attempt to start a node with a disallowed label, K3s will fail to start. As stated by the Kubernetes authors:

> Nodes are not permitted to assert their own role labels. Node roles are typically used to identify privileged or control plane types of nodes, and allowing nodes to label themselves into that pool allows a compromised node to trivially attract workloads (like control plane daemonsets) that confer access to higher privilege credentials.

See [SIG-Auth KEP 279](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/279-limit-node-access/README.md) for more information.

If you want to change node labels and taints after node registration, or add reserved labels, you should use `kubectl`. Refer to the official Kubernetes documentation for details on how to add [taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) and [node labels.](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/#add-a-label-to-a-node)

## Starting the Service with the Installation Script

The installation script will auto-detect if your OS is using systemd or openrc and enable and start the service as part of the installation process.
* When running with openrc, logs will be created at `/var/log/k3s.log`. 
* When running with systemd, logs will be created in `/var/log/syslog` and viewed using `journalctl -u k3s` (or `journalctl -u k3s-agent` on agents).

An example of disabling auto-starting and service enablement with the install script:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_SKIP_START=true INSTALL_K3S_SKIP_ENABLE=true sh -
```

## Running K3s in Docker

There are several ways to run K3s in Docker:

<Tabs>
<TabItem value='K3d' default>

[k3d](https://github.com/k3d-io/k3d) is a utility designed to easily run multi-node K3s clusters in Docker.

k3d makes it very easy to create single- and multi-node k3s clusters in docker, e.g. for local development on Kubernetes.

See the [Installation](https://k3d.io/#installation) documentation for more information on how to install and use k3d.

</TabItem>
<TabItem value="Docker">

To use Docker, `rancher/k3s` images are also available to run the K3s server and agent.
Using the `docker run` command:

```bash
sudo docker run \
  --privileged \
  --name k3s-server-1 \
  --hostname k3s-server-1 \
  -p 6443:6443 \
  -d rancher/k3s:v1.24.10-k3s1 \
  server
```
:::note
You must specify a valid K3s version as the tag; the `latest` tag is not maintained.  
Docker images do not allow a `+` sign in tags, use a `-` in the tag instead.
:::

Once K3s is up and running, you can copy the admin kubeconfig out of the Docker container for use:
```bash
sudo docker cp k3s-server-1:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

</TabItem>
</Tabs>

## SELinux Support

If you are installing K3s on a system where SELinux is enabled by default (such as CentOS), you must ensure the proper SELinux policies have been installed. 

<Tabs>
<TabItem value="Automatic Installation" default>

The [install script](./installation/configuration.md#configuration-with-install-script) will automatically install the SELinux RPM from the Rancher RPM repository if on a compatible system if not performing an air-gapped install. Automatic installation can be skipped by setting `INSTALL_K3S_SKIP_SELINUX_RPM=true`.

</TabItem>

<TabItem value="Manual Installation">

The necessary policies can be installed with the following commands:
```bash
yum install -y container-selinux selinux-policy-base
yum install -y https://rpm.rancher.io/k3s/latest/common/centos/9/noarch/k3s-selinux-1.6-1.el9.noarch.rpm
```

To force the install script to log a warning rather than fail, you can set the following environment variable: `INSTALL_K3S_SELINUX_WARN=true`.
</TabItem>
</Tabs>

### Enabling SELinux Enforcement

To leverage SELinux, specify the `--selinux` flag when starting K3s servers and agents or setting the K3S_SELINUX=true environment variable.
  
This option can also be specified in the K3s [configuration file](./installation/configuration.md#configuration-file).

```
selinux: true
```

Using a custom `--data-dir` under SELinux is not supported. To customize it, you would most likely need to write your own custom policy. For guidance, you could refer to the [containers/container-selinux](https://github.com/containers/container-selinux) repository, which contains the SELinux policy files for Container Runtimes, and the [k3s-io/k3s-selinux](https://github.com/k3s-io/k3s-selinux) repository, which contains the SELinux policy for K3s.

## Enabling Lazy Pulling of eStargz (Experimental)

### What's lazy pulling and eStargz?

Pulling images is known as one of the time-consuming steps in the container lifecycle.
According to [Harter, et al.](https://www.usenix.org/conference/fast16/technical-sessions/presentation/harter),

> pulling packages accounts for 76% of container start time, but only 6.4% of that data is read

To address this issue, k3s experimentally supports *lazy pulling* of image contents.
This allows k3s to start a container before the entire image has been pulled.
Instead, the necessary chunks of contents (e.g. individual files) are fetched on-demand. 
Especially for large images, this technique can shorten the container startup latency.

To enable lazy pulling, the target image needs to be formatted as [*eStargz*](https://github.com/containerd/stargz-snapshotter/blob/main/docs/stargz-estargz.md).
This is an OCI-alternative but 100% OCI-compatible image format for lazy pulling.
Because of the compatibility, eStargz can be pushed to standard container registries (e.g. ghcr.io) as well as this is *still runnable* even on eStargz-agnostic runtimes.

eStargz is developed based on the [stargz format proposed by Google CRFS project](https://github.com/google/crfs) but comes with practical features including content verification and performance optimization.
For more details about lazy pulling and eStargz, please refer to [Stargz Snapshotter project repository](https://github.com/containerd/stargz-snapshotter).

### Configure k3s for lazy pulling of eStargz

As shown in the following, `--snapshotter=stargz` option is needed for k3s server and agent.

```bash
k3s server --snapshotter=stargz
```

With this configuration, you can perform lazy pulling for eStargz-formatted images.
The following example Pod manifest uses eStargz-formatted `node:13.13.0` image (`ghcr.io/stargz-containers/node:13.13.0-esgz`).
When the stargz snapshotter is enabled, K3s performs lazy pulling for this image.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodejs
spec:
  containers:
  - name: nodejs-estargz
    image: ghcr.io/stargz-containers/node:13.13.0-esgz
    command: ["node"]
    args:
    - -e
    - var http = require('http');
      http.createServer(function(req, res) {
        res.writeHead(200);
        res.end('Hello World!\n');
      }).listen(80);
    ports:
    - containerPort: 80
```

## Additional Logging Sources

[Rancher logging](https://ranchermanager.docs.rancher.com/integrations-in-rancher/logging/logging-helm-chart-options) for K3s can be installed without using Rancher. The following instructions should be executed to do so:

```bash
helm repo add rancher-charts https://charts.rancher.io
helm repo update
helm install --create-namespace -n cattle-logging-system rancher-logging-crd rancher-charts/rancher-logging-crd
helm install --create-namespace -n cattle-logging-system rancher-logging --set additionalLoggingSources.k3s.enabled=true rancher-charts/rancher-logging
```

## Additional Network Policy Logging

Packets dropped by network policies can be logged. The packet is sent to the iptables NFLOG action, which shows the packet details, including the network policy that blocked it.

If there is a lot of traffic, the number of log messages could be very high. To control the log rate on a per-policy basis, set the `limit` and `limit-burst` iptables parameters by adding the following annotations to the network policy in question:
* `kube-router.io/netpol-nflog-limit=<LIMIT-VALUE>`
* `kube-router.io/netpol-nflog-limit-burst=<LIMIT-BURST-VALUE>`

Default values are `limit=10/minute` and `limit-burst=10`. Check the [iptables manual](https://www.netfilter.org/documentation/HOWTO/packet-filtering-HOWTO-7.html#:~:text=restrict%20the%20rate%20of%20matches) for more information on the format and possible values for these fields.

To convert NFLOG packets to log entries, install ulogd2 and configure `[log1]` to read on `group=100`. Then, restart the ulogd2 service for the new config to be committed.
When a packet is blocked by network policy rules, a log message will appear in `/var/log/ulog/syslogemu.log`.

Packets sent to the NFLOG netlink socket can also be read by using command-line tools like tcpdump or tshark:
```bash
tcpdump -ni nflog:100
```
While more readily available, tcpdump will not show the name of the network policy that blocked the packet. Use wireshark's tshark command instead to display the full NFLOG packet header, including the `nflog.prefix` field that contains the policy name.

Network Policy logging of dropped packets does not support [policies with an empty `podSelector`](https://github.com/k3s-io/k3s/issues/8008). If you rely on logging dropped packets for diagnostic or audit purposes, ensure that your policies include a pod selector that matches the affected pods.
