# 02 — 核心数据结构

> 对应主文档第二章。本章逐字段解释三个核心结构体的语义、用途与设计意图，标注每个字段"谁设置、谁读取"。

## 2.1 GenericAPIServer

> 文件：`staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go`

`GenericAPIServer` 是 API Server 的运行时实体。它不是 K8s 专属的——它是 `k8s.io/apiserver` 通用库提供的"通用 API 服务器骨架"，K8s、kube-aggregator、各类 aggregator 都基于它构建。K8s 特有的部分通过组合（嵌入 `Config`、注入各 K8s 专属 storage provider）叠加其上。

```go
type GenericAPIServer struct {
    // 发现服务地址（/api、/apis 返回的服务端入口）
    discoveryAddresses    discovery.Addresses

    // 回环客户端配置：apiserver 用它调用自己（聚合层、引导任务）
    LoopbackClientConfig  *restclient.Config

    // 最小请求超时（兜底，防慢请求挂住 goroutine）
    minRequestTimeout     time.Duration

    // 旧版 API 组前缀（区分 legacy core 与 aggregated 组）
    legacyAPIGroupPrefixes sets.String

    // 准入控制器（贯穿 Create/Update/Delete/Connect）
    admissionControl      admission.Interface

    // 安全服务配置（TLS、端口、监听地址）
    SecureServingInfo     *SecureServingInfo

    // HTTP 处理器（持有 FullHandlerChain 与路由容器）
    Handler               *APIServerHandler

    // 授权器（组合多个 authorizer 形成决策链）
    Authorizer            authorizer.Authorizer

    // 生命周期信号（控制 shutdown 各阶段可见性）
    lifecycleSignals      lifecycleSignals

    // 健康检查三类端点（healthz/livez/readyz 语义不同）
    healthz               healthChecker
    livez                 healthChecker
    readyz                healthChecker

    // 关闭超时（优雅停机的最长等待）
    shutdownTimeout       time.Duration
}
```

### 字段语义表

| 字段 | 由谁设置 | 谁读取 | 设计意图 |
|------|---------|--------|---------|
| `discoveryAddresses` | `PrepareRun` 阶段计算 | discovery handler | 让客户端发现可用的 server 入口，支持多入口通告 |
| `LoopbackClientConfig` | `CreateServerChain` 中注入 | 聚合层、admission webhook 调用器 | apiserver 调自己（如 aggregator 转发）必须走回环，绕过外部认证 |
| `minRequestTimeout` | options 配置 | `WithTimeoutForNonLongRunningRequests` | 非长运行请求的兜底超时，防 leak |
| `legacyAPIGroupPrefixes` | 启动配置 | RequestInfo 解析器 | 区分 `/api`（legacy core）与 `/apis`（组化资源）路由前缀 |
| `admissionControl` | `BuildHandlerChain` 前 | REST Storage 各操作 | 在对象落 etcd 前的最后变更/校验点 |
| `Handler` | `New()` 时构造 | `RunWithContext` 启动、shutdown | 单一 HTTP 入口对象 |
| `Authorizer` | config.Complete | `WithAuthorization` | 决策链入口 |
| `lifecycleSignals` | 内部初始化 | shutdown 协调器 | 让各子系统知道"是否还在接受请求""是否已停收"等阶段 |
| `healthz/livez/readyz` | 各子检查注册 | 对应 HTTP 端点 | healthz=进程存活；livez=可处理（含 etcd 可达）；readyz=就绪可接流量。三者解耦便于滚动更新 |
| `shutdownTimeout` | options | `RunWithContext` 退出 | 强制 kill 前的最长优雅停机 |

※ `healthz` 是历史端点，新代码用 `livez`/`readyz` 表达更精确语义。`livez` 失败时 kubelet 会重启 pod；`readyz` 失败时从 Service 端点摘除但仍运行。

## 2.2 APIServerHandler

> 文件：`staging/src/k8s.io/apiserver/pkg/server/handler.go`

`APIServerHandler` 是 HTTP 入口的聚合体，持有"完整处理链"和"两个路由器"。

