# 01 — Engine API

本文件定義 Slogan engine 對外提供的 API 介面。所有操作以 request/response 模式定義，不綁定特定傳輸協定（HTTP、gRPC、函式呼叫皆可）。

---

## API 總覽

| 領域 | 操作 | 說明 |
|------|------|------|
| Definition | Create / Update / Delete / Get / List / Validate / Publish / Deprecate / Archive | 管理 workflow / task / secret definitions |
| Instance | Create / Get / List / Cancel | 管理 workflow instances |
| Event | Send | 發送事件至引擎 |
| Step | Get / List | 查詢 step instances（唯讀） |
| Execution Log | List | 查詢 execution logs（唯讀） |
| Dead-letter | List / Get / Retry / Drop | 管理 dead-letter queue |
| Artifact | Get / List | 查詢 artifact 中繼資料（唯讀） |

---

## Definition API

所有 definition types（Workflow、Task、Secret）共享相同的 lifecycle 狀態機：

```
DRAFT → VALIDATED → PUBLISHED → DEPRECATED → ARCHIVED
                ↘ DRAFT（RevertToDraft）
```

各 API 操作適用於所有 kind，但部分行為依 kind 而異（如 trigger 訂閱僅 Workflow 適用）。Task 與 Secret definitions 的 PUBLISHED 狀態表示可被 workflow steps 引用。

### CreateDefinition

建立新的 definition（workflow / task / secret）。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `content` | string | MUST | YAML 定義內容 |

**行為：**

1. 解析 YAML，識別 `kind`（Workflow / Task / Secret）
2. 驗證 `apiVersion` 與 `kind` 組合的合法性
3. 若同名同版本已存在 → 拒絕（`definition_already_exists`）
4. 儲存至 storage，`lifecycle_state` = DRAFT
5. 回傳 definition record

**Response:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | 產生的唯一識別 |
| `kind` | string | Workflow / Task / Secret |
| `name` | string | `metadata.name` |
| `version` | integer | `metadata.version` |
| `lifecycle_state` | string | DRAFT |

---

### UpdateDefinition

更新 DRAFT 狀態 definition 的 YAML 內容。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `definition_id` | string | MUST | 目標 definition |
| `content` | string | MUST | 新的 YAML 定義內容 |

**行為：**

1. 載入 definition（MUST 為 DRAFT 狀態）
2. 解析新的 YAML，驗證 `kind`、`name`、`version` 與原 definition 一致（不可透過 update 改變 identity）
3. 更新 `content` 與 `parsed_content`
4. 更新 `updated_at`
5. 回傳更新後的 definition record

**Response:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `definition_id` | string | |
| `lifecycle_state` | string | DRAFT |
| `updated_at` | timestamp | 更新時間 |

---

### DeleteDefinition

刪除 DRAFT 狀態的 definition。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `definition_id` | string | MUST | 目標 definition |

**行為：**

1. 載入 definition（MUST 為 DRAFT 狀態）
2. 從 storage 中永久刪除該 definition record
3. 回傳成功

非 DRAFT 狀態的 definition 不可刪除 → 回傳 `invalid_lifecycle_transition`。已進入 VALIDATED 以上狀態的 definition 應透過 deprecate → archive 流程處理。

---

### ValidateDefinition

將 definition 從 DRAFT 轉為 VALIDATED。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `definition_id` | string | MUST | 目標 definition |

**行為：**

1. 載入 definition（MUST 為 DRAFT 狀態）
2. 依 `kind` 執行對應的驗證規則（見 [dsl-spec/v2/16-validation-rules](../dsl-spec/v2/16-validation-rules.md)）：
   - **Workflow**：結構驗證、step 引用完整性、表達式語法、trigger 設定
   - **Task**：backend 設定完整性、input/output schema 合法性
   - **Secret**：data 欄位存在、值為 string
3. 全部通過 → `lifecycle_state` = VALIDATED
4. 任一失敗 → 回傳驗證錯誤列表，狀態不變

**Response（成功）：**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `definition_id` | string | |
| `lifecycle_state` | string | VALIDATED |

**Response（失敗）：**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `errors` | array | 驗證錯誤列表（含 rule、path、message、severity） |

---

### PublishDefinition

將 definition 從 VALIDATED 轉為 PUBLISHED。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `definition_id` | string | MUST | 目標 definition |

**行為：**

1. 載入 definition（MUST 為 VALIDATED 狀態）
2. `lifecycle_state` = PUBLISHED
3. 若為 Workflow definition：啟動 event trigger 訂閱（見 [03-event-router](03-event-router.md)）
4. 回傳更新後的 definition record

---

### DeprecateDefinition

將 definition 從 PUBLISHED 轉為 DEPRECATED。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `definition_id` | string | MUST | 目標 definition |

**行為：**

1. 載入 definition（MUST 為 PUBLISHED 狀態）
2. `lifecycle_state` = DEPRECATED
3. 若為 Workflow definition：停用 event trigger 訂閱
4. 既有 RUNNING / WAITING instances 不受影響

