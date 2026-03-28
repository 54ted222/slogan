# 03 — Event Router

本文件定義事件路由元件：管理 trigger subscriptions 與 wait subscriptions，將收到的事件分發至正確的處理流程。

---

## 職責

Event Router 處理所有進入引擎的事件，執行兩種路由：

| 路由類型 | 來源 | 動作 |
|---------|------|------|
| **Trigger 路由** | Workflow definition 的 event trigger | 建立新的 workflow instance |
| **Wait 路由** | Wait_event step 的 wait_subscription | 恢復等待中的 instance |

---

## 事件處理流程

```
SendEvent API
     ↓
持久化事件
     ↓
┌──────────────────────────────┐
│  Trigger Subscription 匹配   │
│  → 建立新 instances           │
└──────────────────────────────┘
     ↓
┌──────────────────────────────┐
│  Wait Subscription 匹配      │
│  → 恢復等待中的 instances      │
└──────────────────────────────┘
     ↓
回傳結果
```

一個事件 MAY 同時匹配多個 trigger subscriptions 和多個 wait subscriptions。

---

## Trigger Subscription 管理

### 建立時機

- Workflow definition 轉為 PUBLISHED 時，引擎為其 event triggers 建立 trigger subscriptions
- 每個 event trigger 對應一個 subscription

### Subscription 內容

| 欄位 | 說明 |
|------|------|
| `definition_id` | 所屬 workflow definition |
| `event_type` | 訂閱的事件類型 |
| `when_expression` | 過濾條件 CEL 表達式（可為 null） |
| `input_mapping` | 事件到 workflow input 的映射（可為 null） |

### 停用時機

- Definition 轉為 DEPRECATED 或 ARCHIVED → 移除對應的 trigger subscriptions

### 匹配流程

收到事件後，對每個匹配 `event_type` 的 trigger subscription：

1. 若有 `when_expression` → 以事件資料求值 CEL 表達式
   - `true` → 匹配成功
   - `false` → 跳過
   - 求值失敗 → 跳過，記錄錯誤至 log
2. 無 `when_expression` → 匹配成功
3. 匹配成功 → 建立 workflow input：
   - 有 `input_mapping` → 求值每個映射表達式，組合為 input map
   - 無 `input_mapping` → `event.data` 整體作為 input（passthrough）
4. 若有 `input.schema` → 驗證 input
   - 驗證失敗 → 不建立 instance，記錄錯誤至 log
5. 建立 workflow instance（等同 CreateInstance，但由事件觸發）
6. 通知 Scheduler

### Deduplication

同一事件 MAY 匹配同一 workflow definition 的多個 event triggers。引擎 SHOULD 提供 deduplication 機制：

- 建議以 `event_id + definition_id` 作為 deduplication key
- 在 deduplication window（建議 5 分鐘）內不重複建立 instance
- Deduplication 為 SHOULD（非 MUST），實作可根據需求調整

---

## Wait Subscription 管理

### 建立時機

- `wait_event` step 進入 WAITING 狀態時，Scheduler 建立 wait_subscription

### Subscription 內容

見 [dsl-spec/v2/15-runtime-persistence](../dsl-spec/v2/15-runtime-persistence.md) 的 wait_subscription 實體。

### 匹配流程

收到事件後，對每個匹配 `event_type` 的 wait_subscription：

1. 檢查是否已過期（`expires_at` < 當前時間）→ 是 → 跳過（由 Timeout Manager 處理）
2. 載入對應的 workflow instance 狀態（用於 CEL 求值上下文）
3. 若有 `match_expression` → 求值 CEL 表達式
   - 求值上下文包含：`event`（候選事件）、`input`、`steps`、`vars`（instance 資料）
   - `true` → 匹配成功
   - `false` → 跳過
   - 求值失敗 → 跳過，記錄錯誤至 log
4. 無 `match_expression` → 匹配成功
5. 匹配成功：
   a. 刪除 wait_subscription
   b. 刪除對應的 timeout_schedule（若有）
   c. 更新 step instance：state → RUNNING → SUCCEEDED，output = `event.data`
   d. 更新 instance state：WAITING → RUNNING
   e. 通知 Scheduler 推進後續 steps

### 一個事件匹配多個 Wait Subscriptions

- 一個事件 MAY 匹配多個不同 instance 的 wait subscriptions
- 每個匹配的 subscription 獨立處理
- 同一 instance 的同一 step 最多只有一個 wait subscription

---

## 事件持久化

### 事件儲存

引擎 SHOULD 持久化收到的事件，用於：

| 用途 | 說明 |
|------|------|
| 稽核追蹤 | 記錄哪個事件觸發了哪個 instance |
| Replay | Crash recovery 後重新處理未完成的事件 |
| 除錯 | 查詢事件歷史 |

### 事件 Schema

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別 |
| `type` | string | 事件類型 |
| `data` | map | 事件酬載 |
| `source` | string | 事件來源 |
| `timestamp` | timestamp | 事件時間 |
| `created_at` | timestamp | 引擎收到時間 |

### 保留策略

事件資料的保留期限由實作決定。建議：

- 至少保留至所有相關 instances 完成後 24 小時
- 可設定全域保留期限（如 30 天）
- 過期事件可被清理

---

## 內部事件（emit step 產生）

`emit` step 產生的事件透過相同的 Event Router 處理：

1. Step Executor 執行 emit step → 產生事件
2. 事件進入 Event Router（與外部 SendEvent 相同流程）
3. 可觸發其他 workflow 的 trigger subscriptions
4. 可匹配其他 instance 的 wait subscriptions

這確保內部事件與外部事件的行為完全一致。
