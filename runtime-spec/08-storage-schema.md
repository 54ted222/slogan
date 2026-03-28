# 08 — Storage Schema

本文件定義引擎持久化層的完整 schema 與操作規範。補充 [dsl-spec/v2/15-runtime-persistence](../dsl-spec/v2/15-runtime-persistence.md) 中的實體定義，加入實作所需的細節。

---

## 設計原則

- Storage 為抽象介面，不綁定特定資料庫
- 所有操作定義為邏輯操作，由實作對應至具體 SQL / NoSQL / KV 語意
- Step instance 更新 SHOULD 使用 optimistic locking（version column）
- **預設實作**：引擎 MUST 內建 SQLite 作為預設 storage backend，零配置即可啟動。生產環境 MAY 替換為 PostgreSQL 等其他後端

---

## 實體 Schema

### definition（通用）

涵蓋 Workflow、Task、Secret 三種 kind。

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別（自動產生） |
| `kind` | string | `Workflow` / `Task` / `Secret` |
| `name` | string | `metadata.name` |
| `version` | integer | `metadata.version` |
| `content` | text | YAML 定義原文 |
| `parsed_content` | json | 解析後的結構化內容（供引擎直接使用） |
| `lifecycle_state` | string | DRAFT / VALIDATED / PUBLISHED / DEPRECATED / ARCHIVED |
| `backend_type` | string \| null | 僅 Task：bash / http / builtin / stdio |
| `created_at` | timestamp | 建立時間 |
| `updated_at` | timestamp | 最後更新時間 |

**唯一約束：** `(kind, name, version)`

### workflow_instance

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別 |
| `definition_id` | string | 所屬 workflow definition |
| `state` | string | CREATED / RUNNING / WAITING / SUCCEEDED / FAILED / CANCELLED |
| `input` | json | instance 輸入資料 |
| `output` | json \| null | instance 輸出（完成後） |
| `error` | json \| null | 錯誤資訊（失敗時，含 `message`、`code`） |
| `vars` | json | 當前 vars namespace snapshot（assign step 更新） |
| `parent_instance_id` | string \| null | 父 instance（sub_workflow） |
| `parent_step_id` | string \| null | 父 step（sub_workflow） |
| `step_execution_count` | integer | 累計 step 執行次數（`config.max_step_executions` 檢查用） |
| `created_at` | timestamp | 建立時間 |
| `started_at` | timestamp \| null | 開始時間 |
| `completed_at` | timestamp \| null | 完成時間 |
| `version` | integer | Optimistic locking 版本號 |

### step_instance

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別 |
| `workflow_instance_id` | string | 所屬 instance |
| `step_id` | string | 對應 definition 中的 step id |
| `step_type` | string | step 類型（task / assign / if / ...） |
| `parent_step_id` | string \| null | 父容器 step（if/switch/foreach/parallel 內部的 step） |
| `iteration_index` | integer \| null | foreach 迭代索引（僅 foreach 內部 step） |
| `branch_index` | integer \| null | parallel branch 索引（僅 parallel 內部 step） |
| `state` | string | PENDING / READY / RUNNING / SUCCEEDED / FAILED / WAITING / TIMED_OUT / CANCELLED / SKIPPED |
| `input` | json \| null | 求值後的 step 輸入 |
| `output` | json \| null | step 輸出 |
| `error` | json \| null | 錯誤資訊（含 `message`、`code`） |
| `attempt` | integer | 當前嘗試次數（從 1 開始） |
| `recorded_values` | json \| null | 非確定性函式記錄（見 [06-expression-evaluator](06-expression-evaluator.md)） |
| `started_at` | timestamp \| null | 開始時間 |
| `completed_at` | timestamp \| null | 完成時間 |
| `version` | integer | Optimistic locking 版本號 |

**recorded_values 格式：**

```json
{
  "load_order.input.created_at[0]": "2024-01-15T10:30:00Z",
  "load_order.input.id[0]": "550e8400-e29b-41d4-a716-446655440000"
}
```

### wait_subscription

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別 |
| `workflow_instance_id` | string | 所屬 instance |
| `step_id` | string | 所屬 step |
| `event_type` | string | 等待的事件類型 |
| `match_expression` | string | CEL 匹配表達式 |
| `created_at` | timestamp | 建立時間 |
| `expires_at` | timestamp \| null | 過期時間（根據 timeout 計算） |

### timeout_schedule

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別 |
| `workflow_instance_id` | string | 所屬 instance |
| `step_id` | string \| null | 所屬 step（null = workflow 級 timeout） |
| `type` | string | `step_timeout` / `workflow_timeout` |
| `fires_at` | timestamp | 觸發時間 |

### trigger_subscription

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別 |
| `definition_id` | string | 所屬 workflow definition |
| `event_type` | string | 訂閱的事件類型 |
| `when_expression` | string \| null | CEL 過濾表達式（對應 trigger 的 `when`） |
| `input_mapping` | json \| null | 事件到 workflow input 的映射（對應 trigger 的 `input_mapping`） |
| `created_at` | timestamp | 建立時間 |

在 definition PUBLISHED 時建立，DEPRECATED 時刪除。

### event_record

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 事件唯一 ID |
| `type` | string | 事件類型 |
| `source` | string \| null | 事件來源 |
| `data` | json | 事件酬載 |
| `timestamp` | timestamp | 事件發生時間 |
| `created_at` | timestamp | 引擎收到時間 |

