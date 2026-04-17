# 10 — Concurrency

本文件定義 parallel / foreach / async / wait 在引擎內的並行調度、cancel propagation、以及 race condition 處置。

---

## 執行模型前提

Engine 為 **類 Node.js 的單線程 event loop** 模型（見 `01-architecture.md`）：

- 單一 engine 進程 = 單一事件迴圈 thread。該 thread 負責所有 instance 的事件分派與 step transition。
- 所有 I/O 為 non-blocking async；主 loop 絕不阻塞在同步 I/O 或 CPU 密集計算上。
- **同進程內沒有 thread-level 並行**：同時刻 engine 只在處理一個事件；instance 之間、step transition 之間由 event loop 天然串行。
- **跨進程擴展**：多 engine 進程共享 Instance Store，透過 lease 分配 instance 擁有權。

因此以下所謂「並行」皆指 **邏輯並行**（邏輯上多個 step 同時 RUNNING，實際是這些 step 各自在等外部事件；engine 只在事件到達時處理 transition），不是 thread-level 並行。

---

## 並行的層級

| 層級 | 並行單位 | 實現方式 |
|------|----------|----------|
| Cross-process | 不同 engine 進程 | 多進程部署（OS 排程） |
| Cross-instance, 同進程 | workflow / function instance | 同一 event loop，event-driven 交錯 |
| Intra-instance, async-step | foreach iteration / parallel branch（背景） | 同一 event loop，event-driven 交錯 |
| Intra-instance, sync-step | foreach iteration / parallel branch（前景） | 同上 |
| Per-step external I/O | tool process、HTTP client、sub-instance | OS 排程 + non-blocking I/O 回 callback 至 loop |

同一 instance 任何時刻的「邏輯執行 thread」由 lease（跨進程）+ event loop（同進程）共同保證唯一；多 step 同時 RUNNING 是因為這些 step 在等待外部事件（async / wait / sub-instance），engine 並未真的多線程處理同一 instance 的 step transition。

---

## Foreach / Parallel 並行

### 阻塞模式（async: false）

```
def execute_foreach(step):
    items = evaluate(step.items or step.count)
    iterations = [build_iteration(i, item) for i, item in enumerate(items)]

    completed = 0
    failed_indices = []
    in_flight = []
    iter_outputs = [None] * len(iterations)

    while completed < len(iterations):
        # 補滿 concurrency
        while len(in_flight) < step.concurrency and pending(iterations):
            launch_iteration(next_pending(iterations))

        wait_for_any_iteration_completion()       # 透過 step.completed 事件喚醒

        for done in collect_done_iterations():
            in_flight.remove(done)
            completed += 1
            if done.status == "SUCCEEDED":
                iter_outputs[done.index] = done.output
            else:  # FAILED
                failed_indices.append(done.index)
                handle_failure_policy(step, done)
                if step.failure_policy == "fail_fast":
                    cancel_in_flight(in_flight)
                    return foreach_failed(step, done)

    return foreach_done(step, iter_outputs, failed_indices)
```

`failure_policy` 行為：

| 政策 | FAILED iteration 後 | iter_outputs 中 FAILED 位置 |
|------|---------------------|------------------------------|
| `fail_fast`（預設） | 取消未啟動 + 等待 in-flight 完成；step FAILED | n/a（step.output: null） |
| `continue` | 不取消；計入 failed_indices | `null` |
| `ignore` | 不取消；計入 failed_indices；step 仍 SUCCEEDED | `null` |

**取消尚未啟動的 iteration**：直接從 pending queue 移除。
**取消 in-flight iteration**：發 internal `instance.cancel` 至每個子分支；等待至 `cancel_grace_period`。

### 非阻塞模式（async: true）

```
def start_async_foreach(step):
    write_checkpoint(step.RUNNING.async_started)
    publish("step.async_started", { instance_id, step_id })
    schedule_background_iterations(step)
    return immediately(step.status = RUNNING.async_started)
```

背景 scheduler 持續按 concurrency 推進；達終態後寫 checkpoint、發 `step.completed`。

**取消傳播**：父 instance 被取消 / 超時時，背景 scheduler MUST 收到 `instance.cancel` 訊號：

