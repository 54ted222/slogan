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
  max_attempts: 3 # 總嘗試次數（含首次），預設 1 — `1` 即「不重試」
  delay: 2s # 重試間隔，預設 1s
  backoff: exponential # fixed | exponential，預設 fixed
  max_delay: 5m # MAY — backoff 後 sleep 的上限，預設 5m（僅 exponential 生效）
```

- `delay` 支援 CEL 表達式；求值結果 MUST 為 duration string（如 `"30s"`）。非法型別（float / number / 其他 string）→ step FAILED，`error.type == "expression_error.type_error"`
- `backoff: exponential` 時第 N 次重試的實際 sleep = `min(delay × 2^(N-1), max_delay)`；factor 固定為 2
- `backoff: fixed` 時每次 sleep = `delay`，忽略 `max_delay`
- retry 的 sleep MUST 持久化（記錄 `next_attempt_at` 至 checkpoint）；engine 重啟後 MUST 依剩餘時間重排，不重新從 0 開始

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
  condition: ${ steps.load_order.output.status == "cancelled" }
  then:
    - type: fail
      message: "order cancelled"
  else:
    - type: emit
      event: order.processing
```

| 屬性 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `condition` | CEL expression | MUST | 分支條件，求值結果為 boolean |

- `condition` 為 `true` → 執行 `then`；為 `false` → 執行 `else`（若有）
- `condition` 與共通屬性 `when`（前置條件）為不同欄位：`when: false` → 整個 `if` step SKIPPED；`condition` 僅在 `when` 通過後才求值
- Output 為被選中分支最後一個 step 的 output

---

## switch

```yaml
- type: switch
  subject: ${ input.action }
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
    - value: ${ vars.dynamic_action }
      then:
        - type: task
          action: order.cancel
          input: { order_id: ${ input.order_id } }
  default:
    - type: fail
      message: ${ "unknown action: " + input.action }
```

| 屬性 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `subject` | CEL expression | MUST | 待匹配值（任意型別） |
| `cases[].value` | 字面值 or CEL | MUST | 匹配值；與 `subject` 做 `==` 比較 |

- 匹配第一個 `value == subject` 的 case；無匹配且有 `default` → 執行 default；皆無 → SUCCEEDED 且 output 為 `null`
- `value` 為 CEL 時，該 case 被檢查時才求值（lazy）
- **型別相容**：`value == subject` 遵循 CEL `==` 語意；型別不同（如 int vs string、bool vs string）永不匹配（`==` 回 false，不拋錯）。引擎 MAY 在載入期對字面值 `value` 做型別提示警告，但不拒絕載入
- `subject` 與共通屬性 `when`（前置條件）為不同欄位：`when: false` → 整個 `switch` step SKIPPED

---

## foreach

```yaml
- id: reserve_items
  type: foreach
  items: ${ input.items } # 回傳 list 的 CEL 表達式
  # count: ${ vars.max_iterations } # 或以整數迭代（與 items 擇一）
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

- `items` 與 `count` MUST 擇一；兩者皆缺或皆提供 → 載入驗證失敗。`count: N` 等價於 `items: [0..N-1]`，此時 `loop.item == loop.index`。
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

暫停 workflow instance，等待訊號或指定時間。兩種模式互斥：`signals`、`duration`。

### Signals 模式

`signals` 為訊號陣列，每個訊號可為事件訊號或 step 訊號。**任一**訊號匹配即恢復。

```yaml
- id: wait_result
  type: wait
  signals:
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

#### 事件訊號

| 屬性 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `event` | string | MUST | 事件類型 |
| `scope` | string | MAY | 限定事件來源 scope：`workflow` / `project` / `global`；預設接受任一 scope |
| `match` | CEL expression | MAY | 事件過濾條件 |

#### Step 訊號

等待一個 `async: true` 的非阻塞步驟（`foreach` 或 `parallel`）完成。

| 屬性 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `step` | string | MUST | 非阻塞步驟的 ID |

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
  signals:
    - step: reserve_items
  timeout: 5m
  catch:
    - type: fail
      when: ${ error.type == "timeout" }
      message: "reserve timeout"
```

#### 混合訊號

事件訊號與 step 訊號可混合使用，任一匹配即恢復：

```yaml
- id: wait_or_cancel
  type: wait
  signals:
    - step: reserve_items
    - event: order.cancelled
      match: ${ event.data.order_id == input.order_id }
  timeout: 10m
```

#### Output

| 欄位 | 說明 |
|------|------|
| `steps.<id>.output.signal` | 匹配到的訊號類型：`"event"` 或 `"step"` |
| `steps.<id>.output.event` | 事件訊號匹配時：事件類型（如 `payment.confirmed`）；step 訊號時為 `null` |
| `steps.<id>.output.step` | step 訊號匹配時：step ID；事件訊號時為 `null` |
| `steps.<id>.output.data` | 事件訊號匹配時：`event.data`；step 訊號匹配時：該 step 的 output |

```yaml
# 根據匹配到的訊號分支處理
- type: switch
  subject: ${ steps.wait_result.output.event }
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

#### Step 訊號的終態行為

| async step 終態 | wait step 行為 |
|-----------------|----------------|
| SUCCEEDED | wait SUCCEEDED，`output.data` 為該 step 的 output |
| FAILED | wait FAILED，`error.type == "async_step_failed"`、`error.step_id` 為該 step 的 id；`error.cause` 為該 step 的原 error 物件 |
| SKIPPED | wait SKIPPED |

### Duration 模式

```yaml
- type: wait
  duration: 5m
```

### 模式判斷規則

- `signals` 與 `duration` 兩者 MUST 擇一，不可同時存在
- `signals` MUST 為非空陣列

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
- **巢狀 saga**：內層 saga 視為外層的單一 step
  - 內層 saga 自己先完成（含可能的補償）才算「內層 saga step 終態」
  - 外層 saga 觸發補償時，內層 saga 若已成功，只執行外層補償層邏輯；若內層 saga 已處於 compensating，外層等待內層補償結束再繼續外層補償鏈
  - 排序規則：內層整體視為一個時間點（進入 SUCCEEDED 的時戳）參與外層的降序補償
- 無 `compensate` 的 step（Tool 未定義且 step 未覆寫）跳過

#### 補償失敗處理

個別 compensate action 失敗時，引擎 **繼續執行其他 compensate**（best-effort），不中止補償鏈：

- 補償 step 應使用 `retry` 處理暫時錯誤；retry 用盡後仍失敗則記錄，但不阻斷其他補償。
- 所有補償完成後（含失敗），進入 saga `catch`；`error.details.compensation_failures` 為失敗清單：

  ```
  error.type: "saga_failed"
  error.cause: <觸發補償的原 step error>
  error.details.compensation_failures: [
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
