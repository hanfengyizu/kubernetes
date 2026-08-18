# 07 — API Group 安装

> 对应主文档第七章。本章细化 API Group 的注册流程、RESTStorageProvider 的作用、以及内置 API 组的组织方式。

## 7.1 API Group 注册流程

API Server 启动时需要把所有内置资源"安装"成可访问的 HTTP 路由。这个过程的入口是 `InstallAPIGroups`，它遍历每个 `APIGroupInfo`，把对应的 REST Storage 与路由绑定。

```
InstallAPIGroups()
  │
  ├── 为每个 APIGroupInfo:
  │     ├── InstallREST()
  │     │     ├── 创建 GroupVersion
  │     │     ├── 注册 REST 操作
  │     │     └── 安装到 Handler
  │     │
  │     └── 注册 Discovery 信息
  │
  └── 更新路由
```

### 流程细节

1. **遍历 APIGroupInfo**：每个 APIGroupInfo 描述一个组的版本、各版本的 REST Storage 映射、scheme（编解码）、子资源路径等。
2. **InstallREST**：对每个版本，把 `/apis/<group>/<version>/...` 路径与对应 REST Storage 的 CRUD handler 绑定。go-restful 的 WebService 在此构造。
3. **注册 Discovery**：把组信息加入 `/apis` 与 `/apis/<group>` 的 discovery 响应，让 `kubectl api-resources` 能枚举到。
4. **更新路由**：把新 WebService 加入 `GoRestfulContainer`，立即生效。

※ `InstallAPIGroups` 在 `createKubeAPIServer` 阶段调用——先装路由，再进入 `PrepareRun` 让 discovery 就绪信号翻转，之后请求才放行（由 `WithMuxAndDiscoveryComplete` 守门）。

## 7.2 REST Storage Provider

> 文件：`pkg/controlplane/instance.go`

```go
type Instance struct {
    GenericAPIServer *genericapiserver.GenericAPIServer

    // REST Storage Providers
    ClusterAuthenticationInfoCloner clusterauthenticationinfo.Cloner
}

// 各 API 组的 Storage Provider
type RESTStorageProvider interface {
    GroupName() string
    NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, error)
}
```

### RESTStorageProvider 的职责

`RESTStorageProvider` 是"每个 API 组一个"的工厂，负责把该组所有资源的 REST Storage 实例化并打包成 `APIGroupInfo`：

| 方法 | 作用 |
|------|------|
| `GroupName()` | 返回组名（如 `apps`、`batch`） |
| `NewRESTStorage(...)` | 创建组内各资源（deployments/replicasets 等）的 storage，组装成 APIGroupInfo |

参数说明：
- `apiResourceConfigSource`：声明哪些资源/版本该启用（控制"装哪些"）。
- `restOptionsGetter`：提供 etcd 后端配置给各 storage（控制"存哪"）。

### Instance 的角色

`Instance`（`pkg/controlplane/instance.go`）是 K8s 控制平面对 `GenericAPIServer` 的包装。它聚合所有内置组的 RESTStorageProvider，在启动时依次调用 `NewRESTStorage` → `InstallAPIGroups` 完成安装。

## 7.3 内置 API 组

K8s 内置 API 组按功能域划分，每个组独立演进版本：

```
core (v1)
  ├── pods
  ├── services
  ├── configmaps
  ├── secrets
  ├── namespaces
  └── ...

apps (v1)
  ├── deployments
  ├── replicasets
  ├── statefulsets
  └── daemonsets

batch (v1)
  ├── jobs
  └── cronjobs

rbac.authorization.k8s.io (v1)
  ├── roles
  ├── rolebindings
  ├── clusterroles
  └── clusterrolebindings

... 等等
```

### 组的命名规则

| 组 | 前缀 | 版本策略 | 说明 |
|----|------|---------|------|
| core | `/api` | 仅 `v1` | 最古老，无组名，走特殊前缀 |
| `apps` | `/apis/apps` | `v1` | 工作负载（无状态/有状态/守护） |
| `batch` | `/apis/batch` | `v1` | 批处理任务 |
| `rbac.authorization.k8s.io` | `/apis/rbac.authorization.k8s.io` | `v1` | 权限模型 |
| `networking.k8s.io` | `/apis/networking.k8s.io` | `v1` | 网络（Ingress 等） |
| `storage.k8s.io` | `/apis/storage.k8s.io` | `v1` | 存储（StorageClass 等） |

※ core 组是历史遗留：它没有组名（`APIGroup` 为空），路径前缀是 `/api` 而非 `/apis/<group>`。其余所有组都走 `/apis/<group>/<version>/...`。这就是 `RequestInfo` 里 `APIPrefix` 区分 `api`/`apis` 的原因。

### 版本演进

- 同一组可有多个版本并存（如 `v1beta1` → `v1`），路由同时挂载。
- 请求不同版本时，storage 内部做版本转换（由 scheme 转换函数完成），etcd 存的是首选版本。
- 这让客户端可渐进升级，无需一夜全切。

## 7.4 CRD 与聚合层：动态加入组

除了内置组，还有两条动态扩展路径：

| 机制 | 资源 | 注册方式 | 适用 |
|------|------|---------|------|
| CRD | `CustomResourceDefinition` | apiserver 自动识别并装路由 | 自定义资源，同进程存储 |
| Aggregated APIService | `APIService` | aggregator 转发到外部 apiserver | 独立进程的扩展 API（如 metrics-server） |

二者都最终通过 `InstallAPIGroups` 类似机制出现在 `/apis/<group>/<version>` 下，对客户端透明——客户端无需区分内置还是扩展。
