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
apiVersion: function/v6
kind: Function

metadata:
  name: payment.process         # MUST — snake_case + dotted
  version: 1
  description: "處理付款流程"
  recursion_allowed: false      # MAY, 預設 false — 允許本 Function 呼叫自身或進入循環依賴

input_schema: object            # MAY — 輸入 JSON Schema
output_schema: object           # MAY — 輸出 JSON Schema

# 缺省語意：
# - input_schema 缺省 → function instance 建立時不對 input 驗證（任意結構皆接受）；
#   僅 engine 級別的 size limit（instance input ≤ 16 MB，見 08-persistence.md）仍適用
# - output_schema 缺省 → `type: return` 的 output 不驗證；
#   僅 size limit（instance output ≤ 16 MB）與一般 JSON 可序列化性適用
# - callback.*_schema 缺省 → 對應方向不驗證；見 callback 小節

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

## 遞迴呼叫（recursion_allowed）

預設 Function 不可**直接或間接**呼叫自身；載入期 `Tarjan SCC` 偵測出循環 → `registry.dependency_cycle`（見 `runtime-spec/05-task-registry.md`）。

若確有遞迴需要（如走訪樹狀結構），於 function `metadata` 顯式宣告：

```yaml
metadata:
  name: tree.walk
  version: 1
  recursion_allowed: true
```

- `recursion_allowed: true` 僅 **取消載入期的循環檢查**；運行期仍受 `engine.max_function_call_depth`（預設 128）上限保護，超過 → function instance FAILED，`error.type == "max_recursion_depth_exceeded"`
- 僅針對**宣告此 flag 的 function 自身**豁免；若循環經過未宣告 flag 的 function → 仍視為違規
- 多個 Function 互為循環時，**參與循環的每一個 Function** 皆 MUST 設此 flag；否則載入失敗

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

**Schema 宣告完整性**：

- `input_schema` 與 `output_schema` MAY 缺省；缺省時對應方向不做驗證
- 若任一存在，兩者 MUST 都有（語意完整）；僅宣告單邊 → 載入失敗，`error.type: "schema_incomplete"`
- 例外：若 callback handler 端以 `type: return` 無 output 結束，視為 output 為 `null`；output_schema 若不允許 null 則 FAILED（`error.type: "schema_violation"`）

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
| Handler FAILED | 此 step 進入 FAILED，`error` 反映 handler 內傳播至頂層的錯誤；可被 `catch` 捕捉 |
| 超時 | step FAILED，`error.type == "timeout"` |

`timeout` 涵蓋 handler 執行的**整體時長**（從發出 callback 事件算起至 handler 回傳 `return` 為止）。Handler 內各 step 的 `timeout` / `retry` 獨立計時，但不延長外層 callback 的 `timeout`；若 handler 內 step 失敗被其內部 `catch` 處理則不向外傳播，否則向上浮出到 handler 頂層 → 整個 callback step FAILED。Handler output 由引擎在 handler `return` 時依 callback 宣告的 `output_schema` 驗證；不符 → callback step FAILED，`error.type == "schema_violation"`、`error.details.direction == "output"`。

#### 多層 timeout 優先級

當同時存在 callback step `timeout`、function `config.timeout`、workflow `config.timeout` 時，**最短者先生效**（deadline 獨立計算、同時倒數；三者皆非顯式時無時間上限，但仍受上層繼承 — 見 `runtime-spec/02-instance-lifecycle.md` 的「Callback timeout 規則」）：

- Callback step `timeout` 先到 → 本 callback step FAILED（`error.type == "timeout"`），function 其他 step 可繼續（若有 catch）
- Function `config.timeout` 先到 → function instance FAILED，父 task step 以此 failure 呈現
- Workflow `config.timeout` 先到 → workflow FAILED，所有子 function instance 被 cancel

顯式 callback `timeout` 不會「擴張」其外層 function / workflow timeout；即使 callback 設 `timeout: 10m` 而 function 剩 3 分鐘，callback 實際可用時間仍為 3 分鐘。

Step 識別規則：

- 預設以 `name` 作為 step 的識別子（相當於 `id: <name>`），後續 step 以 `steps.<name>.output` 取得 handler 回傳值。
- 若同一 function 內需多次呼叫同一 callback（如 `foreach` 內），MUST 顯式指定 `id` 以避免 ID 衝突；此時 `steps.<id>.output` 生效。
- 緊接的下一步可用 `prev.output`（無需帶名稱）。

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
        action: builtin.echo
        input:
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

#### 多個 callback 時的分流

`callback:` 是 map，以 callback 名稱為 key，每個 key 對應獨立的 handler steps。不同 callback 名稱之間互相隔離，共用 caller 的 `input` / `steps` / `vars`：

```yaml
- type: task
  action: order.flow_with_review       # function 宣告 callback: [review_risk, ask_amount]
  input: { order_id: ${ input.order_id } }
  callback:
    review_risk:
      - id: risk
        type: task
        action: risk.evaluate
        input: { payment_id: ${ callback.input.payment_id } }
      - type: return
        output: { approved: ${ steps.risk.output.level != "high" } }
```

每個 handler 獨立執行；同一 task 內不同 callback 若並行觸發，引擎對 caller namespace 做 snapshot 以避免 race。

### 與 Tool callback 的對應

`type: task` 呼叫 Tool 時亦可使用同一 `callback:` 結構；Tool 透過 backend 協議（exec protocol / http stream / extension）發出 callback 請求，引擎將之路由至同一 handler。Tool 端的協議定義見 `05-tool.md` 的「Callback 協議」章節。

---

## 完整範例

### Function Definition

```yaml
apiVersion: function/v6
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
    signals:
      - event: payment.confirmed
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
apiVersion: function/v6
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
    action: builtin.echo
    input:
      message: "review result: ${ prev.output.approved }"

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
