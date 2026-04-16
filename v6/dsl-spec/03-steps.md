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

- `max_attempts` MUST 為正整數且 ≥ 1；字面值 0 / 負整數 / 非整數 → 載入失敗，`error.type: "invalid_retry_config"`、`details.field: "max_attempts"`
- `max_attempts` 可為 CEL 表達式（`max_attempts: ${ vars.attempts }`）；step 進入 RUNNING 前求值。求值結果非整數或 < 1 → step FAILED，`error.type == "invalid_retry_config"`
- `delay` 支援 CEL 表達式；求值結果 MUST 為 duration string（如 `"30s"`）。非法型別（float / number / 其他 string）→ step FAILED，`error.type == "expression_error.type_error"`
- `backoff: exponential` 時第 N 次重試的實際 sleep = `min(delay × 2^(N-1), max_delay)`；factor 固定為 2
- `backoff: fixed` 時每次 sleep = `delay`，忽略 `max_delay`
- retry 的 sleep MUST 持久化（記錄 `next_attempt_at` 至 checkpoint）；engine 重啟後 MUST 依剩餘時間重排，不重新從 0 開始
- `max_attempts: 1` 與**未宣告 `retry`** 語意等價（皆為「不重試」）；實作 MAY 對兩者不做區分

---

## task

呼叫一個 tool definition（v3 稱 task definition）。

```yaml
- id: load_order
  type: task
  action: order.load # MUST — tool definition 的 name；可附帶 @<version>
  input: # MAY — 傳給 tool 的輸入
    order_id: ${ input.order_id }
  retry: { ... }
  timeout: 30s
  catch: [...]
```

### action 的版本語法

- `action: order.load` — 不指定版本，引擎按「版本解析優先權」決定
- `action: order.load@2` — 明確指定 version 2；找不到 → `registry.action_not_found`
- `action: order/order.load@2` — project 前綴 + version

**版本解析優先權**（高到低，對應 `runtime-spec/05-task-registry.md` 的 `resolve_version`）：

1. Step 明確 `@<version>`
2. Instance action pin（首次解析時記錄；見 `runtime-spec/05-task-registry.md`）
3. Project `project.yaml` 的 `defaults.action_versions.<canonical_name>: <version>`
4. 全域 `registry.default_version_policy`（預設 `highest` — 取 registry 中最大 version；可設為 `lowest` / `require_explicit`）

`highest` 回退可能在 version 升級時改變行為；一旦 instance 首次解析即透過 pin 凍結（見步驟 2），同 instance 內後續 step 不會漂移。Production workflow 仍建議明確寫 `@<version>` 或以 `defaults.action_versions` 釘選，以避免新 instance 受新版本影響。

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
- `vars` map 中 key MUST 為簡單識別字（`^[a-z_][a-z0-9_]*$`）；含 `.` / `/` / 其他 path separator 或 namespace prefix（如 `steps.foo`、`input.bar`）→ 載入失敗，`error.type: "invalid_var_name"`
- `_` 起始的 key 保留給引擎擴充；使用者 assign 以 `_` 開頭 → 載入失敗，`error.type: "invalid_var_name"`（與上同一錯誤碼，`details.reason: "reserved_prefix"`）
- key 不得與全域 namespace 同名（保留字清單：`input` / `steps` / `prev` / `vars` / `loop` / `event` / `env` / `secret` / `error` / `callback` / `context` / `project` / `artifacts`）；違反 → 載入失敗，`error.type: "invalid_var_name"`、`details.reason: "reserved_namespace"`
  - 原因：雖然 `assign` 寫入 `vars.<key>`、與頂層 namespace 不同，但使用者易混淆（讀 `vars.input` 與讀 `input` 語意不同），故載入期即禁止

**Output**：`steps.<id>.output` 為本次 assign 的增量 map（僅含本 step 寫入的 key → value；不含既有 `vars`）。例：

```yaml
- id: set_totals
  type: assign
  vars:
    subtotal: 100
    tax: 8
# steps.set_totals.output == { "subtotal": 100, "tax": 8 }
```

