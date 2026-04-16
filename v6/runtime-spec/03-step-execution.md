# 03 — Step Execution

本文件針對 [dsl-spec/03-steps.md](../dsl-spec/03-steps.md) 定義的每種 step type，給出引擎執行語意。

---

## 通用執行流程

每個 step 由 Engine Loop 依序執行下列階段：

```
1. precondition  → 求值 when；false → SKIPPED；異常 → FAILED
2. resolve       → 解析 input / args / 其他 ${ } 欄位；異常 → FAILED
3. checkpoint A  → 寫入 step RUNNING 狀態
4. execute       → 委派至對應 type handler；可能阻塞、發事件、產生 side effect
5. complete      → 收到 outcome；若 FAILED 進入 retry / catch；若 SUCCEEDED 寫 output
6. checkpoint B  → 寫入終態
7. publish       → 推進 next step；發 step.completed 事件
```

階段 1–2 為**純讀**：未通過時 step 不消耗 retry 計數。階段 3 後的失敗才會觸發 retry。

---

## type: task

### 解析

1. Task Registry resolve `action` → `Action`（Tool / Function / builtin）
2. 失敗 → FAILED `error.type == "registry.action_not_found"`

### 執行

| Action 種類 | 委派對象 | 執行語意 |
|-------------|----------|----------|
| Tool | Backend Driver | spawn / HTTP；單訊息或 NDJSON；按 `idempotent` 決定是否快取 result |
| Function | Engine Loop（建立 Function Instance） | 父 step 進入 `waiting_subinstance`；子 instance 終結後恢復 |
| builtin | 引擎內建實作 | 同步呼叫，無 process / network |

### Callback 路由（Tool）

- 進入 step 時若 caller 有 `callback:`，driver 啟用 NDJSON 模式（exec）或 SSE（http）。
- Tool 發 `{type:"callback", call_id, name, input}` → driver 暫停 reading → engine 執行對應 handler steps（共享 caller namespace）→ handler `return` → engine post `{type:"callback_result", call_id, output}` 至 stdin / `X-Callback-URL`。
- Handler FAILED 時的漣漪：
  - engine 傳回 `{type:"callback_result", call_id, error: {type, message}}`（不含 `output`）
  - Tool 可選擇：
    1. 以 `{type:"result", success:false, error:{...}}` 結束 → caller step FAILED，`error.cause` 為 tool 回報的 error
    2. 以 `{type:"result", success:true, output:{...}}` 結束 → caller step SUCCEEDED（tool 決定忽略 handler 錯誤）
    3. 繼續執行（發更多 callback）→ engine 持續路由至對應 handler
  - 最終 caller step 的終態以 tool 的 `result.success` 為準；`callback_result.error` 僅是通知通道

### Output 寫入

- `idempotent: true` 的 step：若 retry 觸發，driver 先嘗試讀取 instance store 的 cached output（key `(instance_id, step_id, attempt_signature)`）；命中則跳過實際呼叫。
- 非 idempotent：每次 attempt 都實際呼叫；persistence 只記錄最後一次成功的 output。

### Attempt 與 signature 的精確語意

- `attempt` 為 integer，首次執行 = 1；於 step 由 `WAITING` / retry-sleep 進入 `RUNNING`、寫入 checkpoint 的那一刻遞增。
- Engine crash 後重啟：依 checkpoint 中的 `attempts_so_far` 恢復，**不**重新從 0 / 1 開始（例如已重試 2 次並在 attempt=3 中 crash → 重啟後仍以 attempt=3 進入 backend，signature 與 crash 前相同，可命中 cache）。
- `input_snapshot`（CEL 求值結果）於 **首次** 進入 RUNNING 時取樣並寫入 checkpoint；**後續 retry 沿用同一 snapshot**，不重新求值 CEL（避免 `${ now() }` / `${ uuid() }` 造成 signature 漂移、破壞 idempotency）。
  - 唯一例外：`retry.delay`（若為 CEL）每次 retry 前重新求值，屬控制欄位不進 signature。
  - 若使用者確實需要每次 retry 以新值呼叫，請將求值改為該 step 之前的 `assign`，或顯式關閉 `idempotent`。
