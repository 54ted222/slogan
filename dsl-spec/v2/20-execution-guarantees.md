# 20 — Execution Guarantees

本文件定義 task 執行的 delivery guarantee、冪等 key 在 recovery 中的行為、HTTP retry 與 task retry 的區分、以及邊界條件下的 race condition 處理。

---

## Delivery Guarantee

### 整體語意

引擎對 task 執行提供 **at-least-once** 保證：

| 保證 | 說明 |
|------|------|
| At-least-once | 每個 task step 至少被嘗試執行一次。Crash 後 recovery 可能導致重複執行 |
| **非** exactly-once | 引擎無法保證 task 只被執行恰好一次。需要 exactly-once 語意的 task 應透過 `execution.policy` + `idempotency key` 自行實現 |

### 各 Execution Policy 的語意

| Policy | Crash Recovery 行為 | 對 task handler 的要求 |
|--------|--------------------|-----------------------|
| `replayable` | 無條件重跑 | Handler MUST 為冪等，或可接受重複執行（如 read-only 操作、有補償邏輯的操作） |
| `idempotent` | 帶 idempotency key 重跑 | Handler SHOULD 依 idempotency key 去重，確保重複呼叫不產生副作用 |
| `non_repeatable` | 直接標記 FAILED | Handler 不需要冪等。引擎寧可失敗也不重複執行（適用於扣款、發送通知等） |

### emit step 的保證

emit step 提供 **at-least-once** 保證：

- 事件 MUST 在 step 標記為 SUCCEEDED 之前被持久化
- Crash recovery 後，若 step 狀態為 RUNNING → 重新 emit（可能重複發送）
- 下游消費者 SHOULD 能處理重複事件（透過 `event.id` 去重）

---

## Idempotency Key 詳細規格

### Key 格式

```
{workflow_instance_id}:{step_id}:{attempt}
```

- `attempt` 從 1 開始
- 每次 retry 遞增 attempt，產生不同的 key
- Crash recovery 重跑**不**遞增 attempt（使用 crash 前的 attempt 值），確保重跑使用相同的 key

### 傳遞方式

| Backend | 傳遞位置 |
|---------|---------|
| stdio | `context.idempotency_key`（request JSON 欄位）+ `SLOGAN_IDEMPOTENCY_KEY` 環境變數 |
| http | `Idempotency-Key` HTTP header |
| builtin | 引擎內部處理，不外傳 |

### Handler 端實作建議

Tool handler 收到 idempotency key 後 SHOULD：

1. 以 key 查詢是否已處理過（如資料庫記錄、external API 的 idempotency 機制）
2. 若已處理 → 回傳之前的結果，不重複執行
3. 若未處理 → 正常執行，並記錄 key 與結果
4. Key 的保留期限由 handler 自行決定（建議至少 24 小時）

### Recovery 場景

```
Step RUNNING → Engine crash
     ↓
Engine restart → 載入 step instance（state = RUNNING, attempt = N）
     ↓
依 execution.policy 決定：
  replayable    → 重跑（attempt 不變 = N）
  idempotent    → 帶 key 重跑（key = instance:step:N，與 crash 前相同）
  non_repeatable → 標記 FAILED
```

重點：`idempotent` policy 在 crash recovery 時使用與 crash 前相同的 key，讓 handler 能辨識為重複請求。

---

## HTTP Retry 與 Task Retry 的關係

Task step 的 `retry` 機制與 HTTP backend 的 `retry_on_status` 是兩個**獨立的層級**：

### 層級區分

```
Task Step retry（workflow 層）
  ↓ 每次 retry attempt 呼叫一次 Task Executor
  ↓
Task Executor（HTTP backend）
  ↓ 依 retry_on_status 判斷是否為暫時性失敗
  ↓
HTTP 請求結果
```

### 交互行為