若 assign 的某 key CEL 求值失敗 → step FAILED，`error.type == "expression_error"`；`vars` 不發生任何變動（atomic write，見 `runtime-spec/03-step-execution.md`）。

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
- **condition 求值結果嚴格要求 boolean**；非 boolean（null / int / string / list / map）→ step FAILED，`error.type == "expression_error.type_error"`；不做 truthy 自動轉換（避免 `0` / `""` / `[]` 意外成立 false-branch）

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
  - 若該 case 的 `value` CEL 求值失敗 → 整個 switch step FAILED，`error.type == "expression_error"`、`error.details.case_index` 為失敗的 case 索引；後續 case 不再求值（避免部分求值導致的不確定性）
- **型別相容**：`value == subject` 遵循 CEL `==` 語意；型別不同（如 int vs string、bool vs string）永不匹配（`==` 回 false，不拋錯）。引擎 MAY 在載入期對字面值 `value` 做型別提示警告，但不拒絕載入
- `subject` 與共通屬性 `when`（前置條件）為不同欄位：`when: false` → 整個 `switch` step SKIPPED
- `subject` 求值失敗 → step FAILED，`error.type == "expression_error"`；不進入 case 匹配

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

### 迭代內的 step 可見性

`do[]` 中的 step 屬於**每迭代獨立的作用域**，引擎為每次迭代建立一個 frame：

- `loop.item` / `loop.index` 於 frame 生命期內固定，不受其他迭代影響。
- `steps.*` namespace 僅可見**本迭代已完成的 step**；無法跨迭代讀取（例如迭代 3 無法讀迭代 2 的 `steps.foo.output`）。需要跨迭代累積請改用 `vars`（搭配 `concurrency: 1`）或 foreach 完成後讀整個陣列 `steps.<foreach_id>.output[i]`。
- `prev` 僅指向**本迭代內**語法上的前一 step；迭代的第一個 step `prev == null`。
- `when` 於各迭代獨立求值；同一 step id 可能在某些迭代 SKIPPED、某些迭代 SUCCEEDED。
- Output 陣列的位置即使對應迭代 SKIPPED / FAILED / `ignore`（見 `failure_policy`），**MUST 以 `null` 占位**以維持 index 對齊（foreach output 長度恆等於 items / count）；SKIPPED 與 FAILED 在陣列中不可區分，需檢查 `steps.<foreach_id>.output[i] == null` 並搭配 `error.failed_indices` 辨識。

### 嵌套作用域的跨層存取規則

Step 所在位置的 `steps.*` 可讀範圍遵循「可見已完成的祖先作用域」規則：

| 存取方向 | 是否可讀 |
|---------|---------|
| 本迭代 / branch 內已 SUCCEEDED 的 step | ✅ |
| 外層（祖父、曾祖父等）已 SUCCEEDED 的 step（含包裹本迭代的 foreach / parallel / saga / if / switch 之**前**的 steps） | ✅ |
| 包裹本迭代的 foreach / parallel step **自身**（尚未終結） | ❌（未完成；讀取 `steps.<outer_id>.output` 為 `expression_error.identifier_not_found`） |
| **同層兄弟分支**（parallel 內其他 branch、foreach 其他迭代） | ❌ |
| 內層巢狀 step（foreach / parallel / saga / if / switch 的 body 內部 step） | ❌（不暴露至外層 namespace） |
| 已終結兄弟 branch / 迭代（理論上可能，但為避免 race 與混亂 → 統一禁止） | ❌ |

具體來說，在二層 foreach 的**內層**迭代內，`steps.<outer_step_id>` 可讀（其為外層 foreach **之前**已完成的 step，不含外層 foreach 自身），但不可讀同層其他外層迭代的 `steps`。外層 foreach 自己的 output 陣列在外層 foreach 結束後的 step 才可見。

此規則確保 step output 的可見性與「執行順序」嚴格對應，避免並行分支間暴露未 commit 的中間狀態。

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

### Branch 作用域

各 branch 為**獨立作用域**，規則與 foreach 迭代類似：

