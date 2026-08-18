# 08 — Watch 机制

> 对应主文档第八章。本章细化 Watch 的端到端实现、事件模型、资源版本（ResourceVersion）的作用，以及 shutdown 时的优雅断开。

## 8.1 Watch 实现原理

Watch 是 K8s 实时性的基石。相比客户端轮询 `LIST`，Watch 让客户端一次订阅、持续接收增量事件，大幅降低负载与延迟。端到端链路涉及客户端、apiserver、etcd 三方协作。

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

### 关键点

- 客户端发起 `GET /api/v1/pods?watch=true&resourceVersion=<RV>`，挂起长连接。
- apiserver 在 etcd 上建立 watch（从指定 RV 之后），把 etcd 的事件转为 K8s 事件对象流式回写。
- 连接保持直到客户端断开、服务端 shutdown、或出错。

※ `resourceVersion` 是 watch 的"断点续传"游标：客户端记录最后收到的事件 RV，重连时带上，apiserver 从该 RV 之后继续推送，避免漏事件。

## 8.2 Watch 接口

> 文件：`k8s.io/apimachinery/pkg/watch/watch.go`

```go
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

### 事件类型语义

| EventType | 含义 | 触发 |
|-----------|------|------|
| `ADDED` | 新对象创建 | etcd PUT 且原值不存在 |
| `MODIFIED` | 对象更新 | etcd PUT 且原值存在 |
| `DELETED` | 对象删除 | etcd DELETE |
| `BOOKMARK` | 书签（仅含 RV，无对象变化） | 周期性发送，让客户端更新 RV 游标 |
| `ERROR` | 出错（如 RV 过期） | apiserver 无法从该 RV 续传时 |

### Bookmark 的设计意图

⚠ 这是常被忽略但关键的优化。问题：长 watch 连接若只在没有变更时挂起，客户端的 RV 永远停在旧值；连接断开重连时，apiserver 发现"该 RV 之后有大量未推送历史"，可能返回 `410 Gone` 要求重新 LIST。

Bookmark 定期发一个"当前 RV 但无对象变更"的事件，让客户端刷新游标。这样重连时 RV 较新，续传代价小。通过 ListOptions 的 `allowWatchBookmarks` 控制。

## 8.3 ResourceVersion 的语义

`ResourceVersion`（RV）是 etcd 的 `mod_revision`（或 etcd watch 的 compact revision）在 K8s 的暴露：

| 场景 | RV 行为 |
|------|---------|
| GET/LIST 响应 | 返回对象当时的 RV |
| Watch 请求 | 客户端带"从该 RV 之后"订阅 |
| 乐观并发更新 | 更新请求带 `resourceVersion`，apiserver 比对防覆盖 |
| Watch 过期 | RV 被 etcd compact 掉 → 返回 `410 Gone`，需重新 LIST |

※ RV 不是全局递增计数器意义上的"版本号"，而是 etcd 修改序号的字符串化。它保证**单调性**：RV 越大对应的状态越新，但中间可能有空洞（被 compact）。

## 8.4 Watch 的资源消耗与保护

Watch 是长连接、有状态的，对 apiserver 是昂贵资源：

- 每个 watch 在 apiserver 占一个 goroutine + 缓冲区 + etcd watch handle。
- `WithWatchTerminationDuringShutdown` 在停机时主动断开所有 watch，释放资源。
- `WithTimeoutForNonLongRunningRequests` 不杀 watch（长运行豁免），由专门的 watch context 管理。
- PriorityAndFairness 对 watch 也有专门的席位（watch-only 优先级），避免 watch 挤占写请求。

### Shutdown 时序

```
apiserver 收到 SIGTERM
  │
  ├── readyz 翻为 not-ready（摘流量）
  ├── WithWatchTerminationDuringShutdown 通知所有 watch 关闭
  │     └── 客户端收到连接断开，带最后 RV 重连到其他实例
  ├── WithWaitGroup 等待在途非 watch 请求结束
  └── 关闭 etcd 连接
```

客户端侧配合：informers 收到断开后用记录的 RV 重连，新 apiserver 从该 RV 续传。若 RV 已过期，informer 触发 `Relist`（全量 LIST 重建），保证最终一致。

## 8.5 List+Watch 模式

生产客户端（informer）不直接 watch，而是用 **List+Watch** 两阶段：

1. **List**：一次性获取全量对象 + 当前列表 RV。
2. **Watch**：从该 RV 之后订阅增量。

这样既有全量快照又有实时增量，且断连重连只需从 RV 续传，无需重新全量。这是 controller-manager、scheduler 等组件获取状态的统一模式。
