# 06 — 认证、授权、准入控制

> 对应主文档第六章。本章细化三道安全闸门的实现细节、策略选项与组合规则。这三者顺序固定为 Authn → Authorizer → Admission，构成 K8s 的纵深防御。

## 6.1 认证 (Authentication)

认证回答"你是谁"。kube-apiserver 支持**多种认证方式并存**，形成一个认证器链：请求依次尝试每个认证器，任一成功即确定身份并短路。

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

### 九种认证方式对照

| 方式 | 凭证载体 | 适用场景 | 备注 |
|------|---------|---------|------|
| Client Cert | TLS 客户端证书 | 内部组件互信（kubelet↔apiserver） | 证书 CN/ORG 映射为 user/group |
| Token File | `Authorization: Bearer` | 静态长期 token | 需重启才能更换，不推荐生产 |
| Bootstrap Token | `Authorization: Bearer` | 节点加入集群引导 | 短期、自动过期、限权 |
| Service Account | `Authorization: Bearer` | Pod 内访问 API | JWT 形式，挂载到 Pod |
| OIDC | `Authorization: Bearer`（ID Token） | 外部 IdP 集成 | 对接 Keycloak/Auth0 等 |
| Webhook | 任意（由 webhook 解析） | 企业自定义认证 | apiserver 转发请求给 webhook |
| Request Header | 代理注入的头 | 反向代理前置认证 | 信任代理转发的 `X-Remote-User` |
| Anonymous | 无 | 公开资源/未认证兜底 | 默认开，可被授权层拒绝 |

### 认证器链行为

```
Request → [CertAuthn] → [TokenFile] → [BootstrapToken] → [OIDC] → ... → [Anonymous]
              ↓             ↓               ↓              ↓
           成功?           成功?           成功?          成功?
              │是→短路      │是→短路        │是→短路      │
              ↓否           ↓否             ↓否           ↓否
                                                          最终 Anonymous
                                                          （返回 system:anonymous）
```

- 任一认证器成功 → 注入 `user.Info`，后续认证器不再调用。
- 全部失败 → 仍可能以 `system:anonymous` 身份继续（由 Anonymous 认证器兜底），但**能否访问由授权层决定**。
- 链中可穿插 `Webhook` 认证器，按需调用外部服务。

⚠ 顺序很重要：高优先级、低延迟的认证器放前面，Webhook 等慢认证器放后面，避免每个请求都打外部服务。

认证配置文件：`pkg/kubeapiserver/options/authentication.go`

## 6.2 授权 (Authorization)

授权回答"你能做什么"。与认证类似，授权器也组合成链，但决策语义不同：使用**三态决策**（见 [04 章](./04-core-interfaces.md#三态决策)）。

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

### 六种授权模式对照

| 模式 | 决策依据 | 适用场景 | 特点 |
|------|---------|---------|------|
| AlwaysAllow | 无 | 调试 | 危险，仅开发用 |
| AlwaysDeny | 无 | 测试拒绝路径 | 几乎不用 |
| ABAC | 静态策略文件 | 早期方案 | 粒度粗、改要重启，已不推荐 |
| Webhook | 外部服务 | 企业集成 | 灵活但增加延迟 |
| RBAC | Role/ClusterRole + Binding | **生产默认** | 声明式、可热更新 |
| Node | 节点身份 + 限制策略 | kubelet 访问 | 专对节点，配合 RBAC |

### 授权决策流程

```
Request → Authorizer → Decision
                          │
                          ├── DecisionAllow    → 继续
                          ├── DecisionDeny     → 拒绝
                          └── DecisionNoOpinion → 检查下一个授权器
```

### 生产典型组合

`--authorization-mode=Node,RBAC` 是最常见的生产配置：
1. **Node 授权器**先看：若是 kubelet 用其节点证书发起的请求，按节点授权策略决策（允许该节点操作自己 Pod 的状态等）；否则 NoOpinion。
2. **RBAC 授权器**接着看：按 RoleBinding/ClusterRoleBinding 匹配 verb+resource 决策。
3. 链尾仍 NoOpinion → 拒绝（fail-closed）。

※ Node 授权器在 RBAC 之前：节点请求量大且模式固定，先短路可省去 RBAC 全表扫描开销。

## 6.3 准入控制 (Admission Control)

准入回答"这个变更是否允许、是否需要改"。准入只对**写操作**（CREATE/UPDATE/DELETE/CONNECT）生效，读请求不经过准入。准入分**两阶段**：先 Mutating 后 Validation。

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

### 两阶段执行规则

1. **Mutating 阶段**：所有变更插件按配置顺序执行，可修改对象。失败立即拒绝整请求。
2. **对象重新序列化**：变更后的对象需重新序列化，确保 Validation 看到一致形态。
3. **Validation 阶段**：所有验证插件执行，只读校验。任一失败拒绝。
4. 落 etcd。

⚠ Mutating 阶段的插件顺序会影响最终对象（如先注入默认值再校验范围）。生产中应把基础默认值注入放前，业务变更放后。

### 常见准入控制器

| 准入控制器 | 功能 | 阶段 |
| ---------------------------- | ---------- | ---- |
| `NamespaceLifecycle`         | 命名空间生命周期管理（终止中拒绝创建） | Validation |
| `LimitRanger`                | 资源限制范围检查（默认值+上限） | Mutating + Validation |
| `ServiceAccount`             | 服务账户自动挂载/补全 | Mutating |
| `DefaultStorageClass`        | 默认存储类注入 | Mutating |
| `ResourceQuota`              | 资源配额检查（用量统计） | Validation |
| `PodSecurityPolicy`          | Pod 安全策略（已废弃，被 PSA 替代） | Validation |
| `NodeRestriction`            | 节点限制（kubelet 只能改自己节点对象） | Validation |
| `MutatingAdmissionWebhook`   | 变更 Webhook（外部服务） | Mutating |
| `ValidatingAdmissionWebhook` | 验证 Webhook（外部服务） | Validation |

### 准入 vs 授权的区别

| 维度 | 授权 | 准入 |
|------|------|------|
| 关注点 | 身份能否操作资源 | 对象内容是否合法/需修改 |
| 是否修改对象 | 否 | Mutating 可改 |
| 作用对象 | 请求（动词+资源） | 对象内容 |
| 触发时机 | 链中较早 | 落库前最后 |

二者互补：授权管"能不能动"，准入管"动得对不对、动得好不好"。
