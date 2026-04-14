# 03 — Steps

本文件定義所有 step 類型的語法。

---

## Step 類型一覽

| 類型 | 說明 |
|------|------|
| `task` | 呼叫 tool definition |
| `assign` | 設定 vars 變數 |
| `if` | 條件分支 |
| `switch` | 多值分支 |
| `foreach` | 迴圈迭代 |
| `parallel` | 平行分支 |
| `emit` | 發送事件 |
| `wait` | 等待事件或指定時間 |
| `fail` | 中止 workflow 並報錯 |
| `return` | 回傳結果並結束 workflow |
| `agent` | 呼叫 agent definition |
| `saga` | 補償區塊 |

---

## 共通屬性

| 屬性 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `id` | string | MAY | 唯一識別字（`snake_case`） |
| `type` | string | MUST | step 類型 |
| `description` | string | MAY | 人類可讀說明 |
| `when` | CEL expression | MAY | 前置條件，`false` 時 SKIPPED |

部分 step 類型額外支援 `timeout`、`retry`、`catch`、`on_timeout`。

### id 規則

- 非緊鄰的後續 step 需要參照 output 時 MUST 提供 `id`
- 緊接的下一步可透過 `prev` 存取前一步輸出，不需要 `id`
- 省略時引擎自動產生 `_<type>_<index>` 格式的內部 ID
- 使用者定義的 ID MUST NOT 以 `_` 開頭，MUST 全域唯一

### retry

```yaml
retry:
  max_attempts: 3        # 總嘗試次數（含首次），預設 1
  delay: 2s              # 重試間隔，預設 1s
  backoff: exponential   # fixed | exponential，預設 fixed
```

---

## task

呼叫一個 tool definition（v3 稱 task definition）。

```yaml
- id: load_order
  type: task
  action: order.load          # MUST — tool definition 的 name
  input:                      # MAY — 傳給 tool 的輸入
    order_id: ${ input.order_id }
  retry: { ... }
  timeout: 30s
  catch: [...]
  on_timeout: [...]
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
  items: ${ input.items }       # MUST — 回傳 list 的 CEL 表達式
  concurrency: 3                # MAY, 預設 1
  async: true                   # MAY, 預設 false — 非阻塞模式，不等待完成即繼續下一步
  failure_policy: fail_fast     # MAY — fail_fast | continue | ignore, 預設 fail_fast
  do:
    - type: task
      action: inventory.reserve
      input:
        sku: ${ loop.item.sku }
        qty: ${ loop.item.qty }
```

迭代變數：`loop.item`（當前元素）、`loop.index`（索引，從 0 開始）。

- `async: false`（預設）：阻塞模式，等待所有迭代完成後才繼續。Output 為一個 array，索引與 items 一一對應。
- `async: true`：非阻塞模式，啟動後立即繼續下一步。須搭配 `type: wait` 的 `step` 模式取得結果。

---

## parallel

```yaml
- id: finalize
  type: parallel
  async: true                   # MAY, 預設 false — 非阻塞模式，不等待完成即繼續下一步
  failure_policy: wait_all      # MAY — fail_fast | wait_all
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
  event: order.completed        # MUST — 事件類型
  data:                         # MAY — 事件酬載
    order_id: ${ steps.load_order.output.id }
  delay: 30m                    # MAY — 延遲發送
```
### todo
emit 到 其他workflow 


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
  on_timeout:
    - type: fail
      message: "payment timeout"
```

`steps.<id>.output` = `event.data`。

### 多事件（any-of）

同時等待多個不同事件，**任一**匹配即恢復。

```yaml
- id: wait_result
  type: wait
  events:                        # MUST — 事件陣列
    - event: payment.confirmed
      match: ${ event.data.order_id == input.order_id }
    - event: payment.failed
      match: ${ event.data.order_id == input.order_id }
    - event: order.cancelled
      match: ${ event.data.order_id == input.order_id }
  timeout: 30m
  on_timeout:
    - type: fail
      message: "等待逾時"
```

Output：

| 欄位 | 說明 |
|------|------|
| `steps.<id>.output.event` | 匹配到的事件類型（如 `payment.confirmed`） |
| `steps.<id>.output.data` | 該事件的 `event.data` |

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
  step: reserve_items            # MUST — 非阻塞步驟的 id
  timeout: 5m
  on_timeout:
    - type: fail
      message: "reserve timeout"
```

`steps.wait_reserve.output` = 該非阻塞步驟的原始 output（foreach 為 array，parallel 為 array）。

### 模式判斷規則

- `event`、`events`、`duration`、`step` 四者 MUST 擇一，不可同時存在

---

## fail

立即終止 workflow instance，標記為 FAILED。

```yaml
- type: fail
  message: "order has been cancelled"   # MUST
  code: "order_cancelled"               # MAY
```

在 `catch` / `on_timeout` handler 中使用時，視為錯誤重新拋出至上層。

---

## return

結束 workflow instance，標記為 SUCCEEDED。

```yaml
```yaml
- type: return
  output:                       # MAY
    status: "completed"
```

---

## agent

呼叫 agent definition。詳見 [06-agent](06-agent.md)。

```yaml
- id: analyze
  type: agent
  agent: order.analyzer          # MUST — agent definition name
  system: "你是訂單風險等級專家。"  # MAY — 給 agent 附加的系統提示
  prompt: "分析此訂單"            # MAY — 給 agent 附加的任務提示
  input:
    order: ${ steps.load_order.output }
  tools:                         # MAY — 額外附加的 tools
    - type: task
      action: customer.get_history
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
- 反向執行：最後 SUCCEEDED 的 step 最先補償
- 無 `compensate` 的 step（Tool 未定義且 step 未覆寫）跳過

---

## 錯誤處理模型

三層 handler 由內而外：

1. **Step-level `catch`** — 單一 step 失敗時處理
2. **Saga compensate** — saga 範圍內反向補償
3. **Workflow-level `config.catch`** — 未被處理的錯誤最終到達此處

`catch` handler 內可用的 namespace：

| 變數 | 說明 |
|------|------|
| `error.message` | 錯誤訊息 |
| `error.code` | 錯誤碼 |
| `error.step_id` | 發生錯誤的 step id |

`on_timeout` handler 內可用的 namespace：

| 變數 | 說明 |
|------|------|
| `timeout.step_id` | 超時的 step id |
| `timeout.duration` | 設定的 timeout 值 |
