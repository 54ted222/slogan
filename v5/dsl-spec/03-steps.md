# 03 — Steps

本文件定義所有 step 類型的語法。

---

## Step 類型一覽

| 類型       | 說明                    |
| ---------- | ----------------------- |
| `task`     | 呼叫 tool definition    |
| `assign`   | 設定 vars 變數          |
| `if`       | 條件分支                |
| `switch`   | 多值分支                |
| `foreach`  | 迴圈迭代                |
| `parallel` | 平行分支                |
| `emit`     | 發送事件                |
| `wait`     | 等待事件或指定時間      |
| `fail`     | 中止 workflow 並報錯    |
| `return`   | 回傳結果並結束 workflow |
| `agent`    | 呼叫 agent definition   |
| `saga`     | 補償區塊                |

---

## 共通屬性

| 屬性          | 型別           | 必填 | 說明                         |
| ------------- | -------------- | ---- | ---------------------------- |
| `id`          | string         | MAY  | 唯一識別字（`snake_case`）   |
| `type`        | string         | MUST | step 類型                    |
| `description` | string         | MAY  | 人類可讀說明                 |
| `when`        | CEL expression | MAY  | 前置條件。求值結果為 `false` → SKIPPED；求值異常（型別錯誤、引用不存在的 step 等）→ FAILED（由外層 `catch` / saga / workflow catch 處理） |

部分 step 類型額外支援 `timeout`、`retry`、`catch`。

### id 規則

- 非緊鄰的後續 step 需要參照 output 時 MUST 提供 `id`
- 緊接的下一步可透過 `prev` 存取前一步輸出，不需要 `id`
- 省略時引擎自動產生 `_<type>_<index>` 格式的內部 ID
  - `<type>` 為 step 的 `type` 字串原值（如 `task`、`foreach`、`wait`）
  - `<index>` 為該 step 於所在 step 陣列中的 0-based 索引
  - 例：第一個未命名 `type: task` → `_task_0`；同陣列第三個未命名 `type: foreach` → `_foreach_2`
- 使用者定義的 ID MUST NOT 以 `_` 開頭（避開引擎命名空間），MUST 全域唯一

### retry

```yaml
retry:
  max_attempts: 3 # 總嘗試次數（含首次），預設 1
  delay: 2s # 重試間隔，預設 1s
  backoff: exponential # fixed | exponential，預設 fixed
```

---

## task

呼叫一個 tool definition（v3 稱 task definition）。

```yaml
- id: load_order
  type: task
  action: order.load # MUST — tool definition 的 name
  input: # MAY — 傳給 tool 的輸入
    order_id: ${ input.order_id }
  retry: { ... }
  timeout: 30s
  catch: [...]
```

Output 透過 `steps.<id>.output` 或 `prev.output` 存取。

---

## assign

設定 `vars` namespace 中的變數。

```yaml
- type: assign
  vars:
    is_pay: ${ input.action == "pay" }
    payment_input:
      order_id: ${ steps.load_order.output.id }
      amount: ${ steps.load_order.output.amount }
      currency: "USD"
```

- 後續 steps 透過 `vars.<key>` 存取
- 同名變數會被後續 `assign` 覆寫

---

## if

```yaml
- type: if
  when: ${ steps.load_order.output.status == "cancelled" }
  then:
    - type: fail
      message: "order cancelled"
  else:
    - type: emit
      event: order.processing
```

- `when` 為 `true` → 執行 `then`；為 `false` → 執行 `else`（若有）
- Output 為被選中分支最後一個 step 的 output

---

## switch

```yaml
- type: switch
  when: ${ input.action }
  cases:
    - value: "pay"
      then:
        - type: task
          action: payment.process
          input: { order_id: ${ input.order_id } }
    - value: "ship"
      then:
        - type: task
          action: shipment.create
          input: { order_id: ${ input.order_id } }
    - value: ${ input.some_value == "cancel" }
      then:
        - type: task
          action: order.cancel
          input: { order_id: ${ input.order_id } }
  default:
    - type: fail
      message: ${ "unknown action: " + input.action }
```

