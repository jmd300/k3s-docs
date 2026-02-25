---
title: Private Registry Configuration（私有仓库配置）
---
容器
Containerd 可被设置链接私有仓库，并在kubelet需要的时候使用它们拉取镜像。

在启动时，K3s 会检查 `/etc/rancher/k3s/registries.yaml` 是否存在。如果存在，则在生成 containerd 配置时，会使用该文件中包含的仓库配置。

* 如果你想使用一个私有仓库作为一个公共仓库的镜像，比如docker.io, 那你需要在所有你想用这个镜像的节点上配置`registries.yaml`.

* 如果你的私有仓库需要认证，使用自己的TLS证书，或不使用TLS，你需要在所有你需要拉镜像的节点上设置`registries.yaml`.

请注意，服务器节点默认是可调度的。如果你没有为服务器节点添加污点（Taints），那在这些节点上将运行工作负载，请确保在每台服务器上也同样创建 `registries.yaml` 文件。

## Default Endpoint Fallback（隐式默认端点）

Containerd 为所有镜像库都保留了一个隐式的“ "default endpoint（拉镜像时仓库的地址）"。
潜台词：无论你怎么配置，Containerd 脑子里永远记着那个最初的、正版的下载地址，作为备用。

默认端点始终会被当作最后的保底手段，即使你在 `registries.yaml` 里配了其他地址。

重写（Rewrites）不适用于针对默认端点的拉取请求。

比如，当拉 `registry.example.com:5000/rancher/mirrored-pause:3.6`, containerd 将使用默认的地址 `https://registry.example.com:5000/v2`.

* 对于 `docker.io` 默认的地址是 `https://index.docker.io/v2`. 

* 对于其它仓库默认的地址是 `https://<REGISTRY>/v2`, `<REGISTRY>` 是仓库的hostname(域名/IP)和端口（可选）. 

为了被识别为一个仓库，镜像名的第一部分必须至少包含一个句号活或冒号。

因为历史原因，没有仓库地址的镜像名隐式指明是来自 `docker.io` 的。

:::info 版本说明
The `--disable-default-registry-endpoint` 选项自 2024 年 1 月发布的以下版本起可用： v1.26.13+k3s1, v1.27.10+k3s1, v1.28.6+k3s1, v1.29.1+k3s1
:::

节点启动时可以使用 `--disable-default-registry-endpoint` 选项。
当启用此选项时，containerd 将不再回退（fallback）到默认的仓库端点，而仅会从已配置的镜像仓库端点（Mirror Endpoints）拉取镜像；如果启用了分布式仓库（Distributed Registry），也会从中拉取。

如果你的集群处于完全物理隔离（Air-gapped）的环境中，且上游仓库（Upstream Registry）不可访问，或者你希望仅允许部分节点从上游仓库拉取镜像，那么该选项会非常有用。

禁用默认仓库端点的设置，仅对通过 `registries.yaml` 配置过的仓库生效。

如果某个仓库未在 `registries.yaml` 的镜像条目（mirror entry）中被明确配置，系统仍会执行默认的回退（fallback）行为（即尝试直接访问默认地址）。

## Registries Configuration File（仓库配置文件）

The file consists of two top-level keys, with subkeys for each registry:
该文件包含两个顶级键（top-level keys），每个仓库（registry）下面又分别包含各自的子键（subkeys）。
```yaml
mirrors:
  <REGISTRY>:
    endpoint:
      - https://<REGISTRY>/v2
configs:
  <REGISTRY>:
    auth:
      username: <BASIC AUTH USERNAME>
      password: <BASIC AUTH PASSWORD>
      token: <BEARER TOKEN>
    tls:
      # 是双向TLS，客户端需要生成私钥，和ca根证书签名的客户端证书
      # TLS CA 根证书
      ca_file: <PATH TO SERVER CA>
      # 客户端证书
      cert_file: <PATH TO CLIENT CERT>
      # 客户端私钥
      key_file: <PATH TO CLIENT KEY>
      # 如果为true，K3s 在连接仓库时，不再检查仓库的证书是否合法（比如证书是否过期、域名是否匹配、是否由受信任的 CA 签发）。
      insecure_skip_verify: <SKIP TLS CERT VERIFICATION BOOLEAN>
```

### Mirrors（镜像仓库）
`mirrors` 部分定义了各仓库（registries）的名称及其对应的端点（endpoints），例如：

```
mirrors:
  registry.example.com:
    endpoint:
      - "https://registry.example.com:5000"
```

每个镜像（mirror）都必须包含一个名称（name）和一组端点（endpoints）。当从某个仓库拉取镜像时，containerd 会尝试访问这些端点 URL，再加上默认的端点地址，并使用第一个可以正常工作的地址。

