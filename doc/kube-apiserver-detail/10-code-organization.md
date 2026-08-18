# 10 — 代码组织结构

> 对应主文档第十章。本章细化目录布局背后的分层逻辑，以及设计模式在代码中的落地位置。

## 10.1 目录结构

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

### 三层目录的分工

| 层 | 目录 | 角色 | 依赖方向 |
|----|------|------|---------|
| 入口层 | `cmd/kube-apiserver/` | 组装参数、启动进程 | 依赖 pkg 与 staging |
| K8s 专属层 | `pkg/kubeapiserver/`、`pkg/controlplane/` | K8s 特有的认证/授权/准入配置、内置组安装 | 依赖 staging 通用库 |
| 通用库层 | `staging/src/k8s.io/apiserver/` | 通用 API Server 骨架（可被任何 apiserver 复用） | 仅依赖 apimachinery 等基础库 |

※ 关键设计：`staging/src/k8s.io/apiserver` 是**独立可发布的库**（单独的 go module），不依赖任何 K8s 特定代码。这使得 kube-aggregator、metrics-server、CRD-based apiserver 都能复用同一套通用骨架。`cmd` 和 `pkg` 才是 K8s 专属的"组装+定制"。

### staging 机制

`staging/` 下的代码以独立 module 形式存在，主仓库通过 vendor 引用。这样 `k8s.io/apiserver`、`k8s.io/api`、`k8s.io/apimachinery` 等可被外部项目直接 import，形成 K8s 生态的公共基础库。

## 10.2 设计模式

### 工厂模式 — Storage Factory 创建不同资源的存储

落地：`pkg/registry/` 下各资源的 `NewRESTStorage`，以及 `RESTStorageProvider`（见 [07 章](./07-api-group-installation.md#72-rest-storage-provider)）。每个 provider 是一个工厂，按配置产出对应 storage 实例。

### 责任链模式 — Handler Chain 处理请求

落地：`DefaultBuildHandlerChain`（见 [03 章](./03-request-handling.md#31-handler-chain-构建顺序)）。每个 `WithXxx` middleware 是链上一个节点，请求依次穿过。

### 策略模式 — 多种认证/授权方式可插拔

落地：认证器链、授权器链（见 [06 章](./06-authn-authz-admission.md)）。同一接口（`authenticator.Request` / `authorizer.Authorizer`）有多种实现，运行时按配置组合。

### 适配器模式 — REST Storage 适配不同存储后端

落地：`pkg/registry/.../storage` 把 etcd 操作适配为 REST 语义。上层只调用 `Getter.Get`，不感知底层是 etcd 还是其他 KV。CRD storage 是另一适配器，把"自定义资源存储"适配为同一 REST 接口。

### 观察者模式 — Watch 机制实现事件通知

落地：`watch.Interface`（见 [08 章](./08-watch-mechanism.md#82-watch-接口)）。subject 是 etcd/Storage 层，observer 是 watch 连接，事件通过 `ResultChan` 推送。

### 装饰器模式 — Handler Chain 构建

落地：每个 `WithXxx(handler)` 返回一个包裹原 handler 的新 handler，层层嵌套形成完整链。这是 Go 实现 middleware 的惯用法，本质是装饰器。

### 组合优于继承 — 接口隔离

落地：`StandardStorage` 由 `Getter`、`Lister`、`CreaterUpdater` 等小接口组合（见 [04 章](./04-core-interfaces.md#41-rest-storage-接口层次)）。资源按需实现，避免"胖接口"。

## 10.3 模式协作的总览

一个写请求（如 `POST /pods`）穿过的模式：

```
[装饰器] HandlerChain 层层包裹
   → [责任链] 依次经 Panic/RequestInfo/Authn/Authorizer/Audit/...
      → [策略] Authn 用配置的认证器链；Authorizer 用授权器链
         → [工厂] 路由到 PodStorage（由 RESTStorageProvider 创建）
            → [适配器] PodStorage.Get/Create 适配 etcd 操作
               → [观察者] 若有 watcher，触发 watch.Event 推送
```

每个模式各司其职，组合出完整的安全+语义+存储链路。
