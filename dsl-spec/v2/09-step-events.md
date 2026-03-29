# 09 — Steps: emit / wait_event

本文件定義事件相關的兩種 step 類型。

---

## 事件模型概述

事件是系統內的具名訊息，具有標準信封格式：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `type` | string | 事件類型（如 `order.created`） |
| `data` | map | 事件酬載 |
| `id` | string | 事件唯一 ID |
| `timestamp` | timestamp | 事件產生時間 |
| `source` | string | 事件來源 |

事件在本 DSL 中有三種角色：

| 角色 | 機制 | 說明 |
|------|------|------|
| 建立 instance | trigger | 收到事件 → 建立新的 workflow instance |
| 發送事件 | emit | 發布事件至 event bus |
| 恢復 instance | wait_event | 等待事件 → 恢復暫停的 instance |

---

## emit

發送事件至 event bus。

### Schema

```yaml
- type: emit
  event: string                  # MUST — 事件類型
  data: map | CEL expression     # MAY — 事件酬載
```

### 行為

- 將事件發布至 event bus（fire-and-forget）
- 事件 MUST 在 step 標記為 SUCCEEDED 之前被持久化（at-least-once 保證）
- 若 `data` 中的 CEL 表達式求值失敗 → step FAILED
- emit 本身不等待任何回應

### 範例

```yaml
- type: emit
  event: order.completed
  data:
    order_id: ${ steps.load_order.output.id }
    status: "completed"
    completed_at: ${ now() }
```

---

## wait_event

暫停 workflow instance，等待匹配的外部事件。

### Schema

```yaml
- id: string                     # MAY — 需要參照 output 時才必填
  type: wait_event
  event: string                  # MUST — 要等待的事件類型
  match: CEL expression          # MAY — 事件匹配條件，回傳 boolean
  timeout: duration              # MAY — 等待逾時
  on_timeout: [...]              # MAY, step 陣列
```

### 行為

1. Step 進入 RUNNING 狀態
2. 引擎建立 **wait_subscription**，記錄：instance ID、step ID、event type、match expression、expires_at
3. Step 狀態轉為 WAITING，instance 狀態轉為 WAITING
4. Instance 暫停（不佔用運算資源，非 blocking）
5. 當引擎收到匹配的事件：
   - 求值 `match` 表達式（若有），`true` → 匹配成功
   - 無 `match` → 只要 event type 相同即匹配
   - 匹配成功 → step 恢復為 RUNNING → SUCCEEDED
   - 事件的 `data` 成為此 step 的 output（`steps.<id>.output` = `event.data`）
6. Instance 恢復為 RUNNING，繼續後續 steps

### match 的求值上下文

`match` 表達式中可用的 namespaces：

- `event.data`、`event.type`、`event.id` 等（當前候選事件）
- `input`、`steps`、`vars`（workflow instance 的資料）

```yaml
match: ${ event.data.order_id == input.order_id && event.data.status == "confirmed" }
```

### timeout 行為

- 超過 `timeout` 且仍未收到匹配事件 → step TIMED_OUT
- 若有 `on_timeout` → 執行 handler steps
  - Handler 正常完成 → workflow 繼續後續 steps
  - Handler 內使用 `fail` → 錯誤重新拋出，繼續向上層尋找 handler（見 [12-error-handling](12-error-handling.md)）
- 若無 `on_timeout` → 視為未處理錯誤，進入錯誤處理流程（block-level → workflow-level `on_error`），詳見 [12-error-handling](12-error-handling.md)

### 範例

```yaml
- id: wait_payment
  type: wait_event
  event: payment.confirmed
  match: ${ event.data.order_id == steps.load_order.output.id }
  timeout: 30m
  on_timeout:
    - type: emit
      event: payment.timeout
      data:
        order_id: ${ steps.load_order.output.id }
    - type: fail
      message: "payment confirmation timeout"
```

---

## emit 的延遲發送

`emit` step 支援 `delay` 欄位，指定事件在延遲一段時間後才投遞至 Event Router。

### Schema

```yaml
- type: emit
  event: string                  # MUST — 事件類型
  data: map | CEL expression     # MAY — 事件酬載
  delay: duration                # MAY — 延遲時間
```

### 行為

| 情境 | 行為 |
|------|------|
| 無 `delay` | 事件立即投遞（現有行為，不變） |
| 有 `delay` | 事件被排程，在 `delay` 時間後才投遞至 Event Router |

### 延遲事件的持久化

1. emit step 執行時，引擎建立 `delayed_event` 記錄並持久化
2. Step 標記為 SUCCEEDED（不等待事件實際投遞）
3. 引擎的 Timeout Manager（或獨立的 delayed event scheduler）在到期時投遞事件
4. 投遞後刪除 `delayed_event` 記錄

### delayed_event 實體

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別 |
| `event_type` | string | 事件類型 |
| `event_data` | json | 事件酬載 |
| `source_instance_id` | string | 產生此事件的 instance |
| `source_step_id` | string | 產生此事件的 step |
| `deliver_at` | timestamp | 預定投遞時間 |
| `created_at` | timestamp | 建立時間 |

### Crash Recovery

- 引擎重啟時 MUST 重建所有未投遞的 delayed events
- 已過期的 delayed events（`deliver_at` <= 當前時間）→ 立即投遞

### 投遞失敗重試

若延遲事件投遞至 Event Router 時失敗（如 storage 暫時不可用），引擎 SHOULD 重試投遞，建議最多 10 次。超過重試上限 → 記錄錯誤至 execution_log 並刪除該 delayed_event 記錄。

### Instance 取消的影響

若產生 delayed event 的 instance 在事件投遞前被取消：

- 延遲事件**仍然投遞**（fire-and-forget 語意）
- 引擎 MAY 提供設定以在 instance 取消時撤銷未投遞的 delayed events

### 範例

```yaml
# 30 分鐘後發送提醒
- type: emit
  event: order.reminder
  delay: 30m
  data:
    order_id: ${ steps.load_order.output.id }
    message: "your order is waiting for payment"

# 24 小時後自動取消
- type: emit
  event: order.auto_cancel
  delay: 24h
  data:
    order_id: ${ input.order_id }
```
