# Kubernetes API Server 学习总结

## 一、架构概览

### 1.1 核心架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              kube-apiserver                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        Handler Chain (请求处理链)                       │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │  │
│  │  │PanicReco│→│RequestIn│→│Authentic│→│Authoriz │→│Admission│→ ...     │  │
│  │  │  very   │ │  fo     │ │  ation  │ │  ation  │ │ Control │          │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      APIServerHandler (路由层)                          │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐                     │  │
│  │  │  GoRestfulContainer │  │  NonGoRestfulMux    │                     │  │
│  │  │  (REST API 路由)     │  │  (非 API 路由)       │                     │  │
│  │  └─────────────────────┘  └─────────────────────┘                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        REST Storage (存储层)                            │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │  │
│  │  │  Pod    │ │ Service │ │  Node   │ │ConfigMap│ │  ...    │          │  │
│  │  │ Storage │ │ Storage │ │ Storage │ │ Storage │ │         │          │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         Etcd (持久化存储)                               │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心组件

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| **GenericAPIServer** | `staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go` | API Server 核心实现，管理生命周期 |
| **APIServerHandler** | `staging/src/k8s.io/apiserver/pkg/server/handler.go` | HTTP 路由管理 |
| **Config** | `staging/src/k8s.io/apiserver/pkg/server/config.go` | 服务器配置 |
| **Handler Chain** | `staging/src/k8s.io/apiserver/pkg/server/config.go` | 请求处理链 |
| **Admission Control** | `staging/src/k8s.io/apiserver/pkg/admission/interfaces.go` | 准入控制接口 |
| **Authorizer** | `staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go` | 授权接口 |
| **REST Storage** | `staging/src/k8s.io/apiserver/pkg/registry/rest/rest.go` | 资源存储接口 |
| **RequestInfo** | `staging/src/k8s.io/apiserver/pkg/endpoints/request/requestinfo.go` | 请求信息解析 |

---

## 二、核心数据结构

### 2.1 GenericAPIServer

```go
// 文件: staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go
type GenericAPIServer struct {
    // 发现服务地址
    discoveryAddresses    discovery.Addresses

    // 回环客户端配置（用于内部调用）
    LoopbackClientConfig  *restclient.Config

    // 最小请求超时
    minRequestTimeout     time.Duration

    // 旧版 API 组前缀
    legacyAPIGroupPrefixes sets.String

    // 准入控制器
    admissionControl      admission.Interface

    // 安全服务配置
    SecureServingInfo     *SecureServingInfo

    // HTTP 处理器
    Handler               *APIServerHandler

    // 授权器
    Authorizer            authorizer.Authorizer

    // 生命周期管理
    lifecycleSignals      lifecycleSignals

    // 健康检查
    healthz               healthChecker
    livez                 healthChecker
    readyz                healthChecker

    // 关闭信号
    shutdownTimeout       time.Duration
}
```

### 2.2 APIServerHandler

```go
// 文件: staging/src/k8s.io/apiserver/pkg/server/handler.go
type APIServerHandler struct {
    // 完整的处理链
    FullHandlerChain   http.Handler

    // RESTful 容器（处理 API 请求）
    GoRestfulContainer *restful.Container

    // 非 RESTful 路由（健康检查、指标等）
    NonGoRestfulMux    *mux.PathRecorderMux

    // 路由分发器
    Director           http.Handler
}
```

### 2.3 RequestInfo

```go
// 文件: staging/src/k8s.io/apiserver/pkg/endpoints/request/requestinfo.go
type RequestInfo struct {
    IsResourceRequest bool     // 是否为资源请求
    Path              string   // 请求路径
    Verb              string   // HTTP 动词 (GET, POST, etc.)
    APIPrefix         string   // API 前缀 (api, apis)
    APIGroup          string   // API 组
    APIVersion        string   // API 版本
    Namespace         string   // 命名空间
    Resource          string   // 资源类型
    Subresource       string   // 子资源
    Name              string   // 资源名称
    Parts             []string // 路径分段
    FieldSelector     string   // 字段选择器
    LabelSelector     string   // 标签选择器
}
```

