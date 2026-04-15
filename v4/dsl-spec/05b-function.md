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

callback:                       # MAY — 宣告 function 對外發出的具名 callback
  some_name:
    input_schema: object        # MAY — 傳給 caller handler 的資料 schema
    output_schema: object       # MAY — caller handler 回傳的資料 schema

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

## Callback

Callback 提供 function 與 caller 之間的「協同互動」機制：function 在執行過程中暫停，將控制權交還 caller 執行使用端定義的 handler，handler 回傳結果後 function 從原處繼續執行。

適用場景：

- **使用端決策注入**：function 封裝主流程，但需要 caller 在特定時機提供決策（如風險審核、人工確認）
- **回呼資料補充**：function 需要 caller 補上額外資料才能繼續（如付款驗證碼、簽名）
- **與外部互動委派**：function 不直接與外部系統互動，由 caller 決定如何處理（測試替身、不同環境）

### 宣告方式

Function 在頂層 `callback` 區塊宣告對外提供的具名 callback 與其 schema：

```yaml
callback:
  some_name:
    input_schema:
      type: object
      properties:
        payment_id: { type: string }
    output_schema:
      type: object
      properties:
        approved: { type: boolean }
```

宣告為契約：caller 必須為每個 callback 提供同名 handler，否則 function 載入時驗證失敗。

### Function 內：`type: callback` step

Function 中以 `type: callback` step 觸發 callback：

```yaml
- type: callback
  name: some_name              # MUST — 對應 callback 宣告的 key
  input:                       # MAY — 傳給 caller handler 的資料（須符合 input_schema）
    payment_id: ${ steps.create_payment.output.payment_id }
  timeout: 10m                 # MAY — 等待 handler 完成的最大時間
  catch: [...]                 # MAY — 與其他 step 相同的錯誤處理
```

執行行為：

| 階段 | 行為 |
|------|------|
| 進入 step | function instance 暫停，發出 callback 事件給 caller |
| Handler 執行 | caller 執行對應 handler 的 steps（共享 caller 的 namespace） |
| Handler `return` | function instance 恢復，handler output 成為此 step 的 output |
| Handler FAILED | 此 step 進入 FAILED，可被 `catch` 捕捉 |
| 超時 | step FAILED，`error.type == "timeout"` |

`name` 同時作為此 step 的識別子；後續 step 以 `steps.<name>.output` 或 `prev.<name>.output` 取得 handler 回傳值（callback step 不另外宣告 `id`）。

### Caller 端：`callback` 區塊

呼叫端在 `type: task` step 上以 `callback:` map 提供 handler。Map 的 key 對應 function 宣告的 callback 名稱，value 為 step 序列：

```yaml
- type: task
  action: payment.process       # 此 function 宣告了 callback: some_name
  input:
    order_id: ${ input.order_id }
  callback:
    some_name:
      - type: task
        action: buildin.echo
        message: "callback received: ${ callback.input }"
      - type: return
        output:
          approved: true
```

Handler 的求值與資料規則：

| Namespace | 說明 |
|-----------|------|
| `callback.input` | function 端 `type: callback` step 傳入的 input |
| `callback.name` | 觸發的 callback 名稱（同一 task 多種 callback 共用 handler 時可分流） |
| `input` / `steps` / `vars` | caller 自身的 namespace（handler 與 caller 共享） |

Handler 必須以 `type: return` 結束並回傳符合 `output_schema` 的資料；該 output 將作為 function 端 `type: callback` step 的 output。

### 與 Tool callback 的對應

`type: task` 呼叫 Tool 時亦可使用同一 `callback:` 結構；Tool 透過 backend 協議（exec protocol / http stream / extension）發出 callback 請求，引擎將之路由至同一 handler。Tool 端的協議定義另見 `05-tool.md`（待補）。

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
    catch:
      - type: fail
        when: ${ error.type == "timeout" }
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

### 帶 Callback 的 Function

Function 在建立付款後，將是否確認的決策權交還給 caller：

```yaml
apiVersion: function/v4
kind: Function

metadata:
  name: payment.process_with_review
  version: 1
  description: "建立付款並由 caller 決定是否確認"

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

callback:
  review:
    input_schema:
      type: object
      properties:
        payment_id: { type: string }
        amount: { type: number }
    output_schema:
      type: object
      properties:
        approved: { type: boolean }

steps:
  - id: create_payment
    type: task
    action: payment.create
    input:
      order_id: ${ input.order_id }
      amount: ${ input.amount }
    timeout: 30s

  - type: callback
    name: review
    input:
      payment_id: ${ steps.create_payment.output.payment_id }
      amount: ${ input.amount }
    timeout: 10m

  - type: task
    action: buildin.echo
    message: "review result: ${ prev.review.output.approved }"

  - type: return
    output:
      payment_id: ${ steps.create_payment.output.payment_id }
      status: ${ steps.review.output.approved ? "confirmed" : "rejected" }
```

### Caller 提供 Callback Handler

```yaml
steps:
  - id: pay
    type: task
    action: payment.process_with_review
    input:
      order_id: ${ input.order_id }
      amount: ${ input.amount }
    timeout: 15m
    callback:
      review:
        - id: risk_check
          type: task
          action: risk.evaluate
          input:
            payment_id: ${ callback.input.payment_id }
            amount: ${ callback.input.amount }
        - type: return
          output:
            approved: ${ steps.risk_check.output.level != "high" }
    catch:
      - type: fail
        message: ${ "payment failed: " + error.message }
```
