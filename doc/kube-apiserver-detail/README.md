# kube-apiserver 技术细节

本目录是对 `doc/kube-apiserver.md` 的**技术细节细化版**。主文档提供全局视图与章节索引，本目录把每一章拆分为独立文档并深度展开：补充字段语义、调用关系、配置含义、设计权衡与代码路径。

> 阅读建议：先读主文档 `../kube-apiserver.md` 建立整体认知，再按需进入下列任一细项深入。

## 文档导航

| # | 文档 | 主题 | 关键词 |
|---|------|------|--------|
| 01 | [架构概览](./01-architecture-overview.md) | 分层架构、核心组件、模块关系 | GenericAPIServer、Handler、Storage、Etcd |
| 02 | [核心数据结构](./02-core-data-structures.md) | 服务器/处理器/请求信息结构体字段 | GenericAPIServer、APIServerHandler、RequestInfo |
| 03 | [请求处理流程](./03-request-handling.md) | Handler Chain 23 个环节与构建顺序 | 责任链、认证、授权、准入 |
| 04 | [核心接口设计](./04-core-interfaces.md) | REST Storage、Admission、Authorizer 接口层次 | Getter、Creater、Watcher、Mutation |
| 05 | [启动流程](./05-startup-flow.md) | 从 main 到安全服务启动的完整链路 | Cobra、CreateServerChain、PrepareRun |
| 06 | [认证、授权、准入控制](./06-authn-authz-admission.md) | 三道安全闸门的实现与策略 | 9 种认证、6 种授权、两阶段准入 |
| 07 | [API Group 安装](./07-api-group-installation.md) | 资源组注册、REST Storage Provider | InstallAPIGroups、APIGroupInfo |
| 08 | [Watch 机制](./08-watch-mechanism.md) | 实时事件通知的原理与接口 | Interface、Event、Bookmark |
| 09 | [关键配置选项](./09-config-options.md) | 运行选项结构与常用启动参数 | ServerRunOptions、Etcd、SecureServing |
| 10 | [代码组织结构](./10-code-organization.md) | 目录布局与设计模式映射 | cmd、pkg、staging |
| 11 | [扩展点](./11-extension-points.md) | 自定义认证/授权/准入插件 | Interface、Admit、Authorize |
| 12 | [总结](./12-summary.md) | 设计理念与核心机制回顾 | 声明式、Watch、Finalizers |

## 与主文档的关系

```
doc/kube-apiserver.md              ← 概览，约 30 KB，12 章压缩视图
  └── doc/kube-apiserver-detail/   ← 细化，每章独立深展
        ├── README.md (本文件)
        ├── 01-architecture-overview.md
        ├── ...
        └── 12-summary.md
```

## 图例约定

- `文件: 行号` 格式的路径引用指向 kubernetes 源码相对位置（如 `staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go`）。
- ASCII 图用于架构与流程；表格用于枚举与对照；代码块用于结构体与接口签名。
- 细节标注：`※` 表示易被忽略但重要的实现点；`⚠` 表示常见误区。