---

## 三、请求处理流程

### 3.1 Handler Chain 构建顺序

Handler Chain 按照反向顺序构建（从内到外）：

```
1. WithPanicRecovery           - Panic 恢复
2. WithMuxAndDiscoveryComplete - 发现服务完成信号
3. WithRequestReceivedTimestamp - 请求时间戳
4. WithRequestInfo             - 解析请求信息
5. WithRoutine                 - Goroutine 执行（特性门控）
6. WithLatencyTrackers         - 延迟追踪
7. WithHTTPLogging             - HTTP 日志
8. WithRetryAfter              - 关闭期间重试
9. WithHSTS                    - HTTP 严格传输安全
10. WithCacheControl           - 缓存控制
11. WithProbabilisticGoaway    - HTTP/2 GOAWAY 负载均衡
12. WithWatchTerminationDuringShutdown - Watch 请求清理
13. WithWaitGroup              - 请求追踪
14. WithRequestDeadline        - 请求截止时间
15. WithTimeoutForNonLongRunningRequests - 非长运行请求超时
16. WithWarningRecorder        - 警告头管理
17. WithCORS                   - CORS 处理
18. WithAuthentication         - 用户认证
19. WithTracing                - 分布式追踪
20. WithAudit                  - 审计日志
21. WithImpersonation          - 用户模拟
22. WithPriorityAndFairness    - 优先级和公平性（或 WithMaxInFlightLimit）
23. WithAuthorization          - 授权检查
```

### 3.2 请求处理流程图

```
HTTP Request
     │
     ▼
┌─────────────────┐
│ Panic Recovery  │ ← 捕获 panic
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Request Info    │ ← 解析请求路径、动词等
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Authentication  │ ← 验证用户身份
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Authorization   │ ← 检查用户权限
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Admission Ctrl  │ ← 准入控制（变更和验证）
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ REST Storage    │ ← 执行 CRUD 操作
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Etcd        │ ← 持久化存储
└─────────────────┘
```

---

## 四、核心接口设计

### 4.1 REST Storage 接口层次

```go
// 文件: staging/src/k8s.io/apiserver/pkg/registry/rest/rest.go

// 基础存储接口
type Storage interface {
    New() runtime.Object  // 创建空对象
    Destroy()             // 清理资源
}

// 标准存储接口（组合多个子接口）
type StandardStorage interface {
    Getter
    Lister
    CreaterUpdater
    GracefulDeleter
    CollectionDeleter
    Watcher
    Destroy()
}

// 单个资源获取
type Getter interface {
    Get(ctx context.Context, name string, options *metav1.GetOptions) (runtime.Object, error)
}

// 列表获取
type Lister interface {
    List(ctx context.Context, options *metav1.ListOptions) (runtime.Object, error)
}

// 创建
type Creater interface {
    Create(ctx context.Context, obj runtime.Object, createValidation ValidateObjectFunc, options *metav1.CreateOptions) (runtime.Object, error)
}

// 更新
type Updater interface {
    Update(ctx context.Context, name string, objInfo UpdatedObjectInfo, createValidation ValidateObjectFunc, updateValidation ValidateObjectUpdateFunc, options *metav1.UpdateOptions) (runtime.Object, bool, error)
}

// 优雅删除
type GracefulDeleter interface {
    Delete(ctx context.Context, name string, deleteValidation ValidateObjectFunc, options *metav1.DeleteOptions) (runtime.Object, bool, error)
}

// Watch
type Watcher interface {
    Watch(ctx context.Context, options *metav1.ListOptions) (watch.Interface, error)
}

// 连接（如 kubectl exec）
type Connecter interface {
    Connect(ctx context.Context, requestInfo *RequestInfo, responder Responder) (http.Handler, error)
}
```

### 4.2 Admission Control 接口