---

### RevertToDraft

將 definition 從 VALIDATED 退回 DRAFT（需要修改時）。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `definition_id` | string | MUST | 目標 definition |

**行為：**

1. 載入 definition（MUST 為 VALIDATED 狀態）
2. `lifecycle_state` = DRAFT
3. 使用者可重新編輯 content 後再次驗證

---

### ArchiveDefinition

將 definition 從 DEPRECATED 轉為 ARCHIVED。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `definition_id` | string | MUST | 目標 definition |

**前置條件：**

- MUST 為 DEPRECATED 狀態
- 該 definition 下所有 instances MUST 為 terminal 狀態（SUCCEEDED / FAILED / CANCELLED）

---

### GetDefinition

查詢單一 definition。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `definition_id` | string | 二擇一 | 以 ID 查詢 |
| `name` + `version` | string + integer | 二擇一 | 以 name + version 查詢 |

---

### ListDefinitions

列出 definitions（支援篩選與分頁）。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `kind` | string | MAY | 篩選 Workflow / Task / Secret |
| `name` | string | MAY | 篩選 name（前綴匹配） |
| `lifecycle_state` | string | MAY | 篩選狀態 |
| `cursor` | string | MAY | 分頁游標 |
| `limit` | integer | MAY | 每頁筆數，預設 20，上限 100 |

---

## Instance API

### CreateInstance

建立新的 workflow instance（對應 manual trigger）。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `workflow_name` | string | MUST | workflow definition 名稱 |
| `workflow_version` | integer \| "latest" | MAY | 版本，預設 "latest" |
| `input` | map | MAY | 輸入資料 |
| `artifacts` | map | MAY | Input artifact 資訊（見下方格式） |

**artifacts 格式：**

```json
{
  "artifact_name": {
    "uri": "s3://bucket/path/file.csv",
    "content_type": "text/csv",
    "size": 4096
  }
}
```

- Key = workflow `artifacts` block 中宣告的 artifact 名稱
- `uri`（MUST）：artifact 的儲存位置
- `content_type`（MAY）：MIME type
- `size`（MAY）：大小（bytes）
- 引擎建立對應的 `artifact_record`（lifecycle = `input`）

**行為：**

1. 解析版本：`"latest"` → 查找最新 PUBLISHED 版本
2. 驗證 definition 為 PUBLISHED（或 DEPRECATED，仍可執行）
3. 驗證 definition 有 `manual` trigger
4. 若有 `input_schema` → 驗證 input（失敗回傳 `schema_validation_error`）
5. 若有 `required: true` 的 input artifact → 驗證 artifacts 已提供（失敗回傳 `artifact_required`）
6. 若有 `config.secrets` → 檢查所列的 secret definitions 皆已載入（失敗回傳 `secret_not_available`）
7. 建立 workflow_instance record（state = CREATED）
8. 建立 workflow timeout schedule（若 `config.timeout` 有定義）
9. 通知 Scheduler 開始排程（見 [02-scheduler](02-scheduler.md)）
10. 回傳 instance record

**Response:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `instance_id` | string | 產生的 instance 唯一識別 |
| `state` | string | CREATED |
| `definition_id` | string | 對應的 definition |
| `created_at` | timestamp | 建立時間 |

---

### GetInstance

查詢單一 instance 的詳細資訊。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `instance_id` | string | MUST | 目標 instance |

**Response:**

回傳完整的 workflow_instance record，包含 state、input、output、error、時間戳等。

---

### ListInstances

列出 instances（支援篩選與分頁）。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `workflow_name` | string | MAY | 篩選 workflow name |
| `state` | string | MAY | 篩選狀態 |
| `created_after` | timestamp | MAY | 建立時間下限 |
| `created_before` | timestamp | MAY | 建立時間上限 |
| `cursor` | string | MAY | 分頁游標 |
| `limit` | integer | MAY | 每頁筆數 |

---

### CancelInstance

取消一個正在執行的 instance。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `instance_id` | string | MUST | 目標 instance |
| `reason` | string | MAY | 取消原因 |

**行為：**

1. 載入 instance（MUST 為非 terminal 狀態：CREATED / RUNNING / WAITING）
2. 若 instance 為 CREATED → 直接轉為 CANCELLED
3. 若 instance 為 RUNNING / WAITING：
   a. 所有 RUNNING、WAITING、READY 的 step instances → CANCELLED
   b. Instance state → CANCELLED
   c. 若有 child instances（sub_workflow）→ 遞迴 cancel
4. 記錄 cancellation 至 execution_log
5. 不觸發任何 on_error / on_timeout handler

**冪等性：**

對已取消的 instance 呼叫 CancelInstance → 回傳成功（無副作用）。

---

## Event API

### SendEvent

將事件發送至引擎。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | string | MUST | 事件類型（如 `order.created`） |
| `data` | map | MAY | 事件酬載 |
| `id` | string | MAY | 事件唯一 ID（未提供時引擎自動產生） |
| `source` | string | MAY | 事件來源 |

