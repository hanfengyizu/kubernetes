# kubeadm 代码分析

## 概述

kubeadm 是 Kubernetes 官方提供的集群初始化和管理工具，用于快速搭建和管理 Kubernetes 集群。它的主要功能包括：

- 集群初始化 (`kubeadm init`)
- 节点加入集群 (`kubeadm join`)
- 升级集群 (`kubeadm upgrade`)
- 管理控制平面节点

## 代码结构

```
cmd/kubeadm/
├── app/
│   ├── apis/kubeadm/          # API 定义
│   ├── constants/             # 常量定义
│   ├── cmd/                   # 子命令实现
│   ├── features/              # 特性门控
│   ├── phases/                # 各阶段实现
│   ├── preflight/             # 预检查
│   └── util/                  # 工具函数
└── cmd/                       # 主入口
```

## 核心流程

### 1. 初始化流程 (`kubeadm init`)

初始化流程主要在 `cmd/kubeadm/app/cmd/init.go` 中实现，包含以下阶段：

1. **预检查 (preflight)**: 检查系统环境和依赖
2. **生成证书 (certs)**: 为集群组件生成所需证书
3. **生成 kubeconfig (kubeconfig)**: 生成配置文件
4. **控制平面初始化 (control-plane)**: 初始化控制平面组件
5. **上传配置 (upload-config)**: 将配置上传到 ConfigMap
6. **标注控制平面节点 (mark-control-plane)**: 为控制平面节点添加标签和污点
7. **引导 token (bootstrap-token)**: 创建引导 token 用于节点加入
8. **RBAC 配置 (rbac)**: 配置 RBAC 权限
9. **DNS 安装 (addon/dns)**: 安装 CoreDNS

关键文件：
- `cmd/kubeadm/app/phases/init.go` - 初始化各阶段的实现

### 2. 节点加入流程 (`kubeadm join`)

节点加入流程主要在 `cmd/kubeadm/app/cmd/join.go` 中实现：

1. **预检查 (preflight)**: 检查节点环境
2. **发现集群 (discovery)**: 发现并连接到集群的 API Server
3. **下载证书 (certs)**: 从集群下载集群证书
4. **生成 kubeconfig (kubeconfig)**: 为当前节点生成 kubeconfig
5. **标注控制平面节点 (mark-control-plane)**: 如果是控制平面节点
6. **kubelet 配置 (kubelet-start)**: 配置并启动 kubelet

关键文件：
- `cmd/kubeadm/app/phases/join/discovery/discovery.go` - 集群发现

### 3. kubelet 配置实现

kubelet 配置相关代码在 `cmd/kubeadm/app/phases/kubelet/` 目录：

#### kubelet.go

```go
// TryStartKubelet - 尝试启动 kubelet 服务
func TryStartKubelet()

// TryStopKubelet - 暂时停止 kubelet 服务
func TryStopKubelet()

// TryRestartKubelet - 重启 kubelet 服务
func TryRestartKubelet()
```

这些函数通过 `initsystem` 包与系统的 init 系统（如 systemd）交互来管理 kubelet 服务。

### 4. kubeconfig 生成与处理

kubeconfig 相关实现在 `cmd/kubeadm/app/util/kubeconfig/kubeconfig.go`：

#### 核心函数

```go
// CreateBasic - 创建基础 kubeconfig 对象
func CreateBasic(serverURL, clusterName, userName string, caCert []byte) *clientcmdapi.Config

// CreateWithCerts - 使用客户端证书创建 kubeconfig
func CreateWithCerts(serverURL, clusterName, userName string, caCert, clientKey, clientCert []byte) *clientcmdapi.Config

// CreateWithToken - 使用 token 创建 kubeconfig
func CreateWithToken(serverURL, clusterName, userName string, caCert []byte, token string) *clientcmdapi.Config

// ClientSetFromFile - 从文件加载 kubeconfig 并创建 clientset
func ClientSetFromFile(path string) (clientset.Interface, error)

// WriteToDisk - 将 kubeconfig 写入磁盘
func WriteToDisk(filename string, kubeconfig *clientcmdapi.Config) error

// HasAuthenticationCredentials - 检查是否有有效的认证凭据
func HasAuthenticationCredentials(config *clientcmdapi.Config) bool

// EnsureAuthenticationInfoAreEmbedded - 确保认证信息嵌入到 kubeconfig 中
func EnsureAuthenticationInfoAreEmbedded(config *clientcmdapi.Config) error

// EnsureCertificateAuthorityIsEmbedded - 确保 CA 证书嵌入到 kubeconfig 中
func EnsureCertificateAuthorityIsEmbedded(cluster *clientcmdapi.Cluster) error
```