```go
// 文件: staging/src/k8s.io/apiserver/pkg/admission/interfaces.go

// 基础接口
type Interface interface {
    Handles(operation Operation) bool
}

// 变更接口
type MutationInterface interface {
    Interface
    Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error
}

// 验证接口
type ValidationInterface interface {
    Interface
    Validate(ctx context.Context, a Attributes, o ObjectInterfaces) error
}

// 操作类型
type Operation string
const (
    Create  Operation = "CREATE"
    Update  Operation = "UPDATE"
    Delete  Operation = "DELETE"
    Connect Operation = "CONNECT"
)
```

### 4.3 Authorizer 接口

```go
// 文件: staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go

type Authorizer interface {
    Authorize(ctx context.Context, a Attributes) (Decision, string, error)
}

type Decision int
const (
    DecisionDeny Decision = iota
    DecisionAllow
    DecisionNoOpinion
)

// 授权属性接口
type Attributes interface {
    GetUser() user.Info
    GetVerb() string
    IsReadOnly() bool
    GetNamespace() string
    GetResource() string
    GetSubresource() string
    GetName() string
    GetAPIGroup() string
    GetAPIVersion() string
    // ...
}
```

---

## 五、启动流程

### 5.1 入口点

```go
// 文件: cmd/kube-apiserver/apiserver.go
func main() {
    command := app.NewAPIServerCommand()
    code := cli.Run(command)
    os.Exit(code)
}
```

### 5.2 命令创建

```go
// 文件: cmd/kube-apiserver/app/server.go
func NewAPIServerCommand() *cobra.Command {
    s := options.NewServerRunOptions()
    cmd := &cobra.Command{
        RunE: func(cmd *cobra.Command, args []string) error {
            return Run(context.Background(), s)
        },
    }
    // 添加命令行标志
    s.AddFlags(cmd.Flags())
    return cmd
}
```

### 5.3 服务器启动

```go
// 文件: cmd/kube-apiserver/app/server.go
func Run(ctx context.Context, opts *options.ServerRunOptions) error {
    // 1. 创建配置
    server, err := CreateServerChain(ctx, opts.Complete()...)

    // 2. 准备运行
    prepared, err := server.PrepareRun()

    // 3. 运行服务器
    return prepared.RunWithContext(ctx)
}
```

### 5.4 启动流程图

```
main()
  │
  ▼
NewAPIServerCommand() ← 创建 Cobra 命令
  │
  ▼
Parse Flags ← 解析命令行参数
  │
  ▼
CreateServerChain() ← 创建服务器链
  │
  ├── CreateKubeAPIServerConfig()
  │     ├── 创建认证配置
  │     ├── 创建授权配置
  │     └── 创建准入控制配置
  │
  ├── createKubeAPIServer()
  │     ├── InstallAPIGroups()
  │     └── 安装核心 API 组
  │
  └── CreateAggregatorServer()
        └── 创建聚合层
  │
  ▼
PrepareRun() ← 准备运行
  │
  ▼
RunWithContext() ← 启动服务
  │
  ├── 启动健康检查
  ├── 启动非阻塞路由
  └── 启动安全服务
```

---

## 六、认证、授权、准入控制

### 6.1 认证 (Authentication)

支持的认证方式：

```
┌─────────────────────────────────────────────────────────────┐
│                    Authentication Methods                    │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │  Client Cert    │  │  Token File     │                   │
│  │  (客户端证书)    │  │  (静态 Token)   │                   │
│  └─────────────────┘  └─────────────────┘                   │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │  Bootstrap Token│  │  Service Account│                   │
│  │  (引导 Token)   │  │  (服务账户)      │                   │
│  └─────────────────┘  └─────────────────┘                   │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │  OIDC           │  │  Webhook        │                   │
│  │  (OpenID Connect)│  │  (外部认证)     │                   │
│  └─────────────────┘  └─────────────────┘                   │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │  Request Header │  │  Anonymous      │                   │
│  │  (代理认证头)   │  │  (匿名访问)     │                   │
│  └─────────────────┘  └─────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

认证配置文件：`pkg/kubeapiserver/options/authentication.go`

### 6.2 授权 (Authorization)

支持的授权模式：

```go
// 文件: pkg/kubeapiserver/authorizer/modes/modes.go
const (
    ModeAlwaysAllow string = "AlwaysAllow"  // 总是允许
    ModeAlwaysDeny  string = "AlwaysDeny"   // 总是拒绝
    ModeABAC        string = "ABAC"         // 基于属性的访问控制
    ModeWebhook     string = "Webhook"      // 外部 Webhook
    ModeRBAC        string = "RBAC"         // 基于角色的访问控制
    ModeNode        string = "Node"         // 节点授权
)
```

授权决策流程：

```
Request → Authorizer → Decision
                          │
                          ├── DecisionAllow    → 继续
                          ├── DecisionDeny     → 拒绝
                          └── DecisionNoOpinion → 检查下一个授权器