- 匹配第一個符合的 case，無匹配且有 `default` → 執行 default
- `value` 可為字面值或 CEL 表達式

---

## foreach

```yaml
- id: reserve_items
  type: foreach
  items: ${ input.items } # MUST（與 count 擇一）— 回傳 list 的 CEL 表達式
  count: ${ config.max_iterations } # MUST（與 items 擇一）— 整數或回傳整數的 CEL
  concurrency: 3 # MAY, 預設 1
  async: true # MAY, 預設 false — 非阻塞模式，不等待完成即繼續下一步
  failure_policy: fail_fast # MAY — fail_fast | continue | ignore, 預設 fail_fast
  do:
    - type: task
      action: inventory.reserve
      input:
        sku: ${ loop.item.sku }
        qty: ${ loop.item.qty }
```

迭代變數：`loop.item`（當前元素）、`loop.index`（索引，從 0 開始）。

- `items` 與 `count` MUST 擇一。`count: N` 等價於 `items: range(N)`，此時 `loop.item == loop.index`。
- `count` 的值 MUST 為非負整數；負值或非整數 → step FAILED，`error.type == "invalid_count"`。
- `concurrency`：同時執行的迭代上限；`async` 不影響此值。
- `async: false`（預設）：阻塞模式，等待所有迭代完成後才繼續；engine 依 `concurrency` 並行調度。Output 為一個 array，索引與 items 一一對應。
- `async: true`：非阻塞模式，foreach 啟動後立即視為「進入 RUNNING」並繼續下一步；engine 在背景依 `concurrency` 持續調度迭代直到完成。須搭配 `type: wait` 的 `step` 模式取得最終結果。
- 巢狀 `foreach` 時，內層 `loop.item` / `loop.index` 遮蔽外層；需要外層時改用具名 step 的 output 或先 `assign` 保留。
- `failure_policy`：
  - `fail_fast`（預設）：任一迭代 FAILED → 取消尚未開始的迭代，已啟動的迭代繼續執行至終態；foreach FAILED，`error.step_id` 為首個失敗者。
  - `continue`：迭代失敗不中止迴圈；所有迭代完成後若有任一失敗 → foreach FAILED，`error.failed_indices` 為失敗索引清單；output 包含成功迭代的結果（失敗位置為 `null`）。
  - `ignore`：迭代失敗視為 `null` output；foreach 永遠 SUCCEEDED。
- 迭代內部 step 的 `catch` 在 `failure_policy` 之前求值；被 catch 處理的失敗不視為迭代失敗。

---

## parallel

```yaml
- id: finalize
  type: parallel
  async: true                   # MAY, 預設 false — 非阻塞模式，不等待完成即繼續下一步
  failure_policy: continue       # MAY — fail_fast | continue | ignore, 預設 fail_fast
  branches:
    - steps:
        - type: task
          action: notify.customer
          input: { order_id: ${ input.order_id } }
    - steps:
        - type: task
          action: report.generate
          input: { order_id: ${ input.order_id } }
```

- `async: false`（預設）：阻塞模式，等待所有 branches 完成。Output 為一個 array，索引對應 branches 順序。
- `async: true`：非阻塞模式，啟動後立即繼續下一步。須搭配 `type: wait` 的 `step` 模式取得結果。

---

## emit

發送事件至 event bus（fire-and-forget）。

```yaml
- type: emit
  event: order.completed # MUST — 事件類型
  scope: project # MAY — workflow | project | global, 預設 workflow
  data: # MAY — 事件酬載
    order_id: ${ steps.load_order.output.id }
  delay: 30m # MAY — 延遲發送
```

`scope` 控制事件傳播範圍：`workflow`（僅當前 instance）、`project`（同 project）、`global`（跨 project）。