- 補償 step 獨立計數：`compensate_attempt` 首次 = 1，與原 step `attempt` 不共用；即使同一 tool 被多個 saga step 當作補償引用，key 以 `(instance_id, origin_step_id, "compensate", compensate_attempt)` 區分，不會 collision。
- Signature 明確定義：`SHA256_hex(action_name || "\0" || canonical_json(input_snapshot) || "\0" || attempt_as_decimal_string)`；`canonical_json` 採 **RFC 8785 JSON Canonicalization Scheme (JCS)** 規則：
  - 物件 key 以 UTF-16 codepoint 順序遞增排序；相同 key 僅保留最後一個
  - 不輸出空白字元（包括 key/value 分隔符後的空格、換行）
  - 字串以 UTF-8 編碼；數值採 ECMAScript 7.1.12.1 `ToString` 規則（避免 `1.0` vs `1` / `1e2` vs `100` 等同值不同文字表示造成 hash 漂移）
  - `null` / `true` / `false` 以固定字面值輸出；`array` 順序保留（不排序）
  - 跨 engine 實作 MUST 對同一 logical input 產生**相同 bytes**；實作可使用現成 JCS library（如 Go `github.com/cyberphone/json-canonicalization`）

### idempotent + compensate 組合

Tool 同時宣告 `idempotent: true` 與 `compensate` 時，職責分離：

- 原 step 的 idempotency：以 `(instance_id, step_id, attempt_signature)` 去重
- 補償 step 的 idempotency：獨立計算，key 為 `(instance_id, step_id, "compensate", compensate_attempt)`
  - `compensate_attempt` 從 0 開始，retry 時自增；與原 step 的 attempt 不共用計數
  - 補償 tool 應亦宣告 `idempotent: true`；否則引擎無法安全重試補償
- Engine 重啟後依 `compensate_state` checkpoint 決定：
  - `done` → 不重跑
  - `started` 且 compensate tool idempotent → 重跑（可能觸發 cache hit）
  - `started` 且非 idempotent → 標記該 compensate FAILED（`error.type: "lease_lost_unsafe_resume"`），計入 compensation_failures
- 補償 step 的 output 僅用於 engine 觀測，不進入任何 `steps.*` namespace（使用者無法引用）

---

## type: assign

純 namespace 寫入。

```
1. 對 vars map 中每個 key 求值對應 expression
2. 全部成功 → 寫入 vars namespace；step SUCCEEDED，output 為 vars 增量 map
3. 任一失敗 → FAILED；vars 不變動（atomic）
```

vars 的 scope 是 instance-wide；子 instance 不繼承父 vars。

---

## type: if

```
1. 求值共通 when（前置條件）：false → SKIPPED；異常 → FAILED
2. 求值 condition（mandatory, boolean）
3. true  → 執行 then[] 至完成；output = 最後一個 step 的 output
4. false → 執行 else[] 至完成；output = 最後一個 step 的 output；無 else 則 output: null, status SUCCEEDED
5. 子 step 任一 FAILED 且未被內部 catch → 整個 if FAILED
```

`if` step 的 `id` 可被引用：`steps.<if_id>.output` 為被選中分支的最後一個 step output。

---

## type: switch

```
1. 求值共通 when（前置條件）：false → SKIPPED；異常 → FAILED
2. 求值 subject（任意型別）
3. 依序求值 cases[].value，找到第一個 == subject 的 case
4. 沒匹配且有 default → 執行 default[]
5. 沒匹配且無 default → SUCCEEDED, output: null
6. 子 step 失敗處理同 if
```

`cases[].value` 可為字面值或 CEL；CEL 求值在 case 開始檢查時才執行（lazy）。`subject` 與 `value` 的型別需相容才可能匹配（如 `subject` 為 string、`value` 回傳 boolean 則永不匹配）。

---

## type: foreach

### 啟動

1. 求值 `items` 或 `count`
   - `items` 求值結果 MUST 為 list；**任何**非 list 型別（含 null / string / int / map / bool）→ step FAILED，`error.type: "invalid_items"`，`error.details.actual_type` 記錄實際型別（提示使用者以 `default(x, [])` 防禦）
   - `count` 求值結果 MUST 為非負整數：
     - 負數 / 小數 / 非整數型別 → step FAILED，`error.type: "invalid_count"`
2. `count: N` → 內部 items = `[0..N-1]`，`loop.item == loop.index`
3. items 空陣列 → SUCCEEDED, output: `[]`（不進入迭代，do block 不執行）
4. 寫 checkpoint：foreach RUNNING、總迭代數

### 阻塞模式（async: false）

- Engine 依 `concurrency` 並行建立子 step 序列；每個迭代為一個獨立的執行分支，擁有自己的 `loop.*` 與 step namespace（迭代間不共享 `steps.*`）。
- 子 step 完成順序由完成時間決定；output array 仍按原索引排列。
- `failure_policy` 行為見 dsl-spec/03-steps.md；引擎在每個迭代終態時重新評估是否取消其餘。

