# 09 — 关键配置选项

> 对应主文档第九章。本章细化 `ServerRunOptions` 的结构与常用启动参数的含义、影响与默认行为。

## 9.1 服务器运行选项

> 文件：`cmd/kube-apiserver/app/options/options.go`

```go
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

### 结构体分组逻辑

`ServerRunOptions` 采用**按关注点分组**的组合方式，每组对应一个子配置对象：

| 子配置 | 管理范围 | 关键开关示例 |
|--------|---------|-------------|
| `GenericServerRunOptions` | 通用运行参数 | `--max-mutating-requests-inflight`、`--min-request-timeout` |
| `Etcd` | 存储后端 | `--etcd-servers`、`--etcd-prefix`、`--etcd-compaction-interval` |
| `SecureServing` | HTTPS 监听 | `--secure-port`、`--bind-address`、`--tls-cert-file` |
| `Authentication` | 认证方式 | `--oidc-issuer-url`、`--token-auth-file` |
| `Authorization` | 授权模式 | `--authorization-mode`、`--authorization-webhook-config-file` |
| `Admission` | 准入插件 | `--enable-admission-plugins`、`--admission-control-config-file` |

※ 这种分组让每个子模块可独立 `AddFlags`、`Complete`、`Validate`，避免巨型选项结构体难维护。新增配置域只需加一个子配置字段。

## 9.2 常用启动参数

| 参数 | 说明 | 默认/典型值 |
| ------------------------------------ | -------------------- | ----------- |
| `--etcd-servers`                     | Etcd 服务器地址 | `https://127.0.0.1:2379` |
| `--secure-port`                      | HTTPS 端口 | `6443` |
| `--client-ca-file`                   | 客户端 CA 证书 | 必填（生产） |
| `--tls-cert-file`                    | TLS 证书文件 | 必填（生产） |
| `--tls-private-key-file`             | TLS 私钥文件 | 必填（生产） |
| `--authorization-mode`               | 授权模式 | `Node,RBAC` |
| `--enable-admission-plugins`         | 启用的准入插件 | `NodeRestriction,...` |
| `--service-cluster-ip-range`         | Service IP 范围 | 如 `10.96.0.0/12` |
| `--service-account-issuer`           | Service Account 签发者 | `https://kubernetes.default.svc` |
| `--service-account-signing-key-file` | Service Account 签名密钥 | 必填 |

### 参数分组解读

#### 存储（Etcd）

| 参数 | 作用 | 调优要点 |
|------|------|---------|
| `--etcd-servers` | 后端地址列表 | 多实例用逗号分隔，apiserver 会负载均衡 |
| `--etcd-prefix` | key 前缀 | 多集群共用 etcd 时隔离 |
| `--etcd-compaction-interval` | 自动 compaction 间隔 | 影响 watch RV 有效窗口 |
| `--default-watch-cache-size` | watch 缓存大小 | 减少直打 etcd 的 watch |

#### 安全服务（SecureServing）

| 参数 | 作用 | 注意 |
|------|------|------|
| `--secure-port` | HTTPS 端口 | 生产唯一入口 |
| `--bind-address` | 监听地址 | 生产应为 `0.0.0.0` 或具体网卡 |
| `--tls-cert-file` / `--tls-private-key-file` | 服务端证书 | 证书 SAN 须含所有访问入口 |
| `--http2-max-streams-per-connection` | HTTP/2 流上限 | 防 watch 过多耗尽流 |

#### 认证（Authentication）

| 参数 | 作用 |
|------|------|
| `--client-ca-file` | 启用客户端证书认证 |
| `--token-auth-file` | 启用静态 token 文件认证 |
| `--oidc-issuer-url` / `--oidc-client-id` | OIDC 配置 |
| `--service-account-issuer` / `--service-account-signing-key-file` | SA JWT 签发 |

#### 授权（Authorization）

| 参数 | 作用 |
|------|------|
| `--authorization-mode` | 授权器链（逗号分隔） |
| `--authorization-webhook-config-file` | Webhook 授权配置 |
| `--authorization-rbac-super-user` | RBAC 超级用户（绕过 RBAC） |

#### 准入（Admission）

| 参数 | 作用 |
|------|------|
| `--enable-admission-plugins` | 启用的插件（顺序敏感） |
| `--disable-admission-plugins` | 禁用默认插件 |
| `--admission-control-config-file` | 插件配置（如 webhook 配置） |

## 9.3 配置生效时序

```
解析 flags → opts 各子配置
  ↓
opts.Complete()  ← 补默认值、校验、转换
  ↓
CreateServerChain 用各子配置构造认证器/授权器/准入链
  ↓
PrepareRun 应用剩余运行时配置（健康检查阈值等）
  ↓
RunWithContext 启动监听
```

⚠ `--enable-admission-plugins` 的**顺序**对 Mutating 阶段有影响（验证阶段顺序无关）。例如 `NamespaceLifecycle` 应在创建类插件之前，否则可能为已终止命名空间创建对象后被拒。

## 9.4 配置变更的影响边界

| 变更类型 | 是否需重启 | 原因 |
|---------|-----------|------|
| `--etcd-servers` | 是 | 启动时建立连接池 |
| `--authorization-mode` | 是 | 链在启动时组装 |
| RBAC Role/Binding | 否 | 存 etcd，热读 |
| `--enable-admission-plugins` | 是 | 链在启动时组装 |
| AdmissionWebhook 配置（ConfigMap） | 否 | webhook 插件热读配置 |
| TLS 证书 | 支持 hot reload | `--tls-cert-file` 监听文件变化 |

※ RBAC 与 webhook 配置可热更新，是因为它们的"决策数据"在 etcd/ConfigMap，而"决策引擎"（授权器/插件实例）在启动时固定。这是"引擎与数据分离"的配置管理思路。
