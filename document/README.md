# new-api 二次开发导航

本文档面向第一次接触本项目、准备做功能改造或增加上游渠道的开发者。内容依据当前代码结构整理，代码和配置发生变化时应以实际实现为准。

## 推荐阅读顺序

1. [项目总览与目录地图](./01-项目总览与目录地图.md)：先建立模块边界和依赖方向。
2. [启动流程与请求链路](./02-启动流程与请求链路.md)：理解服务如何启动，以及一次 API 请求如何经过路由、鉴权、选渠道、转发和结算。
3. [后端二开方法](./03-后端二开方法.md)：按“新增接口、修改业务、修改数据模型、修改配置”定位入口。
4. [Relay、渠道与协议适配](./04-Relay渠道适配.md)：需要接入模型供应商、协议转换或修改流式响应时阅读。
5. [前端管理台](./05-前端管理台.md)：需要修改页面、表单、API 请求或多语言时阅读。
6. [任务插件与异步任务](./06-任务插件与异步任务.md)：需要接入视频、图片、音乐、轮询任务或 JavaScript 插件时阅读。
7. [数据模型、计费、认证与安全](./07-数据模型计费认证与安全.md)：涉及用户、Token、权限、余额、订阅、日志或安全流程时阅读。
8. [开发、测试与部署](./08-本地开发测试部署.md)：准备运行、调试和提交修改时阅读。
9. [二开路线图与常见问题](./09-二开路线图与常见问题.md)：用于制定改造顺序和快速排查问题。

## 快速定位表

| 目标 | 首先查看 | 然后查看 |
| --- | --- | --- |
| 新增后台 API | `router/*` → `controller/*` | `service/*`、`model/*` |
| 修改 OpenAI/Claude/Gemini 请求 | `router/relay-router.go` | `controller/relay.go`、`relay/*` |
| 新增供应商渠道 | `relay/channel/<provider>/` | `relay/relay_adaptor.go`、`constant/channel.go` |
| 修改模型选择、重试、负载 | `service/channel_select.go` | `model/channel*.go`、`middleware/distributor.go` |
| 修改价格或扣费 | `service/billing*.go` | `service/quota.go`、`types/price_data.go`、`setting/billing_setting/` |
| 新增异步任务 | `relay/channel/task/` 或 `plugins/tasks/` | `service/task_polling.go`、`model/task.go` |
| 修改管理台页面 | `web/src/routes/` | `web/src/features/`、`web/src/lib/api.ts` |
| 修改权限 | `middleware/auth.go` | `service/authz/`、`router/*` |
| 修改数据库字段 | `model/<entity>.go` | `model/main.go`、迁移函数和对应测试 |

## 重要约定

- 根模块、`relaykit/`、`tokenkit/` 是相互独立的 Go 模块；修改后分别构建验证。
- 根模块业务代码使用 `common/json.go` 的 JSON 包装函数，不要直接调用 `encoding/json` 的编解码函数。
- `relaykit/` 不依赖根模块；其 JSON 操作使用 `relaykit/relayconvert/kitutil`。
- 计费修改必须同时检查预扣、结算、退款、日志和异常边界，不能只修改显示价格。
- 前端用户可见文字必须走 i18n；数字和货币格式优先复用 `web/src/lib/format.ts`、`web/src/lib/currency.ts`。
- 不要把二开逻辑直接塞进 `main.go`、路由注册函数或超大的 Controller；应保持路由、接口层、业务层、持久化层边界清楚。
- 本目录是面向本地二开的补充导航，不替代项目已有正式文档和插件 API 规范。

## 关键技术边界

```text
浏览器 / API 客户端
        │
        ▼
Gin 路由 router/ ── 中间件 middleware/
        │
        ▼
Controller controller/ ── DTO dto/
        │
        ├── 管理业务 service/ ── 持久化 model/ ── SQLite/MySQL/PostgreSQL
        │
        └── AI Relay controller/relay.go
                    │
                    ├── relay/common：请求上下文、计费上下文、流状态
                    ├── relay/channel：具体供应商适配器
                    ├── relaykit：协议 DTO 与转换
                    └── service：选渠道、扣费、重试、日志

React 管理台 web/ ── /api 管理接口
异步任务 plugins/tasks 或 relay/channel/task ── service/task_polling.go
```