### 非阻塞模式（async: true）

- foreach 寫入 RUNNING 後立即發 `step.async_started` 事件並返回；父 instance 推進至下一步。
- 引擎在背景以同樣的 `concurrency` 持續調度；達終態後寫入終態 checkpoint，發 `step.completed` 事件。
- 對應的 `wait` step 在訂閱階段檢查當前狀態；若已完成則直接讀取 output。

### Loop 變數遮蔽

- 進入 foreach 時 push 新 `loop` frame；離開時 pop。
- 巢狀 foreach：內層 frame 完全遮蔽外層；CEL 中 `loop` 永遠指當前最內層。

---

## type: parallel

- 與 foreach 類似但分支固定為 `branches[]`。
- 阻塞 / 非阻塞、`failure_policy` 同 foreach。
- 每個 branch 是獨立 step 序列；branch 內無 `loop.*`。

---

## type: emit

```
1. 求值 event 名稱、scope、data、delay
2. 構造事件物件：{ id: uuid(), type: <event>, scope, data, source: instance_id, timestamp: now() }
3. 若 delay > 0 → 寫入延遲 queue；否則直接 publish
4. SUCCEEDED, output: { event_id }
```

- emit 不等待消費者；at-least-once 投遞由 Event Bus 保證。
- delay queue 是持久化的；engine 重啟後仍會在 deadline 觸發。

---

## type: wait

### Signals 模式

`signals` 陣列中每個訊號各自建立訂閱，任一匹配即喚醒：

```
1. 遍歷 signals[]，依類型建立訂閱：
   - 事件訊號（有 event 欄位）：寫入 wait subscription 至 store
     { instance_id, step_id, event_type, match_expression, deadline }
   - Step 訊號（有 step 欄位）：見下「Step 訊號匹配規則」
2. 寫 checkpoint：step RUNNING.waiting_signal
3. 阻塞，無計算成本（engine loop 處理其他 instance）
4. 任一訊號匹配 → 喚醒；取消其餘訂閱
5. timeout 觸發 → step FAILED, error.type == "timeout"
```

訂閱 wake-up 依賴 Engine Loop 收到 `event.matched` 或 `step.completed` 事件後重新 schedule 該 instance。

#### Step 訊號匹配規則

1. **Fast-path 判定**：讀取目標 step 狀態；若狀態屬於 `{SUCCEEDED, FAILED, SKIPPED}` **且** 所在 instance store 的 checkpoint `last_committed_at >= target_step.terminated_at` → 視為已持久化終態，依下步驟 4 決定 wait 終態。
2. **未終態或持久化未確認**：建立 `step.completed` 訂閱並進入阻塞。
3. **事件到達時的 re-read**：
   - 讀取目標 step 最新 state；若 **仍非終態** → 視為事件早到（極少數並發情境），**維持訂閱繼續等待** 下一個 `step.completed`；不報錯（此事件 MAY 重覆投遞至同一 subscription，屬 at-least-once 的正常行為）。
   - 若為終態 → 依下步驟 4 決定 wait 終態。
4. **依目標終態映射 wait step 終態**（與 `dsl-spec/03-steps.md`「Step 訊號的終態行為」表一致）：
   - 目標 SUCCEEDED → wait SUCCEEDED，`output = { signal: "step", event: null, step: <target_id>, data: <target.output> }`
   - 目標 FAILED → wait FAILED，`error.type == "async_step_failed"`、`error.step_id == <target_id>`、`error.cause == <target.error>`；可被 catch 捕捉
   - 目標 SKIPPED → wait SKIPPED（不執行後續 step）
5. **目標 step 永不終態**（例：`async: true` 的 step 所在 instance 已 CANCELLED）：`step.completed` 不會到達，依賴 `wait.timeout` 兜底；deadline 到即 FAILED（`error.type == "timeout"`）。

---

## type: fail

- 不執行任何 I/O；直接構造 error 並 raise 至上層。
- 在 catch handler 內使用時，rethrow 至 catch 外層。

---

## type: return

- 求值 `output`；對父 instance 是「子 instance SUCCEEDED 的 output」。
- 對 workflow / function：觸發 instance 終態流程（含 output_schema 驗證）。

---

## type: callback（function 內部）

