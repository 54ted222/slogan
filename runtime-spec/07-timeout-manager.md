# 07 — Timeout Manager

本文件定義 Timeout Manager 的行為：管理 timeout 排程、觸發、以及 crash recovery 後的重建。

---

## 職責

Timeout Manager 負責：

1. 建立 timeout schedules（step timeout、workflow timeout）
2. 監控 timeout 到期
3. 到期時通知 Scheduler 執行 timeout 處理

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

### Step Timeout 到期

```
timeout_schedule fires
     ↓
載入 step instance 狀態
     ↓
Step 是否仍在 RUNNING？
  ├── 否（已完成）→ 刪除 schedule，結束
  └── 是 →
       ↓
  終止 task 執行（若正在進行）
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
       ├── 有 → 執行 handler（handler 完成後 instance → FAILED）
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
