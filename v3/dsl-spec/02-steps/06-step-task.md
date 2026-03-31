# 06 — Step: task + Task Definition

本文件定義 `task` step（使用端）與 Task Definition `kind: Task`（定義端）。

---

## Part A — task Step（使用端）

`task` step 呼叫一個已定義的 task definition 來執行工作。Task 的定義（怎麼執行）與使用（在 workflow 中呼叫）是分離的。

### Schema

```yaml
- id: string # MAY — 需要參照 output 時才必填
  type: task # MUST
  action: string # MUST — task definition 的 name（兩段式命名：namespace.action）
  version: integer | "latest" # MAY, 預設 "latest"
  input: map | CEL expression # MAY — 傳給 task 的輸入
  resources: array # MAY — artifact 綁定
  execution:
    policy: replayable | idempotent | non_repeatable # MAY, 預設 replayable
  retry:
    max_attempts: integer # MAY, 預設 1（不重試）
    delay: duration # MAY, 預設 1s
    backoff: fixed | exponential # MAY, 預設 fixed
  timeout: duration # MAY
  compensate: [...] # MAY, step 陣列（saga 補償邏輯）
  on_error: [...] # MAY, step 陣列
  on_timeout: [...] # MAY, step 陣列
```

### action 與 Task Definition 的關係

`action` 欄位參照一個 task definition 的 `metadata.name`。所有 task 命名採用兩段式格式 `namespace.action`：

```yaml
# Workflow step（使用端）
- id: load_order
  type: task
  action: order.load # ← 參照 task definition name
  input:
    order_id: ${ input.order_id }
```

### 版本解析

| 值                 | 說明                              |
| ------------------ | --------------------------------- |
| `"latest"`（預設） | 執行時解析為最新的 PUBLISHED 版本 |
| `integer`          | 固定版本號                        |

在正式環境中 SHOULD 使用固定版本號以確保可重現性。

### input 傳遞

`input` 為傳給 task definition 的輸入資料，會在執行前依 task definition 的 `input_schema` 驗證。

```yaml
# 字面值 map
input:
  order_id: ${ input.order_id }
  amount: 100
  currency: "USD"

# 單一 CEL 表達式（回傳 map）
input: ${ vars.payment_input }
```

### output 模型

Task handler 回傳的值即為 step output，後續 steps 透過 `steps.<id>.output` 存取：

```yaml
- id: check
  type: if
  when: ${ steps.load_order.output.status == "active" }
```

Output 會依 task definition 的 `output_schema` 驗證（若有定義）。

### resources 綁定

將 artifact 綁定到 task，指定存取權限：

```yaml
resources:
  - name: order_file # handler 內部使用的名稱
    type: artifact
    ref: order_file # 對應 artifacts 區塊中的 key
    access: read # read | write | read_write
```

詳見 [22-artifacts](../05-runtime/22-artifacts.md)。

### Execution Policy

定義 task 在 crash recovery 時的重跑策略。Policy 定義在 **workflow step** 中，而非 task definition 中。

| Policy           | 說明                                      | Recovery 行為   |
| ---------------- | ----------------------------------------- | --------------- |
| `replayable`     | 可安全重跑，無副作用或有補償邏輯          | 重跑            |
| `idempotent`     | 可重跑，使用 idempotency key 確保結果一致 | 帶 key 重跑     |
| `non_repeatable` | 不可重跑（如扣款、發送通知）              | 直接標記 FAILED |

未指定 `execution.policy` 時，預設為 `replayable`。

### Retry 機制

| 欄位           | 型別     | 預設  | 說明                                       |
| -------------- | -------- | ----- | ------------------------------------------ |
| `max_attempts` | integer  | 1     | 總嘗試次數（含首次），1 = 不重試           |
| `delay`        | duration | 1s    | 重試間隔                                   |
| `backoff`      | string   | fixed | `fixed`: 固定間隔；`exponential`: 指數遞增 |

- 僅在 step FAILED 時重試，TIMED_OUT 不觸發重試
- 重試次數用盡仍失敗 → 觸發 `on_error`（若有），否則 step 最終狀態為 FAILED
- `exponential` backoff：delay × 2^(attempt - 1)

### 範例

