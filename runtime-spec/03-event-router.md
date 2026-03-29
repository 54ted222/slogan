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
| `definition_id` | 所屬 workflow definition（用於查詢 definition 的 name、version 等建立 instance 時所需資訊） |
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
4. 若有 `input_schema` → 驗證 input
   - 驗證失敗 → 不建立 instance，記錄錯誤至 log
5. 建立 workflow instance（等同 CreateInstance，但由事件觸發）
6. 通知 Scheduler

### Deduplication

同一事件 MAY 匹配同一 workflow definition 的多個 event triggers。引擎 MUST 提供 trigger-level deduplication 機制。

#### Deduplication 規則

| 規則 | 說明 |
|------|------|
| Key | `event_id + definition_id`（同一事件對同一 definition 只觸發一次） |
| Window | 5 分鐘（從事件進入引擎時起算） |
| 持久化 | 使用 `event_dedup` 表（見 [08-storage-schema](08-storage-schema.md)）|
| TTL | 記錄保留 5 分鐘後自動清理 |

#### Deduplication Key 衝突處理

| 情境 | 行為 |
|------|------|
| 相同 event_id + definition_id 在 window 內重複到達 | 第二次及之後的事件被忽略（不建立 instance） |
| 不同 event_id、相同 event_type | 各自獨立處理（不衝突） |
| 相同 event_id、不同 definition_id | 各自獨立處理（不衝突） |

#### 同一事件觸發多個 Workflow

一個事件 MAY 匹配**不同** workflow definition 的 event triggers。每個匹配的 definition 獨立建立 instance：

```
Event: order.created
  ├─ 匹配 order_fulfillment definition → 建立 instance A
  ├─ 匹配 order_analytics definition → 建立 instance B
  └─ 匹配 order_notification definition → 建立 instance C
```

同一事件對同一 definition 只觸發一次（deduplication），但對不同 definitions 各觸發一次。

#### 停用 Deduplication

引擎 MAY 提供設定以停用 deduplication（用於測試或特殊場景），但預設 MUST 啟用。

---

## Wait Subscription 管理

### 建立時機

- `wait_event` step 進入 WAITING 狀態時，Scheduler 建立 wait_subscription

### Subscription 內容

見 [dsl-spec/v2/15-runtime-persistence](../dsl-spec/v2/15-runtime-persistence.md) 的 wait_subscription 實體。

### 匹配流程

收到事件後，對每個匹配 `event_type` 的 wait_subscription：

1. 檢查是否已過期（`expires_at` < 當前時間）→ 是 → 跳過（由 Timeout Manager 處理）
2. 載入對應的 workflow instance 當前狀態（用於 CEL 求值上下文）
3. 若有 `match_expression` → 求值 CEL 表達式
   - 求值上下文包含：`event`（候選事件）、`input`、`steps`、`vars`（instance 當前資料）
   - `steps` namespace 為事件到達時的最新 snapshot（包含所有已到達 terminal 狀態的 steps）
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

## HTTP Trigger 處理

Event Router 除了處理事件路由外，亦負責管理 HTTP trigger 的 endpoint 生命週期與請求處理。HTTP trigger 的 DSL 定義見 [dsl-spec/v2/02-triggers](../dsl-spec/v2/02-triggers.md)。

### Endpoint 註冊

當 workflow definition 轉為 PUBLISHED 時，引擎為其每個 `type: http` trigger 註冊對應的 HTTP endpoint：

| 項目 | 說明 |
|------|------|
| URL | 引擎 base URL + trigger 的 `path`（如 `/api/orders/{order_id}/process`） |
| Method | trigger 的 `method`（預設 `POST`） |
| 綁定對象 | `definition_id` + trigger index |

若多個 PUBLISHED definitions 註冊相同的 `method` + `path` 組合，引擎 MUST 拒絕後者並記錄錯誤至 log（endpoint 衝突）。

### 請求處理流程

