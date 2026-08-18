# 11 — 扩展点

> 对应主文档第十一章。本章细化三类扩展点的接口契约、实现步骤与注册方式，并指出每类扩展的边界与陷阱。

扩展点的共同原则：**实现接口 + 注册到链**。K8s 把安全/语义决策都抽象为可插拔接口，扩展者只需实现接口并通过配置注入，无需改动核心代码。

## 11.1 自定义准入控制器

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

### 实现要点

| 步骤 | 内容 |
|------|------|
| 1. 选择接口 | 仅变更实现 `MutationInterface`；仅验证实现 `ValidationInterface`；二者皆有则都实现 |
| 2. 嵌入 `admission.Handler` | 获得 `Handles` 默认实现与资源初始化支持 |
| 3. 实现 `Admit`/`Validate` | 通过 `Attributes` 取操作类型、对象、用户、资源等做决策 |
| 4. 注册插件 | 通过 `plugin.Register` + 配置 `--enable-admission-plugins` 启用 |

### Attributes 提供的决策输入

`admission.Attributes` 接口提供：
- `GetOperation()`：CREATE/UPDATE/DELETE/CONNECT
- `GetObject()` / `GetOldObject()`：新对象与旧对象（UPDATE 时两者都有）
- `GetUserInfo()`：发起者身份
- `GetResource()` / `GetSubresource()` / `GetNamespace()`：定位资源
- `GetName()`：资源名

⚠ 陷阱：
- **Mutating 插件不应同时做强校验**——校验失败会中断链，已变更的其他插件状态难以回滚。校验交给独立的 Validating 插件。
- **Webhook 准入更常用**：大多数场景不需要写 in-tree 插件，用 `MutatingAdmissionWebhook`/`ValidatingAdmissionWebhook` 配外部服务即可，无需改 apiserver 代码、可独立升级。
- **`reinvocationPolicy`**：若你的 Mutating 插件改了对象，可能需要其他插件重跑——通过 `ReinvocationPolicyIfNeeded` 控制。

### Webhook 准入的工作方式

```
apiserver → 准入点 → HTTP POST（admission review）→ 外部 webhook 服务
                                                          ↓
              ← 响应（allow/deny/patch）←─────────────────
```

webhook 返回 `response.allowed=false` 拒绝，或返回 `response.patch` 对对象做 JSON Patch（Mutating 场景）。这是生产中最常见的扩展方式。

## 11.2 自定义授权器

```go
// 实现 authorizer.Authorizer 接口
type MyAuthorizer struct {}

func (a *MyAuthorizer) Authorize(ctx context.Context, attrs authorizer.Attributes) (authorizer.Decision, string, error) {
    // 授权逻辑
    return authorizer.DecisionAllow, "", nil
}
```

### 三态决策的使用

| 情况 | 返回 | 效果 |
|------|------|------|
| 明确允许 | `DecisionAllow, reason, nil` | 短路放行 |
| 明确拒绝 | `DecisionDeny, reason, nil` | 短路拒绝 |
| 不归我管 | `DecisionNoOpinion, "", nil` | 传给链中下一个授权器 |
| 出错 | `0, "", err` | 通常视为拒绝（fail-closed） |

### 注册与组合

自定义授权器需在 apiserver 启动配置中加入授权器链。与认证不同，授权器**较少自定义 in-tree**——生产多用内置 RBAC + Webhook 授权（`--authorization-webhook-config-file`）。

⚠ Webhook 授权的延迟敏感：每个授权决策都要打外部服务，若 webhook 慢会拖垮所有请求。务必给 webhook 设超时并考虑缓存。

### Attributes 的决策维度

见 [04 章 Authorizer](./04-core-interfaces.md#attributes-提供的决策维度)。自定义授权器可基于 user/group/verb/resource/namespace/name 任意组合决策。

## 11.3 自定义认证器

```go
// 实现 authenticator.Request 接口
type MyAuthenticator struct {}

func (a *MyAuthenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
    // 认证逻辑
    return &authenticator.Response{User: &user.DefaultInfo{Name: "test"}}, true, nil
}
```

### 返回值语义

| 返回 | 含义 |
|------|------|
| `(resp, true, nil)` | 认证成功，`resp.User` 注入 context |
| `(nil, false, nil)` | 此认证器无法识别该请求，交给链中下一个 |
| `(nil, false, err)` | 认证过程出错 |

⚠ 关键区别：**返回 false 不等于拒绝**。false 只是"我不认识"，交给下一个认证器；只有所有认证器都 false 时，请求才以 `system:anonymous` 继续（能否访问由授权层决定）。

### 实现要点

- 从 `req.Header`（如 `Authorization`）或 `req.TLS`（客户端证书）提取凭证。
- 验证凭证有效性（签名、过期、吊销等）。
- 构造 `user.Info`（Name、UID、Groups、Extra）返回。
- 缓存昂贵验证结果（如外部 OIDC token 校验），避免每请求打外部服务。

### 注册方式

自定义认证器加入认证器链（与内置 Client Cert、OIDC 等并列）。生产中更常用 Webhook TokenAuth（`--authentication-token-webhook-config-file`）让外部服务校验 token，无需改 apiserver。

## 11.4 三类扩展点的对照

| 维度 | 认证 | 授权 | 准入 |
|------|------|------|------|
| 回答 | 你是谁 | 你能做什么 | 这个变更允许吗/需要改吗 |
| 接口 | `authenticator.Request` | `authorizer.Authorizer` | `admission.Mutation/ValidationInterface` |
| 链行为 | 任一成功短路 | 三态决策传递 | Mutating 全跑→Validation 全跑 |
| 可修改对象 | 否 | 否 | Mutating 可改 |
| 生产常用扩展 | Webhook TokenAuth | Webhook Authz | AdmissionWebhook |
| in-tree 自定义 | 少见 | 少见 | 少见（多走 webhook） |

## 11.5 扩展的优先级：能用 webhook 就不自建 in-tree

K8s 的扩展哲学：**优先用声明式配置的 webhook，其次才是写代码注入 in-tree**。原因：
- webhook 可独立部署、独立升级、独立伸缩，与 apiserver 解耦。
- in-tree 扩展需重新编译 apiserver，版本绑定，运维成本高。
- webhook 是云原生的：可观测、可灰度、可回滚。

仅当性能要求极高（无法接受网络往返）或需要深度介入对象序列化时，才考虑 in-tree 插件。