事件透過 event bus 路由，可觸發其他 workflow 的 `event` trigger（建立新 instance）或恢復其他 instance 的 `wait` step。無匹配接收者時事件被丟棄，不產生錯誤。

---

## wait

暫停 workflow instance，等待事件或指定時間。三種模式互斥：單一事件、多事件、duration。

### 單一事件

```yaml
- id: wait_payment
  type: wait
  event: payment.confirmed
  match: ${ event.data.order_id == input.order_id }
  timeout: 30m
  catch:
    - type: fail
      when: ${ error.type == "timeout" }
      message: "payment timeout"
```

`steps.<id>.output` = `event.data`。

### 多事件（any-of）

同時等待多個不同事件，**任一**匹配即恢復。

```yaml
- id: wait_result
  type: wait
  events: # MUST — 事件陣列
    - event: payment.confirmed
      match: ${ event.data.order_id == input.order_id }
    - event: payment.failed
      match: ${ event.data.order_id == input.order_id }
    - event: order.cancelled
      match: ${ event.data.order_id == input.order_id }
  timeout: 30m
  catch:
    - type: fail
      when: ${ error.type == "timeout" }
      message: "等待逾時"
```

Output：

| 欄位                      | 說明                                       |
| ------------------------- | ------------------------------------------ |
| `steps.<id>.output.event` | 匹配到的事件類型（如 `payment.confirmed`） |
| `steps.<id>.output.data`  | 該事件的 `event.data`                      |

```yaml
# 根據匹配到的事件分支處理
- type: switch
  when: ${ steps.wait_result.output.event }
  cases:
    - value: "payment.confirmed"
      then:
        - type: emit
          event: order.processing
    - value: "payment.failed"
      then:
        - type: fail
          message: "付款失敗"
    - value: "order.cancelled"
      then:
        - type: fail
          message: "訂單已取消"
```

### Duration 模式

```yaml
- type: wait
  duration: 5m
```

### Step 模式

等待一個 `async: true` 的非阻塞步驟（`foreach` 或 `parallel`）完成，並取得其結果。

```yaml
- id: reserve_items
  type: foreach
  async: true
  items: ${ input.items }
  concurrency: 5
  do:
    - type: task
      action: inventory.reserve
      input:
        sku: ${ loop.item.sku }
        qty: ${ loop.item.qty }

# 中間可穿插其他不依賴 reserve_items 結果的步驟
- type: task
  action: analytics.track
  input: { event: "reserve_started" }

# 阻塞等待 foreach 完成
- id: wait_reserve
  type: wait
  step: reserve_items # MUST — 要等待的非阻塞步驟 ID，可為字串或字串陣列
  timeout: 5m
  catch:
    - type: fail
      when: ${ error.type == "timeout" }
      message: "reserve timeout"
```

`step` 為字串陣列時：等待**所有**指定 step 完成才恢復；output 為 map `{<step_id>: <output>, ...}`。
`step` 為單一字串時：output 即該 step 的 output。

所等待的 async step 終態決定 wait step 結果：

| async step 終態 | wait step 行為 |
|-----------------|----------------|
| SUCCEEDED | wait SUCCEEDED，output 為該 step 的 output |
| FAILED | wait FAILED，`error.type == "async_step_failed"`、`error.step_id` 為該 step 的 id；`error.cause` 為該 step 的原 error 物件；output 為 null（由 `catch` 存取 `error.*`） |
| SKIPPED | wait SKIPPED |

陣列模式下任一子 step FAILED → wait FAILED，`error.step_id` 為首個失敗者；其餘 step 仍執行至終態（不取消）。

### 模式判斷規則

- `event`、`events`、`duration`、`step` 四者 MUST 擇一，不可同時存在

---

## fail

立即終止 workflow instance，標記為 FAILED。

```yaml
- type: fail
  message: "order has been cancelled" # MUST
  code: "order_cancelled" # MAY
```

