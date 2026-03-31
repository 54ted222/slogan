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

### definition

統一實體，以 `kind` 區分 Workflow / Task / Secret 三種定義（與 `runtime-spec/08-storage-schema.md` 一致）。

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別（自動產生） |
| `kind` | `Workflow` / `Task` / `Secret` |
| `name` | 定義名稱 |
| `version` | 版本號（正整數） |
| `content` | YAML 定義內容（原始文字） |
| `parsed_content` | 解析後的 JSON 結構 |
| `lifecycle_state` | DRAFT / VALIDATED / PUBLISHED / DEPRECATED / ARCHIVED |
| `backend_type` | stdio / http / builtin / null（僅 Task） |
| `created_at` | 建立時間 |
| `updated_at` | 最後更新時間 |

唯一約束：`(kind, name, version)`

### workflow_instance

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `definition_id` | 所屬 workflow definition |
| `state` | CREATED / RUNNING / WAITING / SUCCEEDED / FAILED / CANCELLED |
| `input` | instance 輸入資料（JSON） |
| `output` | instance 輸出資料（JSON，完成後） |
| `error` | 錯誤資訊 `{message, code}`（失敗時） |
| `vars` | 當前 vars namespace snapshot（JSON，assign step 更新） |
| `correlation_id` | 跨 instance 追蹤用 ID |
| `parent_instance_id` | 父 instance ID（sub_workflow 時） |
| `parent_step_id` | 父 step ID（sub_workflow 時） |
| `step_execution_count` | 累計 step 執行次數（max_step_executions 檢查用） |
| `created_at` | 建立時間 |
| `started_at` | 開始執行時間 |
| `completed_at` | 完成時間 |
| `version` | 樂觀鎖版本號（optimistic locking） |

### step_instance

| 欄位 | 說明 |
|------|------|
| `id` | 唯一識別 |
| `workflow_instance_id` | 所屬 instance |
| `step_id` | 對應 definition 中的 step id |
| `step_type` | step 類型（task / assign / if / switch / foreach / parallel / emit / wait_event / fail / return / sub_workflow） |
| `parent_step_id` | 父 step（容器 step 內的子 step） |
| `iteration_index` | foreach 迭代索引（僅 foreach 內的 step） |
| `branch_index` | parallel 分支索引（僅 parallel 內的 step） |
| `state` | PENDING / READY / RUNNING / SUCCEEDED / FAILED / WAITING / TIMED_OUT / CANCELLED / SKIPPED |
| `input` | step 輸入（JSON，求值後） |
| `output` | step 輸出（JSON） |
| `error` | 錯誤資訊 `{message, code}` |
| `attempt` | 當前嘗試次數 |
| `recorded_values` | 非確定性函式記錄（JSON `{position → value}`） |
| `started_at` | 開始時間 |
| `completed_at` | 完成時間 |
| `version` | 樂觀鎖版本號（optimistic locking） |

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

### Secret 管理

Secret definition（`kind: Secret`）透過統一的 `definition` 實體持久化。引擎啟動時掃描所有 PUBLISHED 的 Secret definitions，解密 `data` 欄位，載入至記憶體快取。

- 解密失敗 → 引擎啟動失敗
- Secret 值 MUST NOT 寫入 execution_log 或 step_instance.output
- 引擎 SHOULD 在 stderr / log 輸出中遮蔽 secret 值

---

## Recovery 機制

### 重啟流程

```
Engine Start
     ↓
Load non-terminal instances (CREATED / RUNNING / WAITING)
     ↓
For each instance: rebuild state from DB
     ↓
For each RUNNING step: apply recovery policy
     ↓
Resume scheduler loop
```

### Instance Recovery 規則

| Instance 狀態 | Recovery 行為 |
|--------------|---------------|
| CREATED | 轉為 RUNNING，開始執行 |
| RUNNING | 依據 step 狀態恢復（見下方 Step Recovery） |
| WAITING | 重建 wait_subscription，繼續等待 |

### Step Recovery 規則

`execution.policy` 僅適用於 `task` 與 `sub_workflow`。其他 step 類型（assign、emit、if、switch、foreach、parallel）在 RUNNING 狀態下的 recovery 行為等同 `replayable`（安全重跑）。

| Step 狀態 | Execution Policy | Recovery 行為 |
|-----------|-----------------|---------------|
| RUNNING | `replayable`（或無 policy） | 重跑 step |
| RUNNING | `idempotent` | 帶 idempotency key 重跑 |
| RUNNING | `non_repeatable` | 標記 FAILED |
| WAITING | — | 重建 wait_subscription，繼續等待 |
| READY | — | 正常執行 |
| PENDING | — | 不動作，等待前置 step |

### 非確定性函式的記錄

為實現 deterministic replay（見 [00-overview](00-overview.md)），引擎 MUST 記錄非確定性函式（`now()`、`uuid()`）的求值結果：

- 首次執行時，`now()` 和 `uuid()` 的回傳值 MUST 與 step instance 一同持久化
- Replay 時，引擎 MUST 使用已記錄的值，而非重新求值
- 記錄格式為 step_instance 的一部分，以 expression 位置為 key、求值結果為 value

### 關鍵保證

- Timeout schedule MUST 在 crash 後被重建（根據 DB 中的 `fires_at`）
- Wait subscription MUST 在 crash 後被重建
- 過期的 timeout / wait 在重啟時立即處理

---

## 並行控制

- Step instance 更新 SHOULD 使用 optimistic locking（version column）
- 多 worker 架構下，使用 lease-based ownership 確保同一 instance 不被多個 worker 同時處理
- Idempotency key = `workflow_instance_id + step_id + attempt`