```yaml
# 基本呼叫
- id: load_order
  type: task
  action: order.load
  input:
    order_id: ${ input.order_id }

# 帶 retry、timeout、compensate
- id: create_payment
  type: task
  action: payment.create
  version: 3
  input:
    order_id: ${ steps.load_order.output.id }
    amount: ${ steps.load_order.output.amount }
  execution:
    policy: non_repeatable
  retry:
    max_attempts: 3
    delay: 2s
    backoff: exponential
  timeout: 30s
  compensate:
    - type: task
      action: payment.refund
      input:
        payment_id: ${ steps.create_payment.output.payment_id }
  on_error:
    - type: emit
      event: payment.failed
      data:
        error: ${ error.message }
```

---

## Part B — Task Definition（`kind: Task`）

Task definition 是獨立的 YAML 文件，定義一個可被 workflow step 呼叫的工作單元。

### 設計動機

- **定義與使用分離**：workflow 描述「呼叫什麼」，task definition 描述「怎麼執行」
- **多 backend 支援**：同一個 task 介面可切換不同的執行方式
- **獨立版本管理**：task 可獨立演進，不影響 workflow definition
- **可重用性**：多個 workflow 可共用同一個 task definition

### 頂層結構

```yaml
apiVersion: task/v3
kind: Task

metadata:
  name: string # MUST — 唯一識別名稱（namespace.action 兩段式命名）
  version: integer # MUST — 版本號，正整數
  description: string # MAY
  labels: map # MAY

input_schema: object # MAY — 輸入 JSON Schema
output_schema: object # MAY — 輸出 JSON Schema

backend:
  type: string # MUST — stdio | http | builtin
  # ... backend 專屬設定
```

### Backend Types

#### stdio

Spawn 子 process，透過 stdin/stdout 交換結構化 JSON 訊息。Tool 可以用任何程式語言實作。

```yaml
backend:
  type: stdio
  command: string # MUST — 啟動 tool process 的命令
  shell: string # MAY — shell 路徑，預設 /bin/sh
  env: map # MAY — 額外環境變數（值支援 ${ } CEL 表達式）
  working_dir: string # MAY — 工作目錄
  config: map # MAY — 傳入 tool 的靜態設定（透過 request JSON）
  timeout_kill_after: duration # MAY — SIGTERM 後等待多久發 SIGKILL，預設 5s
```

**Request**（引擎 → tool）：

```json
{
  "input": {},
  "config": {},
  "context": {
    "idempotency_key": "string | null",
    "instance_id": "string",
    "step_id": "string",
    "attempt": 1
  },
  "artifacts": {}
}
```

**Response**（tool → 引擎）：

```json
{
  "success": true,
  "output": {},
  "error": null
}
```

- Process exit code 0 且 stdout 為有效 response JSON → 依 `success` 欄位判定
- Process exit code ≠ 0 → 失敗
- Process timeout → 先發 SIGTERM，等待 `timeout_kill_after`，再發 SIGKILL

**大小限制**：Request/Response JSON 上限 10 MB，stderr 記錄上限 64 KB。

#### http

呼叫 HTTP/HTTPS endpoint。

```yaml
backend:
  type: http
  url: string # MUST — 完整 URL（支援 ${ } 表達式）
  method: string # MAY — HTTP method，預設 POST
  headers: map # MAY — HTTP headers
  body_mapping: string # MAY — full | none，預設 full
  timeout: duration # MAY — HTTP 請求 timeout
  retry_on_status: array # MAY — 可重試的 HTTP status code（如 [429, 502, 503]）
  response_mapping: # MAY — response 到 output 的映射
    body: string
    status_code: string
```

- Task input 預設以 JSON body 傳送（`Content-Type: application/json`）
- Response body MUST 為 JSON
- HTTP 2xx = 成功，其他 = 失敗
- HTTP backend **不支援** artifact resource binding

#### builtin

呼叫引擎內建的 action。

```yaml
backend:
  type: builtin
  handler: string # MUST — 內建 handler 識別字
  config: map # MAY — handler 專屬設定
```

### Task Definition 範例

```yaml
apiVersion: task/v3
kind: Task

metadata:
  name: order.load
  version: 3
  description: "從資料庫載入訂單"

input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

output_schema:
  type: object
  properties:
    id: { type: string }
    amount: { type: number }
    status: { type: string }

backend:
  type: stdio
  command: "node ./tools/order-tasks/load-order.js"
  config:
    db_pool: "primary"
```

### Task Definition Lifecycle

與 workflow definition 相同：`DRAFT → VALIDATED → PUBLISHED → DEPRECATED → ARCHIVED`

僅 PUBLISHED 狀態的 task 可被 workflow step 呼叫。
