# 03 — 请求处理流程

> 对应主文档第三章。本章逐环节解释 Handler Chain 的 23 个 middleware、它们的构建顺序为何是"反向"、以及每个环节的职责边界与常见误区。这是整个 API Server 技术含量最高的部分。

## 3.1 Handler Chain 构建顺序

> 文件：`staging/src/k8s.io/apiserver/pkg/server/config.go` 中 `DefaultBuildHandlerChain`

Handler Chain 按照**反向顺序**构建（从内到外）——即代码里最后 `wrap` 的是最外层、最先执行。原因是装饰器模式：每次 `wrap` 都返回一个新 handler 把原 handler 包在里面，所以**后包的在最外**。

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

### 逐环节职责

| # | Middleware | 职责 | 关键细节 |
|---|-----------|------|---------|
| 1 | `WithPanicRecovery` | 捕获下游 panic，转 500 + 记日志 | 最外层，保证任何崩溃不致连接挂起 |
| 2 | `WithMuxAndDiscoveryComplete` | 阻塞请求直到路由与 discovery 就绪 | 启动早期请求被挂住在此，避免 404 误报 |
| 3 | `WithRequestReceivedTimestamp` | 记录请求到达时间到 context | 供后续 latency、audit 使用 |
| 4 | `WithRequestInfo` | 解析 URL→RequestInfo 注入 context | 见 [02-核心数据结构](./02-core-data-structures.md#23-requestinfo) |
| 5 | `WithRoutine` | 特性门控下的 goroutine 执行 | 实验性并发路径，默认关 |
| 6 | `WithLatencyTrackers` | 分段延迟埋点 | 用于 metrics `REQUEST_DURATION` |
| 7 | `WithHTTPLogging` | 结构化 HTTP 访问日志 | 默认每请求一行 |
| 8 | `WithRetryAfter` | shutdown 时返回 `Retry-After` 头 | 让客户端重试到其他实例 |
| 9 | `WithHSTS` | 注入 `Strict-Transport-Security` | 强制 HTTPS |
| 10 | `WithCacheControl` | 给 GET 响应加 `Cache-Control` | 防中间缓存误缓存敏感数据，通常 no-store |
| 11 | `WithProbabilisticGoaway` | 随机发 HTTP/2 GOAWAY | 让客户端负载均衡到其他 apiserver，避免连接粘滞 |
| 12 | `WithWatchTerminationDuringShutdown` | shutdown 时优雅断开 watch | 见第 08 章 |
| 13 | `WithWaitGroup` | 用 WaitGroup 跟踪在途请求 | shutdown 时等待所有在途请求结束 |
| 14 | `WithRequestDeadline` | 应用请求级 deadline | 与 timeout 协作，支持 per-request 期限 |
| 15 | `WithTimeoutForNonLongRunningRequests` | 非长运行请求超时 | watch/list 大量数据是长运行，不被此超时杀 |
| 16 | `WithWarningRecorder` | 收集 `Warning` 响应头 | 让 admission 等可向客户端发警告 |
| 17 | `WithCORS` | CORS 跨域 | 浏览器场景才需要 |
| 18 | `WithAuthentication` | 认证，注入 user.Info | 见第 06 章 |
| 19 | `WithTracing` | 分布式追踪 span | OTel/桥接到 tracing 后端 |
| 20 | `WithAudit` | 审计日志 | 记录"谁在什么时间做了什么" |
| 21 | `WithImpersonation` | 处理 `Impersonate-*` 头 | 让高权限用户代理他人身份（需 `impersonate` 权限） |
| 22 | `WithPriorityAndFairness` | 优先级与公平性限流 | APF，防过载核心机制 |
| 23 | `WithAuthorization` | 授权检查 | 见第 06 章 |

⚠ 顺序敏感点：
- `WithAuthentication` 必须在 `WithImpersonation` 之前——先确认真实身份，才能判断是否有权模拟他人。
- `WithAuthorization` 在 `WithPriorityAndFairness` 之后：限流发生在鉴权前会泄露"资源是否存在"（通过拒绝差异），放鉴权后更安全。
- `WithPanicRecovery` 最外层，确保即使下游 panic 也能返回结构化 500。

## 3.2 请求处理流程图

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

## 3.3 顺序为何如此设计

| 设计决策 | 原因 |
|---------|------|
| Panic 在最外 | 任何下游异常都不该让连接挂起 |
| RequestInfo 在 Authn 前 | 让 Authn/Authorizer 能从 context 取资源信息决策 |
| Authn 在 Authorizer 前 | 没有身份就无法授权 |
| Authorizer 在 Admission 前 | 准入是"已授权对象"的语义校验，不是访问控制 |
| Admission 在 Storage 前 | 变更/校验必须在落库前完成 |
| Audit 横跨整个链 | 审计需要请求前后两个快照（admission 时 + 响应后） |
| PriorityAndFairness 在 Authorizer 后 | 限流不应泄露未授权信息 |

## 3.4 长运行 vs 非长运行请求

`WithTimeoutForNonLongRunningRequests` 只对**非长运行请求**生效。区分依据是 `RequestInfo`：

| 请求类型 | 是否长运行 | 超时处理 |
|---------|-----------|---------|
| `WATCH` | 是 | 不被此 middleware 杀，由 watch ctx 或 shutdown 控制 |
| `GET/LIST` 大量数据 | 视情况 | list 大集合可能走长运行路径 |
| `CREATE/UPDATE/DELETE` | 否 | 受 `minRequestTimeout` 约束 |
| `CONNECT`（exec/attach） | 是 | 长连接，单独管理 |

※ 设计意图：watch/exec 等长连接若被粗暴超时杀掉会破坏客户端一致性，故单独走长运行管理路径。
