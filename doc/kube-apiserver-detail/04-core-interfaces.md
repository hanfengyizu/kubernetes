# 04 — 核心接口设计

> 对应主文档第四章。本章细化 REST Storage 的接口层次、Admission 与 Authorizer 的接口契约，解释"为什么是组合式小接口而非大接口"。

## 4.1 REST Storage 接口层次

> 文件：`staging/src/k8s.io/apiserver/pkg/registry/rest/rest.go`

REST Storage 采用**接口组合**而非单一巨型接口——每个资源 storage 只实现它需要的操作。一个只读资源不必实现 `Creater`，一个不支持 watch 的资源不必实现 `Watcher`。这是接口隔离原则（ISP）的典型应用。

```go
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

### 接口选择矩阵

| 资源示例 | 实现的接口 | 不实现的接口 | 原因 |
|---------|-----------|-------------|------|
| Pod | StandardStorage + Connecter | — | 全功能，支持 exec/log |
| ConfigMap | StandardStorage | Connecter | 无子资源连接需求 |
| Binding（pod 绑定） | Storage（create-only） | Getter/Lister 等 | 仅接受创建，不可读 |
| Event | StandardStorage | — | 但 TTL 自动清理 |
| Status 子资源 | Updater（仅 status） | Creater/Deleter | 只允许更新 status 字段 |
| Scale 子资源 | Updater + Getter | — | 缩放语义受限 |

※ `GracefulDeleter` 的"优雅"指支持 `gracePeriodSeconds`：删除请求先标记 `deletionTimestamp`，等 finalizer 清理完才真正从 etcd 移除。

### Update 的 `bool` 返回值

`Update` 返回 `(obj, created, err)`。第二个 `bool` 表示**是否是新建**——因为 K8s 的 Update 在对象不存在时可由某些 storage 转为 Create（upsert 语义）。调用方据此判断审计动词。

## 4.2 Admission Control 接口

> 文件：`staging/src/k8s.io/apiserver/pkg/admission/interfaces.go`

```go
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

### 设计要点

- **`Handles(operation)` 优化**：每个插件声明自己关心哪些操作。`Handles` 返回 false 时整个插件链跳过该插件，避免无谓调用。
- **Mutation 与 Validation 分离**：变更插件可修改对象（注入默认值、补全字段），验证插件只读校验。两者**不可混用**——一个 Mutating 插件若同时校验失败会中断整个链，导致已变更状态难以回滚。
- **两阶段执行顺序**：所有 Mutating 插件先执行完 → 对象序列化 → 再执行所有 Validation 插件。这样验证看到的是最终形态，避免"先验证再变更"导致验证结果失效。

⚠ 误区：MutatingWebhook 与 ValidatingWebhook 的执行顺序并非"交替"，而是"先全部 Mutating，再全部 Validation"。这意味着 MutatingWebhook 的变更会被后续 ValidatingWebhook 检查到。

## 4.3 Authorizer 接口

> 文件：`staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go`

```go
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

### 三态决策

| Decision | 含义 | 链式行为 |
|----------|------|---------|
| `DecisionAllow` | 显式允许 | 短路：立即放行，不再询问后续授权器 |
| `DecisionDeny` | 显式拒绝 | 短路：立即拒绝，不再询问 |
| `DecisionNoOpinion` | 不置可否 | 传递给链中下一个授权器 |

授权器组合成链（如 `[Node, RBAC, Webhook]`）。`NoOpinion` 的存在让"各授权器只管自己懂的领域"成为可能——例如 Node 授权器只对 kubelet 发起的请求有意见，其他一律 NoOpinion 交给 RBAC。

⚠ 链尾若全是 NoOpinion，最终决策为**拒绝**（fail-closed）。这是安全默认：未知情况拒绝而非放行。

### Attributes 提供的决策维度

授权基于"6W"决策：
- **Who**：`GetUser()` + 用户组 + extra
- **What**：`GetResource()` + `GetSubresource()` + `GetAPIGroup/Version()`
- **Where**：`GetNamespace()`（cluster vs namespaced）
- **How**：`GetVerb()` + `IsReadOnly()`（read-only 可放宽某些限制）
- **Which**：`GetName()`（针对具体对象）

RBAC 的 RoleBinding 即基于这些维度匹配 `verb + resource + group + namespace`。
