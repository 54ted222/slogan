# 05 — Tool Definition（`kind: Tool`）

本文件定義 `kind: Tool` 的 YAML 結構（v3 的 `kind: Task`）。Tool definition 描述一個可被 workflow step 呼叫的工具及其執行後端。

---

## 設計動機

- **定義與使用分離**：workflow 描述「呼叫什麼」（`type: task` step），tool definition 描述「怎麼執行」
- **多 backend 支援**：同一個 tool 介面可切換不同的執行方式
- **統一命名**：兩段式 `namespace.action` 命名，同時作為 agent 的 function call 名稱

---

## 頂層結構

```yaml
apiVersion: tool/v4
kind: Tool

metadata:
  name: order.load              # MUST — snake_case + dotted 命名
  version: 3                    # MUST
  description: "從資料庫載入訂單" # MAY

input_schema: object            # MAY — 輸入 JSON Schema
output_schema: object           # MAY — 輸出 JSON Schema

idempotent: bool                # MAY, 預設 false — 是否為冪等操作

compensate:                     # MAY — 預設補償操作
  action: string                #   MUST — 補償用的 tool name
  input: map                    #   MAY — 補償輸入（支援 ${ }，可引用原 step 的 output）

backend:
  type: stdio | http | builtin  # MUST
  # ... backend 專屬設定
```

---

## Backend Types

### stdio

Spawn 子 process，透過 stdin/stdout 交換 JSON。

```yaml
backend:
  type: stdio
  command: "node ./tools/order/load.js"   # MUST — 啟動命令
  shell: "/bin/sh"                         # MAY
  env:                                     # MAY — 環境變數（支援 ${ }）
    DB_URL: ${ secret.DATABASE_URL }
  working_dir: "./tools"                   # MAY
  config:                                  # MAY — 傳入 tool 的靜態設定
    db_pool: "primary"
```

Request（引擎 → tool）：

```json
{
  "input": { ... },
  "config": { ... },
  "context": {
    "idempotency_key": "string | null",
    "instance_id": "string",
    "step_id": "string",
    "attempt": 1
  }
}
```

Response（tool → 引擎）：

```json
{
  "success": true,
  "output": { ... },
  "error": null
}
```

### http

呼叫 HTTP endpoint。採用預設 OpenAPI 協議：request body = `input_schema`，response body = `output_schema`。

```yaml
backend:
  type: http
  url: "https://api.payment.example.com/v1/charges"  # MUST
  method: POST                    # MAY, 預設 POST
  headers:                        # MAY
    Authorization: "Bearer ${ secret.PAYMENT_API_KEY }"
  timeout: 10s                    # MAY
  retry_on_status: [429, 502, 503]  # MAY
```

### builtin

引擎內建 handler。

```yaml
backend:
  type: builtin
  handler: echo                   # MUST — handler 識別字
  config: {}                      # MAY
```

---

## 在 Workflow 中使用

Tool definition 透過 `type: task` step 呼叫（`action` 對應 `metadata.name`）：

```yaml
# Tool Definition
apiVersion: tool/v4
kind: Tool
metadata:
  name: order.load
  version: 1
input_schema:
  type: object
  properties:
    order_id: { type: string }
  required: [order_id]
backend:
  type: stdio
  command: "node ./tools/order/load.js"

---

# Workflow Step（使用端）
- id: load_order
  type: task
  action: order.load           # 參照 tool definition name
  input:
    order_id: ${ input.order_id }
```

---

## 在 Agent 中使用

Tool definition 也可作為 agent 的 tool（自動轉為 function call schema）：

```yaml
# Agent Definition 的 tools
tools:
  - type: task
    action: order.load
  - type: task
    action: order.update
```

LLM 看到的 function call：name = `order.load`，parameters = `input_schema`，description = `metadata.description`。

---

## 完整範例

```yaml
apiVersion: tool/v4
kind: Tool

metadata:
  name: payment.create
  version: 2
  description: "建立付款請求"

input_schema:
  type: object
  properties:
    order_id:
      type: string
    amount:
      type: number
    currency:
      type: string
      default: "TWD"
  required: [order_id, amount]

output_schema:
  type: object
  properties:
    payment_id:
      type: string
    status:
      type: string

compensate:
  action: payment.refund
  input:
    payment_id: ${ output.payment_id }

backend:
  type: http
  url: "https://api.payment.example.com/v1/charges"
  method: POST
  headers:
    Authorization: "Bearer ${ secret.PAYMENT_API_KEY }"
  timeout: 10s
  retry_on_status: [429, 502, 503]
```

## todo
type: stdio
要加入一個 text 的概念 可以使用 CEL  
text將會取代預設的json格式，改以純字串傳給 command, 
可以使用 CEL ，且有 backend 變數空間 可以拿 context


tools 返回
也要可以定義 輸出格式
也可以 mapping 輸出格式，預設是一個字串格式輸出

探討支援持續性輸出的可能性
stdio command 只接受字串輸出入，json 會被轉成string 傳輸
