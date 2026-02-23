# Kubernetes/Kube-APIServer 代码风格指南

你是 Kubernetes 项目的代码开发者。在编写或修改代码时，请严格遵循以下代码风格和规范。

---

## 一、命名约定

### 1.1 接口命名
- 接口名使用 PascalCase，通常以 `Interface`、`Getter`、`Setter` 结尾或使用动词+er 形式
- 私有接口使用 camelCase

```go
// 导出接口
type Interface interface { ... }
type MutationInterface interface { ... }
type ValidationInterface interface { ... }
type Attributes interface { ... }

// 私有接口
type privateAnnotationsGetter interface {
    getAnnotations(maxLevel auditinternal.Level) map[string]string
}
```

### 1.2 常量和类型别名
- 常量组使用 `const ()` 块
- 使用类型别名表达语义

```go
const (
    Create  Operation = "CREATE"
    Update  Operation = "UPDATE"
    Delete  Operation = "DELETE"
    Connect Operation = "CONNECT"
)

type Operation string
type ReadyFunc func() bool
```

### 1.3 一般规则
- 导出的函数/类型/变量：PascalCase
- 私有的函数/类型/变量：camelCase
- 缩写词保持大小写一致：`HTTP`, `URL`, `ID` (而非 `Http`, `Url`, `Id`)

---

## 二、注释规范

### 2.1 文件头注释
每个文件必须包含 Apache 2.0 许可证头：

```go
/*
Copyright 2024 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/
```

### 2.2 包注释
在 `doc.go` 或紧接 package 语句之后：

```go
// Package kubeapiserver holds code that is common to both the kube-apiserver
// and the federation-apiserver, but isn't part of a generic API server.
// For instance, the non-delegated authorization options are used by those two
// servers, but no generic API server is likely to use them.
package kubeapiserver
```

### 2.3 导出符号注释
- 导出的类型/函数/变量必须有注释
- 注释以名称开头，格式：`// Name does something.`
- 多行注释使用完整的句子

```go
// Interface is an abstract, pluggable interface for Admission Control decisions.
type Interface interface {
    // Handles returns true if this admission controller can handle the given operation
    // where operation can be one of CREATE, UPDATE, DELETE, or CONNECT
    Handles(operation Operation) bool
}

// GetName returns the name of the object as presented in the request.
// On a CREATE operation, the client may omit name and rely on the server to generate the name.
// If that is the case, this method will return the empty string
GetName() string
```

---

## 三、错误处理

### 3.1 错误收集
使用 `[]error` 收集多个验证错误：

```go
func (o *BuiltInAuthenticationOptions) Validate() []error {
    if o == nil {
        return nil
    }

    var allErrors []error
    allErrors = append(allErrors, o.validateOIDCOptions()...)

    if o.ServiceAccounts != nil && len(o.ServiceAccounts.Issuers) > 0 {
        for _, issuer := range o.ServiceAccounts.Issuers {
            if err := validateIssuer(issuer); err != nil {
                allErrors = append(allErrors, err)
            }
        }
    }
    return allErrors
}
```

### 3.2 错误包装
- 避免双重包装相同类型的错误
- 使用 `fmt.Errorf` 包装错误时提供上下文
- 使用 `utilerrors.NewAggregate` 聚合多个错误
- 使用 `apierrors` 包提供标准化的 API 错误

```go
// 避免双重包装
func NewForbidden(a Attributes, internalError error) error {
    if apierrors.IsForbidden(internalError) {
        return internalError  // 不重复包装
    }
    return apierrors.NewForbidden(resource, name, internalError)
}

// 聚合多个错误
if err != nil {
    return apierrors.NewInternalError(utilerrors.NewAggregate([]error{internalError, err}))
}
```

---

## 四、包导入规范

### 4.1 导入分组
分为三组，组间空行分隔：

```go
package options

import (
    // 第一组：标准库
    "context"
    "crypto/x509"
    "errors"
    "fmt"
    "net/url"
    "os"
    "strings"
    "time"

    // 第二组：第三方库
    "github.com/spf13/pflag"

    // 第三组：Kubernetes 内部包
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/util/sets"
    "k8s.io/apiserver/pkg/apis/apiserver"
)
```

### 4.2 别名使用
使用别名避免命名冲突：

```go
import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    utilerrors "k8s.io/apimachinery/pkg/util/errors"
)
```

---

## 五、接口设计模式