- 尚未啟動的 iteration：從 pending queue 移除，該迭代的所有 step 標記為 FAILED（`error.type == "cancelled"`；step 層無 CANCELLED 狀態）；iteration 計入 foreach 的 failed_indices
- 已啟動的 iteration：依 `Cancel propagation` 章節流程發 cancel（exec SIGTERM / http close / sub-instance cancel）
- 等待 `cancel_grace_period` 後，未終態者強制 KILL
- 全部 iteration 達終態後：async foreach step **本身** 無獨立 CANCELLED state（step state enum 見 `08-persistence.md` 僅 WAITING / RUNNING / SUCCEEDED / FAILED / SKIPPED）；父 instance 若為 CANCELLED 路徑 → step 標記為 FAILED（`error.type == "cancelled"`），instance 本身進入 CANCELLED 終態；父 instance 若為 FAILED 路徑（timeout）→ step FAILED 向上傳播至 instance FAILED

避免「父 instance 已終結但背景 iteration 仍在跑」的僵屍 task。

對應的 `wait` step 在訂閱時：
- 若目標 step 已終態 → 直接讀取
- 否則訂閱 `step.completed` 事件

### Parallel 與 Foreach 共用 scheduler

`parallel.branches[]` 等同 `foreach` items 為 `[branch_0, branch_1, ...]`；scheduler 邏輯相同。差異：
- `concurrency` 預設為 `len(branches)`（全並行）
- `branches[]` 結構固定，不需 items 求值

### Concurrency slot 釋放時機（含 catch / compensate）

並行迭代（foreach iteration / parallel branch）的 concurrency slot **在迭代達到終態後才釋放**；「終態」定義為以下全部完成：

1. 該迭代的**最後一個 step** 終結（SUCCEEDED / FAILED / SKIPPED）
2. 若該 step FAILED 且迭代內有 `catch` handler，**catch 全部 step 亦達終態**
3. 若迭代本身位於 saga 內而進入 compensating，**該迭代對應的 compensate sequence 亦終態**

含義：`failure_policy: continue` / `ignore` 下，FAILED 迭代進入 catch 時 **concurrency slot 尚未釋放**；scheduler 不會啟動下一個 pending 迭代，直到本迭代的 catch 完成。此設計：

- 確保 catch handler 於穩定 concurrency 語意下執行（不會與新啟動迭代搶資源）
- 保留「先完成者先釋放 slot」的 happens-after 順序；使用者可於 catch 內假設本迭代的 `steps.*` 已定型
- `fail_fast` 下同樣適用：FAILED 迭代的 catch 完成後才進 foreach / parallel 的終結；其他 in-flight 迭代繼續至終態或被 cancel（見「取消 in-flight iteration」）

若 catch handler 本身執行時間很長，使用者應以 step-level `timeout` 或 tool-level `timeout` 控制；不建議將昂貴的補償邏輯放在 foreach / parallel 的 catch 內（改以 saga 結構表達更清晰）。

---

## Cancel propagation

```
graph: workflow_inst → step → sub_step → ... → tool_process
                     → sub_instance → ...
```

cancel 訊號從上向下傳播（狀態機：`RUNNING.*` → `RUNNING.cancelling` → `CANCELLED`）：

```
def cancel_instance(inst, reason):
    # 進入 RUNNING.cancelling 子態；state 仍為 RUNNING，不直接跳 CANCELLED
    inst.state    = RUNNING
    inst.substate = "cancelling"
    inst.cancel_requested = true
    inst.cancel_reason = reason
    write_checkpoint(inst)
    for child in inst.live_children():
        publish("instance.cancel", { target: child.id, reason })
    for in_flight_step in inst.live_steps():
        signal_step_cancel(in_flight_step)
    schedule_grace_deadline(inst, cancel_grace_period)
    # grace 到期或所有子節點終結後，於終結 checkpoint 將 state 改為 CANCELLED
```

step 接收 cancel：
- 若為 task step：發 cancel 至 sub-instance / tool driver
  - exec backend：對 process group SIGTERM；grace 後 SIGKILL
  - http backend：close connection
  - extension：cancel handler `ctx`（Go `context.Context` / Python `asyncio.CancelledError` / WASM component interrupt）；詳見 `06-tool-backend.md` 的「Cancel 傳播契約」
- 若為 wait：刪除 subscription；step → FAILED，`error.type == "cancelled"`（step 層無 CANCELLED 狀態，見 `04-expression-evaluation.md` StepRef 可見狀態規則）
- 若為 foreach/parallel：對所有 in-flight iteration / branch 發 cancel
- 若為 saga 在 compensating：補償執行不被中止（避免半補償狀態），但 saga 完成補償後直接終結

`cancel_grace_period` 預設 30s；可由 workflow.config 或 system config 覆寫。

