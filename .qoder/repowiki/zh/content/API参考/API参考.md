# API参考

<cite>
**本文档中引用的文件**  
- [transport.go](file://transport.go)
- [server/server.go](file://server/server.go)
- [handler.go](file://handler.go)
- [signer.go](file://signer.go)
- [types.go](file://types.go)
</cite>

## 目录
1. [X402Transport API](#x402transport-api)  
2. [X402Server API](#x402server-api)  
3. [签名器 API](#签名器-api)  
4. [核心数据结构](#核心数据结构)

## X402Transport API

`X402Transport` 是实现 `transport.Interface` 接口的客户端传输层，支持 x402 支付协议。它基于 StreamableHTTP 实现，并增加了支付处理逻辑。

### New 函数

创建一个新的 `X402Transport` 实例。

```go
func New(config Config) (*X402Transport, error)
```

**参数说明：**
- `config`：配置对象，包含以下字段：
  - `ServerURL` (string)：目标服务器的 URL
  - `Signer` (PaymentSigner)：用于签署支付的签名器
  - `PaymentCallback` (func(amount *big.Int, resource string) bool)：支付前的回调函数，用于决定是否执行支付
  - `HTTPClient` (*http.Client)：可选的自定义 HTTP 客户端
  - `OnPaymentAttempt` (func(PaymentEvent))：支付尝试时的回调
  - `OnPaymentSuccess` (func(PaymentEvent))：支付成功时的回调
  - `OnPaymentFailure` (func(PaymentEvent, error))：支付失败时的回调

**返回值：**
- `*X402Transport`：新创建的传输实例
- `error`：如果配置无效（如 URL 格式错误或签名器为 nil），返回错误

**使用示例：**
```go
transport, err := x402.New(x402.Config{
    ServerURL: "https://example.com",
    Signer:    signer,
    PaymentCallback: func(amount *big.Int, resource string) bool {
        return true // 自动批准支付
    },
})
if err != nil {
    log.Fatal(err)
}
```

**Section sources**
- [transport.go](file://transport.go#L77-L114)

## X402Server API

`X402Server` 是一个包装了 MCP 服务器的结构体，增加了 x402 支付支持。

### NewX402Server 函数

创建一个新的 x402 启用的 MCP 服务器。

```go
func NewX402Server(name, version string, config *Config) *X402Server
```

**参数说明：**
- `name` (string)：服务器名称
- `version` (string)：服务器版本
- `config` (*Config)：服务器配置，包含支付工具等设置

**返回值：**
- `*X402Server`：新创建的服务器实例

**Section sources**
- [server/server.go](file://server/server.go#L17-L27)

### AddPayableTool 方法

向服务器添加一个需要支付的工具。

```go
func (s *X402Server) AddPayableTool(
    tool mcp.Tool,
    handler server.ToolHandlerFunc,
    requirements ...PaymentRequirement,
)
```

**参数说明：**
- `tool` (mcp.Tool)：要注册的工具定义
- `handler` (server.ToolHandlerFunc)：处理该工具调用的函数
- `requirements` (...PaymentRequirement)：一个或多个支付要求，定义了支付方式、金额、网络等

**行为说明：**
- 该方法会将工具添加到基础 MCP 服务器中
- 同时将支付要求注册到服务器配置中，供客户端在收到 402 响应时使用
- 如果未提供任何支付要求，会触发 panic

**使用示例：**
```go
server.AddPayableTool(
    myTool,
    myHandler,
    x402.PaymentRequirement{
        Scheme:            "exact",
        Network:           "base-sepolia",
        MaxAmountRequired: "1000000", // 1 USDC
        Asset:             "0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2", // USDC on Base
        PayTo:             "0xRecipientAddress",
        Resource:          "/tool/my-tool",
        Description:       "My paid tool",
        MaxTimeoutSeconds: 300,
    },
)
```

**Section sources**
- [server/server.go](file://server/server.go#L35-L53)

## 签名器 API

签名器负责签署 x402 支付授权。系统提供了多种签名器实现。

### PaymentSigner 接口

所有签名器必须实现的接口。

```go
type PaymentSigner interface {
    SignPayment(ctx context.Context, req PaymentRequirement) (*PaymentPayload, error)
    GetAddress() string
    SupportsNetwork(network string) bool
    HasAsset(asset, network string) bool
    GetPaymentOption(network, asset string) *ClientPaymentOption
}
```

**方法说明：**
- `SignPayment`：根据支付要求创建并签署支付授权
- `GetAddress`：返回签名器的地址
- `SupportsNetwork`：检查是否支持指定网络
- `HasAsset`：检查是否在指定网络上拥有指定资产
- `GetPaymentOption`：获取匹配的客户端支付选项

**Section sources**
- [signer.go](file://signer.go#L23-L38)

### NewPrivateKeySigner 函数

使用十六进制私钥创建签名器。

```go
func NewPrivateKeySigner(privateKeyHex string, options ...ClientPaymentOption) (*PrivateKeySigner, error)
```

**参数说明：**
- `privateKeyHex` (string)：十六进制格式的私钥（可选 0x 前缀）
- `options` (...ClientPaymentOption)：一个或多个客户端支付选项，定义了支持的网络、资产和优先级

**返回值：**
- `*PrivateKeySigner`：新创建的签名器
- `error`：如果私钥无效或未配置支付选项，返回错误

**Section sources**
- [signer.go](file://signer.go#L48-L78)

### NewMnemonicSigner 函数

使用助记词创建签名器。

```go
func NewMnemonicSigner(mnemonic string, derivationPath string, options ...ClientPaymentOption) (*MnemonicSigner, error)
```

**参数说明：**
- `mnemonic` (string)：BIP-39 标准的助记词
- `derivationPath` (string)：密钥派生路径（如 "m/44'/60'/0'/0/0"），可为空，使用默认路径
- `options` (...ClientPaymentOption)：客户端支付选项

**返回值：**
- `*MnemonicSigner`：新创建的签名器
- `error`：如果助记词无效或派生失败，返回错误

**Section sources**
- [signer.go](file://signer.go#L255-L297)

### NewKeystoreSigner 函数

使用加密的 keystore 文件创建签名器。

```go
func NewKeystoreSigner(keystoreJSON []byte, password string, options ...ClientPaymentOption) (*KeystoreSigner, error)
```

**参数说明：**
- `keystoreJSON` ([]byte)：keystore JSON 文件内容
- `password` (string)：解密密码
- `options` (...ClientPaymentOption)：客户端支付选项

**返回值：**
- `*KeystoreSigner`：新创建的签名器
- `error`：如果密码错误或 keystore 无效，返回错误

**Section sources**
- [signer.go](file://signer.go#L305-L330)

## 核心数据结构

基于 `types.go` 定义的关键数据结构。

### PaymentRequirement

表示服务器要求的支付方式。

**字段：**
- `Scheme` (string)：支付方案（如 "exact"）
- `Network` (string)：区块链网络（如 "base-sepolia"）
- `MaxAmountRequired` (string)：最大需支付金额（字符串形式的大整数）
- `Asset` (string)：资产地址
- `PayTo` (string)：收款地址
- `Resource` (string)：资源标识符
- `Description` (string)：描述
- `MimeType` (string, 可选)：MIME 类型
- `OutputSchema` (interface{}, 可选)：输出模式
- `MaxTimeoutSeconds` (int)：最大超时时间（秒）
- `Extra` (map[string]string, 可选)：额外参数

**Section sources**
- [types.go](file://types.go#L8-L30)

### PaymentRequirementsResponse

402 响应体结构。

**字段：**
- `X402Version` (int)：x402 协议版本
- `Error` (string)：错误消息
- `Accepts` ([]PaymentRequirement)：可接受的支付方式列表

**Section sources**
- [types.go](file://types.go#L32-L37)

### PaymentPayload

客户端发送的已签名支付载荷。

**字段：**
- `X402Version` (int)：协议版本
- `Scheme` (string)：支付方案
- `Network` (string)：网络
- `Payload` (PaymentPayloadData)：包含签名和授权信息

**Section sources**
- [types.go](file://types.go#L39-L44)

### PaymentPayloadData

支付载荷数据。

**字段：**
- `Signature` (string)：EIP-712 签名
- `Authorization` (PaymentAuthorization)：授权信息

**Section sources**
- [types.go](file://types.go#L46-L51)

### PaymentAuthorization

EIP-3009 风格的授权数据。

**字段：**
- `From` (string)：付款人地址
- `To` (string)：收款人地址
- `Value` (string)：金额
- `ValidAfter` (string)：有效开始时间（Unix 时间戳）
- `ValidBefore` (string)：有效结束时间（Unix 时间戳）
- `Nonce` (string)：随机数

**Section sources**
- [types.go](file://types.go#L53-L62)

### SettlementResponse

支付结算响应。

**字段：**
- `Success` (bool)：是否成功
- `Transaction` (string)：交易哈希
- `Network` (string)：网络
- `Payer` (string)：付款人地址
- `ErrorReason` (string, 可选)：错误原因

**Section sources**
- [types.go](file://types.go#L64-L71)

### ClientPaymentOption

客户端支付选项（内部使用）。

**字段：**
- 继承 `PaymentRequirement` 的所有字段
- `Priority` (int)：优先级（数值越低优先级越高）
- `MaxAmount` (string)：客户端愿意支付的最大金额
- `MinBalance` (string)：最低余额阈值
- `ChainID` (*big.Int)：链 ID（用于签名）

**Section sources**
- [types.go](file://types.go#L73-L82)