在 `catch` handler 中使用時，視為錯誤重新拋出至上層。

---

## return

結束 workflow instance，標記為 SUCCEEDED。

```yaml
- type: return
  output: # MAY
    status: "completed"
```

---

## agent

呼叫 agent definition。詳見 [06-agent](06-agent.md)。

```yaml
- id: analyze
  type: agent
  agent: order.analyzer # MUST — agent definition name
  system: "你是訂單風險等級專家。" # MAY — 附加於 definition system 之後的系統提示
  prompt: "分析此訂單" # MAY — 附加於 definition prompt 之後的任務提示
  input:
    order: ${ steps.load_order.output }
  tools: # MAY — 疊加至 definition tools 的額外工具（字串簡寫或 {action, name?, description?, input?}）
    - customer.get_history
  timeout: 2m
  catch: [...]
```

---

## saga

包裹一組需要交易性補償的 steps。

```yaml
- type: saga
  steps:
    - id: debit
      type: task
      action: payment.debit
      input: { amount: ${ input.amount } }

    - id: create_order
      type: task
      action: order.create
      input: { payment_ref: ${ steps.debit.output.transaction_id } }

    - type: task
      action: notification.send
      input: { message: "訂單已建立" }

  catch:
    - type: fail
      message: "交易已回滾"
```

### compensate 來源

每個 `type: task` step 的補償操作來自 Tool definition 的 `compensate` 欄位（見 [05-tool](05-tool.md)）。Step 層級可覆寫：

```yaml
- type: saga
  steps:
    - id: debit
      type: task
      action: payment.debit
      input: { amount: ${ input.amount } }
      compensate:                  # 覆寫 Tool definition 的預設 compensate
        action: payment.void
        input: { transaction_id: ${ steps.debit.output.transaction_id } }
```

### 執行流程

1. 依序執行 `steps`
2. 任一 step FAILED → 反向執行已 SUCCEEDED steps 的 compensate
3. 補償完成 → 進入 `catch`

### 補償規則

- 僅 SUCCEEDED 的 step 會執行其 compensate
- 反向執行：以「step 進入 SUCCEEDED 的時間戳」降序補償
- 序列內的 step 遵循「後成功者先補償」
- 若 saga 內含 `parallel` / `foreach`：
  - 不同分支 / 不同迭代間**無全序保證**，補償可能並行進行
  - 同一分支或同一迭代內部仍遵循時間戳降序
  - 分支間有隱含依賴時，補償順序由 tool 作者自行以 idempotent 設計處理；引擎不提供依賴宣告
- 無 `compensate` 的 step（Tool 未定義且 step 未覆寫）跳過

#### 補償失敗處理

個別 compensate action 失敗時，引擎 **繼續執行其他 compensate**（best-effort），不中止補償鏈：

- 補償 step 應使用 `retry` 處理暫時錯誤；retry 用盡後仍失敗則記錄，但不阻斷其他補償。
- 所有補償完成後（含失敗），進入 saga `catch`；`error.compensation_failures` 為失敗清單：

  ```
  error.compensation_failures: [
    { step_id: "debit", error: { type, message, code } },
    ...
  ]
  ```

- 若 saga `catch` 為空，workflow 以 `error.type == "saga_failed"` 向上傳播。

---

## 錯誤處理模型

三層 handler 由內而外：

1. **Step-level `catch`** — 單一 step 失敗時處理
2. **Saga compensate** — saga 範圍內反向補償
3. **Workflow-level `config.catch`** — 未被處理的錯誤最終到達此處

`catch` handler 內可用的 namespace：

| 變數            | 說明                             |
| --------------- | -------------------------------- |
| `error.type`    | 錯誤類型（`timeout` 等） buildin |
| `error.message` | 錯誤訊息                         |
| `error.code`    | 錯誤碼 業務客製化                |
| `error.step_id` | 發生錯誤的 step id               |
