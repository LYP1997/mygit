# Relay、渠道与协议适配

## 1. Relay 的三层结构

### Host 层：统一入口

- `router/relay-router.go`：把 URL 映射到 `RelayFormat`。
- `controller/relay.go`：统一请求处理、错误、重试和计费钩子。
- `relay/common/relay_info.go`：保存用户、Token、模型、渠道、流状态、计费和转换链。

### 协议层：请求和响应处理

- `relay/compatible_handler.go`：普通 OpenAI 兼容文本路径。
- `relay/claude_handler.go`：Claude Messages。
- `relay/gemini_handler.go`：Gemini。
- `relay/responses_handler.go`、`relay/responses_request.go`：OpenAI Responses。
- `relay/image_handler.go`、`relay/audio_handler.go`、`relay/embedding_handler.go`、`relay/rerank_handler.go`：专项协议。
- `relay/helper/`：请求验证、流扫描、模型映射、参数和错误辅助。

### 渠道层：供应商差异

`relay/channel/<provider>/` 通常包含：

- `adaptor.go`：实现 `channel.Adaptor` 或 `channel.TaskAdaptor`；
- 请求转换和 URL 构造；
- Header/鉴权；
- 同步或流式响应解析；
- 供应商错误映射；
- 渠道专用测试。

当前已有许多渠道目录，例如 `openai`、`claude`、`gemini`、`aws`、`ali`、`deepseek`、`vertex`、`volcengine`、`xai`、`advancedcustom` 等。新增前先检查是否已有兼容实现，避免重复渠道。

## 2. 新增普通同步渠道的阅读和实现顺序

建议从以下文件开始：

1. `relay/channel/adapter.go`：确认接口方法。
2. 一个最接近的渠道，例如 `relay/channel/openai/` 或 `relay/channel/claude/`。
3. `relay/relay_adaptor.go`：确认渠道类型到 Adaptor 的工厂注册。
4. `constant/channel.go`：增加渠道类型、名称或能力枚举。
5. `model/channel.go`：确认渠道配置如何存储和读取。
6. `service/channel_select.go`：确认模型、分组、可用性和重试行为。
7. `controller/channel-test.go`：增加渠道测试路径，确保管理台“测试渠道”可用。

普通 `Adaptor` 关键职责：

```text
Init
 → GetRequestURL
 → SetupRequestHeader
 → ConvertOpenAIRequest / ConvertClaudeRequest / ConvertGeminiRequest
 → DoRequest
 → DoResponse
```

如果供应商本身就是 OpenAI 兼容协议，仍要明确处理：Base URL 拼接、模型名映射、Header 覆盖、流式 usage、错误状态码和响应格式。

## 3. 新增异步任务渠道

如果供应商返回任务 ID，需要实现 `TaskAdaptor` 的主要阶段：

```text
ValidateRequestAndSetAction
 → EstimateBilling
 → BuildRequestURL / BuildRequestHeader / BuildRequestBody
 → DoRequest
 → ParseResponse
 → 保存 model.Task
 → service/task_polling.go 周期轮询
 → FetchTask / ParseTaskResult
 → 结算、写日志、保存产物
```

核心接口在 `relay/channel/adapter.go`。不要在 `ParseResponse` 中直接写客户端响应；它应返回与传输无关的 `TaskSubmitResponse`，由 Host 统一渲染。

任务适配器还可能实现：

- `OpenAIVideoConverter`：将结果转换成 OpenAI Video；
- `TaskArtifactProvider`：列出视频、图片、音频等产物；
- `TaskContentRequestProvider`：构造产物下载请求；
- `TaskUsageFactsProvider`：提供分层计费所需的用量事实。

## 4. 模型名、渠道类型和协议格式

需要区分三个名字：

- 客户端模型名：用户请求中的模型；
- 计费模型名：用于价格查询；
- 上游模型名：实际发送给供应商的模型。

`RelayInfo` 中分别保留这些信息。不要用上游模型名去替代渠道选择所需的客户端模型名，也不要因为价格别名修改真实路由模型。

协议格式由 `relaykit/types` 中的 `RelayFormat` 表达。跨协议转换会追加 `RequestConversionChain`，例如 OpenAI Chat → Claude 或 OpenAI Responses → Gemini。转换实现优先放 `relaykit/relayconvert`，Host 专用逻辑留在根模块。

## 5. 流式和可选字段

- 可选的数值、布尔字段使用指针加 `omitempty`，这样可以区分“未提供”和显式 `0`、`false`。
- 供应商支持 `StreamOptions` 时，要确认并更新 `streamSupportedChannels`。
- 流转换必须测试文本增量、工具调用、思考内容、finish reason、usage 和错误事件。
- SSE 不能按普通 JSON 一次性读完；使用已有的 stream scanner/handler。
- WebSocket 要同时检查升级、鉴权、断开、重试和关闭码。

## 6. 图片、音频和异步计费

新协议如果会改变扣费数量，必须从请求验证开始追踪到：

```text
请求数量/时长校验
 → EstimateBilling 或 usage facts
 → 预扣
 → 上游响应中的实际数量
 → Settle / Refund
 → consume log / task log
```

图片数量不能简单用 JSON `data` 数组长度统计；要按有效 URL、Base64 payload 和流完成事件的规则统计。任务插件返回的用量也必须经过 Host 校验。

## 7. 什么时候用 Relay，什么时候用插件

优先使用 Go Relay 的场景：

- 同步文本、Embedding、音频等稳定协议；
- 需要复杂的跨协议转换；
- 需要高性能或严格的 Go 类型约束。

优先使用 Task Plugin 的场景：

- 供应商是异步视频、图片、音乐任务；
- 希望通过数据库上传/替换供应商驱动，不重新编译主程序；
- 使用 Host Protocol（`openai_responses`、`openai_video`、`openai_image`）接入。

## 8. Relay 改造自检表

- [ ] 路由映射到正确的 `RelayFormat`。
- [ ] 渠道类型、工厂注册和管理台渠道表单已接入。
- [ ] Base URL、Header、API Key、模型映射和代理策略正确。
- [ ] 普通和流式响应均能解析。
- [ ] 上游错误映射为统一 `NewAPIError`。
- [ ] 重试不会复用脏的流转换状态。
- [ ] 计费预扣、结算、退款和日志路径完整。
- [ ] 请求中的 `0`、`false` 等显式值不会因 `omitempty` 丢失。
- [ ] 单元测试、golden 测试和必要的渠道测试已补齐。