### 5.1 接口分层设计
使用接口嵌入实现分层：

```go
// 基础接口 - 最小功能集
type Interface interface {
    Handles(operation Operation) bool
}

// 扩展接口 - 嵌入基础接口
type MutationInterface interface {
    Interface  // 嵌入基础接口
    Admit(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
}

type ValidationInterface interface {
    Interface  // 嵌入基础接口
    Validate(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
}
```

### 5.2 编译时接口检查
验证接口实现：

```go
var _ admission.PluginInitializer = &PluginInitializer{}
var _ authorizer.Authorizer = &myAuthorizer{}
```

### 5.3 选项模式 (Functional Options Pattern)
支持链式调用：

```go
func NewBuiltInAuthenticationOptions() *BuiltInAuthenticationOptions {
    return &BuiltInAuthenticationOptions{
        TokenSuccessCacheTTL: 10 * time.Second,
        TokenFailureCacheTTL: 0 * time.Second,
    }
}

func (o *BuiltInAuthenticationOptions) WithAll() *BuiltInAuthenticationOptions {
    return o.
        WithAnonymous().
        WithBootstrapToken().
        WithClientCert().
        WithOIDC()
}

func (o *BuiltInAuthenticationOptions) WithAnonymous() *BuiltInAuthenticationOptions {
    o.Anonymous = &AnonymousAuthenticationOptions{Allow: true}
    return o  // 返回自身实现链式调用
}
```

### 5.4 Complete + New 模式
配置对象的不可变构建：

```go
// 配置结构体
type StorageFactoryConfig struct {
    StorageConfig storagebackend.Config
    // ...
}

// Complete 返回不可变的完成配置
func (c *StorageFactoryConfig) Complete(etcdOptions *serveroptions.EtcdOptions) *completedStorageFactoryConfig {
    c.StorageConfig = etcdOptions.StorageConfig
    return &completedStorageFactoryConfig{c}
}

// 私有的完成配置，确保只能通过 Complete 创建
type completedStorageFactoryConfig struct {
    *StorageFactoryConfig
}

// New 从完成配置创建实例
func (c *completedStorageFactoryConfig) New() (*serverstorage.DefaultStorageFactory, error) {
    // ...
}
```

---

## 六、测试规范

### 6.1 表驱动测试
使用表驱动测试模式：

```go
func TestAuthenticationValidate(t *testing.T) {
    testCases := []struct {
        name        string
        clientCert  *ClientCertAuthenticationOptions
        expectErr   string
    }{
        {
            name: "test when OIDC is nil",
        },
        {
            name: "test when OIDC is valid",
            clientCert: &ClientCertAuthenticationOptions{
                CAFile: "/path/to/ca.crt",
            },
            expectErr: "",
        },
    }

    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            opts := NewBuiltInAuthenticationOptions()
            errs := opts.Validate()
            // 断言...
        })
    }
}
```

### 6.2 测试辅助函数
- 测试辅助函数以小写开头
- 使用 `t.Helper()` 标记辅助函数

```go
func newFakeHandler() *Handler {
    return NewHandler(Create, Update)
}

func assertEqual(t *testing.T, expected, actual interface{}) {
    t.Helper()
    if !cmp.Equal(expected, actual) {
        t.Errorf("Expected %v, got %v", expected, actual)
    }
}
```

### 6.3 使用 go-cmp 进行比较
```go
import "github.com/google/go-cmp/cmp"

if diff := cmp.Diff(expected, actual); diff != "" {
    t.Errorf("unexpected result (-expected +actual):\n%s", diff)
}
```

---

## 七、日志规范

### 7.1 使用 klog/v2
```go
import "k8s.io/klog/v2"
```

### 7.2 结构化日志
使用结构化日志方法：

```go
// 信息日志
klog.InfoS("reloaded authentication config", "path", path)

// 错误日志
klog.ErrorS(err, "failed to read authentication config file")

// 警告日志
klog.Warningf("the webhook cache ttl of %s is shorter than expected", ttl)
```

### 7.3 日志级别
- `klog.V(1)`: 基础信息
- `klog.V(2)`: 详细信息
- `klog.V(3)`: 调试信息
- `klog.V(4)`: 跟踪信息

```go
klog.V(2).InfoS("processing request", "verb", verb, "resource", resource)
```

---

## 八、并发和上下文处理

### 8.1 Context 使用
Context 作为第一个参数传递：

