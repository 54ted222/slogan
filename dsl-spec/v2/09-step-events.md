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