#### Redirects （重定向）

如果将私有仓库用作另一个仓库的镜像，比如，当配置一个 [pull through cache](https://docs.docker.com/registry/recipes/mirror/)时,
镜像的拉取是被透明地重定向到列出的endpoints的。原镜像的名字通过`ns` 参数传递给了镜像endpoint.

例如，如果你为 `docker.io`   配置了一个镜像（mirror）：
```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://registry.example.com:5000"
```

后续拉取 `docker.io/rancher/mirrored-pause:3.6` 时将按照这个镜像拉取： `registry.example.com:5000/rancher/mirrored-pause:3.6`.

#### Rewrites （重写）

每个镜像（mirror）都可以拥有一组重写规则（rewrites），这些规则利用正则表达式，在从镜像仓库拉取镜像时，对镜像名称进行匹配与转换。

如果私有仓库中的组织/项目结构与它所镜像的原始仓库不同，那么该功能会非常有用。
重写规则仅对“镜像名称”进行匹配和转换，而不涉及“标签（tag）”。

例如，通过以下配置，当拉取镜像 `docker.io/rancher/mirrored-pause:3.6` 时，系统会透明地将其替换为从 `registry.example.com:5000/mirrorproject/rancher-images/mirrored-pause:3.6` 拉取。

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://registry.example.com:5000"
    rewrite:
      # rancher开头的，匹配其第2部分，然后重写为另一个路径放到$1位置 
      "^rancher/(.*)": "mirrorproject/rancher-images/$1"
```

:::info 版本说明
自 2024 年 1 月发布的版本（v1.26.13+k3s1、v1.27.10+k3s1 等）起，重写规则（Rewrites）不再应用于[Default Endpoint](#default-endpoint-fallback) 。  
在这些版本之前，重写规则也会被应用于默认端点。这会导致一种情况：如果镜像无法从镜像端点拉取，且在原始仓库中也不存在经过改名后的镜像，K3s 就会无法从原始仓库拉取该镜像。
::::

如果你想在直接从某个仓库拉取镜像时应用重写规则（即：该仓库并非作为其它上游仓库的镜像时），你必须提供一个与默认端点不一致的镜像端点（mirror endpoint），（比如加个端口号），好让系统把它当成一个独立的镜像端点来看待。

在 `registries.yaml` 中，如果镜像端点（mirror endpoints）与默认端点完全匹配，则会被忽略；除非已禁用回退（fallback）机制，否则默认端点始终会被放在最后尝试，且不会应用任何重写规则。

例如，如果你有一个位于 `https://registry.example.com/` 的仓库，并且希望在显式拉取 `registry.example.com/rancher/mirrored-pause:3.6` 时应用重写规则，你可以添加一个带端口号的镜像端点。

因为该镜像端点与默认端点不匹配 —— **`"https://registry.example.com:443/v2" != "https://registry.example.com/v2"`** —— 尽管它实际上与默认端点指向同一个位置，但该端点仍会被接受为镜像（mirror），并应用重写规则。
```yaml
mirrors:
 registry.example.com
   endpoint:
     - "https://registry.example.com:443"
   rewrite:
     "^rancher/(.*)": "mirrorproject/rancher-images/$1"
```
请注意，在使用镜像（mirrors）和重写规则（rewrites）时，镜像在本地仍将以原始名称存储。  
例如，即便镜像是从名称不同的镜像端点拉取的，`crictl image ls` 仍会显示该镜像在节点上以 `docker.io/rancher/mirrored-pause:3.6` 形式存在。

### Configs

`configs` 部分定义了每个镜像仓库（mirror）的 TLS 和凭据（ credential ）配置。你可以为每个镜像仓库分别定义 `auth`（认证）和/或 `tls`（传输层安全协议）。

`tls` 部分包含以下内容：

| 配置项（Directive）| 解释（Description）                                                                          |
|------------------------|--------------------------------------------------------------------------------------|
| `cert_file`            | 将用于向镜像仓库进行身份验证的客户端证书路径。
| `key_file`             | 将用于向镜像仓库进行身份验证的客户端私钥路径。              |
| `ca_file`              | 定义 CA 证书路径，用于验证镜像仓库的服务器证书文件。 |
| `insecure_skip_verify` | 布尔值（True/False），定义是否跳过对该镜像仓库的 TLS 验证。          |

`auth` 部分包含
auth 部分由用户名/密码或身份验证令牌（token）组成：

| 配置项（Directive）  | 解释（Description）                                             |
|------------|---------------------------------------------------------|
| `username` | 私有镜像仓库基础认证（Basic Auth）的用户名            |
| `password` | 私有镜像仓库基础认证（Basic Auth）的用户密码          |
| `auth`     | 私有镜像仓库基础认证（Basic Auth）的身份验证令牌（Token） |

以下是在不同模式下使用私有镜像仓库的基础示例：

### Wildcard Support（通配符支持）

:::info 版本说明
通配符支持在2024年3月的版本可用: v1.26.15+k3s1, v1.27.12+k3s1, v1.28.8+k3s1, v1.29.3+k3s1
:::


`"*"` 通配符条目可用于 `mirrors（镜像）` 和 `configs（配置）`部分，为所有镜像仓库提供默认配置。

默认配置（通配符配置）仅在该镜像仓库没有具体对应的配置条目时才会被使用。请注意，星号（*）必须加引号。

在以下示例中，所有镜像仓库都将使用一个本地镜像仓库作为镜像源。除 `docker.io` 外，所有镜像仓库都将禁用 TLS 验证。
```yaml
mirrors:
  "*":
    endpoint:
      - "https://registry.example.com:5000"
configs:
  "docker.io":
  "*":
    tls:
      insecure_skip_verify: true
```

### With TLS
以下示例展示了在使用 TLS（安全传输层协议）时，如何在每个节点上配置  `/etc/rancher/k3s/registries.yaml` 文件。

<Tabs>
<TabItem value="With Authentication">

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://registry.example.com:5000"
configs:
  "registry.example.com:5000":
    auth:
      username: xxxxxx # this is the registry username
      password: xxxxxx # this is the registry password
    tls:
      cert_file: # path to the cert file used in the registry
      key_file:  # path to the key file used in the registry
      ca_file:   # path to the ca file used in the registry
```

</TabItem>
<TabItem value="Without Authentication">

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://registry.example.com:5000"
configs:
  "registry.example.com:5000":
    tls:
      cert_file: # path to the cert file used in the registry
      key_file:  # path to the key file used in the registry
      ca_file:   # path to the ca file used in the registry
```
</TabItem>
</Tabs>

### Without TLS

以下示例展示了在“不使用” TLS（即不使用 HTTPS/加密传输）时，如何在每个节点上配置  `/etc/rancher/k3s/registries.yaml` 文件。

<Tabs>
<TabItem value="With Authentication">

```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://registry.example.com:5000"
configs:
  "registry.example.com:5000":
    auth:
      username: xxxxxx # this is the registry username
      password: xxxxxx # this is the registry password
```

</TabItem>
<TabItem value="Without Authentication">

```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://registry.example.com:5000"
```
</TabItem>
</Tabs>

> 在没有 TLS 通信（非加密）的情况下，你需要在端点（endpoints）地址中明确指定 `http://`，否则系统会默认使用 https。

为了使镜像仓库（Registry）的修改生效，你需要在每个节点上重启 K3s。

## Troubleshooting Image Pulls（解决镜像拉取失败的问题）
当 Kubernetes 拉取镜像遇到问题时，Kubelet 显示的错误信息可能仅仅反映了：针对‘默认地址’发起的那次拉取尝试所返回的最终错误。这会产生一种假象，让你觉得你所配置的那些镜像仓库地址根本没被用到。

请查看位于节点上 `/var/lib/rancher/k3s/agent/containerd/containerd.log` 路径下的 containerd 日志，以获取有关故障根本原因的详细信息。  
请注意： 您必须查看该 Pod 所在节点上的日志。  
您可以执行以下命令来确认 Pod 被调度到了哪个节点：  
`kubectl get pod -o wide -n 命名空间 Pod名称`
然后查看输出结果中的 NODE 列。

## Adding Images to the Private Registry（向私有镜像仓库添加镜像）

将镜像镜像（Mirroring）到私有仓库需要一台安装了 Docker 或其他第三方工具的主机，并且该工具能够执行拉取（pull）和推送（push）镜像的操作。

以下步骤假设您拥有一台安装了 dockerd（Docker 守护进程）和 docker 命令行工具（CLI）的主机，并且该主机能够同时访问 docker.io（Docker Hub）和您的私有镜像仓库。

1. 从 GitHub 上获取您所使用的对应版本的 `k3s-images.txt` 文件。

2. 从 docker.io 拉取 k3s-images.txt 文件中列出的每一个 K3s 镜像。
示例：`docker pull docker.io/rancher/mirrored-pause:3.6`

3. 对镜像重新打标签为私有仓库的名字
示例：`docker tag docker.io/rancher/mirrored-pause:3.6 registry.example.com:5000/rancher/mirrored-pause:3.6`

4. 将镜像push到私有仓库  
私有仓库的推送可能需要先 `docker login 仓库地址`  
示例：`docker push registry.example.com:5000/rancher/mirrored-pause:3.6`