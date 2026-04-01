# 15 — Runtime & Persistence

本文件定義 runtime 架構需求與持久化模型。

---

## Runtime 架構概述

```
Event / API / Timer
       ↓
  Scheduler Loop
       ↓
  Find READY steps
       ↓
  Execute (async)
       ↓
  Update state (DB)
```

### 核心原則

| 原則 | 說明 |
|------|------|
| 非同步執行 | 所有 task 執行為 async，不阻塞 scheduler |
| 非 blocking pause | `wait_event` 不佔用執行資源 |
| DB 為唯一 truth | 記憶體狀態僅為快取，crash 後從 DB 恢復 |
| 多 instance 交錯 | 單一 scheduler 可同時處理多個 instance |

---

## 持久化實體

引擎 MUST 持久化以下實體：

### workflow_definition

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `name` | workflow name |
| `version` | 版本號 |
| `content` | YAML 定義內容 |
| `lifecycle_state` | DRAFT / VALIDATED / PUBLISHED / DEPRECATED / ARCHIVED |
| `created_at` | 建立時間 |
| `updated_at` | 最後更新時間 |

### task_definition

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `name` | task name（dotted namespace，如 `order.load`） |
| `version` | 版本號 |
| `content` | YAML 定義內容 |
| `backend_type` | bash / http / builtin / sdk |
| `lifecycle_state` | DRAFT / VALIDATED / PUBLISHED / DEPRECATED / ARCHIVED |
| `created_at` | 建立時間 |
| `updated_at` | 最後更新時間 |

### workflow_instance

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `definition_id` | 所屬 workflow definition |
| `state` | CREATED / RUNNING / WAITING / SUCCEEDED / FAILED / CANCELLED |
| `input` | instance 輸入資料 |
| `output` | instance 輸出資料（完成後） |
| `error` | 錯誤資訊（失敗時） |
| `parent_instance_id` | 父 instance ID（sub_workflow 時） |
| `parent_step_id` | 父 step ID（sub_workflow 時） |
| `created_at` | 建立時間 |
| `started_at` | 開始執行時間 |
| `completed_at` | 完成時間 |

### step_instance

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `workflow_instance_id` | 所屬 instance |
| `step_id` | 對應 definition 中的 step id |
| `state` | PENDING / READY / RUNNING / SUCCEEDED / FAILED / WAITING / TIMED_OUT / SKIPPED |
| `input` | step 輸入（求值後） |
| `output` | step 輸出 |
| `error` | 錯誤資訊 |
| `attempt` | 當前嘗試次數 |
| `started_at` | 開始時間 |
| `completed_at` | 完成時間 |

### wait_subscription

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `workflow_instance_id` | 所屬 instance |
| `step_id` | 所屬 step |
| `event_type` | 等待的事件類型 |
| `match_expression` | CEL 匹配表達式 |
| `created_at` | 建立時間 |
| `expires_at` | 過期時間（根據 timeout 計算） |

### timeout_schedule

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `workflow_instance_id` | 所屬 instance |
| `step_id` | 所屬 step（null = workflow 級 timeout） |
| `fires_at` | 觸發時間 |
| `type` | `step_timeout` / `workflow_timeout` |

### expression_cache

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `workflow_instance_id` | 所屬 instance |
| `step_id` | 所屬 step |
| `expression_position` | 表達式在 step 定義中的位置識別 |
| `function_name` | 非確定性函式名稱（`now` / `uuid`） |
| `value` | 函式回傳值（JSON 序列化） |
| `created_at` | 記錄時間 |

用於 deterministic replay：runtime 首次求值 `now()` 或 `uuid()` 時 MUST 記錄回傳值；replay 時 MUST 使用記錄值而非重新求值。

---

### execution_log

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `workflow_instance_id` | 所屬 instance |
| `step_id` | 所屬 step |
| `event_type` | 狀態轉換類型（如 `state_changed`、`retry`、`event_received`） |
| `data` | 事件詳細資料 |
| `created_at` | 記錄時間 |

Execution log 為 append-only，用於稽核與除錯。

### artifact_record

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `workflow_instance_id` | 所屬 instance |
| `name` | artifact 名稱 |
| `kind` | file / data |
| `lifecycle` | input / output / intermediate |
| `uri` | 儲存位置 |
| `size` | 大小（bytes） |
| `content_type` | MIME type |
| `created_at` | 建立時間 |

---

## Recovery 機制

### 重啟流程

```
Engine Start
     ↓
Load non-terminal instances (RUNNING / WAITING)
     ↓
For each instance: rebuild state from DB
     ↓
For each RUNNING step: apply recovery policy
     ↓
Resume scheduler loop
```

### Recovery 規則

| Step 狀態 | Execution Policy | Recovery 行為 |
|-----------|-----------------|---------------|
| RUNNING | `replayable` | 重跑 step |
| RUNNING | `idempotent` | 帶 idempotency key 重跑 |
| RUNNING | `non_repeatable` | 標記 FAILED |
| WAITING | — | 重建 wait_subscription，繼續等待 |
| READY | — | 正常執行 |
| PENDING | — | 不動作，等待前置 step |

### 關鍵保證

- Timeout schedule MUST 在 crash 後被重建（根據 DB 中的 `fires_at`）
- Wait subscription MUST 在 crash 後被重建
- 過期的 timeout / wait 在重啟時立即處理

---

## 並行控制

- Step instance 更新 SHOULD 使用 optimistic locking（version column）
- 多 worker 架構下，使用 lease-based ownership 確保同一 instance 不被多個 worker 同時處理
- Idempotency key = `workflow_instance_id + step_id + attempt`