- `steps.*` 僅可見**本 branch 已完成**的 step；branches 間無法互讀 output。跨 branch 共享資料請於 parallel 前以 `assign` 寫入 `vars`（注意 branch 內對 `vars` 的寫入僅對**本 branch**可見；parallel 結束後不保留，避免並發 race）。
- `prev` 僅指向本 branch 語法上的前一 step。
- `when` 於各 branch 內獨立求值；不同 branch 的同名 step id **MUST** 由載入驗證器拒絕（parallel 內所有 branch 的 step id 集合必須互斥、且不與外層衝突），以避免 `steps.<id>` 解析歧義。
- Output 陣列長度固定等於 branches 數；若某 branch 的最後一個 step SKIPPED 或整個 branch 為空 → 該索引位置為 `null`。`failure_policy: continue` / `ignore` 時失敗 branch 亦以 `null` 占位。

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

#### event 名稱規則

- 格式：**dotted snake_case**，每段符合 `^[a-z][a-z0-9_]*$`，段間以 `.` 分隔；例：`order.completed` / `payment.webhook.received`
- 長度限制：整體 ≤ 255 chars；段數 ≤ 8
- 不支援 `/` 分隔；命名空間靠 dotted prefix 表達（而非路徑）
- **保留前綴**（engine 內部使用，`emit` 不得宣告）：
  - `engine.*`：引擎生命週期事件
  - `internal.*`：engine 內部事件（scope: internal）
  - `bus.*`：event bus 自身事件（如 `bus.dead_letter`）
  - `tool.*`：tool backend 事件（如 `tool.stream` / `tool.callback`）
  - `step.*`、`instance.*`、`wait.*`、`event.*`、`lifecycle.*`、`trigger.*`、`workflow.*`、`function.*`：引擎內部流程事件
- 違反格式或使用保留前綴 → 載入失敗，`registry.invalid_event_name`
- `wait.signals[].event` / event trigger 的 `event` 欄位亦套同一規則

#### data 大小限制

- `data` 序列化後 MUST ≤ `engine.event_bus_max_data_bytes`（預設 1 MB，見 `runtime-spec/08-persistence.md`）
- 驗證時機：**emit step 進入 RUNNING、CEL 求值後、排入 event bus / delayed_events 前**；超限 → step FAILED，`error.type == "event_too_large"`、`error.details.size` 保留實際 bytes
- `delay > 0` 時：驗證仍於排入 delayed_events 前執行；已入 queue 的 event 保證在 deadline 到達時符合大小限制（不會在投遞瞬間失敗）
- 若 emit 本身被 `catch` 捕捉或於 `workflow.config.catch` 中執行，大小限制仍適用（不因位置豁免）

#### data 巢狀 CEL 的求值原子性

`data` 通常為 map，內含任意層的 CEL 表達式（`${ ... }`）。引擎以**原子方式**求值：

- 先求值所有 CEL 欄位 → 組成完整 data 結構；任一欄位失敗 → **整個 emit 失敗**，不投遞事件
- 失敗的 error.type 為第一個失敗 CEL 對應的 expression_error 子類型；`error.details.field` 為 dotted 路徑（如 `data.items[2].price`）
- **部分投遞不允許**：不會將成功欄位投遞、失敗欄位設 null；此設計確保消費者看到的事件結構一致
- `event.id` 於求值成功後才分配（失敗的 emit 不消耗 id / sequence）

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
  duration: 5m            # MUST — 字面 duration string 或 CEL 求 duration string