| HTTP 結果 | `retry_on_status` 匹配 | Task Executor 回報 | Task Step retry |
|-----------|----------------------|-------------------|----------------|
| 2xx | — | 成功 | 不觸發 |
| 429 | 在列表中 | 可重試失敗 | 觸發（若 attempts 未用盡） |
| 500 | 不在列表中 | 永久性失敗 | 觸發（若 attempts 未用盡） |
| 連線失敗 | — | 可重試失敗 | 觸發（若 attempts 未用盡） |

### 重要區別

- **HTTP backend 不自行重試**：`retry_on_status` 僅影響 Task Executor 回報的失敗類型，不會在 HTTP 層做重試
- **Task step retry 控制所有重試**：無論是暫時性還是永久性失敗，都由 task step 的 `retry` 設定統一控制
- **每次 retry attempt = 一次完整的 HTTP 請求**

### 可重試失敗 vs 永久性失敗

| 失敗類型 | 說明 | retry 行為 |
|---------|------|-----------|
| 可重試（transient） | 暫時性問題，重試可能成功 | 正常 retry（respect delay + backoff） |
| 永久性（permanent） | 不可能透過重試解決 | 仍然 retry（引擎不區分），直到 attempts 用盡 |

引擎**不區分**可重試與永久性失敗的 retry 行為。無論失敗類型，只要 `max_attempts` 未用盡就重試。區分的用途在於讓使用者在 `on_error` handler 中透過 `error.code` 判斷失敗原因。

---

## Race Condition 處理

### Timeout 與 Event 同時到達

當 `wait_event` step 的 timeout 和匹配事件幾乎同時到達時：

```
Timeline:
  t=0    wait_event 進入 WAITING
  t=30m  timeout fires_at

  Case A: event 在 t=29m59s 到達，timeout 在 t=30m 觸發
  Case B: timeout 在 t=30m 觸發，event 在 t=30m01s 到達
```

#### 解決規則

引擎 MUST 使用以下規則確保唯一贏家：

1. **Atomic check-and-update**：無論是 event 匹配還是 timeout 觸發，更新 step 狀態前 MUST 檢查 step 是否仍為 WAITING
2. **先到先贏**：第一個成功將 step 狀態從 WAITING 更新為其他狀態的操作為贏家
3. **Optimistic locking**：使用 step_instance 的 version column 防止並行更新

| 贏家 | 輸家行為 |
|------|---------|
| Event 先到 | Timeout 觸發時發現 step 不再是 WAITING → 忽略 |
| Timeout 先到 | Event 匹配時發現 step 不再是 WAITING → 忽略（事件不會被消費，其他 subscription 可能匹配） |

#### 對 Wait Subscription 的清理

贏家操作 MUST 在同一 transaction 中：
- 刪除 wait_subscription
- 刪除 timeout_schedule（若存在）

### Cancellation 與 Step 執行的 Race

當 instance 被取消（API cancel 或 parent timeout）時，可能有 step 正在執行：

| Step 狀態 | Cancellation 行為 |
|-----------|------------------|
| RUNNING（task） | 引擎嘗試終止 task 執行（stdio: SIGTERM → SIGKILL；http: 不等待 response）。Step → CANCELLED |
| RUNNING（sub_workflow） | Child instance → CANCELLED |
| WAITING | 刪除 wait_subscription，step → CANCELLED |
| READY | Step → CANCELLED（不執行） |
| PENDING | Step → CANCELLED |

### Parallel Branch 失敗與 fail_fast 的 Race

當 `parallel` 的 `failure_policy` 為 `fail_fast` 時，一個 branch 失敗要取消其他 branches：

1. Branch A 失敗
2. 引擎設定 parallel step 為 FAILED
3. 引擎對其他仍在執行的 branches 發送 cancellation
4. 其他 branches 的 step MAY 在收到 cancellation 之前完成 → 完成結果被忽略

引擎 MUST 確保：
- Parallel step 的最終狀態只被設定一次
- 使用 optimistic locking 防止多個 branch 同時嘗試設定 parallel step 狀態
