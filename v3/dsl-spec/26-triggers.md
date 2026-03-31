# 26 — Triggers

本文件定義 workflow instance 的啟動機制。

---

## 觸發模型

Trigger 的唯一職責是 **建立新的 workflow instance**。

- 一個 workflow definition MAY 定義多個 triggers
- 每個 trigger 獨立運作，各自可建立 instance
- `triggers` 陣列 MUST 至少包含一個 trigger

```yaml
triggers:
  - type: manual
  - type: event
    event: order.created
    when: ${ event.data.source == "api" }
```

---

## manual Trigger

透過 API 或 CLI 手動啟動 workflow instance。

### Schema

```yaml
- type: manual
```

無額外欄位。

### 行為

- 呼叫端提供 input 資料（需符合 `input_schema`，詳見 [04-input-output-schema](04-input-output-schema.md)）
- 引擎建立 instance，狀態設為 CREATED，隨即轉為 RUNNING（詳見 [23-lifecycle](23-lifecycle.md)）

---

## event Trigger

當系統收到匹配的事件時，自動建立 workflow instance。

### Schema

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | string | MUST | 固定值 `event` |
| `event` | string | MUST | 要訂閱的事件類型 |
| `when` | CEL expression | MAY | 過濾條件，MUST 回傳 boolean |
| `input_mapping` | map | MAY | 事件資料到 workflow input 的映射 |

```yaml
- type: event
  event: order.created
  when: ${ event.data.source == "api" && event.data.amount > 100 }
  input_mapping:
    order_id: ${ event.data.order_id }
    action: ${ event.data.type }
    notify_customer: ${ default(event.data.notify, true) }
```

> **注意**：trigger 的 `when` 欄位用於事件過濾條件，與曾提議但已否決的 step-level `when` 條件守衛無關（該提議已由 `if` step 取代，詳見 [08-step-control-flow](08-step-control-flow.md)）。trigger `when` 的行為維持不變。

### 行為

1. 引擎訂閱指定的 event type
2. 收到事件時，若有 `when` 條件，以事件資料求值 CEL 表達式
3. 條件為 `true`（或無 `when`）→ 建立新 instance
4. 條件為 `false` → 忽略該事件
5. `when` 求值失敗 → 忽略該事件並記錄錯誤

### when 的求值上下文

`when` 表達式中可用的 namespace：

| Namespace | 說明 |
|-----------|------|
| `event.type` | 事件類型 |
| `event.data` | 事件酬載 |
| `event.id` | 事件唯一 ID |
| `event.timestamp` | 事件時間戳 |
| `event.source` | 事件來源 |

注意：`input`、`steps`、`vars` 在 trigger 的 `when` 中不可用（instance 尚未建立）。關於 CEL 表達式的完整 namespace 定義，詳見 [03-expressions](03-expressions.md)。

---

## Trigger 與 Input 的關係

### manual trigger

Input 由呼叫端明確提供。

### event trigger

Event trigger 透過 `input_mapping` 將事件資料轉換為 workflow input：

#### 有 input_mapping（建議）

- 每個 key 為 workflow input 的欄位名
- 每個 value 為 CEL 表達式，在 `event` namespace 下求值
- 映射結果 MUST 符合 `input_schema`（若有定義，詳見 [04-input-output-schema](04-input-output-schema.md)）
- 映射表達式求值失敗 → instance 不建立，記錄錯誤

```yaml
- type: event
  event: order.created
  input_mapping:
    order_id: ${ event.data.order_id }
    action: "pay"                          # 字面值
    notify_customer: ${ has(event.data.notify) ? event.data.notify : true }
    items: ${ event.data.line_items }
```

#### 無 input_mapping（隱式 passthrough）

- `event.data` 整體作為 workflow input
- `event.data` MUST 直接符合 `input_schema`
- 適用於事件結構與 input schema 完全一致的情況

---

## 多 Trigger 範例