**Saga 補償豁免 `cancel_grace_period`**：saga 一旦進入 `compensating`，其補償序列 MUST 執行至結束（不論 `cancel_grace_period` 是否到期），以避免半補償狀態；`cancel_grace_period` 不適用於 compensating 中的 saga step。其他並行分支仍受 grace period 約束，grace 到期後強制終結；但父 instance 不終結為 CANCELLED，直至最後一個 compensating saga 完成（SUCCEEDED 或 `compensation_failures` 計完）。若 compensate tool 本身需要上限，應以 tool 層 `timeout` 控制（而非 cancel grace）。

---

## Race condition 處置

### Wait subscription vs event arrival

事件到達瞬間 subscription 可能正在被 unsubscribe（timeout 觸發）。處置：

```
1. event 投遞與 subscription 取消由 bus 序列化（同 subscription_id 互斥）
2. 若 subscription 已取消，event 丟棄
3. 若 event 已開始投遞（已寫 step.completed），timeout 不再觸發（檢查 step.state）
```

### 多個訊號同時匹配 signals 模式

`signals: [...]` 的 any-of：第一個被引擎處理的 match 喚醒，其餘訂閱被取消。**「第一個被處理」**由 bus 投遞順序決定，非業務時間戳；建議使用者不依賴具體哪個訊號勝出。

### Lease vs cancel

cancel 訊號到達時，該 instance 可能由另一個 engine 進程持有 lease。處置：

```
1. cancel 訊號以無鎖 UPDATE 寫入 instance.cancel_requested = true、cancel_reason = <reason>
   （不需要 lease；見 `08-persistence.md` 的 instances schema）
2. 同 transaction 內 INSERT 一筆 outbox internal event `instance.cancel`
   （target: instance_id，scope: internal）以喚醒 lease 持有進程
3. Lease 持有進程 MUST 於下列時機讀取 cancel_requested 並進入 CANCELLED 流程：
   a. 每次 event loop dispatch 的起頭（收到 internal event `instance.cancel` 即觸發）
   b. 每次 checkpoint commit 前的 re-read（作為保底，防 bus 投遞漏失）
   c. Lease renew UPDATE 時，於 RETURNING 同時讀回 cancel_requested 欄位
4. 發現 cancel_requested → 取消 in-flight step、進入 CANCELLED 流程
```

- 寫入 `cancel_requested = true` 的 transaction commit 後，該值對所有讀者（READ COMMITTED 以上）立即可見；lease 持有進程於下一次讀取必能觀測到
- Lease 持有進程 crash（在 bus 訊號投遞前或讀取後）→ lease 過期後由其他 engine 接管；接管者於取得 lease 的 UPDATE 結果中即看到 `cancel_requested = true`，直接進入 CANCELLED 流程，不會遺失 cancel
- 不依賴 SERIALIZABLE：cancel 是**單調設定**（false → true，不回寫 false），即使 READ COMMITTED 下多次讀到不同值（stale → fresh），最終仍會觀測到 true

#### Cancel 偵測與執行的時序模型

Cancel 請求從觸發到完成需區分兩個階段：

| 階段 | 延遲上限 | 說明 |
|------|---------|------|
| **偵測延遲（detection）** | `min(bus 投遞延遲, checkpoint 間隔, lease TTL)` | 從 `cancel_requested = true` commit 到 lease 持有進程觀測到的時間 |
| **執行延遲（execution）** | `cancel_grace_period`（預設 30s） | 從 lease 持有進程開始執行 cancel 到所有 in-flight step / tool 終結；**自「lease 持有進程接收到 cancel」起算**，不含偵測延遲 |

**時序保證**：

- **典型情境（lease 持有者健在）**：總延遲 ≤ bus 投遞延遲 + `cancel_grace_period` ≈ ms 級 + 30s
- **持有者 crash 情境**：總延遲 ≤ lease TTL + `cancel_grace_period` ≈ 30s + 30s = 60s（實作建議 lease TTL ≤ 30s 以保證此上限）
- `cancel_grace_period` **不包含** lease 接管時間；接管者取得 lease 的那一刻重新開始 grace 倒數（避免「crash 前已耗用部分 grace」造成不確定性）
- **Saga 補償豁免**：compensating 中的 saga step 不受 `cancel_grace_period` 限制（見上「Saga 補償豁免」）；此類 instance 的總延遲由 compensate tool 自身 `timeout` 決定

**實作要點**：