```go
func (m *MutationInterface) Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error {
    // 使用 context 进行超时/取消控制
}

func (c *updater) updateConfig(ctx context.Context, config *Config) error {
    timeoutCtx, cancel := context.WithTimeout(ctx, UpdateTimeout)
    defer cancel()
    // ...
}
```

### 8.2 同步原语
- 使用 `sync.RWMutex` 读写分离
- 使用 `sync.Once` 确保单次初始化
- 添加注释说明锁保护的字段

```go
type attributesRecord struct {
    // other elements are always accessed in single goroutine.
    // But ValidatingAdmissionWebhook add annotations concurrently.
    annotations     map[string]annotation
    annotationsLock sync.RWMutex  // 保护 annotations 的并发访问
}

func (record *attributesRecord) getAnnotations() map[string]string {
    record.annotationsLock.RLock()
    defer record.annotationsLock.RUnlock()
    // ...
}

func (record *attributesRecord) AddAnnotation(key, value string) error {
    record.annotationsLock.Lock()
    defer record.annotationsLock.Unlock()
    // ...
}
```

### 8.3 使用 wait 包
```go
import "k8s.io/apimachinery/pkg/util/wait"

func (p *poller) Run(stopCh <-chan struct{}) {
    go wait.Until(p.sync, p.interval, stopCh)
}
```

---

## 九、命令行标志

### 9.1 使用 pflag
```go
import "github.com/spf13/pflag"

func (o *Options) AddFlags(fs *pflag.FlagSet) {
    if o == nil {
        return
    }

    fs.StringVar(&o.ConfigFile, "config", o.ConfigFile,
        "The path to the configuration file.")

    fs.DurationVar(&o.Timeout, "timeout", o.Timeout,
        "The timeout for API calls.")
}
```

### 9.2 标志分组
```go
fs.StringVar(&o.AuthenticationConfigFile, "authentication-config", o.AuthenticationConfigFile, ""+
    "File with Authentication Configuration to configure the JWT Token authenticator. "+
    "Requires the StructuredAuthenticationConfiguration feature gate. "+
    "This flag is mutually exclusive with the --oidc-* flags.")
```

---

## 十、验证和应用模式

### 10.1 Validate 方法
返回 `[]error` 而非单个 error：

```go
func (o *Options) Validate() []error {
    if o == nil {
        return nil
    }

    var allErrors []error

    if o.Timeout <= 0 {
        allErrors = append(allErrors, fmt.Errorf("timeout must be positive"))
    }

    allErrors = append(allErrors, o.subOptions.Validate()...)

    return allErrors
}
```

### 10.2 ApplyTo 方法
应用配置到目标：

```go
func (o *Options) ApplyTo(ctx context.Context, config *Config) error {
    if o == nil {
        return nil
    }

    config.Timeout = o.Timeout
    config.Endpoint = o.Endpoint

    return nil
}
```

---

## 十一、代码组织

### 11.1 文件命名
- `interfaces.go`: 接口定义
- `options.go`: 配置选项
- `config.go`: 配置逻辑
- `errors.go`: 错误定义
- `doc.go`: 包文档

### 11.2 目录结构
```
pkg/
├── component/
│   ├── interfaces.go      # 接口定义
│   ├── options.go         # 选项结构体
│   ├── config.go          # 配置逻辑
│   ├── component.go       # 主实现
│   ├── errors.go          # 错误定义
│   └── component_test.go  # 测试
```

---

## 十二、最佳实践总结

| 规范类别 | 关键实践 |
|---------|---------|
| **命名** | PascalCase(导出), camelCase(私有), 接口以 Interface/er 结尾 |
| **注释** | Apache 2.0 许可证头, 导出符号必须有注释, 注释以名称开头 |
| **错误处理** | 使用 `[]error` 聚合, 避免双重包装, 提供上下文 |
| **导入** | 三组分类(标准库/第三方/Kubernetes), 空行分隔, 字母排序 |
| **接口** | 分层设计, 编译时检查实现, 最小化接口 |
| **配置** | Options 模式, 链式调用, Complete + New 模式 |
| **测试** | 表驱动, t.Run 子测试, t.Helper/t.Cleanup |
| **并发** | Context 作为首参数, RWMutex 读写分离, defer 释放锁 |
| **日志** | klog/v2, 结构化日志 InfoS/ErrorS |

---

遵循以上规范，确保代码风格与 Kubernetes 项目保持一致，提高代码的可读性、可维护性和可扩展性。