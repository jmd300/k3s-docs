---
title: Configuration Options （配置选项）
---
这页重点介绍了在首次设置 K3s 时常用的选项。有关更深入的内容，请参考 [高级选项和配置](../advanced.md) 以及 [server](../cli/server.md)￼和 [agent](../cli/agent.md)￼ 命令的文档。

## Configuration with install script （使用安装脚本配置）
如 [快速入门指南](../quick-start.md)￼中所述，你可以使用 https://get.k3s.io 提供的安装脚本，在基于 systemd 和 openrc 的系统上将 K3s 安装为服务。

您可以使用 `INSTALL_K3S_EXEC`、`K3S_`环境变量和命令标志的组合，将配置传递给服务配置。

带前缀的环境变量、`INSTALL_K3S_EXEC` 的值和尾部的 shell 参数都会被保存到服务配置中。

安装后，可以通过编辑环境文件、编辑服务配置或使用新选项重新运行安装程序来更改配置。

为了说明这一点，以下命令都会导致相同的行为，即注册一个没有 `flannel` 带有 `token` 的服务器：

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -s - --flannel-backend none --token 12345
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --flannel-backend none" K3S_TOKEN=12345 sh -s -
curl -sfL https://get.k3s.io | K3S_TOKEN=12345 sh -s - server --flannel-backend none

# 未设置 K3S_URL 就被认为是 server 节点
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--flannel-backend none --token 12345" sh -s - 
curl -sfL https://get.k3s.io | sh -s - --flannel-backend none --token 12345
```

在注册代理时，以下命令都会导致相同的行为：

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent --server https://k3s.example.com --token mypassword" sh -s -
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent" K3S_TOKEN="mypassword" sh -s - --server https://k3s.example.com
curl -sfL https://get.k3s.io | K3S_URL=https://k3s.example.com sh -s - agent --token mypassword
curl -sfL https://get.k3s.io | K3S_URL=https://k3s.example.com K3S_TOKEN=mypassword sh -s - # agent is assumed because of K3S_URL
```

对于所有环境变量的细节，看[环境变量](../reference/env-variables.md)