- Lease 持有進程**進入 cancelling 子態時**寫入 `cancelling_started_at` 欄位（`instances` schema 擴充）；`cancel_grace_period` 自此時刻倒數
- 若 `cancelling_started_at + cancel_grace_period < now()` → engine 強制終結所有 in-flight step（exec SIGKILL、http force close、extension context cancel）
- Lease 接管時：若 `cancelling_started_at` 非 null，接管者**重置** `cancelling_started_at = now()` 並重新倒數（理由：原持有者可能 crash 於 cancel 發送前或 grace 中段，接管者無法確定子節點實際收到 cancel 的時間）
- 此欄位於 instance 終結（CANCELLED / SUCCEEDED / FAILED）時寫入終態的 checkpoint 中，供 observability 重建 cancel timeline

### Foreach 並行 iteration 的 vars 寫入

iteration 內的 `type: assign` 寫入 vars **共享於 instance**（vars 是 instance 級 namespace）。

**可見性規則（happens-after 語意）**：

| 讀取位置 | 看得到的寫入 |
|----------|--------------|
| 同一 iteration 內後續 step | 同 iteration 前面 `assign` 的所有寫入（立即可見） |
| 其他 iteration 內的 step | 不保證可見；可能看到初始值或其他 iteration 的部分寫入 |
| foreach 結束後的 step | 所有 iteration 的最終寫入（last-write-wins by assign 完成時間） |

實作層面：

- vars 寫入 MUST 為 atomic（單一 key 級別，以 DB row-level lock 達成）；無須提供整體 transaction
- 多 iteration 並行 assign 同一 key → 最終值由 **最後一個完成 assign 的 iteration 決定**（last-write-wins）；若需確定性輸出請用 step output 而非 vars
- **建議**在 foreach 內僅讀 vars，不寫；若必須寫請以 `loop.index` 區分 key（`vars["result_" + string(loop.index)] = ...`）

### Tool callback 與 result 競爭

tool 在發出 callback 後立即發 result（未等 callback_result）的情況：

```
1. driver 收到 callback → 啟動 handler 路徑
2. 立即收到 result → 若仍有 pending callback handler，等待 handler 完成
3. handler 完成 → 寫 callback_result 至 stdin（tool 已 close stdin？）
   - 若 stdin 已關閉，記錄 warning 但不影響 step.SUCCEEDED（result 已收到）
4. step.output 取自 result，不依賴 callback_result 是否被 tool 收到
```

引擎以 result 為終態的權威來源；callback 為 side-effect 通道。

---

## Concurrency 限制

實作 SHOULD 暴露下列可調限制：

| 設定 | 作用 |
|------|------|
| `engine.max_inflight_instances` | 單一 engine 進程同時處理的 instance 上限 |
| `engine.max_inflight_steps_per_instance` | foreach/parallel 全域 concurrency 上限（cap） |
| `engine.tool_process_pool_size` | 同時 spawn 的 process 上限 |
| `engine.http_pool_size` | http backend connection pool |
| `engine.event_bus_buffer_size` | 內部事件 buffer |
| `engine.callback_in_flight_per_step` | 單 step 同時等待中的 callback 上限 |
| `engine.cpu_worker_pool_size` | CPU-bound 下放（CEL 求值 / 大 JSON 處理）的 worker thread 數量 |
| `engine.max_function_call_depth` | Function 遞迴深度上限（預設 128） |

超過上限時：
- foreach concurrency 被 cap 到 max；不報錯
- inflight instance 上限：trigger 被延後（accept rate-limit）；或由其他 engine 進程接手
- process pool 滿：tool spawn 排隊；無 timeout 直至 step.timeout 觸發

---

## Deadlock 防範

引擎不偵測使用者層級 deadlock（如 wait 等待永遠不到的 event），由 timeout 處理。

引擎內部潛在 deadlock：

| 場景 | 防範 |
|------|------|
| 父 instance wait 子 instance、子 wait 父 | 父子 instance 同進程；engine 在啟動子 instance 時不阻塞 event loop（async dispatch） |
| 多 callback 互相等待對方 result | 每 callback 各自獨立 timeout |
| Saga 補償需要 wait 外部事件 | 補償 step 的 timeout 是必要；建議補償用 idempotent tool 而非 wait |

---

## 進度保證

- 若沒有外部事件到達，但 lease 仍持有，engine MUST 在「最近一個 wait 的 deadline」前喚醒並處理 timeout。
- timer 由 Engine Loop 內部維護；MUST 持久化（記錄 deadline at），engine 重啟後重排。
- 「永遠等不到」的 wait 受 wait.timeout 限制；無 timeout 則受 workflow.timeout 限制。
