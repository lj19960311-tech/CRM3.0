# 主服务 · 智能助手（Agent）对外对接文档

面向 **H5 / Web / 小程序 / 业务系统** 接入多式联运智能助手。本文只覆盖 **主服务**（`multimodal_transport`）上对外暴露的 Agent 相关 HTTP；独立 Agent 进程自身的直连协议见 [`api_agent.md`](./api_agent.md)（调试用，生产请走主服务）。

| 项 | 说明 |
|---|---|
| Base URL | `https://{host}/multimodal-transport` |
| 本地默认 | `http://127.0.0.1:8579/multimodal-transport` |
| Content-Type | `application/json; charset=utf-8` |
| 流式 | Server-Sent Events（`text/event-stream`） |
| OpenAPI | `{Base URL}/docs` |
| 实现 | 反代：`core/web/routes/agent_proxy.py`；历史/落库：`agent_chat.py`；能力卡：`assistant_capability_admin.py` |

---

## 1. 架构与选型

```
外部客户端（已登录）
        │
        ▼
主服务  /api/agent/chat 等          ← 对外正式入口（用户登录鉴权）
        │  出站加 X-Agent-Service-Token + X-Anon-Id
        ▼
独立 Agent  /multimodal-transport-agent/chat
        │  查方案/铁路等再回调主服务业务 API
        ▼
主服务  业务查询 + /api/agent/runtime/*（落库/画像）
```

| 入口 | 路径 | 是否推荐对外 | 鉴权 |
|---|---|---|---|
| **主服务反代** | `POST /api/agent/chat` | **是** | 用户登录 |
| Agent 直连 | `POST …/multimodal-transport-agent/chat` | 否（仅联调） | 默认无用户登录校验 |

外部对接请固定走主服务；不要把 Agent 直连地址暴露给公网客户端。

---

## 2. 鉴权

### 2.1 用户登录（C 端必选）

对话、历史、能力卡、提醒收件箱等均要求已登录用户：

| 方式 | 凭证 | `anon_id` 前缀 |
|---|---|---|
| 飞书 | Session Cookie | `feishu:` |
| 手机号 | Session Cookie | `phone:` |
| 微信小程序 | `Authorization: Bearer <access_token>` 或 Session | `wechat:` |

未登录 → `401 {"detail":"请先登录"}`。

登录方式细则见 [`user_login_embed.md`](./user_login_embed.md)；总鉴权表见 [`api_external.md`](./api_external.md) §1.2。

**注意：**

- 纯 Header `X-Anon-Id` **不能**代替用户登录调用本批 C 端接口。
- 主服务转发对话时会把登录态映射为 `X-Anon-Id` 等头传给 Agent；客户端无需、也不应自造服务令牌。

### 2.2 Agent 服务令牌（服务间，非浏览器）

独立 Agent / 机器人回写主服务（落库、画像、反馈）时使用：

| Header | 说明 |
|---|---|
| `X-Agent-Service-Token` | 与主服务、Agent 环境变量 `AGENT_SERVICE_TOKEN` 相同 |
| `X-Anon-Id` | 用户标识，如 `feishu:ou_xxx`（写入类建议必带） |

适用：`/api/agent/runtime/*`、部分反馈接口。浏览器对接 **不要** 使用此令牌。

细则见 [`api_history_chat_feedback.md`](./api_history_chat_feedback.md) §1.5。

### 2.3 本文不覆盖

| 类别 | 说明 |
|---|---|
| 管理后台对话历史 | `/api/agent/chat-history`（无 `/mine`）、备注等，需 `agent-history` 模块权限 |
| Agent `/admin/*`、禁运检查等 | 服务间接口，不经主服务 C 端反代 |
| 飞书 / 企微机器人通道 | 长连接或 Webhook，非本文 HTTP |

---

## 3. 接口一览（对外）

路径均相对于 `/multimodal-transport`。

