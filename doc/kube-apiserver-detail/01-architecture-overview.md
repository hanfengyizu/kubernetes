# 01 — 架构概览

> 对应主文档第一章。本章细化分层架构、核心组件职责边界，以及请求自入口到 Etcd 的纵向数据流。

## 1.1 分层架构总览

kube-apiserver 采用**严格分层**设计，每一层只与相邻层交互，层与层之间通过 Go interface 解耦。这种设计带来三个工程价值：① 每层可独立测试；② 任一层可被替换（如 Etcd→其他存储、go-restful→其他路由器）；③ 安全检查集中在入口层，业务逻辑（Storage 层）无需重复鉴权。

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

### 四层职责

| 层 | 职责 | 关键关注点 | 不关心 |
|----|------|-----------|--------|
| **Handler Chain** | 横切关注点：panic 恢复、认证、授权、审计、限流、超时 | 安全、可观测性、过载保护 | 资源具体语义 |
| **APIServerHandler（路由层）** | URL→Handler 映射、RESTful 路由分发 | 路径解析、动词分发 | 业务逻辑 |
| **REST Storage（存储层）** | 资源 CRUD 语义、策略校验、与 etcd 交互 | 资源生命周期、字段默认值、子资源 | HTTP 协议细节 |
| **Etcd** | 强一致 KV 持久化 | 数据可靠性、Watch 通知 | Kubernetes 资源模型 |

※ 路由层有两条路径：`GoRestfulContainer` 处理 `/api`、`/apis` 下的 REST 资源请求；`NonGoRestfulMux` 处理健康检查（`/healthz`）、指标（`/metrics`）、调试（`/debug/pprof`）等非资源路由。

## 1.2 核心组件清单

| 组件 | 文件位置 | 职责 | 依赖方向 |
|------|---------|------|---------|
| **GenericAPIServer** | `staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go` | API Server 核心实现，管理生命周期、聚合各子系统 | 依赖 Config、Handler、Authorizer |
| **APIServerHandler** | `staging/src/k8s.io/apiserver/pkg/server/handler.go` | HTTP 路由管理、FullHandlerChain 持有 | 依赖 restful.Container、PathRecorderMux |
| **Config** | `staging/src/k8s.io/apiserver/pkg/server/config.go` | 服务器配置聚合体、构建 Handler Chain | 依赖各子配置（认证/授权/准入） |
| **Handler Chain** | `staging/src/k8s.io/apiserver/pkg/server/config.go` | 请求处理链组装 | 由 `DefaultBuildHandlerChain` 构建 |
| **Admission Control** | `staging/src/k8s.io/apiserver/pkg/admission/interfaces.go` | 准入控制接口（变更+验证） | 由插件链组合实现 |
| **Authorizer** | `staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go` | 授权接口 | 由 RBAC/Node/Webhook 等组合 |
| **REST Storage** | `staging/src/k8s.io/apiserver/pkg/registry/rest/rest.go` | 资源存储接口 | 由各资源 storage 实现适配 etcd |
| **RequestInfo** | `staging/src/k8s.io/apiserver/pkg/endpoints/request/requestinfo.go` | 请求信息解析（资源/动词/命名空间） | 由 `WithRequestInfo` 注入 context |

## 1.3 纵向请求流

一个 `GET /api/v1/namespaces/default/pods/nginx` 请求的纵向穿透：

```
HTTP Request
  → [Panic Recovery]      ← 捕获下游 panic，转 500
  → [RequestInfo]         ← 解析出 verb=GET, resource=pods, ns=default, name=nginx
  → [Authentication]      ← 验证客户端身份，注入 user.Info 到 context
  → [Authorization]      ← 检查 user 对 pods 资源的 get 权限
  → [Audit]               ← 记录请求元数据
  → [PriorityAndFairness] ← 限流/排队（防止过载）
  → [Route]               ← GoRestfulContainer 匹配到 pods handler
  → [Admission]           ← GET 只读，通常跳过；写请求经 Mutating→Validation
  → [PodStorage.Get]      ← REST Storage 执行读
  → [Etcd]                ← 底层 KV 读
  ← 返回 Pod 对象（经 Storage→Route→Chain 反向封装响应）
```

⚠ 误区：认证和授权在 Admission **之前**。准入控制处理的是已通过鉴权后的对象变更，不是访问控制的第一道闸门。访问控制的第一道闸门是认证。

## 1.4 横向扩展点

架构在每一层都预留了扩展接口，使外部可介入而不修改核心：

| 层 | 扩展机制 | 典型用例 |
|----|---------|---------|
| 认证 | `authenticator.Request` 接口 | OIDC、Webhook、自定义身份提供者 |
| 授权 | `authorizer.Authorizer` 接口 | 自定义授权策略 |
| 准入 | `admission.MutationInterface`/`ValidationInterface` | MutatingAdmissionWebhook、ValidatingAdmissionWebhook |
| 路由 | 聚合层（Aggregator）/ APIService | metrics-server、自定义 CRD API |
| 存储 | CRD（CustomResourceDefinition） | 自定义资源类型 |

详见 [11-扩展点](./11-extension-points.md)。
