# Service

Kubernetes 中的 **Service** 是一种抽象层，用于将一组 Pod 暴露为稳定的网络服务，其底层原理涉及 **网络代理、负载均衡和 DNS 解析** 等多个组件的协同工作。

* **服务发现**：通过 DNS 或环境变量暴露服务名称（如 `my-service.default.svc.cluster.local`）。
* **负载均衡**：将请求均匀分发到后端 Pod。
* **稳定访问**：屏蔽 Pod IP 的动态变化（如扩缩容、重启）。

## 两种服务发现方式

### （早）基于环境变量

每当 Pod 启动时，Kubernetes 会自动为集群中的 Service 设置环境变量，这样 Pod 内部的应用可以使用这些环境变量来发现 Service。

变量格式：

```bash
<SERVICE_NAME>_SERVICE_HOST=<service-ip>
<SERVICE_NAME>_SERVICE_PORT=<port>
```

**限制**

* **Pod 必须在 Service 创建后启动，否则不会获取环境变量！**
* **只适用于简单的服务发现，不支持负载均衡。**
* **无法解析 Pod 级别的 DNS，只能解析 Service。**

### （现）基于 DNS（CoreDNS）

Kubernetes 主要使用 CoreDNS 进行服务发现，解析 Service 或 Pod 的域名。

*   Service DNS 格式：

    ```bash
    <service-name>.<namespace>.svc.cluster.local
    ```
*   Pod DNS 格式（可选）:

    ```bash
    <pod-ip>.<pod-name>.<namespace>.pod.cluster.local
    ```

一般访问一个 Service，CoreDNS 会返回一个 ClusterIP：

```bash
nslookup my-service.default.svc.cluster.local

Name: my-service.default.svc.cluster.local
Address: 10.100.200.1
```

这里的 `10.100.200.1` 就是 `my-service` 的 ClusterIP。

在 Headless Service 中，CoreDNS 就会直接返回后端所有 Pod 的真实地址：

```bash
nslookup my-db.default.svc.cluster.local

Name: my-db.default.svc.cluster.local
Addresses: 10.244.1.5
           10.244.2.7
```

应用可以**手动选择**连接哪个 Pod。

