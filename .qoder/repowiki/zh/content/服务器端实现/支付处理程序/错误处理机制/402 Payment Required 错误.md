# 402 Payment Required 错误

<cite>
**Referenced Files in This Document**   
- [server/handler.go](file://server/handler.go)
- [server/types.go](file://server/types.go)
- [server/requirements.go](file://server/requirements.go)
- [errors.go](file://errors.go)
</cite>

## 目录
1. [实现机制概述](#实现机制概述)
2. [HTTP状态码设计原因](#http状态码设计原因)
3. [PaymentRequirements402Response结构体详解](#paymentrequirements402response结构体详解)
4. [实际JSON响应示例](#实际json响应示例)
5. [Accepts列表填充机制](#accepts列表填充机制)
6. [Verbose日志调试作用](#verbose日志调试作用)

## 实现机制概述

`X402Handler`中的`sendPaymentRequiredError`函数实现了JSON-RPC 2.0规范的402错误响应机制。该函数在检测到客户端请求需要付费的工具但未提供支付凭证时被调用，通过构造符合JSON-RPC 2.0标准的错误响应来通知客户端支付要求。

该函数的调用流程始于`X402Handler.ServeHTTP`方法，当系统解析客户端请求并确认所请求的工具需要支付但请求中未包含支付信息时，会调用`sendPaymentRequiredError`函数。此过程确保了只有在明确需要支付且支付信息缺失的情况下才会返回402错误。

**Section sources**
- [server/handler.go](file://server/handler.go#L217-L235)
- [server/handler.go](file://server/handler.go#L32-L214)

## HTTP状态码设计原因

`sendPaymentRequiredError`函数中一个关键的设计决策是使用`w.WriteHeader(http.StatusOK)`将HTTP状态码设置为200，而非标准的402。这种设计遵循了x402规范的特殊要求：错误信息被封装在JSON-RPC响应体中，而不是依赖HTTP状态码。

这种设计的主要原因包括：
1. **协议一致性**：保持与JSON-RPC 2.0协议的一致性，所有响应都通过标准的JSON-RPC错误格式传达
2. **错误信息丰富性**：HTTP 200状态码允许在响应体中包含详细的支付要求信息，而标准HTTP 402响应通常只包含简单的错误消息
3. **中间件兼容性**：避免被网络中间件或代理服务器拦截或修改402状态码的响应
4. **统一错误处理**：所有错误类型（包括402）都通过相同的JSON-RPC错误处理机制处理

这种设计使得错误处理更加灵活和可扩展，同时保持了API的简洁性和一致性。

**Section sources**
- [server/handler.go](file://server/handler.go#L217-L235)

## PaymentRequirements402Response结构体详解

`PaymentRequirements402Response`结构体定义了402错误响应的具体内容，包含以下关键字段：

- **X402Version**：表示x402协议的版本号，当前为1，用于确保客户端和服务器之间的协议兼容性
- **Error**：描述性错误消息，明确指出"Payment required to access this resource"（访问此资源需要支付）
- **Accepts**：包含一个或多个`PaymentRequirement`对象的数组，详细列出可接受的支付选项

`PaymentRequirement`结构体进一步定义了每个支付选项的具体要求：
- **Scheme**：支付方案，如"exact"表示精确金额支付
- **Network**：区块链网络，如"base"或"base-sepolia"
- **MaxAmountRequired**：所需支付的最大金额
- **Asset**：支付资产的合约地址
- **PayTo**：收款方地址
- **Resource**：被访问资源的标识符
- **Description**：支付要求的描述信息
- **MimeType**：响应的MIME类型
- **MaxTimeoutSeconds**：支付的有效期（秒）

**Section sources**
- [server/types.go](file://server/types.go#L19-L23)
- [server/types.go](file://server/types.go#L4-L16)

## 实际JSON响应示例

当客户端请求一个需要支付的工具但未提供支付凭证时，系统会返回如下格式的JSON响应：

```json
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "error": {
    "code": 402,
    "message": "Payment required",
    "data": {
      "x402Version": 1,
      "error": "Payment required to access this resource",
      "accepts": [
        {
          "scheme": "exact",
          "network": "base",
          "maxAmountRequired": "1.0",
          "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
          "payTo": "0x1234567890123456789012345678901234567890",
          "resource": "mcp://tools/paywalled-tool",
          "description": "Access to premium tool",
          "mimeType": "application/json",
          "maxTimeoutSeconds": 60,
          "extra": {
            "name": "USD Coin",
            "version": "2"
          }
        }
      ]
    }
  }
}
```

此响应示例展示了客户端请求付费工具但未提供支付凭证时的完整错误格式，包含了协议版本、错误描述和具体的支付要求。

**Section sources**
- [server/handler.go](file://server/handler.go#L217-L235)
- [server/types.go](file://server/types.go#L19-L23)

## Accepts列表填充机制

`Accepts`列表中的支付要求从`Config.PaymentTools`配置中提取并填充。具体机制如下：

1. 当`X402Handler.ServeHTTP`处理请求时，会根据请求的工具名称从`h.config.PaymentTools`映射中查找对应的支付要求
2. 找到的`[]PaymentRequirement`数组直接作为`sendPaymentRequiredError`函数的`requirements`参数传递
3. 在返回402错误前，系统会确保每个支付要求的`Resource`字段被正确设置为`mcp://tools/{toolName}`格式
4. 如果`MimeType`字段为空，则默认设置为`application/json`

开发者可以通过`RequireUSDCBase`和`RequireUSDCBaseSepolia`等辅助函数来创建标准化的支付要求，这些函数预设了USDC在Base网络上的常见支付参数，简化了配置过程。

**Section sources**
- [server/handler.go](file://server/handler.go#L32-L214)
- [server/requirements.go](file://server/requirements.go#L7-L39)

## Verbose日志调试作用

Verbose日志在此过程中扮演着重要的调试角色，当`config.Verbose`设置为`true`时，系统会输出详细的日志信息：

1. **请求追踪**：记录每个请求的来源和方法类型，帮助识别流量模式
2. **支付决策**：明确记录工具是否需要支付以及检查支付信息的过程
3. **支付要求详情**：在发送402错误前，详细列出所有可用的支付选项，包括网络、资产、金额和收款地址
4. **流程监控**：跟踪支付验证、结算等关键步骤的执行情况

这些日志信息对于调试支付流程、验证配置正确性以及监控系统行为至关重要，特别是在开发和测试阶段能够快速定位问题。

**Section sources**
- [server/handler.go](file://server/handler.go#L32-L214)
- [server/handler.go](file://server/handler.go#L217-L235)