```
1. 求值 input；驗證符合 callback 宣告的 input_schema
2. 寫 checkpoint：function instance RUNNING.suspended_callback
3. 發送 callback request 給 caller instance（含 caller step_id、callback name、call_id、input）
4. caller engine 收到後執行對應 handler steps（共享 caller namespace；不影響 caller step 進度）
5. handler return → 驗證符合 output_schema → 回覆 callback_result
6. function instance 喚醒，step SUCCEEDED with output
7. timeout 在 step 1 寫入 deadline；deadline 到 → FAILED, error.type == "timeout"
```

caller engine 對 handler 的執行：

- 為 handler 建立短命的「callback frame」：擁有獨立 `steps.*` namespace（避免與 caller 主流程衝突），共享 caller 的 `input` / `vars`。
- handler 內 step 對 `prev` 的解析以 handler steps 為準（不指向 caller 主流程）。
- handler `return` 結束；若未 return 則最後一個 step output 視為 handler output。

---

## type: saga

```
1. 進入 RUNNING；初始化 compensation log = []
2. 依序執行 steps：
   - step SUCCEEDED 且有 compensate（tool 預設或 step 覆寫） → 推入 compensation log
3. 任一 step FAILED 未被內部 catch →
   a. 進入 RUNNING.compensating
   b. 依時間戳降序執行 compensation log 中的所有 compensate
      - 每個 compensate 在 Instance Store 維護 compensate_state: pending → started → done | failed
      - started 中的 compensate 崩潰後重啟，engine 依 compensate 的 idempotent 屬性決定是否重跑（建議補償 tool 設為 idempotent: true）
      - done 狀態的 compensate 重啟後不再重跑（從 checkpoint 直接跳過）
   c. 補償失敗不中止其他補償；累計至 error.details.compensation_failures
   d. 全部補償完成 → step FAILED；error.type == "saga_failed"，error.details.compensation_failures: [...]
   e. 進入 saga.catch（若有）；catch 也可消化錯誤令 saga SUCCEEDED
4. 全部 step SUCCEEDED → SUCCEEDED, output: null（saga 不暴露 sub-step output）
```

並行子 step 的補償時序見 dsl-spec/03-steps.md「補償規則」。

---

## Retry

step `retry` 在 step 達 FAILED 時介入：

```
attempts ← 1
while step FAILED and attempts < max_attempts:
    sleep_duration ←
        min(delay × 2^(attempts-1), max_delay)  if backoff == exponential
        delay                                    if backoff == fixed
    next_attempt_at ← now() + sleep_duration
    寫 checkpoint（含 attempts、next_attempt_at）
    sleep_until(next_attempt_at)   # 可被 engine 重啟中斷，重啟後依 next_attempt_at 繼續等
    attempts += 1
    重新進入「3. checkpoint A」（覆寫舊 RUNNING checkpoint）
    執行 → 取得新 outcome
若仍 FAILED → 進入 catch
```

- `delay` / `max_delay` 支援 CEL，每次 attempt 前求值；MUST 回傳 duration string（如 `"30s"`）。非 duration → step FAILED (`expression_error.type_error`)。
- `backoff: exponential` 的乘數固定為 2；無上限時實務上會指數爆炸，故 `max_delay` 預設 `5m`（使用者可覆寫）。
- attempt 計數與 `next_attempt_at` 都寫入 checkpoint，engine 重啟可正確還原 — 不重新從 0 計時。

---

## Catch

`catch` 是一個 step 序列，可使用 `error.*` namespace。

```
1. step FAILED 且 retry 用盡
2. 若有 catch[] → 視為一段子 steps，依序執行
3. catch 中的 step 也可能 FAILED；catch 不再重新進入 catch（避免無限）
4. catch 全部 SUCCEEDED → 原 step 視為已處理，status SUCCEEDED；output 為 catch 最後一個 step 的 output（若有）
5. catch 中觸發 type: fail → 錯誤向上拋
6. catch 自身有未處理錯誤 → 錯誤向上拋
```

`when` 在 catch 內 step 上仍可用，常見搭配：

```yaml
catch:
  - type: fail
    when: ${ error.type == "timeout" }
    message: ...
  - type: emit
    event: step.recovered
```

---

## Lifecycle init / destroy

- init 不在 step pipeline；由 Tool Registry 在 resolve 階段檢查並按需呼叫。
- 第一次解析某 tool → 從 Resource Pool 查 cache；miss → 執行 init backend → 寫 cache。
- init 失敗：所有依賴此 tool 的 step 在 RUNNING checkpoint 之前 FAIL（`error.type == "lifecycle_init_failed"`）。
- destroy 在 instance 終結時觸發；失敗僅記錄，不影響 instance 最終狀態。
