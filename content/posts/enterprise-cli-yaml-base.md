---
title: "别再让后端手写脚本了：我搭了一套可复用的企业 CLI 基座，命令扩展靠 YAML"
description: "这是我给公司内部写的 CLI 基座。不是讲趋势，是讲实战：认证、环境、打包先收敛，业务命令 YAML 化。看完你能照着自己搭一套。"
date: 2026-04-18T12:00:00+08:00
draft: false
categories: ["技术", "工程实践"]
tags: ["CLI", "Go", "工程架构", "开发效能"]
slug: "enterprise-cli-yaml-base"
images: ["/images/enterprise-cli-yaml-base/cover.svg"]
---

内部工具越堆越多，每个新服务都要重新写一遍登录、鉴权、日志、错误处理。

新来的同事拿到一堆脚本，不知道哪个能跑、哪个已经废了。测试环境和生产环境的 token 混着用，一不小心就把测试数据写到生产。

这是我搭这套 CLI 基座的真实原因：**不是为了让工程师更极客，而是为了把内部能力收敛成一个人和 Agent 都能调用的统一入口。**

这个项目我已经用在企业内部，核心代码公开。看完你能照着自己搭一套。

---

## 一、我为什么把企业后台能力做成 CLI

先说判断。

GUI、API、CLI 这三层，很多企业只做了前两层：后台页面给人点，API 给系统调。但中间那层——**对人足够友好、对脚本足够稳定、对 Agent 足够结构化**——一直是空的。

后果就是：
- 业务同学要操作后台，点到手软
- 工程师要写脚本，每个脚本都重新处理一遍认证、日志、错误
- Agent 要调用业务，得直接拼 API，参数错了都不知道怎么报错

CLI 正好卡在中间。

但企业 CLI 不是把几个 API 包成命令就完了。我搭这套基座时，核心就一个原则：**稳定层先代码化，业务层再规格化。**

什么意思？

认证、环境、HTTP 请求、Header 注入、打包发布——这些是稳定层，一次搭好，长期复用。

业务命令——这些是变化层，用 YAML 规格化，扩展时不改核心代码。

这样搭出来的 CLI，命令越多，扩展越快。而不是命令越多，代码越乱。

---

## 二、这套 CLI 基座的骨架

项目结构长这样：

```
.
├── cmd/                        # Cobra 命令入口
│   ├── root.go                 # 根命令，加载 YAML 动态命令
│   ├── auth_login.go           # 登录回调
│   ├── token_refresh.go        # Token 刷新
│   └── oauth_injected.go       # OAuth 凭据注入（构建时生成）
├── internal/
│   ├── auth/                   # 认证层：登录、token、device_id
│   ├── httpx/                  # 请求层：HTTP client、Header 注入、401 refresh
│   ├── runtime/                # 执行层：YAML 解析、Cobra 命令注册、请求执行
│   └── store/                  # 存储层：profile、token 本地持久化
├── specs/
│   ├── groups/                 # 业务命令规格
│   │   └── order/
│   │       ├── group.yaml      # 订单组配置
│   │       └── commands/
│   │           └── list.yaml   # 订单查询命令
│   └── embed.go                # go:embed 把 YAML 编译进二进制
└── scripts/
    └── build-with-oauth.sh     # 多环境、多平台打包脚本
```

**核心分层：**

1. **认证层**（`internal/auth/` + `cmd/auth_login.go`）
   - OAuth 登录回调
   - Token 持久化
   - Device ID 生成

2. **请求层**（`internal/httpx/`）
   - 统一 HTTP client
   - Header 注入（Authorization、X-Device-Id、X-Request-Id 等）
   - 401 自动刷新一次

3. **命令层**（`cmd/` + `internal/runtime/` + `specs/`）
   - 核心命令代码化（auth、version）
   - 业务命令 YAML 化（order、user、payment...）

4. **打包层**（`scripts/build-with-oauth.sh`）
   - 构建时固定环境
   - 多平台矩阵打包
   - OAuth 凭据安全注入