```

### 6.3 准入控制 (Admission Control)

准入控制阶段：

```
┌─────────────────────────────────────────────────────────────────┐
│                        Admission Phases                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request ──→ [Mutating Phase] ──→ [Validation Phase] ──→ Etcd   │
│                 │                      │                        │
│                 ▼                      ▼                        │
│         ┌──────────────┐       ┌──────────────┐                │
│         │ MutatingWebhook│     │ ValidatingWebhook│             │
│         │ LimitRanger    │     │ ResourceQuota    │             │
│         │ Initializer    │     │ PodSecurityPolicy│             │
│         │ ...            │     │ ...              │             │
│         └──────────────┘       └──────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

常见准入控制器：

| 准入控制器 | 功能 |
|-----------|------|
| `NamespaceLifecycle` | 命名空间生命周期管理 |
| `LimitRanger` | 资源限制范围检查 |
| `ServiceAccount` | 服务账户管理 |
| `DefaultStorageClass` | 默认存储类 |
| `ResourceQuota` | 资源配额检查 |
| `PodSecurityPolicy` | Pod 安全策略 |
| `NodeRestriction` | 节点限制 |
| `MutatingAdmissionWebhook` | 变更 Webhook |
| `ValidatingAdmissionWebhook` | 验证 Webhook |

---

## 七、API Group 安装

### 7.1 API Group 注册流程

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

### 7.2 REST Storage Provider

```go
// 文件: pkg/controlplane/instance.go
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

### 7.3 内置 API 组

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

---

## 八、Watch 机制

### 8.1 Watch 实现原理

```
Client                    API Server                    Etcd
  │                           │                           │
  │──── GET /watch/pods ────→│                           │
  │                          │──── Watch ───────────────→│
  │                          │                           │
  │                          │←──── Event (PUT) ─────────│
  │←──── Event (ADDED) ──────│                           │
  │                          │                           │
  │                          │←──── Event (DELETE) ──────│
  │←──── Event (DELETED) ────│                           │
  │                          │                           │
  ...                        ...                         ...
```

### 8.2 Watch 接口

```go
// 文件: k8s.io/apimachinery/pkg/watch/watch.go
type Interface interface {
    Stop()
    ResultChan() <-chan Event
}

type Event struct {
    Type   EventType
    Object runtime.Object
}