```

規則：

- `duration` 於 step 進入 RUNNING 前求值；若非 string 或不符 Duration 格式 → step FAILED（`error.type == "invalid_duration_format"`，不建立 subscription）
- 求值結果 `0s` 或負值 → step FAILED，`error.type == "invalid_duration"`
- 到期 deadline = `step RUNNING checkpoint commit 時刻 + duration 求值結果`；一次性決定並持久化（邏輯同 wait signals 的 deadline；見 `runtime-spec/07-event-bus.md`）
- 期間不可被事件喚醒（無 signals）；僅 `instance.cancel` / workflow timeout 可中止
- Output：`{ signal: "duration", duration_ms: <實際 sleep 毫秒> }`；正常結束時 `duration_ms` 恰為求值結果；被 cancel 中斷不產 output（step 終態為 CANCELLED / FAILED）

### 模式判斷規則

- `signals` 與 `duration` 兩者 MUST 擇一，不可同時存在
- `signals` MUST 為非空陣列
- `duration` 不得與 `timeout` 欄位共存（duration 本身即為時限）；同時宣告 → 載入失敗，`error.type: "invalid_wait_config"`

---

## fail

立即終止 workflow instance，標記為 FAILED。

```yaml
- type: fail
  message: "order has been cancelled" # MUST — 非空字串；長度 ≤ 2048 chars
  code: "order_cancelled"             # MAY — 業務錯誤碼；見下規則
```

### message / code 格式

- `message`：MUST 為非空字串；求值後若為空字串 / null → step FAILED，`error.type == "invalid_fail_config"`、`error.details.field == "message"`
- `message` 長度 ≤ 2048 chars；超過 → 截斷至 2048 並加 `…`（observability 事件記錄 truncation）
- `code`：MAY；若提供，格式 `^[a-z][a-z0-9_.]{0,63}$`（dotted snake_case，首字母小寫，≤ 64 chars）；違反 → 載入失敗 `invalid_fail_config`
- `code` 的空字串 `""` 視為**未提供**（與省略 `code` 欄位等價）；`error.code` 為 null
- CEL 表達式求值後 code 若為 null → 同上，視為未提供
- `catch` 中讀取 `error.code` 時，若原 `fail` 未提供 code 或提供空字串，`has(error.code) == false`、`error.code == null`

在 `catch` handler 中使用時，視為錯誤重新拋出至上層。

---

## return

結束 **當前 instance**，標記為 SUCCEEDED。

```yaml
- type: return
  output: # MAY
    status: "completed"