这个骨架的价值：**新业务命令不一定要改 Go 代码，填 YAML 就行。**

![CLI 四层骨架图](/images/enterprise-cli-yaml-base/cli-layers.svg)

---

## 三、核心代码公开

### 1. 根命令如何加载 YAML 动态命令

`cmd/root.go` 是入口。它做两件事：挂核心命令，加载 YAML 动态命令。

```go
// cmd/root.go 核心逻辑
func Execute() {
    root := &cobra.Command{
        Use:   "cli",
        Short: "企业内部统一命令行工具",
    }
    
    // 挂核心命令
    root.AddCommand(authCmd)
    root.AddCommand(versionCmd)
    
    // 加载 YAML 动态命令
    loadYAMLCommands(root)
    
    root.Execute()
}

func loadYAMLCommands(root *cobra.Command) {
    specs, err := runtime.LoadSpecs()  // 从 embed 读取 YAML
    if err != nil {
        return  // YAML 加载失败不影响核心命令
    }
    
    commands := runtime.BuildCommands(specs)  // YAML 转 Cobra 命令
    for _, cmd := range commands {
        root.AddCommand(cmd)
    }
}
```

**设计意图：**

- 核心命令（auth、version）是硬编码，保证稳定性
- 业务命令是动态加载，扩展时不改代码
- YAML 加载失败不影响核心功能，CLI 还能用

---

### 2. 业务命令如何 YAML 化

这是这套方案最值钱的地方。

**group.yaml**（订单组配置）：

```yaml
# specs/groups/order/group.yaml
group: order
short: "订单相关命令"
domains:
  test: https://api.test.example.com
  prod: https://api.example.com
services:
  order-service:
    test: https://api.test.example.com/order
    prod: https://api.example.com/order
```

**list.yaml**（订单查询命令）：

```yaml
# specs/groups/order/commands/list.yaml
group: order
command: list
short: "查询订单列表"
request:
  method: GET
  service: order-service
  path: /list
flags:
  - name: keyword
    type: string
    short: k
    usage: "关键词搜索"
  - name: biz-ids
    type: int_array
    short: i
    usage: "业务 ID 列表"
  - name: limit
    type: int
    default: 20
    usage: "返回数量上限"
```

**执行效果：**

```bash
$ ./cli order list --help
查询订单列表

Usage:
  cli order list [flags]

Flags:
  -k, --keyword string    关键词搜索
  -i, --biz-ids int[]     业务 ID 列表
      --limit int         返回数量上限 (default 20)
```

**设计意图：**

- 业务命令扩展变成"填规格"，不是"写代码"
- 参数自动映射到 Cobra，不用手写 flag 解析
- 请求地址按环境自动回退，不用每个命令都写一遍

---

### 3. 执行器如何把 YAML 转成 HTTP 请求

`internal/runtime/executor.go` 负责执行。

```go
// internal/runtime/executor.go 核心逻辑
func (e *Executor) Execute(cmd *CommandSpec, args map[string]interface{}) (*Response, error) {
    // 1. 构建请求 URL
    baseURL := cmd.ResolveHTTPBaseURL()  // 按环境回退
    url := baseURL + cmd.Request.Path
    
    // 2. 构建请求参数
    var req *http.Request
    if cmd.Request.Method == "GET" {
        query := buildQuery(args)  // 参数转 query string
        req, _ = http.NewRequest("GET", url+"?"+query, nil)
    } else {
        body := buildBody(args)    // 参数转 JSON body
        req, _ = http.NewRequest("POST", url, bytes.NewReader(body))
    }
    
    // 3. 用带 401 refresh 的 client 发送
    client := NewClientWith401(e.store)
    resp, err := client.Do(req)
    
    // 4. 统一错误处理
    if err != nil {
        return nil, fmt.Errorf("请求失败：%w", err)
    }
    
    return parseResponse(resp)
}
```

**设计意图：**

- 所有业务命令共用一个执行器，错误输出统一
- GET/POST 自动区分，参数自动构建
- 401 refresh 逻辑收口在执行层，业务命令不用各自处理