### 5. 配置上传实现

配置上传到 ConfigMap 的实现在 `cmd/kubeadm/app/phases/uploadconfig/uploadconfig.go`：

```go
// UploadConfiguration 将 InitConfiguration 保存到 ConfigMap
// ConfigMap 名称为 "kubeadm-config"，位于 "kube-system" namespace
func UploadConfiguration(cfg *kubeadmapi.InitConfiguration, client clientset.Interface) error
```

该函数会：
1. 创建名为 `kubeadm:nodes-kubeadm-config` 的 Role，赋予读取 ConfigMap 的权限
2. 创建 RoleBinding，将权限绑定到 `system:bootstrappers:kubeadm:default-node-token` 和 `system:nodes` 组

### 6. 控制平面节点标注

控制平面节点标注实现在 `cmd/kubeadm/app/phases/markcontrolplane/markcontrolplane.go`：

```go
// MarkControlPlane 为控制平面节点添加标签和污点
func MarkControlPlane(client clientset.Interface, controlPlaneName string, taints []v1.Taint) error
```

添加的标签：
- `node-role.kubernetes.io/control-plane=""` - 控制平面角色标签
- `node.kubernetes.io/exclude-from-external-load-balancers=""` - 排除在外部负载均衡器之外

添加的污点：
- `node-role.kubernetes.io/control-plane:NoSchedule` - 默认污点

### 7. CRI Socket 标注

CRI Socket 标注实现在 `cmd/kubeadm/app/phases/patchnode/patchnode.go`：

```go
// AnnotateCRISocket 为节点添加 CRI socket 注解
func AnnotateCRISocket(client clientset.Interface, nodeName string, criSocket string) error

// RemoveCRISocketAnnotation 移除节点的 CRI socket 注解
func RemoveCRISocketAnnotation(client clientset.Interface, nodeName string) error
```

## 配置文件

### InitConfiguration

控制 kubeadm init 行为的配置：

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
bootstrapTokens:
  - token: abcdef.0123456789abcdef
    ttl: 24h
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs: {}
```

### ClusterConfiguration

集群范围配置：

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: 1.31.0
controlPlaneEndpoint: ""
clusterName: kubernetes
networking:
  serviceSubnet: 10.96.0.0/12
  podSubnet: ""
  dnsDomain: cluster.local
```

### JoinConfiguration

控制 kubeadm join 行为的配置：

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: abcdef.0123456789abcdef
    apiServerEndpoint: 192.168.1.100:6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
```

## 特性门控

kubeadm 支持的特性门控定义在 `cmd/kubeadm/app/features/features.go` 中：

- `IPv6DualStack` - IPv6 双栈支持
- `RootlessControlPlane` - 无根控制平面
- `PublicKeysECDSA` - 使用 ECDSA 公钥
- 等

## 常量定义

重要常量在 `cmd/kubeadm/app/constants/constants.go` 中定义：

- `KubeadmConfigConfigMap` - "kubeadm-config"
- `ClusterConfigurationConfigMapKey` - "ClusterConfiguration"
- `LabelNodeRoleControlPlane` - "node-role.kubernetes.io/control-plane"
- `NodeBootstrapTokenAuthGroup` - "system:bootstrappers:kubeadm:default-node-token"
- `NodesGroup` - "system:nodes"