:::info Note
如果在运行安装脚本时设置了配置，但在重新运行安装脚本时没有重新设置，原始值将会丢失。
[配置文件](#configuration-file) ￼的内容不由安装脚本管理。  

如果你希望你的配置独立于安装脚本，你应该使用配置文件，而不是将环境变量或参数传递给安装脚本。
:::

## Configuration with binary （使用二进制文件配置）
安装脚本主要用于将 K3s 配置为系统服务运行。  

如果你选择不使用安装脚本，可以通过从我们的 [GitHub release page](https://github.com/k3s-io/k3s/releases/latest) 下载二进制文件，将其放置在你的路径中并执行它来运行 K3s。这对于永久安装来说并不特别有用，但在进行快速测试时可能会有用，尤其是那些不需要将 K3s 作为系统服务来管理的测试。

```bash
curl -Lo /usr/local/bin/k3s https://github.com/k3s-io/k3s/releases/download/v1.26.5+k3s1/k3s; chmod a+x /usr/local/bin/k3s
```

你可以通过设置 K3S_ 环境变量来传递配置：
```bash
# K3s 配置文件的权限为 644
K3S_KUBECONFIG_MODE="644" k3s server
```

或者命令标志：
```bash
k3s server --write-kubeconfig-mode=644
```

k3s 代理也可以这样配置：
```bash
k3s agent --server https://k3s.example.com --token mypassword
```
配置 k3s server 的细节参考[`k3s server` documentation](../cli/server.md). 
配置 k3s agent 的细节参考[`k3s agent` documentation](../cli/agent.md).  

你还可以使用 --help 标志查看所有可用选项及其相应的环境变量。

:::info Matching Flags  

匹配服务器节点上的关键标志非常重要。例如，如果在主节点上使用 --disable servicelb 或 --cluster-cidr=10.200.0.0/16 标志，但在其他服务器节点上没有设置相同的标志，节点将无法加入集群。它们将打印类似如下的错误：
`failed to validate server configuration: critical configuration value mismatch.`
请参阅上面的服务器配置文档，了解哪些标志必须在服务器节点上保持一致。

:::

## Configuration with container image（使用容器镜像配置）
k3s 容器镜像（docker.io/rancher/k3s￼）支持与 GitHub 发布页面上提供的二进制文件相同的配置方法。

## Configuration File
除了使用环境变量和 CLI 参数配置 K3s 之外，K3s 还可以使用配置文件。  
配置文件会被加载，无论 k3s 是如何安装或执行的。

默认情况下，配置文件从 `/etc/rancher/k3s/config.yaml` 加载，且附加文件会按照字母顺序从 `/etc/rancher/k3s/config.yaml.d/*.yaml` 加载。

这个路径可以通过 `--config` CLI 标志或 `K3S_CONFIG_FILE` 环境变量进行配置。当覆盖默认配置文件名称时，附加文件目录路径也会随之修改。

下面是一个基本的 `server` 配置文件示例：

```yaml title="/etc/rancher/k3s/config.yaml"
# 这个配置指定了 Kubernetes 配置文件的权限，所有者可以读写文件，其他用户只能读取。
write-kubeconfig-mode: "0644"

# 这个配置指定了一个额外的 TLS Subject Alternative Name (SAN)，用于在 TLS 证书中添加额外的主机名。在这个例子中，foo.local 是一个 SAN，用于为该 K3s 服务器生成包含此名称的证书，便于在集群中的客户端与服务器进行安全通信。
tls-san:
  - "foo.local"
node-label:
  - "foo=bar"
  - "something=amazing"
cluster-init: true
```

“这相当于以下 CLI 参数：
```bash
k3s server \
  --write-kubeconfig-mode "0644"    \
  --tls-san "foo.local"             \
  --node-label "foo=bar"            \
  --node-label "something=amazing"  \
  --cluster-init
```
通常，CLI 参数映射到各自的 YAML 键，重复的 CLI 参数会在 YAML 文件中表示为列表。布尔标志在 YAML 文件中表示为 true 或 false。  

也可以同时使用配置文件和 CLI 参数。在这种情况下，值将从两个来源加载，但 CLI 参数会优先。对于可重复的参数，如    `--node-label`，CLI 参数会覆盖列表中的所有值。

### Value Merge Behavior（值合并行为）

If present in multiple files, the last value found for a given key will be used. A `+` can be appended to the key to append the value to the existing string or slice, instead of replacing it. All occurrences of this key in subsequent files will also require a `+` to prevent overwriting the accumulated value.

如果在多个文件中存在，给定键的最后一个值将被使用。可以在键后附加一个 +，将值追加到现有的字符串或切片中，而不是替换它。在后续文件中所有该键的出现也需要使用 +，以防止覆盖已累积的值。

下面是来自多个配置文件合并的值示例：
```yaml title="/etc/rancher/k3s/config.yaml"
token: boop
node-label:
  - foo=bar
  - bar=baz
```

```yaml title="/etc/rancher/k3s/config.yaml.d/test1.yaml"
write-kubeconfig-mode: 600
# 污点设置
node-taint:
  - alice=bob:NoExecute
```

```yaml title="/etc/rancher/k3s/config.yaml.d/test2.yaml"
# 根据加载顺序覆盖此配置
write-kubeconfig-mode: 777
node-label:
  - other=what
  - foo=three
# 补充污点配置
node-taint+:
  - charlie=delta:NoSchedule
```

这将导致最终配置为：
```yaml
write-kubeconfig-mode: 777
token: boop
# 覆盖后的配置
node-label:
  - other=what
  - foo=three
# 合并后的配置
node-taint:
  - alice=bob:NoExecute
  - charlie=delta:NoSchedule
```

## Putting it all together（将所有内容汇总在一起）
上述所有选项可以组合成一个单一的示例。  
一个 `config.yaml` 文件会被创建在 `/etc/rancher/k3s/config.yaml`：

```yaml
token: "secret"
debug: true
```
然后，安装脚本通过环境变量和标志的组合来运行：

```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="server" sh -s - 
# 禁用 Flannel 网络插件
--flannel-backend none
```
或你已经安装了k3s可执行文件  

这将导致一个具有以下配置的服务器：  
- kubeconfig 文件权限为 644  
- Flannel 后端设置为 none  
- Token 设置为 secret  
- 启用调试日志  

```bash
K3S_KUBECONFIG_MODE="644" k3s server --flannel-backend none
```

## Kubelet Configuration Files
Kubernetes 支持通过 CLI 标志和配置文件配置 kubelet。  

通过 CLI 标志配置 kubelet 已经被弃用很久，但它仍然被支持，并且是设置基本选项的最简单方法。
一些高级的 kubelet 配置只能通过配置文件进行设置。  

欲了解更多信息，请参阅 Kubernetes 文档中的 [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)￼ 和 [通过配置文件设置 kubelet 参数](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)。  

:::info Version Gate
对 kubelet drop-in（补丁式）配置文件或主配置文件的支持（即下文的选项 1 和 2）仅适用于 v1.32 及以上版本。   
对于较旧的版本，你应该直接使用 kubelet 命令行参数（即下文的选项 3）。
:::
K3s 使用一套默认的 kubelet 配置，该配置文件存储在  `/var/lib/rancher/k3s/agent/etc/kubelet.conf.d/00-k3s-defaults.conf` 路径下。如果你想修改这些默认配置参数，可以通过以下三种方式来实现：

1. 放一个 drop-in 配置文件到 `/var/lib/rancher/k3s/agent/etc/kubelet.conf.d/` *(推荐)*  

2. 通过使用这个标志 `--kubelet-arg=config=$PATHTOFILE`, `$PATHTOFILE` 是包含kubelet参数的配置文件  (e.g. `/etc/rancher/k3s/kubelet.conf`) 或者这个参数`--kubelet-arg=config-dir=$PATHTODIR`, `$PATHTODIR` 是一个文件夹路径，它包含包含kubelet配置参数的文件 (e.g. `/etc/rancher/k3s/kubelet.conf.d`)

3. 通过使用标志 `--kubelet-arg=$FLAG`，其中 `$FLAG` 是一个 kubelet 配置参数（例如 image-gc-high-threshold=100）。
image-gc-high-threshold=100：表示容器镜像垃圾回收的高阈值为 100%。当节点的容器镜像占用磁盘空间超过 100% 时，垃圾回收机制将触发。

在混合使用 kubelet CLI 标志和配置文件附加项时，请注意 [order of precedence](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/#kubelet-configuration-merging-order). 