```
HTTP Request
     ↓
路由匹配（method + path）
     ↓ 找不到 → 404 Not Found
Content-Type 檢查
     ↓ 非 JSON → 415 Unsupported Media Type
Authentication 檢查
     ↓ 失敗 → 401 / 403
建構 request context
     ↓
求值 input_mapping CEL 表達式
     ↓ 求值失敗 → 400 Bad Request
驗證 input_schema
     ↓ 驗證失敗 → 400 Bad Request
建立 workflow instance
     ↓
┌─────────────────────────────────────┐
│  async 模式 → 立即回傳 202 Accepted  │
│  sync 模式  → 等待 instance 完成     │
└─────────────────────────────────────┘
```

### Request Context 建構

引擎從 HTTP request 中提取以下 namespace，供 `input_mapping` CEL 表達式求值：

| Namespace | 型別 | 說明 |
|-----------|------|------|
| `request.body` | map | HTTP request body（JSON 解析後） |
| `request.headers` | map\<string, string\> | HTTP headers |
| `request.query` | map\<string, string\> | URL query parameters |
| `request.path_params` | map\<string, string\> | URL path parameters（`{param}` 對應值） |
| `request.method` | string | HTTP method（如 `POST`） |

求值流程：

1. 解析 request body 為 JSON map
2. 提取 headers、query parameters、path parameters
3. 組合為 `request` context 物件
4. 對 `input_mapping` 中的每個欄位求值 CEL 表達式
5. 若無 `input_mapping` → `request.body` 整體作為 workflow input（passthrough）
6. 若有 `input_schema` → 驗證求值結果，失敗回傳 HTTP 400

### Response 模式

#### async 模式（預設）

建立 instance 後立即回傳，不等待執行完成：

- HTTP Status: `202 Accepted`
- Response body:

```json
{
  "instance_id": "inst_abc123",
  "status": "created"
}
```

#### sync 模式

建立 instance 後等待其執行完成，受 `response.timeout` 限制（預設 30s）：

| 情境 | HTTP Status | Response |
|------|-------------|----------|
| Instance 成功完成 | `response.success_status`（預設 200） | `{ "instance_id": "...", "status": "succeeded", "output": { ... } }` |
| Instance 失敗 | `response.error_status`（預設 500） | `{ "instance_id": "...", "status": "failed", "error": { ... } }` |
| 等待逾時 | `202 Accepted` | `{ "instance_id": "...", "status": "running" }` |

sync 模式逾時時回傳 202 而非錯誤碼，因為 instance 仍在執行中，呼叫端可透過 API 後續查詢結果。

### Authentication

HTTP trigger endpoint 的認證與授權機制由引擎實作決定，詳見 [runtime-spec/12-authentication](12-authentication.md)。

引擎 SHOULD 支援在 HTTP trigger endpoint 層級設定認證策略（如 API key、Bearer token、webhook signature verification）。

### Endpoint 生命週期

| 事件 | 動作 |
|------|------|
| Definition → PUBLISHED | 為每個 http trigger 註冊 endpoint |
| Definition → DEPRECATED | 移除對應的 endpoints |
| Definition → ARCHIVED | 移除對應的 endpoints |
| Definition 更新（新版本 PUBLISHED） | 以新版本的 triggers 替換舊 endpoints |

引擎 MUST 確保 endpoint 的新增與移除為原子操作，避免在切換期間出現不一致狀態。

### 錯誤處理

| HTTP Status | 條件 | Response Body |
|-------------|------|---------------|
| `400 Bad Request` | input_mapping CEL 求值失敗（`trigger_mapping_error`）、input_schema 驗證失敗（`trigger_input_invalid`） | `{ "error": "<error_code>", "detail": "..." }` |
| `401 Unauthorized` | 未提供 API key 或 API key 無效 | `{ "error": "authentication_required" }` |
| `403 Forbidden` | API key 權限不足或 workflow 不在 allowed_workflows 內 | `{ "error": "insufficient_permissions" }` |
| `404 Not Found` | 無匹配的 method + path endpoint | `{ "error": "not_found" }` |
| `405 Method Not Allowed` | path 存在但 method 不匹配 | `{ "error": "method_not_allowed" }` |
| `415 Unsupported Media Type` | Content-Type 非 `application/json` | `{ "error": "unsupported_media_type" }` |

Authentication 行為詳見 [runtime-spec/12-authentication](12-authentication.md)。400 Bad Request 的 `error` 欄位使用 [dsl-spec/v2/12-error-handling](../dsl-spec/v2/12-error-handling.md) 定義的標準錯誤碼。

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