```go
type APIServerHandler struct {
    // 完整的处理链（含所有 middleware + 路由）
    FullHandlerChain   http.Handler

    // RESTful 容器（处理 /api、/apis 下的 REST 资源请求）
    GoRestfulContainer *restful.Container

    // 非 RESTful 路由（健康检查、指标、调试等）
    NonGoRestfulMux    *mux.PathRecorderMux

    // 路由分发器（决定请求走哪个容器/多路复用）
    Director           http.Handler
}
```

### 三个处理器的分工

| 成员 | 处理对象 | 典型路径 | 特点 |
|------|---------|---------|------|
| `FullHandlerChain` | 一切请求 | 所有 | 入口，套满 middleware 后再分发 |
| `GoRestfulContainer` | REST 资源 | `/api/v1/...`、`/apis/apps/v1/...` | 基于 go-restful，支持内容协商、子资源 |
| `NonGoRestfulMux` | 非资源 | `/healthz`、`/metrics`、`/debug/*` | 轻量 mux，不经过资源鉴权语义 |

`Director` 的作用：当请求到达路由层，`Director` 判断它是否属于已注册的 REST 路由；若是交给 `GoRestfulContainer`，否则交给 `NonGoRestfulMux`。聚合层（aggregated APIServices）的路由也通过 Director 注入。

⚠ `FullHandlerChain` ≠ `GoRestfulContainer`。前者包含认证/授权/审计等所有 middleware 包裹着后者；直接访问 `GoRestfulContainer` 会**绕过安全检查**——只有内部回环或特定场景才这么做。

## 2.3 RequestInfo

> 文件：`staging/src/k8s.io/apiserver/pkg/endpoints/request/requestinfo.go`

`RequestInfo` 是把原始 HTTP 请求"翻译"为 Kubernetes 语义的中间产物。Handler Chain 中的 `WithRequestInfo` 解析后注入 context，下游（授权、准入、storage）统一从 context 取用，而非各自重新解析。

```go
type RequestInfo struct {
    IsResourceRequest bool     // 是否为资源请求（区别于 /healthz 等）
    Path              string   // 请求路径
    Verb              string   // 语义动词（GET/LIST/CREATE/UPDATE/DELETE/WATCH...）
    APIPrefix         string   // API 前缀 (api / apis)
    APIGroup          string   // API 组（core 为空字符串）
    APIVersion        string   // API 版本 (v1, v1beta1...)
    Namespace         string   // 命名空间（cluster-scoped 资源为空）
    Resource          string   // 资源类型 (pods, services)
    Subresource       string   // 子资源 (log, exec, status, scale)
    Name              string   // 资源名称
    Parts             []string // 路径分段（原始切片，便于回溯）
    FieldSelector     string   // 字段选择器（list 请求）
    LabelSelector     string   // 标签选择器（list 请求）
}
```

### Verb 的语义映射

`Verb` 不是 HTTP 方法，而是 K8s 语义动词。映射规则：

| HTTP 方法 + 路径形态 | RequestInfo.Verb | 说明 |
|---------------------|------------------|------|
| `GET /pods` | `LIST` | 集合读取 |
| `GET /pods/nginx` | `GET` | 单体读取 |
| `POST /pods` | `CREATE` | 创建 |
| `PUT /pods/nginx` | `UPDATE` | 全量更新 |
| `PATCH /pods/nginx` | `PATCH` | 部分更新 |
| `DELETE /pods/nginx` | `DELETE` | 删除 |
| `GET /pods?watch=true` | `WATCH` | 长连接 watch |
| `POST /pods/nginx/exec` | `CONNECT` | 子资源连接（exec/attach/portforward） |

※ 这层抽象的价值：授权（RBAC）基于 `Verb`+`Resource`+`Namespace` 决策，与 HTTP 方法解耦。例如 `LIST` 和 `GET` 在 RBAC 里是不同动词（`list` vs `get`），便于细粒度授权。

### 生命周期与可见性

```
HTTP Request
  → WithRequestInfo 解析 RequestInfo
  → 注入 context (requestContextKey)
  → Authorization 从 context 取 RequestInfo 决策
  → Admission 从 context 取 RequestInfo 组装 Attributes
  → REST Storage 用 RequestInfo 定位资源
```

`IsResourceRequest` 是关键开关：非资源请求（如 `/metrics`）走简化路径，不进入资源鉴权/准入语义，直接由 `NonGoRestfulMux` 处理。