type EventType string
const (
    Added    EventType = "ADDED"
    Modified EventType = "MODIFIED"
    Deleted  EventType = "DELETED"
    Bookmark EventType = "BOOKMARK"
    Error    EventType = "ERROR"
)
```

---

## 九、关键配置选项

### 9.1 服务器运行选项

```go
// 文件: cmd/kube-apiserver/app/options/options.go
type ServerRunOptions struct {
    GenericServerRunOptions  *genericoptions.ServerRunOptions
    Etcd                     *genericoptions.EtcdOptions
    SecureServing            *genericoptions.SecureServingOptionsWithLoopback
    Authentication           *kubeoptions.BuiltInAuthenticationOptions
    Authorization            *kubeoptions.BuiltInAuthorizationOptions
    Admission                *kubeoptions.AdmissionOptions
    // ...
}
```

### 9.2 常用启动参数

| 参数 | 说明 |
|------|------|
| `--etcd-servers` | Etcd 服务器地址 |
| `--secure-port` | HTTPS 端口 |
| `--client-ca-file` | 客户端 CA 证书 |
| `--tls-cert-file` | TLS 证书文件 |
| `--tls-private-key-file` | TLS 私钥文件 |
| `--authorization-mode` | 授权模式 |
| `--enable-admission-plugins` | 启用的准入插件 |
| `--service-cluster-ip-range` | Service IP 范围 |
| `--service-account-issuer` | Service Account 签发者 |
| `--service-account-signing-key-file` | Service Account 签名密钥 |

---

## 十、代码组织结构

### 10.1 目录结构

```
kubernetes/
├── cmd/kube-apiserver/              # 入口点
│   ├── apiserver.go                 # main 函数
│   └── app/
│       ├── server.go                # 服务器创建和启动
│       └── options/                 # 命令行选项
│
├── pkg/kubeapiserver/               # Kubernetes 特定实现
│   ├── admission/                   # 准入控制初始化
│   ├── authenticator/               # 认证器配置
│   ├── authorizer/                  # 授权器配置
│   └── options/                     # 选项定义
│
├── pkg/controlplane/                # 控制平面
│   └── instance.go                  # API Server 实例
│
└── staging/src/k8s.io/apiserver/    # 通用 API Server 库
    ├── pkg/
    │   ├── server/                  # 服务器核心
    │   │   ├── genericapiserver.go  # 通用服务器
    │   │   ├── config.go            # 配置和 Handler Chain
    │   │   └── handler.go           # HTTP 处理器
    │   │
    │   ├── admission/               # 准入控制接口
    │   ├── authentication/          # 认证接口
    │   ├── authorization/           # 授权接口
    │   ├── endpoints/               # API 端点
    │   └── registry/                # REST 注册
    │       └── rest/                # REST 接口
    │
    └── plugin/pkg/                  # 插件实现
        ├── authenticator/           # 认证插件
        └── authorizer/              # 授权插件
```

### 10.2 设计模式

1. **工厂模式** - Storage Factory 创建不同资源的存储
2. **责任链模式** - Handler Chain 处理请求
3. **策略模式** - 多种认证/授权方式可插拔
4. **适配器模式** - REST Storage 适配不同存储后端
5. **观察者模式** - Watch 机制实现事件通知

---

## 十一、扩展点

### 11.1 自定义准入控制器

```go
// 实现 admission.Interface
type MyAdmissionPlugin struct {
    *admission.Handler
}

func (p *MyAdmissionPlugin) Admit(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) error {
    // 变更逻辑
    return nil
}

func (p *MyAdmissionPlugin) Validate(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) error {
    // 验证逻辑
    return nil
}
```

### 11.2 自定义授权器

```go
// 实现 authorizer.Authorizer 接口
type MyAuthorizer struct {}

func (a *MyAuthorizer) Authorize(ctx context.Context, attrs authorizer.Attributes) (authorizer.Decision, string, error) {
    // 授权逻辑
    return authorizer.DecisionAllow, "", nil
}
```

### 11.3 自定义认证器

```go
// 实现 authenticator.Request 接口
type MyAuthenticator struct {}

func (a *MyAuthenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
    // 认证逻辑
    return &authenticator.Response{User: &user.DefaultInfo{Name: "test"}}, true, nil
}
```

---

## 十二、总结

Kubernetes API Server 是一个高度模块化、可扩展的系统：

1. **分层架构** - Handler Chain → Router → Storage → Etcd
2. **接口抽象** - 通过接口定义扩展点，支持插件化
3. **可配置性** - 通过命令行参数和配置文件灵活配置
4. **安全性** - 多层次的安全机制（认证、授权、准入控制）
5. **可观测性** - 完善的日志、指标、追踪支持

核心设计理念：
- **声明式 API** - 用户声明期望状态，系统维护实际状态
- **Watch 机制** - 实时事件通知，减少轮询开销
- **Finalizers** - 资源删除前的清理机制
- **Owner References** - 资源依赖关系和级联删除