### 3.1 对话与反代（须登录）

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/api/agent/chat` | 智能助手对话（JSON 或 SSE）；转发到 Agent `/chat` |
| `GET` | `/api/agent/reminder/inbox` | 到期提醒收件箱；转发 Agent `/reminder/inbox` |
| `GET` | `/api/agent/capabilities` | 助手页能力卡（启用配置 ∩ Agent 工具清单） |
| `GET` | `/api/agent/upstream/health` | 探活上游 Agent（内部/调试，OpenAPI 可隐藏） |
| `GET` | `/api/agent/upstream/agents` | 上游工具清单（内部/调试） |

评测入库 ` /api/agent/eval/capture*` 亦须登录，一般仅内部 Debug，外部产品对接可忽略。

### 3.2 对话历史 · C 端（须登录，仅本人）

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/api/agent/chat-history/mine` | 我的对话轮次列表 |
| `GET` | `/api/agent/chat-history/mine/sessions` | 我的会话列表（分页） |
| `DELETE` | `/api/agent/chat-history/mine/sessions?session_id=` | 软删除指定会话 |
| `GET` | `/api/agent/chat-history/mine/by-session?session_id=` | 单会话全部轮次 |
| `GET` | `/api/agent/chat-history/mine/{row_index}` | 单条轮次详情 |

保留最近约 **30** 天。字段与分页细节见 [`api_history_chat_feedback.md`](./api_history_chat_feedback.md) §4。

### 3.3 运行时落库（须登录 *或* Agent 令牌）

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/api/agent/runtime/chat-turns` | 追加一轮对话 |
| `GET` | `/api/agent/runtime/chat-turns/recent` | 某会话最近 N 轮 |
| `GET` | `/api/agent/runtime/chat-turns/exists` | 会话是否已有轮次 |

普通前端一般 **不必** 自己调落库：走 `/api/agent/chat` 时由 Agent 侧按配置写入。仅自建编排或机器人通道才需要。

画像类 `/api/agent/runtime/profile*` 为 Agent↔主服务内部契约，见独立仓 `docs/profile_api_contract.md`，不作为对外产品接口。

### 3.4 相关：反馈（须登录）

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/api/feedback` | 提交反馈 |
| `GET` | `/api/feedback/mine` | 我的反馈 |
| `GET` | `/api/feedback/{ticket_no}` | 按工单号查公开进度（公开） |

详见 [`api_history_chat_feedback.md`](./api_history_chat_feedback.md) §5。

---

## 4. 核心：`POST /api/agent/chat`

### 4.1 请求

**鉴权：** 用户登录。  
**行为：** 主服务校验登录后，将 body 原样转发到 `AGENT_SERVICE_BASE_URL/chat`，并注入服务令牌与用户标识头。

**Body（ChatRequest）**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `message` | string | 是 | 用户输入，非空 |
| `session_id` | string \| null | 否 | 会话 ID；不传则新建，多轮须回传 |
| `stream` | boolean \| null | 否 | `true`=SSE；`false`=整包 JSON；`null` 跟随 Agent 全局配置 |
| `schemes_query_mode` | `"block"` \| `"stream"` \| null | 否 | 方案查询传输；过程标签仅在 `stream=true` 时推给前端 |
| `persist_history` | boolean | 否 | 默认 `true`；`false` 时不写对话历史（如 debug 重跑） |

```bash
# Block（整包 JSON）
curl -s -X POST 'http://127.0.0.1:8579/multimodal-transport/api/agent/chat' \
  -H 'Content-Type: application/json' \
  -H 'Cookie: <session>' \
  -d '{
    "message": "查询广州国际港到株洲北的煤炭价差系数",
    "stream": false
  }'
```

```bash
# Stream（SSE）
curl -N -X POST 'http://127.0.0.1:8579/multimodal-transport/api/agent/chat' \
  -H 'Content-Type: application/json' \
  -H 'Cookie: <session>' \
  -d '{
    "message": "广州市南沙区附近有哪些铁路站可以发玉米",
    "stream": true,
    "session_id": "<上一轮返回的 session_id>"
  }'
```

### 4.2 Block 响应（摘要）

| 字段 | 类型 | 说明 |
|---|---|---|
| `session_id` | string | 会话 ID（后续多轮回传） |
| `intent` | string | 意图码 |
| `sub_agent` | string \| null | 子能力名 |
| `tool_calls` | array | 工具调用记录 |
| `answer` | string | 最终回答（Markdown / 结构化文案） |
| `usage` | object | token 用量 |
| `ui_tag` | string \| null | `Form`=唤起表单卡；`Result`=结果；其它多为 null |
| `form_values` | object \| null | Form 预填值 |
| `audience` | `"internal"` \| `"external"` | 渠道受众打标 |

