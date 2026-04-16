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

- 尚未啟動的 iteration：從 pending queue 移除，狀態直接標記為 CANCELLED
- 已啟動的 iteration：依 `Cancel propagation` 章節流程發 cancel（exec SIGTERM / http close / sub-instance cancel）
- 等待 `cancel_grace_period` 後，未終態者強制 KILL
- 全部 iteration 達終態後寫 async foreach 的終態 checkpoint（CANCELLED）

避免「父 instance 已終結但背景 iteration 仍在跑」的僵屍 task。

對應的 `wait` step 在訂閱時：
- 若目標 step 已終態 → 直接讀取
- 否則訂閱 `step.completed` 事件

### Parallel 與 Foreach 共用 scheduler

`parallel.branches[]` 等同 `foreach` items 為 `[branch_0, branch_1, ...]`；scheduler 邏輯相同。差異：
- `concurrency` 預設為 `len(branches)`（全並行）
- `branches[]` 結構固定，不需 items 求值

---

## Cancel propagation

```
graph: workflow_inst → step → sub_step → ... → tool_process
                     → sub_instance → ...
```

cancel 訊號從上向下傳播：

```
def cancel_instance(inst, reason):
    inst.state = CANCELLED.pending
    write_checkpoint(inst)
    for child in inst.live_children():
        publish("instance.cancel", { target: child.id, reason })
    for in_flight_step in inst.live_steps():
        signal_step_cancel(in_flight_step)
    schedule_grace_deadline(inst, cancel_grace_period)
```

step 接收 cancel：
- 若為 task step：發 cancel 至 sub-instance / tool driver
  - exec backend：對 process group SIGTERM；grace 後 SIGKILL
  - http backend：close connection
  - extension：呼叫 handler.cancel()
- 若為 wait：刪除 subscription；step → CANCELLED
- 若為 foreach/parallel：對所有 in-flight iteration / branch 發 cancel
- 若為 saga 在 compensating：補償執行不被中止（避免半補償狀態），但 saga 完成補償後直接終結

`cancel_grace_period` 預設 30s；可由 workflow.config 或 system config 覆寫。

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
1. cancel 訊號寫入 instance.cancel_requested = true（不需要 lease）
2. lease 持有進程在每次 checkpoint 邊界檢查此 flag
3. 發現 cancel_requested → 進入 CANCELLED 流程
```

最差情況下 lease TTL 內 cancel 不生效；可由實作提供 lease invalidation 加速（例如透過 event bus 廣播 `instance.cancel` 給擁有者進程的 event loop，讓其在下一個事件分派 tick 處理）。

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