**行為：**

1. 補全事件信封（自動產生 `id`、填入 `timestamp`）
2. 持久化事件
3. 交由 Event Router 處理（見 [03-event-router](03-event-router.md)）：
   a. 匹配 trigger subscriptions → 建立新 instances
   b. 匹配 wait subscriptions → 恢復等待中的 instances
4. 回傳事件 ID

**Response:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `event_id` | string | 事件唯一 ID |
| `triggered_instances` | array | 因此事件建立的 instance IDs |
| `resumed_instances` | array | 因此事件恢復的 instance IDs |

---

## Step API

### GetStepInstance

查詢單一 step instance。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `instance_id` | string | MUST | 所屬 workflow instance |
| `step_id` | string | MUST | step 識別字 |

---

### ListStepInstances

列出某 instance 下的所有 step instances。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `instance_id` | string | MUST | 所屬 workflow instance |
| `state` | string | MAY | 篩選狀態 |

---

## Execution Log API

### ListExecutionLogs

查詢某 instance 的 execution logs。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `instance_id` | string | MUST | 所屬 workflow instance |
| `step_id` | string | MAY | 篩選特定 step |
| `event_type` | string | MAY | 篩選事件類型（如 `state_changed`、`retry`、`dead_letter`） |
| `cursor` | string | MAY | 分頁游標 |
| `limit` | integer | MAY | 每頁筆數，預設 50，上限 200 |

**Response:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `logs` | array | execution_log records |
| `cursor` | string \| null | 下一頁游標 |

每筆 log record 包含 `id`、`workflow_instance_id`、`step_id`、`event_type`、`data`、`created_at`。

---

## Dead-letter API

### ListDeadLetterEvents

列出 dead-letter queue 中的事件。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `reason` | string | MAY | 篩選原因（`no_match` / `routing_failed` / `poison`） |
| `created_after` | timestamp | MAY | 建立時間下限 |
| `cursor` | string | MAY | 分頁游標 |
| `limit` | integer | MAY | 每頁筆數，預設 20 |

**Response:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `events` | array | dead_letter_event records |
| `cursor` | string \| null | 下一頁游標 |

---

### GetDeadLetterEvent

查詢單一 DLQ 事件詳情。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `dlq_id` | string | MUST | DLQ 記錄 ID |

**Response:**

回傳完整的 dead_letter_event record。

---

### RetryDeadLetterEvent

重新投遞 DLQ 事件至 Event Router。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `dlq_id` | string | MUST | DLQ 記錄 ID |

**行為：**

1. 載入 dead_letter_event record
2. 將事件重新投遞至 Event Router
3. 投遞成功 → 從 DLQ 移除，回傳成功
4. 投遞再次失敗 → 保留在 DLQ，更新 `attempt_count`，回傳錯誤

---

### DropDeadLetterEvent

從 DLQ 移除事件（放棄處理）。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `dlq_id` | string | MUST | DLQ 記錄 ID |

**行為：**

從 storage 刪除該 dead_letter_event record。

---

## Artifact API

### GetArtifact

查詢單一 artifact 的中繼資料。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `instance_id` | string | MUST | 所屬 workflow instance |
| `name` | string | MUST | artifact 名稱 |

**Response:**

回傳 artifact_record（`id`、`name`、`kind`、`lifecycle`、`uri`、`size`、`content_type`、`created_at`、`updated_at`）。

---

### ListArtifacts

列出某 instance 的所有 artifacts。

**Request:**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `instance_id` | string | MUST | 所屬 workflow instance |
| `lifecycle` | string | MAY | 篩選 `input` / `output` / `intermediate` |

**Response:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `artifacts` | array | artifact_record 陣列 |

---

## 錯誤回傳格式

所有 API 錯誤 MUST 回傳統一格式：

```json
{
  "error": {
    "code": "definition_not_found",
    "message": "definition with id 'abc123' not found",
    "details": {}
  }
}
```

### API 錯誤碼

| 錯誤碼 | 說明 |
|--------|------|
| `definition_not_found` | Definition 不存在 |
| `definition_already_exists` | 同名同版本已存在 |
| `invalid_lifecycle_transition` | 不允許的 lifecycle 狀態轉換 |
| `instance_not_found` | Instance 不存在 |
| `instance_not_cancellable` | Instance 已為 terminal 狀態 |
| `schema_validation_error` | Input 不符合 schema |
| `validation_failed` | Definition 驗證失敗 |
| `no_manual_trigger` | Definition 沒有 manual trigger |
| `no_published_version` | 找不到 PUBLISHED 版本 |
| `archive_precondition_failed` | 尚有非 terminal instance |
| `secret_not_available` | config.secrets 所列的 secret 未載入 |
| `definition_identity_mismatch` | UpdateDefinition 時新 YAML 的 kind/name/version 與原 definition 不一致 |
| `dlq_event_not_found` | DLQ 事件不存在 |
| `artifact_not_found` | Artifact 不存在 |
| `payload_too_large` | Event data 或 step output 超過大小上限 |
