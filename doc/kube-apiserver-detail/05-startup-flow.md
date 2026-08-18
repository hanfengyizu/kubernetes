# 05 — 启动流程

> 对应主文档第五章。本章细化从 `main()` 到安全服务监听的完整链路，解释每个阶段做什么、为什么这样分阶段。

## 5.1 入口点

> 文件：`cmd/kube-apiserver/apiserver.go`

```go
func main() {
    command := app.NewAPIServerCommand()
    code := cli.Run(command)
    os.Exit(code)
}
```

入口极简：构造 Cobra 命令 → `cli.Run` 执行 → 退出码。`cli.Run` 封装了信号处理、panic 恢复、退出码标准化等横切逻辑，让各组件入口保持一致。

## 5.2 命令创建

> 文件：`cmd/kube-apiserver/app/server.go`

```go
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

`options.NewServerRunOptions()` 创建默认配置对象，`AddFlags` 把每个配置项注册为 Cobra flag。这里体现了"配置即结构体"模式：命令行参数与 Go 字段一一映射，避免了字符串键值散落。

## 5.3 服务器启动

> 文件：`cmd/kube-apiserver/app/server.go`

```go
func Run(ctx context.Context, opts *options.ServerRunOptions) error {
    // 1. 创建配置
    server, err := CreateServerChain(ctx, opts.Complete()...)

    // 2. 准备运行
    prepared, err := server.PrepareRun()

    // 3. 运行服务器
    return prepared.RunWithContext(ctx)
}
```

### 三阶段分离的意义

| 阶段 | 做什么 | 为什么独立 |
|------|--------|-----------|
| `opts.Complete()` | 校验参数、补全默认值、转换格式 | 把"原始 flag"变为"已校验配置"，后续不再处理 nil/非法 |
| `CreateServerChain` | 构建服务器对象树（config→kube→aggregator） | 纯构造，无副作用，便于测试 |
| `PrepareRun` | 安装健康检查、discovery、注册路由 | "装弹"阶段，可在此刻做依赖就绪检查 |
| `RunWithContext` | 真正监听端口、接受连接 | 唯一阻塞阶段，受 ctx 控制可优雅停机 |

※ 这种"构造→准备→运行"三段式让优雅停机成为可能：`RunWithContext` 监听 ctx.Done()，收到信号后停止接受新连接、等待在途请求、关闭存储。

## 5.4 启动流程图

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

### CreateServerChain 内部顺序

服务器链是**嵌套**结构：aggregator 包 kube-apiserver，kube-apiserver 包 genericapiserver。构建顺序由内向外：

1. **`CreateKubeAPIServerConfig`**：组装 K8s 专属配置——认证器链、授权器链、准入插件链、服务 IP 范围、service account 签发者等。
2. **`createKubeAPIServer`**：基于 config 创建 `Instance`（K8s 版 GenericAPIServer），调用 `InstallAPIGroups` 把 core/apps/batch 等内置组的 REST 路由装上去。
3. **`CreateAggregatorServer`**：在 kube-apiserver 之上套聚合层，处理 `APIService` 资源——把对 `/apis/<group>/<version>/...` 的请求按 APIService 定义转发到对应扩展 apiserver（如 metrics-server）。

※ 聚合层在最外，因此一个请求可能先被 aggregator 拦截转发，未命中才回落到内置 kube-apiserver 路由。

## 5.5 RunWithContext 的运行时行为

```
RunWithContext(ctx)
  ├── 启动健康检查 server（healthz/livez/readyz 端点）
  ├── 启动非阻塞路由（debug、metrics 等不经过完整 chain 的端点）
  ├── 启动安全服务（HTTPS，套满 HandlerChain）
  ├── 注册信号处理（SIGTERM/SIGINT → 触发 shutdown）
  └── 阻塞等待 ctx.Done()
        │
        ▼ (收到停机信号)
  ├── 停止接受新连接
  ├── WithWaitGroup 等待在途请求结束（最多 shutdownTimeout）
  ├── WithWatchTerminationDuringShutdown 断开 watch
  ├── 关闭 etcd 连接
  └── 返回
```

### 健康检查三端点的启动顺序

`healthz` 最先就绪（进程活着即返回 200）；`livez` 次之（需 etcd 可达）；`readyz` 最后（需所有路由装完、admission 就绪）。这个阶梯让 kubelet/负载均衡器能精确感知"能接流量"的时机，避免流量打到尚未就绪的实例。

## 5.6 配置完成（Complete）的关键校验

`opts.Complete()` 阶段会做大量交叉校验，典型如：
- `--service-cluster-ip-range` 必须是合法 CIDR 且与 `--service-node-port-range` 不冲突
- `--authorization-mode` 列表中的每个模式都有对应配置
- TLS 证书与私钥匹配
- etcd servers 非空

校验失败在此刻报错，比"跑到一半崩"友好得多——这是"快速失败"原则在启动流程的体现。
