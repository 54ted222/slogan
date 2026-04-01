# 11 — Step: sub_workflow

`sub_workflow` 步驟呼叫另一個 workflow definition，建立子 instance 並等待其完成。

---

## 設計動機

- **組合復用**：將常見的子流程（如付款、通知）封裝為獨立的 workflow definition
- **關注分離**：子流程有獨立的 input/output schema、lifecycle、版本
- **獨立測試**：子 workflow 可獨立執行與測試

---

## Schema

```yaml
- id: string                          # MAY — 需要參照 output 時才必填
  type: sub_workflow                   # MUST
  workflow: string                     # MUST — 要呼叫的 workflow definition name
  version: integer | "latest"         # MAY, 預設 "latest"
  input: map | CEL expression         # MAY — 傳給子 workflow 的輸入
  timeout: duration                   # MAY
  execution_policy: replayable | idempotent | non_repeatable  # MAY, 預設 replayable
  retry:
    max_attempts: integer             # MAY
    delay: duration                   # MAY
    backoff: fixed | exponential      # MAY
  on_error: [...]                     # MAY, step 陣列
  on_timeout: [...]                   # MAY, step 陣列
```

---

## 執行語意

1. Parent step 進入 RUNNING 狀態
2. 引擎建立 **child workflow instance**：
   - Definition = `workflow` 指定的 workflow definition
   - Version = `version` 指定的版本
   - Input = `input` 求值的結果
3. Child instance 獨立執行（擁有自己的 step instances、state）
4. Parent step 等待 child instance 完成
5. Child SUCCEEDED → parent step SUCCEEDED，`steps.<id>.output` = child 的 return output
6. Child FAILED → parent step FAILED（可被 `on_error` 捕捉）

---

## 資料隔離

Child workflow **不可** 存取 parent 的任何 namespace：

| Namespace | Parent → Child | Child → Parent |
|-----------|---------------|----------------|
| `input` | 透過 `input` 傳入 ✅ | — |
| `steps` | ❌ | 透過 return output ✅ |
| `vars` | ❌ | ❌ |
| `loop` | ❌ | ❌ |
| `artifacts` | 透過 input 映射 ✅ | — |

這是硬性邊界，確保子 workflow 的可組合性與獨立性。

---

## 版本解析

| 值 | 說明 |
|----|------|
| `"latest"` | 執行時解析為最新的 PUBLISHED 版本 |
| `integer` | 固定版本號 |

- 預設為 `"latest"`
- 在正式環境中 SHOULD 使用固定版本號，確保可重現性
- 若指定版本不存在或非 PUBLISHED 狀態 → step FAILED

---

## 巢狀深度限制

- 子 workflow MAY 再呼叫 sub_workflow（遞迴 / 間接遞迴）
- 引擎 MUST 設定最大巢狀深度限制（建議上限 10 層）
- 超過深度限制 → step FAILED（錯誤碼 `max_depth_exceeded`）

---

## 錯誤傳播

Child workflow 的失敗資訊可在 parent 的 `on_error` handler 中存取：

| 變數 | 說明 |
|------|------|
| `error.message` | child workflow 的 fail message |
| `error.code` | child workflow 的 fail code |
| `error.step_id` | parent 中的 sub_workflow step id |

---

## Timeout 行為

- Parent step 的 `timeout` 是 sub_workflow 的最長執行時間
- 若 parent step timeout 觸發 → child instance 被 CANCELLED
- Child 的 workflow 級 `on_timeout` 不會被觸發（因為是被外部取消，非自身 timeout）
- Parent 的 `on_timeout` handler 觸發

---

## 範例

### 基本呼叫

```yaml
- id: process_payment
  type: sub_workflow
  workflow: payment_processing
  version: 3
  input:
    order_id: ${ steps.load_order.output.id }
    amount: ${ steps.load_order.output.amount }
  timeout: 5m
```

### 帶錯誤處理

```yaml
- id: process_payment
  type: sub_workflow
  workflow: payment_processing
  input:
    order_id: ${ steps.load_order.output.id }
    amount: ${ steps.load_order.output.amount }
  timeout: 5m
  execution_policy: non_repeatable
  on_error:
    - type: emit
      event: payment.failed
      data:
        order_id: ${ steps.load_order.output.id }
        error: ${ error.message }
    - type: fail
      message: ${ "payment failed: " + error.message }
  on_timeout:
    - type: fail
      message: "payment sub-workflow timeout"
```

### 被呼叫的子 workflow

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: payment_processing
  version: 3

triggers:
  - type: manual

input:
  schema:
    type: object
    properties:
      order_id:
        type: string
      amount:
        type: number
    required: [order_id, amount]

output:
  schema:
    type: object
    properties:
      payment_id:
        type: string
      status:
        type: string
    required: [payment_id, status]

steps:
  - id: create_payment
    type: task
    action: payment.create
    input:
      order_id: ${ input.order_id }
      amount: ${ input.amount }
    execution_policy: non_repeatable
    timeout: 30s

  - id: wait_confirmed
    type: wait_event
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
