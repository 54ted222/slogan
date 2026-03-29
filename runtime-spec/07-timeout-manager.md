# 07 — Timeout Manager

本文件定義 Timeout Manager 的行為：管理 timeout 排程、觸發、以及 crash recovery 後的重建。

---

## 職責

Timeout Manager 負責：

1. 建立 timeout schedules（step timeout、workflow timeout）
2. 監控 timeout 到期
3. 到期時通知 Scheduler 執行 timeout 處理
4. 管理 delayed event 投遞排程
5. 管理 scheduled trigger 觸發排程

---

## Timeout 類型

| 類型 | 設定位置 | 起算點 | 影響範圍 |
|------|---------|-------|---------|
| **Step timeout** | step 的 `timeout` 欄位 | step 進入 RUNNING 時 | 該 step |
| **Wait timeout** | `wait_event` 的 `timeout` 欄位 | step 進入 WAITING 時 | 該 wait_event step |
| **Workflow timeout** | `config.timeout` | instance 從 CREATED 建立時 | 整個 instance |

---

## 排程建立

### Step Timeout

Step Executor 在 step 進入 RUNNING 時，若 step 有 `timeout` 設定：

1. 計算 `fires_at` = `now()` + timeout duration
2. 建立 timeout_schedule record：
   - `type` = `step_timeout`
   - `instance_id` = 當前 instance
   - `step_id` = 當前 step
   - `fires_at` = 計算值

適用 step types：`task`、`sub_workflow`。

### Wait Timeout

Step Executor 在 `wait_event` step 進入 WAITING 時：

1. 計算 `fires_at` = `now()` + timeout duration
2. 建立 timeout_schedule record：
   - `type` = `step_timeout`
   - `instance_id` = 當前 instance
   - `step_id` = 當前 step
   - `fires_at` = 計算值
3. 同時更新 wait_subscription 的 `expires_at` = `fires_at`

### Workflow Timeout

Engine API 在建立 instance 時，若 definition 有 `config.timeout`：

1. 計算 `fires_at` = `now()` + config.timeout duration
2. 建立 timeout_schedule record：
   - `type` = `workflow_timeout`
   - `instance_id` = 當前 instance
   - `step_id` = null
   - `fires_at` = 計算值

---

## 到期監控

Timeout Manager MUST 持續監控 timeout_schedule，在 `fires_at` 到期時觸發處理。

### 監控策略

| 策略 | 說明 |
|------|------|
| Polling | 定期查詢 `fires_at <= now()` 的 schedules（建議間隔 1s） |
| Timer-based | 為每個 schedule 設定 in-memory timer |
| 混合 | 近期 schedules 用 timer，遠期用 polling |

具體策略由實作決定。引擎 MUST 確保到期的 timeout 在合理時間內被處理（建議延遲 < 2s）。

---

## 到期處理

### 並行安全

Timeout 到期處理 MUST 以原子方式執行狀態檢查與轉換。實作 SHOULD 使用以下策略之一：

- **Pessimistic locking**：載入 step/instance 狀態時使用 row-level lock（如 `SELECT FOR UPDATE`）
- **Optimistic locking**：使用 step_instance 的 `version` 欄位，若 version 已變更則放棄處理（其他 worker 已處理）

這防止 Timeout Manager 與 Step Executor 同時修改同一 step 的 race condition。

### Step Timeout 到期

```
timeout_schedule fires
     ↓
載入 step instance 狀態（需取得 lock 或檢查 version）
     ↓
Step 是否仍在 RUNNING？
  ├── 否（已完成）→ 刪除 schedule，結束
  └── 是 →
       ↓
  終止執行（task 執行或 sub_workflow child instance）
  若為 sub_workflow step → child instance → CANCELLED
    （child 的 config.on_timeout 不觸發，因為是外部取消）
       ↓
  Step state → TIMED_OUT
       ↓
  刪除 timeout_schedule
       ↓
  檢查 on_timeout handler
       ├── 有 → 建立 handler steps，執行
       └── 無 → 視為 FAILED，進入 error handling flow
       ↓
  通知 Scheduler
```

### Wait Timeout 到期

```
timeout_schedule fires
     ↓
載入 step instance 狀態
     ↓
Step 是否仍在 WAITING？
  ├── 否（已收到事件）→ 刪除 schedule，結束
  └── 是 →
       ↓
  刪除 wait_subscription
       ↓
  Step state → TIMED_OUT
       ↓
  Instance state → RUNNING（從 WAITING 恢復以處理 timeout）
       ↓
  刪除 timeout_schedule
       ↓
  檢查 on_timeout handler
       ├── 有 → 建立 handler steps，執行
       └── 無 → 視為 FAILED，進入 error handling flow
       ↓
  通知 Scheduler
```

