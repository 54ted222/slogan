# 17 — Agent Hooks 與 Streaming

本文件定義 agent 的 hook 機制（生命週期關鍵節點的自訂邏輯）與 streaming 輸出支援。

---

## Agent Hooks

Agent hooks 允許在 agent 生命週期的關鍵節點執行自訂邏輯，用於審計、監控、權限檢查、資料轉換等橫切關注點。

### 7 個觸發點

| Hook | 觸發時機 | 可用資料 | 典型用途 |
|------|---------|---------|---------|
| `on_session_start` | Agent session 初始化完成後 | `session.id`、`input` | 初始化外部資源、記錄啟動 |
| `on_iteration_start` | 每輪迭代開始前 | `session.iteration`、`session.messages` | 注入動態 context、檢查前置條件 |
| `on_before_llm_call` | 送 messages 給 LLM 前 | `hook.messages` | 修改 messages、注入系統訊息 |
| `on_after_llm_call` | 收到 LLM 回應後 | `hook.response`、`hook.has_tool_calls` | 記錄回應、攔截特定內容 |
| `on_before_tool_call` | 執行 tool 前 | `hook.tool_name`、`hook.tool_input` | 權限檢查、輸入驗證、審計記錄 |
| `on_after_tool_call` | Tool 執行完成後 | `hook.tool_name`、`hook.tool_output`、`hook.success` | 結果轉換、錯誤處理 |
| `on_session_end` | Agent session 結束時 | `hook.final_output`、`hook.status` | 清理資源、記錄結果、發送通知 |

### 觸發順序

系統預設 loop 中，一輪完整迭代的 hook 觸發順序：

```
on_session_start                     （僅第一輪前）
  │
  ├── on_iteration_start
  │     │
  │     ├── on_before_llm_call
  │     ├── [LLM 呼叫]
  │     ├── on_after_llm_call
  │     │
  │     ├── on_before_tool_call      （每個 tool call 各觸發一次）
  │     ├── [Tool 執行]
  │     ├── on_after_tool_call
  │     │
  │     └── （回到 on_iteration_start，若有下一輪）
  │
  └── on_session_end                 （session 結束時）
```

---

## Hook 配置

在 agent definition 的 `hooks` 欄位定義，每個 hook 指向 workflow step 陣列：

```yaml
apiVersion: agent/v3
kind: Agent

metadata:
  name: order.analyzer
  version: 1

# ... model, system_prompt, tools 等 ...

hooks:
  on_before_tool_call:
    - type: if
      when: ${ hook.tool_name == "order.delete" }
      then:
        - type: task
          action: audit.log
          input:
            action: "agent_tool_call"
            tool: ${ hook.tool_name }
            input: ${ hook.tool_input }
            session_id: ${ session.id }

  on_session_end:
    - type: task
      action: metrics.record
      input:
        session_id: ${ session.id }
        iterations: ${ session.iteration }
        tool_calls: ${ session.tool_call_count }
        status: ${ hook.status }
        output: ${ hook.final_output }
```

### Hook Step 可用的 Step 類型

Hook steps 支援與一般 workflow 相同的 step 類型：`task`、`assign`、`if`、`switch`、`emit`、`fail` 等。

但不支援 `return`（hook 不能中斷 agent 流程）和 `wait_event`（hook 必須同步完成）。

若 hook 中的 `fail` step 被觸發，agent session 將中止並標記為 FAILED。

---

## Hook Namespace

Hook steps 中除了標準 namespace（`input`、`session`）外，額外可存取 `hook` namespace，包含該 hook 觸發時的上下文資料。

### 各 Hook 的 `hook.*` 變數

| Hook | 可用變數 |
|------|---------|
| `on_session_start` | （無額外 hook 變數，使用 `session.id`、`input`） |
| `on_iteration_start` | （無額外 hook 變數，使用 `session.iteration`、`session.messages`） |
| `on_before_llm_call` | `hook.messages`（即將送給 LLM 的 messages 陣列） |
| `on_after_llm_call` | `hook.response`（LLM 回傳的 assistant message）、`hook.has_tool_calls`（bool） |
| `on_before_tool_call` | `hook.tool_name`（string）、`hook.tool_input`（map） |
| `on_after_tool_call` | `hook.tool_name`（string）、`hook.tool_output`（any）、`hook.success`（bool） |
| `on_session_end` | `hook.final_output`（any）、`hook.status`（`SUCCEEDED` 或 `FAILED`） |

### 範例：審計所有高風險 tool 呼叫

```yaml
hooks:
  on_before_tool_call:
    - type: if
      when: ${ hook.tool_name in ["order.delete", "payment.refund", "customer.deactivate"] }
      then:
        - type: task
          action: audit.log
          input:
            level: "warning"
            event: "high_risk_tool_call"
            tool: ${ hook.tool_name }
            input: ${ hook.tool_input }
            session_id: ${ session.id }
            iteration: ${ session.iteration }
```