```yaml
triggers:
  # 手動啟動（測試或管理用途）
  - type: manual

  # 新訂單事件（僅 API 來源），帶 input_mapping
  - type: event
    event: order.created
    when: ${ event.data.source == "api" }
    input_mapping:
      order_id: ${ event.data.order_id }
      action: "pay"
      notify_customer: ${ has(event.data.notify) ? event.data.notify : true }
      items: ${ event.data.line_items }

  # 新訂單事件（僅金額 > 1000），passthrough
  - type: event
    event: order.created
    when: ${ event.data.amount > 1000 }
```

以上設定表示：同一事件可能同時觸發多個 trigger（若都符合條件），各自建立獨立的 instance。引擎 MUST 提供 trigger-level deduplication 機制：同一 `event_id` 對同一 definition 在 5 分鐘窗口內僅觸發一次。同一事件對不同 definitions 各自獨立觸發。

---

## scheduled Trigger

依據 cron 表達式或固定間隔，定時建立 workflow instance。

### Schema

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | string | MUST | 固定值 `scheduled` |
| `cron` | string | MUST（二擇一） | Cron 表達式（5 欄位標準格式） |
| `interval` | duration | MUST（二擇一） | 固定間隔（如 `5m`、`1h`） |
| `timezone` | string | MAY | IANA timezone（如 `Asia/Taipei`），預設 UTC |
| `input` | map | MAY | 每次觸發時提供的固定 input |
| `input_mapping` | map | MAY | 動態 input（可使用 `schedule` namespace） |

`cron` 與 `interval` MUST 擇一，不可同時存在。

### Cron 格式

5 欄位標準 cron 表達式：

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, 0=Sunday)
│ │ │ │ │
* * * * *
```

支援的特殊字元：`*`（任意）、`,`（列舉）、`-`（範圍）、`/`（間隔）。

### input_mapping 的求值上下文

`input_mapping` 表達式中可用的 namespace：

| Namespace | 說明 |
|-----------|------|
| `schedule.fired_at` | 排程觸發時間（timestamp） |
| `schedule.iteration` | 從 definition PUBLISHED 起累計的觸發次數（int） |

```yaml
- type: scheduled
  cron: "0 9 * * MON-FRI"
  timezone: "Asia/Taipei"
  input_mapping:
    report_date: ${ schedule.fired_at }
    batch_id: ${ "batch_" + string(schedule.iteration) }
```

### 行為

1. Definition PUBLISHED 時，引擎根據 `cron` 或 `interval` 建立排程
2. 每次排程觸發時建立新的 workflow instance
3. Instance input = `input`（靜態）或 `input_mapping` 求值結果
4. 若有 `input_schema` → 驗證 input（詳見 [04-input-output-schema](04-input-output-schema.md)）
5. Definition DEPRECATED 時，停止排程（不再觸發新 instance）

### 錯漏觸發處理

| 情境 | 行為 |
|------|------|
| 引擎停機期間錯過排程 | 引擎重啟後 MUST NOT 補觸發（fire-and-forget）。引擎 MAY 提供設定以啟用補觸發（catch-up） |
| 前一次觸發的 instance 仍在執行 | 仍然建立新 instance（不等待前一個完成） |

### 併發控制

引擎 MAY 支援 scheduled trigger 的併發控制：

| 設定 | 說明 |
|------|------|
| `allow_concurrent: true`（預設） | 即使前一個 instance 仍在執行，仍建立新 instance |
| `allow_concurrent: false` | 若前一個由此 trigger 建立的 instance 仍在執行，跳過本次觸發 |

```yaml
- type: scheduled
  interval: 5m
  allow_concurrent: false
  input:
    task: "cleanup"
```

### 範例

```yaml
triggers:
  - type: manual

  # 每天早上 9 點（台北時間）
  - type: scheduled
    cron: "0 9 * * *"
    timezone: "Asia/Taipei"
    input:
      report_type: "daily"

  # 每 30 分鐘
  - type: scheduled
    interval: 30m
    allow_concurrent: false
    input:
      task: "sync"