### execution_log

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別 |
| `workflow_instance_id` | string | 所屬 instance |
| `step_id` | string \| null | 所屬 step（workflow 級事件為 null） |
| `event_type` | string | 事件類型（見下方） |
| `data` | json | 事件詳細資料（見下方） |
| `created_at` | timestamp | 記錄時間 |

Execution log 為 append-only，用於稽核與除錯。

#### event_type 與 data 格式

| event_type | data 欄位 | 說明 |
|------------|----------|------|
| `state_changed` | `{ entity: "instance"\|"step", from_state, to_state }` | 狀態轉換 |
| `retry` | `{ step_id, attempt, error_message, delay_ms }` | 重試事件 |
| `event_received` | `{ event_id, event_type, matched_subscription_id }` | 收到匹配事件 |
| `timeout_fired` | `{ timeout_type, step_id }` | Timeout 觸發 |
| `handler_invoked` | `{ handler_level: "step"\|"block"\|"workflow", handler_type: "on_error"\|"on_timeout" }` | Error/timeout handler 啟動 |
| `task_log` | `{ content }` | Task stderr 輸出（截取前 64KB） |
| `cancellation` | `{ reason, cancelled_steps: [...] }` | Instance 取消 |

### artifact_record

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 唯一識別 |
| `workflow_instance_id` | string | 所屬 instance |
| `name` | string | artifact 名稱（對應 `artifacts` 區塊的 key） |
| `kind` | string | `file` / `data` |
| `lifecycle` | string | `input` / `output` / `intermediate` |
| `uri` | string | 儲存位置（由 runtime 產生） |
| `size` | integer | 大小（bytes） |
| `content_type` | string | MIME type |
| `created_at` | timestamp | 建立時間 |
| `updated_at` | timestamp | 最後更新時間 |

### instance_lease

| 欄位 | 型別 | 說明 |
|------|------|------|
| `instance_id` | string | 被租借的 instance |
| `worker_id` | string | 持有 lease 的 worker |
| `acquired_at` | timestamp | 取得時間 |
| `expires_at` | timestamp | 過期時間 |

### event_dedup

| 欄位 | 型別 | 說明 |
|------|------|------|
| `event_id` | string | 事件 ID |
| `definition_id` | string | 觸發的 definition |
| `created_at` | timestamp | 記錄時間 |

用於 trigger 去重。記錄 SHOULD 設定 TTL（建議 5 分鐘）。

---

## Artifact 存取控制

引擎 MUST 在 task 執行前後強制執行 artifact 存取規則：

| 規則 | 說明 |
|------|------|
| 寫入 input artifact | 拒絕：step → FAILED |
| 同時 write 存取 | 拒絕：引擎 MUST 確保同一 artifact 同一時間僅一個 step 以 write/read_write 存取 |
| 多 step 同時 read | 允許 |
| 寫入後更新 record | MUST 更新 `uri`、`size`、`content_type`、`updated_at` |

### Artifact 清理

| Lifecycle | 清理策略 |
|-----------|---------|
| `input` | Instance 到達 terminal 狀態後保留（供查詢） |
| `output` | Instance 到達 terminal 狀態後保留（供查詢） |
| `intermediate` | Instance 到達 terminal 狀態後 MAY 清理 |

---

## Secret 管理

引擎在啟動時 MUST：

1. 掃描已 PUBLISHED 的 Secret definitions
2. 解密 `data` 中的 key-value pairs
3. 載入至記憶體快取（`secret` namespace）

### 錯誤處理

| 情況 | 行為 |
|------|------|
| 解密失敗（key 不正確） | 引擎啟動失敗（MUST log 錯誤的 secret name） |
| Secret definition 不存在 | `secret.XXX` 表達式求值回傳 `null` |

### Secret 安全規範

- Secret 值 MUST NOT 出現在 `execution_log`、`step_instance.output`、`workflow_instance.output`
- Task stderr/stdout 中含有 secret 值時，引擎 SHOULD 遮蔽為 `***`
- Secret 值 MUST NOT 被 Expression Evaluator 記錄至 `recorded_values`

---

## 資料保留策略

| 實體 | 保留策略 |
|------|---------|
| definition | 永久保留 |
| workflow_instance | 引擎 SHOULD 提供可設定的保留期限（如 90 天） |
| step_instance | 隨 workflow_instance 保留 |
| execution_log | 引擎 SHOULD 提供可設定的保留期限 |
| event_record | 引擎 SHOULD 提供可設定的保留期限（建議 7 天） |
| artifact（intermediate） | Instance 完成後 MAY 立即清理 |
| event_dedup | TTL 5 分鐘 |

具體保留策略由部署設定決定。

---

## SQLite 預設實作

引擎以 SQLite 作為預設 storage backend，降低部署門檻。

### 啟用方式

零配置：引擎啟動時若未指定 storage 設定，自動建立 SQLite 資料庫檔案（預設路徑：`./slogan.db`）。

### SQLite 特定設定

| 設定 | 值 | 說明 |
|------|---|------|
| `journal_mode` | WAL | 支援讀寫並行 |
| `busy_timeout` | 5000ms | 避免 SQLITE_BUSY 錯誤 |
| `foreign_keys` | ON | 啟用外鍵約束 |
| `synchronous` | NORMAL | 平衡效能與安全性 |

### 限制

- SQLite 為單檔案資料庫，不支援多 worker 跨主機部署。多 worker 架構 MUST 使用 PostgreSQL 等支援網路連線的後端
- Instance lease 機制在 SQLite 模式下仍有效（同一 process 內的多 worker threads）
- 高併發寫入場景下效能受限，適用於開發、測試與低流量生產環境