```

### output 求值失敗

`output` 欄位的 CEL 若求值失敗（identifier 不存在、型別錯誤等）：

- 本 instance 立即轉為 **FAILED**（**不**進入 SUCCEEDED），`error.type == "expression_error"`、`error.fragment` 為失敗的 CEL 片段
- 失敗後仍依一般 FAILED 終結流程處理：走 `workflow.config.catch`（若存在，且該 step 在 workflow 層級）→ 寫 checkpoint → 發 `instance.failed` 事件
- `return` 本身**不支援** `catch`（語法上不列；catch 為長時 I/O step 的選項，return 為瞬時）；若需先求值再決定 return，請於前一 step 以 `assign` 將值寫入 `vars`，再 `return output: ${ vars.x }`
- 於 saga 內 `return` 且 output 求值失敗：因 return 直接結束 instance，saga 補償**不會**觸發（已完成的 saga step 其 compensate 於 return 之前已省略，因 saga 視同 SUCCEEDED）；若需確保補償，請先檢查 output 可得性後再 return

### 作用域規則

| 位置 | 行為 |
|------|------|
| Workflow 頂層 steps[] | 結束整個 workflow instance；output 為 return 值 |
| Function 頂層 steps[] | 結束該 function instance；output 回傳給呼叫端 task step |
| `foreach.do[]` | **只結束當前迭代**；該迭代視為 SUCCEEDED，output 為 return 值；foreach 繼續其他迭代 |
| `parallel.branches[i].steps[]` | 只結束該 branch；其他 branch 不受影響 |
| `if.then` / `if.else` / `switch.cases[].then` | 結束外層最接近的 instance（workflow 或 function）；分支結構不建立新 scope |
| `saga.steps[]` | 結束外層最接近的 instance；saga 不捕獲 return |
| `catch[]` | 結束外層最接近的 instance |

若需在 foreach / parallel 內提前中止整個 workflow，使用 `type: fail`（錯誤向上傳播）或外層 `if` 判斷後於迴圈外 `return`。

### output_schema 驗證層級

- `foreach.do[]` / `parallel.branches[].steps[]` 內的 `return` **只結束該迭代 / branch**，該 return 的 output 寫入 foreach 的 output array 位置或 branch 的 output slot；**不**觸發 workflow / function 層的 `output_schema` 驗證。
- Workflow / function 層的 `output_schema` 僅於「instance 整體終結」時驗證一次（見 `runtime-spec/02-instance-lifecycle.md`）。
- Function instance 的 `output_schema` 驗證於**子 instance 內完成**；父 task step 收到的 output 已通過驗證，父層不重驗（若需額外驗證，請在 task step 後以 `if` / `assign` 顯式處理）。
- 以 `type: fail` 中止不經 `output_schema`；instance FAILED，error 依 fail 的 message / type 決定。

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
- **時間戳 tiebreak**：若兩 step 的 `ended_at` 毫秒精度相同（parallel branch 或批次 checkpoint 所致），依以下規則決定順序（皆為降序）：
  1. `step_path` 字串（lexicographic 降序，例：`saga.0.parallel.branches.2.0` > `saga.0.parallel.branches.1.0`）
  2. 若 `step_path` 相同（理論上不應發生） → 以 `started_at` 降序
  3. 若仍相同 → 以 `(instance_id, step_id)` 字串降序（作為最終決定性來源）
- 此 tiebreak 確保同一組 input 下，補償順序於不同 engine 實作間**完全決定性**，避免冪等性設計依賴實作細節
- 若 saga 內含 `parallel` / `foreach`：
  - 不同分支 / 不同迭代間**無全序保證**，補償可能並行進行（受 `concurrency` 限制）
  - 同一分支或同一迭代內部仍遵循時間戳降序
  - 分支間有隱含依賴時，補償順序由 tool 作者自行以 idempotent 設計處理；引擎不提供依賴宣告
- **巢狀 saga**：內層 saga 視為外層的單一 step
  - 內層 saga 自己先完成（含可能的補償）才算「內層 saga step 終態」
  - 外層 saga 觸發補償時，內層 saga 若已成功，只執行外層補償層邏輯；若內層 saga 已處於 compensating，外層等待內層補償結束再繼續外層補償鏈
  - 排序規則：內層整體視為一個時間點（進入 SUCCEEDED 的時戳）參與外層的降序補償
- 無 `compensate` 的 step（Tool 未定義且 step 未覆寫）跳過

#### Saga 內的 `type: emit` 語意

`emit` 發出事件是 **不可撤銷的 side effect**：

- 已投遞至 Event Bus 的事件即使 saga 進入補償，也不會被「收回」
- 引擎不提供 emit 的 compensate 鉤子；使用者若需「撤銷通知」，應由補償邏輯**額外發一個撤銷事件**（如 `order.rolled_back`），下游消費者自行以 event type 或 correlation id 區分
- Saga 補償完成後可在 `catch` 區塊內以 `type: emit` 發送 rollback 事件（見 99-examples.md 範例 4）
- `emit.delay > 0` 的延遲事件：若 saga 在 delay deadline 前進入補償，已在 delayed_events 表中但未投遞的事件**會被刪除**（與 instance cancel 相同機制），不會投遞

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

### catch 消化失敗後的 step output

當 step 層 `catch` **成功消化錯誤**（catch 內未 re-raise，最終 step 視為 SUCCEEDED），`steps.<id>.output` 的值規則：

- catch 區塊最後一個 step 的 output 即為本 step 的 output（無視原始失敗 step 的 partial output）
- 若 catch 最後 step 為 `type: emit` / `type: fail`（無 meaningful output）→ `steps.<id>.output == null`
- 若 catch 內執行 `type: return` → 不寫入 `steps.<id>.output`（`return` 結束外層 instance，見「return 作用域」）
- 若 catch 自身 FAILED（re-raise）→ 原 step 終態為 FAILED，`steps.<id>.output == null`、`steps.<id>.error` 為 catch 自身或 re-raised 錯誤

此規則確保 `has(steps.<id>.output)` 與 `steps.<id>.status == "SUCCEEDED"` 的語意一致：output 存在 ⇔ step 成功（不論是否經 catch 消化）。
