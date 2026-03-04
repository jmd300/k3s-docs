---
title: "内置的镜像仓库镜像源"
---
# Embedded Registry Mirror

:::info Version Gate
自 2024 年 1 月发布的以下版本起，内置镜像仓库镜像源（Embedded Registry Mirror）作为一项实验性功能正式推出：v1.26.13+k3s1、v1.27.10+k3s1、v1.28.6+k3s1、v1.29.1+k3s1。
:::

K3s 内置了 [Spegel](https://github.com/spegel-org/spegel) — 这是一个无状态的分布式 OCI 镜像源（Registry Mirror）。它支持在 Kubernetes 集群的各个节点之间，以 P2P（点对点）的方式共享容器镜像。
该分布式镜像代理功能默认处于禁用状态。

## 启用分布式 OCI 镜像仓库镜像源

为了启用内置镜像仓库镜像功能，Server 节点在启动时必须带上 `--embedded-registry` 标志，或者在配置文件中设置 `embedded-registry: true`。
该选项启用后，集群中的所有节点均可使用内置镜像代理。

当在集群级别启用该功能后，所有节点都将在 6443 端口上托管（运行）一个本地 OCI 仓库，并通过 5001 端口在点对点（P2P）网络中发布可用镜像列表。

只要是集群中任一节点的 containerd 镜像库（Image Store）里存在的镜像，其他集群成员都可以在无法访问外部镜像仓库的情况下直接拉取。

通过 [air-gap image tar files](./airgap.md#manually-deploy-images-method) 导入的镜像会被“固定”（pinned）在 containerd 中，以确保它们始终可用，且不会被 Kubelet 的垃圾回收（GC）机制清理掉。

### 要求
启用内置镜像仓库镜像功能后，所有节点必须能够通过各自的内部 IP 地址，在 TCP 端口 5001 和 6443 上实现互访。

如果节点之间无法互访，镜像拉取的时间可能会变长；因为 containerd 会先尝试连接分布式镜像仓库，在请求超时或失败后，才会回退（fallback）到其他地址。

## 启用仓库镜像源
针对某个镜像仓库（Registry）启用镜像功能后，节点既可以从其他节点拉取该仓库的镜像，也可以将该仓库的镜像共享给其他节点。
如果某个仓库仅在部分节点上启用了镜像功能，而其它节点未启用，那么只有启用了该功能的节点之间才会互相交换该仓库的镜像。

为了启用上游容器仓库的镜像共享功能，各节点必须在 `registries.yaml` 文件的 `mirrors` 部分为该仓库添加一个条目。
该仓库不需要列出任何后端接入点（endpoints），只要存在该条目即可。
例如，若要对 `docker.io` 和 `registry.k8s.io` 的镜像启用分布式共享，请在所有集群节点上配置如下内容的 `registries.yaml`：

```yaml
mirrors:
  docker.io:
  registry.k8s.io:
```

镜像仓库的接入点（Endpoints）仍可照常添加。
在以下配置中，镜像拉取尝试将首先搜索“内置镜像源”（P2P 共享），接着尝试 `mirror.example.com`，最后才会连接 `docker.io`：

```yaml
mirrors:
  docker.io:
    endpoint:
      - https://mirror.example.com
```

如果你是直接使用私有仓库，而非将其作为上游仓库的镜像源，你仍然可以按照启用公共仓库的相同方式来开启分布式镜像源功能——只需将其列在 mirrors 部分即可：
```yaml
mirrors:
  mirror.example.com:
```
如果一个节点没有针对任何镜像仓库启用镜像功能，那么该节点将不会以任何形式参与到分布式镜像仓库网络中。

想了解有关 `registries.yaml` 文件结构的更多信息，看[Private Registry Configuration](./private-registry.md).。

### 默认Endpoint回退机制
默认情况下，在从配置了镜像接入点（mirror endpoints）的仓库拉取镜像时，如果镜像源不可用，containerd 将会自动回退（fall back）到默认地址。

如果你想关闭这个，只从配置的镜像源和/或内置镜像源拉取镜像，看私有仓库配置文档 [Default Endpoint Fallback](./private-registry.md#default-endpoint-fallback) 部分。

请注意，如果你使用了 `--disable-default-endpoint` 选项，并希望仅允许从某个特定仓库直接拉取镜像，而禁止拉取其他仓库，你可以显式地为该仓库提供一个接入点（endpoint），从而允许镜像拉取行为回退至该仓库本身。

```yaml
mirrors:
  docker.io:           # no default endpoint, pulls will fail if not available on a node
  registry.k8s.io:     # no default endpoint, pulls will fail if not available on a node
  mirror.example.com:  # explicit default endpoint, can pull from upstream if not available on a node
    endpoint:
      - https://mirror.example.com
```

### Latest Tag

当未为容器镜像指定标签时，隐式默认标签是 `latest`。此标签通常会更新，以指向镜像的最新版本。由于该标签会根据拉取的时间指向镜像的不同版本，因此分布式仓库不会从其它节点拉取 latest 标签的镜像。这迫使 containerd 访问上游仓库或仓库镜像源，以确保对 latest 标签所指代内容的一致。

这与 Kubernetes 在为容器镜像使用 latest 标签时所遵循的 [special `imagePullPolicy` defaulting](https://kubernetes.io/docs/concepts/containers/images/#imagepullpolicy-defaulting)逻辑保持一致。

可以通过为 K3s 服务设置 `K3S_P2P_ENABLE_LATEST=true` 环境变量来启用 `latest` 标签的镜像。这是不支持的且不推荐的，原因如上所述。

## Security

### Authentication（认证）

访问内置镜像源（embedded mirror）的 Registry API 需要持有一个有效的客户端证书，该证书必须由集群的客户端证书颁发机构（Client CA）签名。

访问分布式哈希表（DHT）的点对点（P2P）网络需要持有一个由服务端节点（Server Nodes）控制的预共享密钥（PSK）。
节点之间通过预共享密钥和由集群证书颁发机构（CA）签名的证书进行双重认证。

### Potential Concerns（潜在顾虑）

:::warning
分布式镜像仓库（Distributed Registry）基于点对点（P2P）原理构建，并假设所有集群成员之间拥有同等的权限级别和信任程度。
如果这与您集群的安全态势（Security Posture）不符，则不应启用内置的分布式镜像仓库。
:::

The embedded registry may make available images that a node may not otherwise have access to.
For example, if some of your images are pulled from a registry, project, or repository that requires authentication via Kubernetes Image Pull Secrets, or credentials in `registries.yaml`,
the distributed registry will allow other nodes to share those images without providing any credentials to the upstream registry.

Users with access to push images into the containerd image store on one node may be able to use this to 'poison' the image for other cluster nodes,
as other nodes will trust the tag advertised by the node, and use it without checking with the upstream registry.
If image integrity is important, you should use image digests instead of tags, as the digest cannot be poisoned in this manner.

## Sharing Air-gap or Manually Loaded Images

Image sharing is controlled based on the source registry.
Images loaded directly into containerd via [air-gap tarballs](./airgap.md?airgap-load-images=Manually+Deploy+Images), [pre-imported](../add-ons/import-images.md#pre-import-images) or loaded directly into containerd's image store using the `ctr` command line tool, will be shared between nodes if they are tagged as being from a registry that is enabled for mirroring.

Note that the upstream registry that the images appear to come from does not actually have to exist or be reachable.
For example, you could tag images as being from a fictitious upstream registry, and import those images into containerd's image store.
You would then be able to pull those images from all cluster members, as long as that registry is listed in `registries.yaml`

## Pushing Images

The embedded registry is read-only, and cannot be pushed to directly using `docker push` or other common tools that interact with OCI registries.

Images can be manually made available via the embedded registry by running `ctr -n k8s.io image pull` to pull an image,
or by loading image archives created by `docker save` via the `ctr -n k8s.io image import` command or the [pre-import feature](../add-ons/import-images.md#pre-import-images).
Note that the `k8s.io` namespace must be specified when managing images via `ctr` in order for them to be visible to the kubelet.
