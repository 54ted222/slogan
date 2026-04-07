# 05b — Function Definition（`kind: Function`）

本文件定義 `kind: Function` 的 YAML 結構。Function 是以 steps 定義的可重用邏輯單元，擁有獨立的變數空間，透過 `type: task` step 呼叫。

---

## 設計動機

- **組合復用**：將常見子流程（如付款、通知）封裝為獨立定義
- **資料隔離**：擁有獨立的 `input`、`steps`、`vars` 空間，不可存取呼叫端的 namespace
- **統一呼叫**：與 Tool 共用 `type: task` 呼叫方式，對使用端透明

### Function vs Tool

| | Function | Tool |
|--|----------|------|
| 定義方式 | `steps`（DSL） | `backend`（外部程式） |
| 執行位置 | 引擎內部 | 外部 process / HTTP |
| 呼叫方式 | `type: task` | `type: task` |
| 變數空間 | 獨立 | — |

---

## 頂層結構

```yaml
apiVersion: function/v4
kind: Function

metadata:
  name: payment.process         # MUST — snake_case + dotted
  version: 1
  description: "處理付款流程"

input_schema: object            # MAY — 輸入 JSON Schema
output_schema: object           # MAY — 輸出 JSON Schema

steps:                          # MUST — 步驟序列
  - ...
```

---

## 呼叫方式

與 Tool 相同，透過 `type: task` step 呼叫：

```yaml
- id: process_payment
  type: task
  action: payment.process       # Function 的 metadata.name
  input:
    order_id: ${ steps.load_order.output.id }
    amount: ${ steps.load_order.output.amount }
  timeout: 5m
  catch: [...]
```

- `steps.<id>.output` = Function 的 `return` output
- Function FAILED → 呼叫端 step FAILED（可被 `catch` 捕捉）

---

## 資料隔離

Function 內部**不可**存取呼叫端的任何 namespace：

| Namespace | 呼叫端 → Function | Function → 呼叫端 |
|-----------|-------------------|-------------------|
| `input` | 透過 `input` 傳入 | — |
| `steps` | — | 透過 `return` output |
| `vars` | — | — |

---

## 完整範例

### Function Definition

```yaml
apiVersion: function/v4
kind: Function

metadata:
  name: payment.process
  version: 1
  description: "建立付款並等待確認"

input_schema:
  type: object
  properties:
    order_id: { type: string }
    amount: { type: number }
  required: [order_id, amount]

output_schema:
  type: object
  properties:
    payment_id: { type: string }
    status: { type: string }

steps:
  - id: create_payment
    type: task
    action: payment.create
    input:
      order_id: ${ input.order_id }
      amount: ${ input.amount }
    timeout: 30s

  - id: wait_confirmed
    type: wait
    event: payment.confirmed
    match: ${ event.data.payment_id == steps.create_payment.output.payment_id }
    timeout: 30m
    on_timeout:
      - type: fail
        message: "payment confirmation timeout"

  - type: return
    output:
      payment_id: ${ steps.create_payment.output.payment_id }
      status: "confirmed"
```

### 在 Workflow 中呼叫

```yaml
steps:
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  - id: pay
    type: task
    action: payment.process
    input:
      order_id: ${ steps.load_order.output.id }
      amount: ${ steps.load_order.output.amount }
    timeout: 35m
    catch:
      - type: fail
        message: ${ "payment failed: " + error.message }

  - type: return
    output:
      payment_id: ${ steps.pay.output.payment_id }
      status: "completed"
```
