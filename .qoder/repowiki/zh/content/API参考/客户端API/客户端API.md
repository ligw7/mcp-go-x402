# 客户端API

<cite>
**本文档引用的文件**
- [transport.go](file://transport.go)
- [signer.go](file://signer.go)
- [types.go](file://types.go)
- [handler.go](file://handler.go)
- [client_options.go](file://client_options.go)
- [errors.go](file://errors.go)
</cite>

## 目录
1. [简介](#简介)
2. [X402Transport结构体](#x402transport结构体)
3. [New函数详解](#new函数详解)
4. [PaymentHandler与PaymentSigner](#paymenthandler与paymentsigner)
5. [关键数据结构](#关键数据结构)
6. [使用示例](#使用示例)
7. [常见问题与解决方案](#常见问题与解决方案)

## 简介

本文档详细介绍了mcp-go-x402客户端API的核心组件，重点是`transport.go`中的`X402Transport`结构体及其公共方法。该客户端实现了x402支付协议，能够自动处理HTTP 402（支付要求）响应。当服务器要求支付时，客户端会使用配置的`PaymentSigner`自动创建并签名支付，然后重试请求。

`X402Transport`包装了一个基础的HTTP传输，实现了`transport.Interface`，使其可以无缝集成到现有的MCP（Model Context Protocol）客户端架构中。其核心功能是拦截402错误，通过`PaymentHandler`处理支付逻辑，并将支付信息以标准化的方式（通过`X-PAYMENT`头或JSON-RPC的`_meta`字段）发送回服务器。

**Section sources**
- [transport.go](file://transport.go#L33-L63)
- [handler.go](file://handler.go#L10-L13)

## X402Transport结构体

`X402Transport`是客户端的核心结构体，负责管理与服务器的通信、会话状态和支付流程。

### 字段说明

- **serverURL**: 服务器的URL，所有请求都将发送到此地址。
- **httpClient**: 用于发送HTTP请求的客户端，支持自定义超时和配置。
- **handler**: `*PaymentHandler`，负责处理支付相关的逻辑，包括选择支付方式和创建签名支付。
- **sessionID**: `atomic.Value`，存储从服务器获取的会话ID，用于在请求中标识会话。
- **protocolVersion**: `atomic.Value`，存储协商的协议版本。
- **initialized**: `chan struct{}`，一个信号通道，当会话初始化完成后关闭，用于同步。
- **notificationHandler**: `func(mcp.JSONRPCNotification)`，用于处理从服务器收到的通知。
- **requestHandler**: `transport.RequestHandler`，用于处理服务器可能发起的反向请求（如采样请求）。
- **onPaymentAttempt, onPaymentSuccess, onPaymentFailure**: 事件回调函数，分别在支付尝试、成功和失败时触发。
- **closed**: `chan struct{}`，表示传输是否已关闭。
- **paymentRecorder**: `*PaymentRecorder`，用于测试，记录所有支付事件。

### 公共方法

`X402Transport`实现了`transport.Interface`接口，提供了以下方法：

- **Start(ctx context.Context) error**: 启动传输。此实现中，由于使用HTTP短连接，该方法不执行任何操作，直接返回nil。
- **Close() error**: 关闭传输。它会关闭内部通道，并尝试向服务器发送一个DELETE请求以优雅地关闭会话。
- **SetProtocolVersion(version string)**: 设置协议版本，该版本将包含在后续请求的头信息中。
- **SendRequest(ctx context.Context, request transport.JSONRPCRequest) (*transport.JSONRPCResponse, error)**: 发送一个JSON-RPC请求。这是核心方法，它会：
  1.  首先尝试发送请求。
  2.  如果收到402错误，它会调用`handlePaymentRequired`来处理支付。
  3.  处理服务器响应，包括JSON、SSE（Server-Sent Events）流。
- **SendNotification(ctx context.Context, notification mcp.JSONRPCNotification) error**: 发送一个JSON-RPC通知。
- **SetNotificationHandler(handler func(mcp.JSONRPCNotification))**: 设置通知处理器。
- **SetRequestHandler(handler transport.RequestHandler)**: 设置请求处理器，用于处理来自服务器的入站请求。
- **GetSessionId() string**: 获取当前的会话ID。

**Section sources**
- [transport.go](file://transport.go#L33-L63)
- [transport.go](file://transport.go#L117-L120)
- [transport.go](file://transport.go#L123-L164)
- [transport.go](file://transport.go#L167-L169)
- [transport.go](file://transport.go#L175-L208)
- [transport.go](file://transport.go#L698-L723)
- [transport.go](file://transport.go#L726-L730)
- [transport.go](file://transport.go#L733-L737)
- [transport.go](file://transport.go#L740-L747)

## New函数详解

`New(config Config) (*X402Transport, error)` 是创建 `X402Transport` 实例的构造函数。

### Config结构体参数

`Config` 结构体定义了创建 `X402Transport` 所需的所有配置。

- **ServerURL (string)**: 必需。MCP服务器的URL。
- **Signer (PaymentSigner)**: 必需。一个实现了 `PaymentSigner` 接口的实例，用于对支付进行签名。这是支付功能的核心。
- **PaymentCallback (func(amount *big.Int, resource string) bool)**: 可选。一个回调函数，允许客户端在实际签名前决定是否支付。函数接收支付金额和资源描述，返回true表示同意支付，false表示拒绝。如果未提供，客户端将默认批准所有支付。
- **HTTPClient (*http.Client)**: 可选。自定义的HTTP客户端。如果为nil，将使用一个默认超时为2分钟的客户端。
- **OnPaymentAttempt, OnPaymentSuccess, OnPaymentFailure (func)**: 可选。支付生命周期事件的回调函数，用于监控和日志记录。

### 返回值

- **成功**: 返回一个指向新创建的 `*X402Transport` 实例的指针和 `nil` 错误。
- **失败**: 返回 `nil` 和一个错误。可能的错误包括：
  - `invalid server URL: ...`: `ServerURL` 字符串无法被正确解析。
  - `signer cannot be nil`: `Signer` 参数为nil。
  - 创建 `PaymentHandler` 时发生的其他错误。

**Section sources**
- [transport.go](file://transport.go#L77-L114)
- [transport.go](file://transport.go#L68-L68)

## PaymentHandler与PaymentSigner

`PaymentHandler` 和 `PaymentSigner` 是支付流程中的两个关键组件。

### PaymentHandler

`PaymentHandler` 是支付逻辑的协调者。它持有 `PaymentSigner` 和 `HandlerConfig`，并负责：

1.  **ShouldPay**: 根据 `PaymentCallback` 或默认策略，决定是否应该为给定的支付要求付款。
2.  **selectPaymentMethod**: 从服务器提供的多个支付选项（`accepts`）中，根据客户端配置的 `ClientPaymentOption` 选择最佳的支付方式。选择逻辑基于：
    *   **优先级 (Priority)**: 优先级数字越小，优先级越高。
    *   **金额 (Amount)**: 在相同优先级下，选择金额更小的选项。
    *   **客户端限制**: 检查 `MaxAmount`，确保所需金额不超过客户端愿意支付的上限。
3.  **CreatePayment**: 这是主要的支付创建方法。它调用 `selectPaymentMethod` 选择支付方式，然后调用 `ShouldPay` 进行确认，最后委托给 `PaymentSigner` 的 `SignPayment` 方法来生成签名的支付。

**Section sources**
- [handler.go](file://handler.go#L10-L13)
- [handler.go](file://handler.go#L37-L55)
- [handler.go](file://handler.go#L58-L82)
- [handler.go](file://handler.go#L85-L152)

### PaymentSigner接口

`PaymentSigner` 是一个接口，定义了所有签名器必须实现的方法。它抽象了签名的具体实现，允许使用不同的密钥管理方式。

```go
type PaymentSigner interface {
    SignPayment(ctx context.Context, req PaymentRequirement) (*PaymentPayload, error)
    GetAddress() string
    SupportsNetwork(network string) bool
    HasAsset(asset, network string) bool
    GetPaymentOption(network, asset string) *ClientPaymentOption
}
```

#### 方法说明

- **SignPayment**: 使用签名者的私钥为给定的 `PaymentRequirement` 生成一个 `PaymentPayload`。它使用EIP-712标准对支付授权进行签名。
- **GetAddress**: 返回签名者的以太坊地址（如 `0x...`）。
- **SupportsNetwork**: 检查签名者是否支持指定的网络（如 "base", "ethereum"）。
- **HasAsset**: 检查签名者是否在指定网络上拥有指定的资产（如USDC）。
- **GetPaymentOption**: 根据网络和资产，返回客户端配置的 `ClientPaymentOption`，其中包含签名所需的 `ChainID` 等信息。

### 不同的签名器实现

该库提供了多种 `PaymentSigner` 的实现：

- **PrivateKeySigner**: 使用一个十六进制编码的私钥进行签名。这是最直接的实现。
- **MnemonicSigner**: 使用BIP-39助记词和BIP-32 HD派生路径（如 `m/44'/60'/0'/0/0`）来派生私钥。
- **KeystoreSigner**: 使用一个加密的Keystore文件（如Geth或Parity生成的UTC文件）和密码来解密并获取私钥。
- **MockSigner**: 一个用于测试的模拟签名器，生成假的签名，不涉及真实的私钥。

**Section sources**
- [signer.go](file://signer.go#L23-L38)
- [signer.go](file://signer.go#L41-L45)
- [signer.go](file://signer.go#L233-L253)
- [signer.go](file://signer.go#L333-L336)

## 关键数据结构

这些结构体定义了支付流程中使用的数据格式。

### PaymentRequirement

服务器在402响应中提供的支付要求。

- **Scheme (string)**: 支付方案，如 "exact"。
- **Network (string)**: 支付网络，如 "base"。
- **MaxAmountRequired (string)**: 所需的最大金额，以字符串形式的整数表示（Wei）。
- **Asset (string)**: 资产地址，如USDC的合约地址。
- **PayTo (string)**: 收款人地址。
- **Resource (string)**: 被请求的资源描述。
- **MaxTimeoutSeconds (int)**: 支付授权的有效期（秒）。
- **Extra (map[string]string)**: 额外信息，如资产名称和版本。

### PaymentPayload

客户端发送的已签名支付。

- **X402Version (int)**: 协议版本。
- **Scheme (string)**: 支付方案。
- **Network (string)**: 支付网络。
- **Payload (PaymentPayloadData)**: 包含签名和授权信息的嵌套结构。
  - **Signature (string)**: EIP-712签名。
  - **Authorization (PaymentAuthorization)**: 授权数据。
    - **From (string)**: 付款人地址。
    - **To (string)**: 收款人地址。
    - **Value (string)**: 支付金额。
    - **ValidAfter/ValidBefore (string)**: 授权有效的时间窗口。
    - **Nonce (string)**: 唯一的随机数，防止重放攻击。

### PaymentEvent

表示支付生命周期中的一个事件（尝试、成功、失败）。

- **Type (PaymentEventType)**: 事件类型 (`attempt`, `success`, `failure`)。
- **Resource (string)**: 相关资源。
- **Method (string)**: 相关的JSON-RPC方法。
- **Amount (*big.Int)**: 支付金额。
- **Network (string)**: 网络。
- **Asset (string)**: 资产。
- **Recipient (string)**: 收款人。
- **Error (error)**: 失败时的错误信息。
- **Timestamp (int64)**: 事件发生的时间戳。

**Section sources**
- [types.go](file://types.go#L9-L27)
- [types.go](file://types.go#L30-L35)
- [types.go](file://types.go#L38-L55)
- [types.go](file://types.go#L58-L68)
- [types.go](file://types.go#L71-L81)
- [types.go](file://types.go#L84-L91)

## 使用示例

以下是如何创建 `X402Transport` 实例并注册支付回调的示例。

```go
// 1. 创建一个签名器 (例如，使用私钥)
signer, err := x402.NewPrivateKeySigner(
    "your-private-key-hex",
    // 配置客户端接受的支付选项
    x402.AcceptUSDCBase().WithPriority(1),
    x402.AcceptUSDCBaseSepolia().WithPriority(2),
)
if err != nil {
    log.Fatal(err)
}

// 2. 创建 X402Transport 配置
config := x402.Config{
    ServerURL: "https://your-mcp-server.com",
    Signer:    signer,
    // 可选：支付前的确认回调
    PaymentCallback: func(amount *big.Int, resource string) bool {
        fmt.Printf("请求支付 %s Wei 以访问资源: %s\n", amount.String(), resource)
        return true // 自动批准
    },
    // 可选：支付事件回调
    OnPaymentAttempt: func(event x402.PaymentEvent) {
        fmt.Printf("支付尝试: %s %s 到 %s\n", event.Amount.String(), event.Asset, event.Recipient)
    },
    OnPaymentSuccess: func(event x402.PaymentEvent) {
        fmt.Printf("支付成功: 交易 %s\n", event.Transaction)
    },
    OnPaymentFailure: func(event x402.PaymentEvent, err error) {
        fmt.Printf("支付失败: %v\n", err)
    },
}

// 3. 创建 X402Transport 实例
transport, err := x402.New(config)
if err != nil {
    log.Fatal(err)
}

// 4. 使用 transport 发送请求 (支付将自动处理)
ctx := context.Background()
request := transport.JSONRPCRequest{
    ID:     mcp.NewRequestId(1),
    Method: "mcp.resource.get",
    Params: map[string]any{"resource": "premium-content"},
}

response, err := transport.SendRequest(ctx, request)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("响应: %+v\n", response)
```

**Section sources**
- [client_options.go](file://client_options.go#L7-L21)
- [client_options.go](file://client_options.go#L24-L38)
- [signer.go](file://signer.go#L65-L106)
- [transport.go](file://transport.go#L77-L114)

## 常见问题与解决方案

### 签名失败 (Signing Failed)

- **原因**: `ErrSigningFailed` 错误通常由以下原因引起：
  - 私钥、助记词或Keystore文件无效。
  - 用于签名的 `ClientPaymentOption` 缺少 `ChainID`。
  - 网络或资产不匹配。
- **解决方案**:
  - 仔细检查私钥/助记词/Keystore文件的正确性。
  - 确保在创建 `PrivateKeySigner`、`MnemonicSigner` 或 `KeystoreSigner` 时，提供的 `ClientPaymentOption` 包含正确的 `ChainID`。
  - 使用 `AcceptUSDCBase()` 等辅助函数来确保配置正确。

### 网络不支持 (Unsupported Network)

- **原因**: 服务器要求的网络（如 "polygon"）不在客户端配置的 `ClientPaymentOption` 中。
- **解决方案**:
  - 在创建签名器时，添加对所需网络的支持。例如，为 `PrivateKeySigner` 添加一个针对Polygon的 `ClientPaymentOption`。
  - 检查 `SupportsNetwork` 方法的返回值以进行调试。

### 支付被拒绝 (Payment Declined)

- **原因**: `PaymentCallback` 回调返回了 `false`，或者 `PaymentHandler` 无法找到可接受的支付方式（`ErrNoAcceptablePayment`）。
- **解决方案**:
  - 检查 `PaymentCallback` 的逻辑，确保它在预期情况下返回 `true`。
  - 确认客户端的 `ClientPaymentOption` 与服务器的 `PaymentRequirement` 在 `Scheme`、`Network` 和 `Asset` 上匹配。
  - 检查 `MaxAmount` 限制，确保服务器要求的金额没有超过客户端的设置。

**Section sources**
- [errors.go](file://errors.go#L10-L22)
- [signer.go](file://signer.go#L112-L129)
- [handler.go](file://handler.go#L85-L152)
- [handler.go](file://handler.go#L37-L55)