---

## Streaming

Agent 執行可能耗時數分鐘，streaming 讓外部系統即時接收部分結果。

### 三種 Stream 層級

| 層級 | 說明 | 適用場景 |
|------|------|---------|
| `token` | LLM 回應逐 token 串流 | Chat UI、即時互動 |
| `iteration` | 每輪迭代結果完整送出 | 進度追蹤、Dashboard |
| `none`（預設） | 僅最終結果 | 批次處理、後台任務 |

### 配置

在 agent definition 或 step 層級的 `config` 中設定：

```yaml
# Agent definition 層級
config:
  stream: token                   # MAY, 預設 none

# Step 層級覆寫
- type: agent
  agent: order.analyzer
  prompt: "..."
  config:
    stream: iteration             # 覆寫 agent definition 的預設
```

---

## Token-level Streaming

引擎在 `agent.llm_call` 時啟用 LLM 的 streaming API，透過 SSE 即時推送增量文字。

### SSE Event 格式

```yaml
event: agent.stream.token
data:
  session_id: "asess_xxx"
  iteration: 3
  delta: "根據分析結果，"            # 增量文字
  timestamp: "2026-03-31T10:00:00Z"
```

每個 SSE event 包含一個 `delta` 文字片段。客戶端將所有 delta 串接即可取得完整回應。

### Stream 結束信號

```yaml
event: agent.stream.token_end
data:
  session_id: "asess_xxx"
  iteration: 3
  has_tool_calls: true             # 是否有後續 tool call
```

---

## Iteration-level Streaming

每輪迭代完成後發出完整的迭代結果，包含 LLM 回應與 tool call 結果。

### SSE Event 格式

```yaml
event: agent.stream.iteration
data:
  session_id: "asess_xxx"
  iteration: 3
  message:
    role: assistant
    content: "根據分析結果，需要進一步確認..."
  tool_calls:
    - name: order.load
      input: { order_id: "ORD-001" }
  tool_results:
    - tool_call_id: "tc_001"
      content: { ... }
```

### Session 結束事件

無論哪種 stream 層級，session 結束時都會發出：

```yaml
event: agent.stream.session_end
data:
  session_id: "asess_xxx"
  status: "SUCCEEDED"              # SUCCEEDED | FAILED
  output: { ... }                  # final output（若成功）
  error: { code: "...", message: "..." }  # error 資訊（若失敗）
```

---

## Stream API

### 訂閱端點

```
GET /agents/{session_id}/stream
```

回傳 SSE（Server-Sent Events）串流。客戶端透過標準 SSE 協議連線，接收 agent session 的即時事件。

### 行為

| 情境 | 回應 |
|------|------|
| Session 存在且 RUNNING | 建立 SSE 連線，開始推送事件 |
| Session 存在且已結束 | 回傳 `agent.stream.session_end` 事件後關閉 |
| Session 不存在 | HTTP 404 |
| `config.stream` 為 `none` | HTTP 409（不支援 streaming） |

### 重新連線

SSE 支援 `Last-Event-ID` header。客戶端斷線重連時帶上最後收到的 event ID，server 從該點繼續推送，避免遺漏事件。

---

## 自訂 Loop 中的 Streaming

自訂 loop 可透過以下方式支援 streaming：

### `agent.llm_call` 啟用 token streaming

在 `agent.llm_call` 的 input 中設定 `stream: true`：

```yaml
- id: response
  type: task
  action: agent.llm_call
  input:
    messages: ${ session.messages }
    stream: true                   # 啟用 token-level stream
```

啟用後，`agent.llm_call` 在執行過程中即時推送 `agent.stream.token` 事件，完成後正常回傳完整結果作為 step output。

### 手動發送自訂 stream event

透過 `emit` step 發送自訂 stream event：

```yaml
loop:
  steps:
    - id: response
      type: task
      action: agent.llm_call
      input:
        messages: ${ session.messages }
        stream: true

    # 手動發送自訂進度事件
    - type: emit
      event: agent.stream.custom
      data:
        progress: ${ session.iteration }
        partial_result: ${ steps.response.output.message.content }

    - type: if
      when: ${ !steps.response.output.has_tool_calls }
      then:
        - type: return
          output: ${ steps.response.output.message.content }

    - id: tool_results
      type: task
      action: agent.exec_tool_calls
      input:
        tool_calls: ${ steps.response.output.tool_calls }

    - type: task
      action: agent.set_messages
      input:
        messages: ${ session.messages.concat(steps.tool_results.output.results) }
```

自訂 emit 的事件會透過相同的 SSE 連線推送給訂閱端，event name 以 `agent.stream.` 為前綴。

---

## 相關文件

- [09-step-events](../02-steps/09-step-events.md) — emit 與 wait_event step
- [16-agent-loop](16-agent-loop.md) — Agent step schema 與自訂 loop
- [15-agent-skills](15-agent-skills.md) — Agent skills 與 agentskills.io 規格