---

### 4. 统一 Header 注入

`internal/httpx/transport.go` 负责注入。

```go
// internal/httpx/transport.go
func (t *Transport) RoundTrip(req *http.Request) (*http.Response, error) {
    // 从 profile 读取 token 和 device_id
    profile, _ := t.store.GetProfile()
    
    // 注入统一 Header
    req.Header.Set("Authorization", "Bearer "+profile.Token)
    req.Header.Set("X-Device-Id", profile.DeviceID)
    req.Header.Set("X-Request-Id", generateRequestID())
    req.Header.Set("X-Source", "cli")
    req.Header.Set("X-Client-Version", getVersion())
    
    // OAuth 链路不带 X-Device-Id，业务链路带
    if t.isOAuthRequest(req) {
        req.Header.Del("X-Device-Id")
    }
    
    return t.base.RoundTrip(req)
}
```

**设计意图：**

- 所有请求自动带认证和追踪信息
- OAuth 请求和业务请求的 Header 策略分开
- 新命令不用重复写 Header 注入逻辑

---

### 5. 401 自动刷新一次

`internal/httpx/client.go` 处理 token 过期。

```go
// internal/httpx/client.go
func NewClientWith401(store Store) *http.Client {
    return &http.Client{
        Transport: &RefreshTransport{
            store: store,
            base:  http.DefaultTransport,
        },
    }
}

type RefreshTransport struct {
    store  Store
    base   http.RoundTripper
    once   bool  // 只刷新一次
}

func (t *RefreshTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    resp, err := t.base.RoundTrip(req)
    if err != nil {
        return nil, err
    }
    
    // 401 且还没刷新过，刷新一次再重试
    if resp.StatusCode == 401 && !t.once {
        t.once = true
        err := refreshToken(t.store)  // 调用刷新接口
        if err != nil {
            return resp, nil  // 刷新失败，返回原 401
        }
        
        // 重试一次
        return t.base.RoundTrip(req)
    }
    
    return resp, nil
}
```

**设计意图：**

- 业务命令不用各自处理 token 过期
- 只刷新一次，避免无限重试
- 刷新失败返回原 401，让上层决定怎么处理

![登录 +Token Refresh 流程图](/images/enterprise-cli-yaml-base/auth-flow.svg)

---

### 6. 多环境打包脚本

`scripts/build-with-oauth.sh` 解决企业 CLI 的真实问题：怎么把不同环境的凭据、安全边界、发布产物管理清楚。

```bash
#!/bin/bash
# scripts/build-with-oauth.sh

ENV=${1:-test}  # 默认 test 环境
SUPPORTED_ENVS=("test" "prod" "uat")

# 1. 校验环境参数
if [[ ! " ${SUPPORTED_ENVS[@]} " =~ " ${ENV} " ]]; then
    echo "错误：不支持的环境 $ENV"
    exit 1
fi

# 2. 生成临时 Go 源码注入 OAuth 凭据
cat > cmd/oauth_injected.go <<EOF
//go:build $ENV
// +build $ENV

package cmd

const (
    OAuthClientID = "$OAUTH_CLIENT_ID_${ENV}"
    OAuthClientSecret = "$OAUTH_CLIENT_SECRET_${ENV}"
)
EOF

# 3. 多平台矩阵打包
PLATFORMS=("darwin-arm64" "darwin-amd64" "linux-amd64" "windows-amd64")
for platform in "${PLATFORMS[@]}"; do
    GOOS=$(echo $platform | cut -d'-' -f1)
    GOARCH=$(echo $platform | cut -d'-' -f2)
    
    CGO_ENABLED=0 GOOS=$GOOS GOARCH=$GOARCH go build \
        -tags $ENV \
        -o dist/cli-${ENV}-${platform} \
        .
done

# 4. 清理临时文件
rm -f cmd/oauth_injected.go
```

**设计意图：**

- 构建时固定环境，运行时不让切
- 产物命名带环境，避免混用
- 用 build tag 注入凭据，不用 ldflags（避免特殊字符导致构建/运行异常）