> 详情看 [CoreDNS](Service.md#coredns)

## 组件

### kube-proxy

监听 API Server 的 Service 和 Endpoint 变化，配置本地网络规则（iptables/IPVS），将请求转发到 Pod。

详情看 [kube-proxy](Service.md) 笔记。

### Endpoints/EndpointSlice

### CoreDNS

> 在 Kubernetes 1.11 之前，`kube-dns` 是默认的 **DNS 解析服务**，用于解析集群内部 `Service` 和 `Pod` 的域名。它是 Kubernetes 内部服务发现的重要组件，但在 Kubernetes 1.13 之后被 **CoreDNS** 取代。

CoreDNS 是 Kubernetes 默认的 **DNS 解析服务**，负责集群内部的 **服务发现**，用于将 Service、Pod 以及外部域名解析成 IP 地址。它是一个基于 **插件架构** 的轻量级 DNS 服务器，并且可以扩展解析逻辑。CoreDNS 负责：

* 解析 **Service 名称**（如 `my-service.default.svc.cluster.local`）。
* 解析 **Pod 名称**（可选）。
* 解析 **外部域名**（如 `google.com`）。
* 提供 **DNS 负载均衡** 和 **缓存** 机制。

#### 工作流程

当 Pod 需要解析一个域名时：

1. **Pod 内部 DNS 查询**：
   *   Pod 里的应用（如 Nginx）执行 DNS 查询，比如：

       ```bash
       nslookup my-service.default.svc.cluster.local
       ```
   * 这个请求被发送到 Pod 内置的 `resolv.conf` 配置的 DNS 服务器。
2. **DNS 请求到达 CoreDNS（监听 53 端口）**：
   * CoreDNS 监听 `10.0.0.10:53`（默认 Service IP）。
   * CoreDNS 解析域名，并返回对应的 **ClusterIP** 或 **Pod IP**。
3. **CoreDNS 解析逻辑**
   * 如果域名是 **Kubernetes Service**（如 `my-service.default.svc.cluster.local`），CoreDNS 查找 `kube-apiserver` 获取 Service IP 并返回。
   *   如果是 **外部域名**（如 `google.com`），CoreDNS 会向上转发到外部 DNS 服务器（如 `8.8.8.8`），**使用 `forward . /etc/resolv.conf`**，`resolv.conf` 可能包含外部 DNS，如：

       ```nginx
       nameserver 8.8.8.8
       ```

#### 配置文件

CoreDNS 通过 ConfigMap 配置，默认配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

**解析规则分析**

| **配置项**                      | **作用**                      |
| ---------------------------- | --------------------------- |
| `.:53`                       | CoreDNS 监听 `53` 端口          |
| `errors`                     | 记录 DNS 解析错误                 |
| `health`                     | 提供健康检查                      |
| `ready`                      | 检查 CoreDNS 是否准备就绪           |
| `kubernetes`                 | 解析 Kubernetes 内部 DNS        |
| `prometheus`                 | 暴露 DNS 监控指标                 |
| `forward . /etc/resolv.conf` | 将外部 DNS 请求转发到 `resolv.conf` |
| `cache 30`                   | 缓存解析结果 30 秒                 |
| `loop`                       | 防止 DNS 死循环                  |
| `reload`                     | 自动重载配置                      |
| `loadbalance`                | 负载均衡多个 A 记录                 |

#### 查看 CoreDNS

查看 CoreDNS 状态：

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

如果 Pod 持续 Crash，可以查看日志：

```bash
kubectl -n kube-system logs -l k8s-app=kube-dns
```

#### 插件机制

coreDNS 采用 **插件架构**，可以扩展 DNS 功能。常见插件：

| **插件名称**      | **作用**                            |
| ------------- | --------------------------------- |
| `kubernetes`  | 解析 Kubernetes 内部 Service / Pod 名称 |
| `forward`     | 将外部域名查询转发到上游 DNS                  |
| `cache`       | 缓存 DNS 解析结果，提高查询速度                |
| `loadbalance` | 负载均衡多个 A 记录                       |
| `rewrite`     | 修改 DNS 查询请求（如 URL 重写）             |
| `autopath`    | 自动完成短域名                           |
| `hosts`       | 解析本地 `/etc/hosts` 文件              |
| `log`         | 记录 DNS 查询日志                       |

#### CoreDNS 负载均衡

CoreDNS 默认使用 **轮询（Round Robin）** 方式负载均衡多个 A 记录。

* 适用于 Service 类型 `ClusterIP` 和 `Headless Service`。

```bash
# 查询
nslookup my-db.default.svc.cluster.local

# 返回
Name: my-db.default.svc.cluster.local
Addresses: 10.244.1.5
           10.244.2.7
```

应用可以**手动选择**连接哪个 Pod。

## 服务类型

> \[!important]
>
> Service 使用 selector 选中 Pod 的逻辑：
>
> * Pod 是处于就绪状态
> * Pod 的标签是 Service 标签的集合（同一个命名空间下）

### ClusterIP

通过标签选择器 selector，选中 Pod，用于为一组 `Pod` 提供一个 **集群内可访问的虚拟 IP 地址**，让其他 Pod 通过该 IP 进行通信。

#### 工作原理

1. **service 创建时，kube-apiserver 分配 ClusterIP**
   * kube-apiserver 分配 Service 一个 ClusterIP（如 `10.100.200.1`）。
   * Endpoints 记录 `selector` 匹配的 Pod IP。
   * kube-proxy 监听 kube-apiserver，更新 iptables / IPVS 规则。
2. **Pod 访问 ClusterIP**
   * Pod 访问 `10.100.200.1:80`（Service 地址）。
   * kube-proxy 会查找与该 Service 关联的 Endpoints 列表，并根据流量转发规则将请求路由到对应的 Pod。
3. **流量到达 Pod**
   * 目标 Pod 处理请求，并返回响应数据。

### NodePort

当 NodePort 创建时，service 会在当前的 Node 的网卡上绑定一个端口。如果是 kube-proxy 是 IPVS 模式，这个 IP 和端口号就变成了 当前 IPVS 集群的 VIP，后端的 RS（Real Server）就是Node 中的 Pod。

**高可用架构**：

![service-nodeport-highavailability.drawio](../.gitbook/assets/service-nodeport-highavailability.drawio.svg)

在上图的 Pod 中有三次负载均衡，分别是 NodePort（IPVS） 四层、nginx 七层、ClusterIP（IPVS） 四层。

### LoadBalancer

依赖云提供商，分配一个外部负载均衡器 LASS（如 AWS ELB、GCP LB），API Server 会向 LASS API 发起请求，LASS 会创建相应的 LB 将流量转发到集群内的 Pod，自动创建一个 `NodePort` 和 `ClusterIP`，作为负载均衡器的后端。

> **为什么 LoadBalancer 为什么会自动创建 NodePort 和 ClusterIP ？**
>
> 由于 LASS 无法直接访问 Pod，只能访问 Node，所以所以 Kubernetes **必须为 LoadBalancer Service 创建 NodePort**，再由 kube-proxy 进行流量分发。
>
> **所有 Service（无论什么类型）都会自动创建 ClusterIP**，这样集群内部的 Pod 仍然可以通过 `service-name:port` 访问这个 Service。

在传统环境中可以手动搭建 IPVS + Keepalived 实现基础功能。

![service-loadbalancer](../.gitbook/assets/service-loadbalancer.svg)

**流量转发路径**：

```bash
用户流量 -> 云负载均衡器 (LoadBalancer) -> Kubernetes 节点 (Node) -> NodePort -> kube-proxy -> Pod
```

#### 工作原理

当你创建一个 `LoadBalancer` 类型的 Service 时，Kubernetes 通过云控制器（Cloud Controller Manager, CCM）与云提供商的 API 交互，具体流程如下：

1.  **用户创建 LoadBalancer Service**：

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # AWS 示例
    spec:
      type: LoadBalancer
      selector:
        app: my-app
      ports:
        - protocol: TCP
          port: 80
          targetPort: 8080
    ```
2.  **Kubernetes 处理 LoadBalancer 资源**:

    * Kubenetes 发现 `spec.type: LoadBalancer`，确定需要创建一个外部负载均衡器。
    * 调用云控制器管理器（CCM），由 `kube-controller-manager` 触发 `Service Controller`，调用 **Cloud Provider API** 创建负载均衡器。

    > CCM（Cloud Controller Manager）是 **Kubernetes 与云厂商 API 交互的桥梁**，负责所有云相关功能（节点、存储、路由、LB），它从 `kube-controller-manager` 中拆分出来，专门负责与云平台集成。
    >
    > CCM 本身不直接创建负载均衡器，而是调用 `Service Controller` 进行管理。Service Controller 运行在 CCM 内部，它是 CCM 的一部分
3. **云提供商创建负载均衡器**：
   * **CCM 调用云厂商 API 创建负载均衡器**
     * 例如，在 AWS，CCM 会调用 ELB API 创建一个 **Elastic Load Balancer (ELB)**。
     * 在 GCP，CCM 会创建 **Google Cloud Load Balancer (GCLB)**。
     * 在 Azure，CCM 会创建 **Azure Load Balancer**。
   * **云负载均衡器分配公网 IP**
     * 云提供商分配一个外部 IP（`EXTERNAL-IP`），如 `52.15.34.21`。
     * 你可以通过 `kubectl get svc` 查看。
4. **负载均衡器转发流量到 Kubernetes 节点**：
   * **负载均衡器将流量转发到 Kubernetes NodePort**
     * LoadBalancer 监听 80 端口，并将请求转发到 Kubernetes **所有可用节点** 的 `NodePort`。
     * 例如，如果 NodePort 为 `32567`，流量会转发到 `NODE_IP:32567`。
5. **kube-proxy 转发流量给Pod 处理**

### ExternalName

用于将集群内部的服务名映射到**外部域名**。这个过程中没有任何代理被创建，将 Service 的流量转发到一个外部 DNS 名称，而不是 Kubernetes 集群内的 Pod，不分配 ClusterIP，也不与 Pod 关联。其核心原理是通过 **DNS CNAME 记录** 实现透明代理，无需创建 Endpoints 或 Pod 选择器。

> **CNAME**
>
> CNAME（Canonical Name，规范名称）是一种 **DNS 记录**，用于将一个域名 **指向** 另一个域名，而不是直接解析为 IP 地址。
>
> CNAME 不能直接用于 `example.com`，只能用于子域名（如 `www.example.com`）。
>
> 示例：
>
> ```bash
> www.example.com.  300  IN  CNAME  example.com.
> example.com.      300  IN  A      192.168.1.1
> ```
>
> 在 ExternalName 中示例：
>
> ```bash
> my-service.default.svc.cluster.local.  5 IN CNAME external.example.com.
> ```

**无代理转发**：不依赖 `kube-proxy`，`ExternalName` 不创建 ClusterIP 或 iptables 规则，仅修改 DNS 解析结果，直接依赖外部服务的 DNS 解析机制。

> 在 Kubenetes 1.11 版本以后默认为 CoreDNS。

**场景**：

* 用于集群内部需要访问外部服务的场景。
* 例如：将流量转发到外部数据库或第三方 API。

#### 工作原理

当 Pod 访问一个 `ExternalName` 类型的 Service 时，Kubernetes 并不会提供 Cluster IP，而是通过 **CoreDNS** 直接解析为 **CNAME 记录**，具体流程如下：

1. **Pod 发起 DNS 解析请求**：

* 假设 Kubernetes 内部有一个 `ExternalName` 类型的 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-external-service
  namespace: default
spec:
  type: ExternalName
  externalName: example.com
```

* Pod 内部尝试解析 `my-external-service.default.svc.cluster.local`。

2. **CoreDNS 处理请求**：

* `kubernetes` 插件检测到该 Service 类型是 `ExternalName`，并返回 CNAME 记录：

```bash
my-external-service.default.svc.cluster.local.  5 IN CNAME example.com.
```

* 这意味着 CoreDNS 只是返回 `example.com`，并不会解析它的 IP 地址。

3. **Pod 再次向外部 DNS 服务器查询**：

* Pod 的 DNS 解析器会向外部 DNS 服务器（如 `8.8.8.8`）请求 `example.com` 的 IP。
* 外部 DNS 解析 `example.com` 并返回相应的 A 记录（如 `93.184.216.34`）。

4. **Pod 直接访问外部服务**：
   * Pod 直接向 `93.184.216.34` 发起 TCP/UDP 请求（如 HTTP、gRPC）。
   * Kubernetes 本身不参与后续的网络流量转发。

**注意：**

* ExternalName 服务仅在 DNS 层面进行映射，不涉及流量代理或负载均衡。
* 集群内的应用程序需要能够解析和访问指定的外部 DNS 名称。
* ExternalName 服务不支持所有协议，主要用于 TCP 和 UDP 协议。

### Headless Service

## Endpoints

## 流量策略

### spec.internalTrafficPolicy

**internalTrafficPolicy** 字段，用于控制**集群内部流量**的路由策略，主要影响同一集群内 Pod 之间的通信行为，适用于所有 Service 类型。

可选值：

* `Cluster`（默认）：从集群内任何位置访问 Service 的 ClusterIP 时，流量被负载均衡到集群中所有健康的端点（跨节点），适用于全局负载均衡。
* `Local`：当一个 Pod 访问 Service 时，流量只会被路由到与该 Pod 位于统一节点上的后端 Pod，若节点无可用端点，则视为该服务无可用实例（即使其他节点有端点），这样的流量会被 Drop。

### spec.externalTrafficPolicy

**externalTrafficPolicy** 字段，用于控制**外部流量**如何路由到后端 Pod，仅对 **`NodePort`** 和 **`LoadBalancer`** 生效，不适用于 `ClusterIP`，可选值：

* `Cluster`（默认）：当外部流量到达一个 Node 时，该节点会将其负载均衡到集群中任意节点上的所有后端 Pod。
* `Local`：当外部流量到达一个节点时，该节点只会将流量转发到位于本节点上的后端 Pod，如果本节点没有后端 Pod，流量会被丢弃。（即使其他节点有端点），这样的流量会被 Drop。

### spec.sessionAffinity

**sessionAffinity** 字段用于会话保持，底层原理依赖于 IPVS 的持久化连接。

IPVS 中的持久化连接（Persistent Connection）用来保证来自同一客户端 IP 的多个请求 **在指定的时段内** 都会绕过负载均衡算法，被持续定向到同一 RS（Real Server）。当时间定时器到期之后，来自该客户端新的连接才会再次收到负载均衡规则的约束。

```bash
ipvsadm -A -t 192.168.66.100:80 -s rr -p 120
```

`-s`：负载均衡算法，`rr` 表示 Round Robin 轮询。

`-p` ：持久化连接的定时器，单位为秒。

**sessionAffinity** 字段值：

* `None`
* `ClientIP`：打开持久化连接，默认 100800 秒（3 小时）。

可以通过一下命令查看 ：

```bash
kubectl explain svc.spec.sessionAffinityConfig.clientIP
```

### spec.publishNotReadyAddresses

**publishNotReadyAddresses** 是 Service 的一个布尔类型字段，主要用于控制是否将未就绪的 Pod 地址（`NotReadyAddresses`）发布到 DNS 记录中。

* `false`：不发布未就绪的地址，默认情况下，Kubernetes 仅将已通过\*\*就绪探针（ReadinessProbe）\*\*的 Pod IP 发布到 DNS 记录（如 `my-service.namespace.svc.cluster.local`），未就绪的 Pod 会被排除在外。
* `true`：强制发布所有 Pod 的地址，DNS 会包含所有 Pod 的 IP（包括 `NotReadyAddresses`），无论其就绪状态如何。适用于无头服务。

`publishNotReadyAddresses` 仅影响 DNS 的发布行为，不改变 Endpoints 中 `NotReadyAddresses` （列出未通过就绪检查的 Pod IP）字段的生成逻辑。