完整字段与意图表见 [`api_agent.md`](./api_agent.md) §4。

### 4.3 SSE 事件（`stream=true`）

响应 `Content-Type: text/event-stream`。常见事件：

| event | 说明 |
|---|---|
| `intent` | 意图分类完成 |
| `tool_start` / `tool_result` | 工具调用 |
| `progress` | 过程提示（如方案查询过程标签） |
| `delta` | 回答增量，客户端拼接 |
| `done` | 结束；含完整 `session_id` / `answer` / `usage` 等 |
| `error` | 失败，流终止 |

典型序列：`intent` →（`tool_*`）→ `delta`* → `done`。

### 4.4 错误

| HTTP | 场景 |
|---|---|
| `401` | 未登录 |
| `400` | `message` 为空等（上游或本地校验） |
| `502` / `503` | 上游 Agent 不可达或未配置 `AGENT_SERVICE_BASE_URL` |
| `500` | Agent 编排失败 |

---

## 5. 其它 C 端接口（摘要）

### 5.1 `GET /api/agent/reminder/inbox`

拉取当前用户到期提醒，读后从收件箱移除。须登录。Query：`limit`（默认约 20，有上限）。

### 5.2 `GET /api/agent/capabilities`

助手欢迎语与能力卡片列表（与 Agent 已加载工具求交）。Agent 不可达时可能返回空 `items` + `agent_error`，前端宜做静态兜底。

### 5.3 `GET /api/agent/chat-history/mine/sessions`

| Query | 说明 |
|---|---|
| `page` / `page_size` | 分页 |
| `session_id` | 模糊筛选 |
| `channel` | `api` / `feishu` / `wecom` / `cli` |

### 5.4 `GET /api/agent/chat-history/mine/by-session`

| Query | 必填 | 说明 |
|---|---|---|
| `session_id` | 是 | 会话 ID |

非本人或不存在 → `404 {"detail":"会话不存在或已过期"}`。

### 5.5 `DELETE /api/agent/chat-history/mine/sessions`

| Query | 必填 | 说明 |
|---|---|---|
| `session_id` | 是 | 要软删除的会话 |

---

## 6. 推荐对接流程

1. **完成用户登录**（飞书 / 手机号 / 微信），拿到 Session 或 Bearer。  
2. （可选）`GET /api/agent/capabilities` 渲染能力介绍。  
3. `POST /api/agent/chat`，`stream` 按产品选择；保存响应 `session_id`。  
4. 多轮澄清 / 追问：**同一 `session_id`** 再次 `POST /api/agent/chat`。  
5. 侧栏历史：`mine/sessions` → `mine/by-session`。  
6. （可选）`GET /api/agent/reminder/inbox` 轮询到期提醒。  
7. 问题反馈：`POST /api/feedback`。

```text
登录 → capabilities（可选）→ chat（拿 session_id）→ 多轮 chat
                              ↘ chat-history/mine/*（历史）
                              ↘ reminder/inbox（提醒）
```

---

## 7. 运维侧依赖（对接方无需配置，需知）

主服务环境变量（由平台配置）：

| 变量 | 作用 |
|---|---|
| `AGENT_SERVICE_BASE_URL` | 独立 Agent 根地址（含 `/multimodal-transport-agent`） |
| `AGENT_SERVICE_TOKEN` | 主服务 → Agent、Agent → 主服务 的服务间令牌 |

未配置 Agent 基址时，`/api/agent/chat` 会失败；属部署问题，不是客户端参数错误。

---

## 8. 相关文档

| 文档 | 内容 |
|---|---|
| [`api_external.md`](./api_external.md) | 主服务对外 API 总览 |
| [`api_agent.md`](./api_agent.md) | 独立 Agent 直连协议（意图、SSE 细节） |
| [`api_history_chat_feedback.md`](./api_history_chat_feedback.md) | 历史 / 落库 / 反馈字段级说明 |
| [`user_login_embed.md`](./user_login_embed.md) | 用户登录嵌入 |
| 独立仓 `multimodal_transport_agent` | Agent 运行与能力说明 |