---

## 四、几个反常识的设计选择

### 1. 构建时固定环境，运行时不让切

很多团队做 CLI，喜欢做一个二进制既能跑 test 又能跑 prod，用 flag 切换。

我选择反过来：**一个二进制只对应一个环境。**

为什么？

- Profile、凭据、token、网关地址，全在构建期固定
- 运行时想跑错环境都难
- 测试环境产物和生产环境产物物理隔离

**企业 CLI 真正要做的是治理，不是灵活。**

---

### 2. 401 只刷新一次，不无限重试

见过太多脚本 token 过期后无限重试，把认证服务打挂。

这套方案里，每个请求只 refresh 一次，retry 一次。刷新失败就返回 401，让上层决定怎么处理。

**自动化的前提是克制。**

---

### 3. OAuth 链路和业务请求的 Header 区分

OAuth 请求不带 `X-Device-Id`，业务请求带。

这个细节是踩坑踩出来的。有些网关会校验设备 ID 的一致性，OAuth 链路带了反而过不去。

**企业 CLI 能上线跑起来，靠的就是这些细节。**

---

### 4. 不用 ldflags 注入 secret

很多教程教人用 `-ldflags -X` 注入 secret。我选择生成临时 Go 源码 + build tag。

原因：
- ldflags 注入的字符串有特殊字符时，运行即崩
- build tag 注入是编译期行为，更安全
- 临时文件构建完就删，不留痕迹

---

## 五、你要自己搭，按这个顺序来

### 最小落地步骤

1. **先固定 1 个二进制只对应 1 个环境**
   - 不要运行时切换
   - 产物命名带环境

2. **先把登录、token、profile、HTTP Header 注入收敛好**
   - 这是稳定层，一次搭对

3. **先做 1 个 group + 1 个 command 的 YAML 试点**
   - 跑通一个真实接口
   - 验证动态注册机制

4. **把 YAML 内嵌进二进制，不要依赖运行时外部配置**
   - 用 go:embed
   - 避免部署时丢配置文件

5. **加上 401 refresh once**
   - 业务命令不用各自处理 token 过期

6. **再做打包脚本，输出多环境、多平台产物**
   - 测试、生产物理隔离

7. **最后才扩业务命令数量**
   - 稳定层先搭好，扩展是水到渠成

---

### 验证方式

```bash
# 1. 新增 YAML 命令后，验证动态注册
go build -o cli .
./cli <group> <command> --help

# 2. 打包 test 环境后验证登录
./scripts/build-with-oauth.sh test
./dist/cli-test-darwin-arm64 auth login

# 3. 手动观察 profile 文件
# 位置：~/.cli/profiles/<profile>.json
# 确认 token/device_id/env/base_url 存在

# 4. 故意触发业务请求 401
# 确认 CLI 只 refresh 一次、retry 一次

# 5. 新增一个 group + command 规格
# 不改核心代码也能扩一个业务命令
```

---

### 常见坑

| 坑 | 表现 | 解法 |
|----|------|------|
| 运行时自由切环境 | profile、凭据、token 混用 | 构建时固定环境 |
| ldflags 注入 secret | 特殊字符导致运行即崩 | 用 build tag + 临时源码 |
| YAML 当配置文件 | 没有统一执行器，越改越乱 | YAML 只是规格，执行器要代码化 |
| OAuth 和业务请求混用 Header | 认证链路被网关拒绝 | 区分 Header 策略 |
| 只扩命令不做打包分发 | 别人没法安装升级 | 打包脚本做多环境多平台矩阵 |
| 只给人用不考虑 Agent | 输出、错误、参数不稳定 | 加上 --json，输出结构化 |

---

## 六、最后

AI 时代，企业真正缺的不是更多 API，而是一层能同时服务工程师、脚本和 Agent 的稳定执行界面。

CLI 正在变成这层界面。

但企业 CLI 不是脚本堆，是能力收口。

**稳定层先代码化，业务层再规格化。**

这样搭出来的 CLI，命令越多，扩展越快。