### Workflow Timeout 到期

```
timeout_schedule fires
     ↓
載入 instance 狀態
     ↓
Instance 是否仍為非 terminal？
  ├── 否（已完成）→ 刪除 schedule，結束
  └── 是 →
       ↓
  所有 RUNNING、WAITING、READY 的 steps → CANCELLED
       ↓
  終止所有進行中的 task 執行
       ↓
  刪除所有相關的 timeout_schedules 和 wait_subscriptions
       ↓
  取消所有 child instances（sub_workflow）
       ↓
  檢查 config.on_timeout handler
       ├── 有 → 建構 timeout namespace（timeout.step_id = null、timeout.duration = config.timeout）
       │        → 執行 handler（handler 完成後 instance → FAILED）
       └── 無 → instance → FAILED
       ↓
  刪除 workflow timeout_schedule
```

---

## 排程清理

Timeout schedules 在以下情況 MUST 被刪除：

| 情況 | 動作 |
|------|------|
| Step 正常完成（SUCCEEDED） | 刪除該 step 的 timeout_schedule |
| Wait 收到匹配事件 | 刪除該 step 的 timeout_schedule |
| Instance 到達 terminal 狀態 | 刪除該 instance 所有 timeout_schedules |
| Instance 被取消 | 刪除該 instance 所有 timeout_schedules |

---

## Crash Recovery

引擎重啟時，Timeout Manager MUST：

1. 載入所有未到期的 timeout_schedules（instance 為非 terminal 狀態）
2. 對已過期的 schedules（`fires_at` <= 當前時間）→ 立即觸發到期處理
3. 對未過期的 schedules → 重新建立監控

### 時間漂移

Crash recovery 期間可能產生時間漂移（實際過期時間晚於 `fires_at`）。引擎 MUST 仍按規則處理，不因延遲而改變行為（如已過期的 wait_event 仍執行 timeout flow）。

---

## Delayed Event 投遞

Timeout Manager 同時負責 delayed event 的到期投遞（見 [dsl-spec/v2/09-step-events](../dsl-spec/v2/09-step-events.md) emit delay 規格）。

### 投遞流程

```
delayed_event.deliver_at 到期
     ↓
載入 delayed_event 記錄
     ↓
組合事件信封（type, data, 自動產生 id, timestamp = deliver_at）
     ↓
投遞至 Event Router（與 emit step 產生的即時事件相同流程）
     ↓
刪除 delayed_event 記錄
```

### 投遞失敗

若投遞至 Event Router 失敗（如 storage 不可用）：

- 不刪除 delayed_event 記錄
- 引擎 SHOULD 在下一個監控週期重試
- 重試次數由引擎設定決定（建議上限 10 次）
- 超過重試上限 → 記錄錯誤至 log，刪除記錄

### Crash Recovery

引擎重啟時，MUST 載入所有未投遞的 delayed_events，已過期者立即投遞。

---

## Scheduled Trigger 排程

Timeout Manager 負責 scheduled trigger 的定時觸發。

### 觸發流程

```
scheduled_trigger.next_fire_at 到期
     ↓
檢查 allow_concurrent（若 false → 查詢是否有非 terminal instance）
     ├── 有非 terminal instance → 跳過本次觸發
     └── 無（或 allow_concurrent = true）→
          ↓
     求值 input / input_mapping（見下方 Namespace）
          ↓
     建立 workflow instance（等同 CreateInstance）
          ↓
     遞增 iteration_count
          ↓
     計算 next_fire_at（依 cron 或 interval）
          ↓
     更新 scheduled_trigger 記錄
```

### input_mapping Namespace

`input_mapping` 求值時，Expression Evaluator 提供 `schedule` namespace（與 [dsl-spec/v2/02-triggers](../dsl-spec/v2/02-triggers.md) 定義一致）：

| 變數 | 型別 | 說明 |
|------|------|------|
| `schedule.fired_at` | timestamp | 本次觸發的時間（`next_fire_at` 到期時間） |
| `schedule.iteration` | integer | 自 definition PUBLISHED 以來的累計觸發次數（= `iteration_count + 1`，因 `iteration_count` 在建立 instance 後才遞增） |

靜態 `input`（非 `input_mapping`）不需要 namespace，直接作為 instance input。

### Crash Recovery

引擎重啟時：

- 載入所有 scheduled_trigger 記錄
- 已過期的 `next_fire_at` → 依據引擎設定決定是否補觸發（預設不補觸發）
- 更新 `next_fire_at` 為下一次應觸發的時間