```

---

## http Trigger

透過 HTTP endpoint 直接觸發 workflow instance。引擎為每個 http trigger 自動產生 REST API endpoint。

### Schema

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | string | MUST | 固定值 `http` |
| `method` | string | MAY | HTTP method，預設 `POST` |
| `path` | string | MUST | URL path（相對於引擎 base URL） |
| `input_mapping` | map | MAY | HTTP request 到 workflow input 的映射 |
| `response` | object | MAY | 自訂 HTTP response 行為 |

```yaml
- type: http
  method: POST
  path: /api/orders
  input_mapping:
    order_id: ${ request.body.order_id }
    items: ${ request.body.items }
    source: ${ request.headers["X-Source"] }
```

### input_mapping 的求值上下文

| Namespace | 說明 |
|-----------|------|
| `request.body` | HTTP request body（JSON 解析後的 map） |
| `request.headers` | HTTP headers（map\<string, string\>） |
| `request.query` | URL query parameters（map\<string, string\>） |
| `request.path_params` | URL path parameters（若 path 含 `{param}`） |
| `request.method` | HTTP method（string） |

### Path Parameters

Path 中 MAY 使用 `{param}` 定義路徑參數：

```yaml
- type: http
  path: /api/orders/{order_id}/process
  input_mapping:
    order_id: ${ request.path_params.order_id }
```

### Response 行為

| 設定 | 說明 |
|------|------|
| `response.mode` | `async`（預設）或 `sync` |

#### async 模式（預設）

1. 收到 HTTP request
2. 驗證 input
3. 建立 instance
4. 立即回傳 HTTP 202 Accepted：

```json
{
  "instance_id": "inst_abc123",
  "status": "created"
}
```

#### sync 模式

1. 收到 HTTP request
2. 建立並執行 instance
3. 等待 instance 完成（受 `response.timeout` 限制）
4. 回傳 instance output：

```json
{
  "instance_id": "inst_abc123",
  "status": "succeeded",
  "output": { ... }
}
```

sync 模式設定：

| 欄位 | 說明 |
|------|------|
| `response.timeout` | 等待 instance 完成的最大時間（預設 30s） |
| `response.success_status` | 成功時的 HTTP status code（預設 200） |
| `response.error_status` | 失敗時的 HTTP status code（預設 500） |

```yaml
- type: http
  path: /api/validate
  response:
    mode: sync
    timeout: 10s
    success_status: 200
    error_status: 422
```

### Content-Type

- Request body MUST 為 JSON（`Content-Type: application/json`）
- 非 JSON request → HTTP 415 Unsupported Media Type
- Response 一律為 JSON

### 範例

```yaml
triggers:
  - type: manual

  - type: http
    method: POST
    path: /api/orders
    input_mapping:
      order_id: ${ request.body.order_id }
      action: ${ default(request.body.action, "pay") }
      source: ${ default(request.headers["X-Source"], "http") }
    response:
      mode: async
```

---

## 完整範例

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: order_processing
  version: 1

triggers:
  - type: manual

  - type: event
    event: order.created
    when: ${ event.data.source == "api" }
    input_mapping:
      order_id: ${ event.data.order_id }
      action: "pay"

  - type: scheduled
    cron: "0 */6 * * *"
    timezone: "Asia/Taipei"
    input:
      mode: "batch_cleanup"

  - type: http
    method: POST
    path: /api/orders
    input_mapping:
      order_id: ${ request.body.order_id }
    response:
      mode: async

input_schema:
  type: object
  properties:
    order_id:
      type: string
    action:
      type: string
    mode:
      type: string

steps:
  - id: process
    type: task
    action: order.process
    input:
      order_id: ${ input.order_id }
```

---

## 相關文件

- [02-document-structure](02-document-structure.md) — `triggers` 在 Workflow 頂層結構中的位置
- [03-expressions](03-expressions.md) — CEL 表達式與 `event`、`schedule`、`request` namespace
- [04-input-output-schema](04-input-output-schema.md) — Input schema 驗證
- [09-step-events](09-step-events.md) — `emit` / `wait_event` 步驟（與 event trigger 共用事件系統）
- [23-lifecycle](23-lifecycle.md) — Instance 狀態機（CREATED → RUNNING）
- [24-instance-labels](24-instance-labels.md) — 建立 instance 時可同時指定 labels
