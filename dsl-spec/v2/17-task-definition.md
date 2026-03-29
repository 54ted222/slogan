# 17 — Task Definition

Task definition 是獨立的 YAML 文件，定義一個可被 workflow step 呼叫的工作單元。Task definition 與 workflow definition 分離，擁有獨立的 `kind`、版本與生命週期。

---

## 設計動機

- **定義與使用分離**：workflow 描述「呼叫什麼」，task definition 描述「怎麼執行」
- **多 backend 支援**：同一個 task 介面可切換不同的執行方式
- **獨立版本管理**：task 可獨立演進，不影響 workflow definition
- **可重用性**：多個 workflow 可共用同一個 task definition

---

## 頂層結構

```yaml
apiVersion: task/v2
kind: Task

metadata:
  name: string              # MUST — 唯一識別名稱（dotted namespace）
  version: integer           # MUST — 版本號，正整數
  description: string        # MAY
  labels: map                # MAY

input_schema: object             # MAY — 輸入 JSON Schema

output_schema: object             # MAY — 輸出 JSON Schema

backend:
  type: string               # MUST — stdio | http | builtin
  # ... backend 專屬設定
```

### 欄位說明

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `apiVersion` | string | MUST | 固定值 `task/v2` |
| `kind` | string | MUST | 固定值 `Task` |
| `metadata` | object | MUST | 識別與描述 |
| `input_schema` | object | MAY | 輸入 schema |
| `output_schema` | object | MAY | 輸出 schema |
| `backend` | object | MUST | 執行後端設定 |

---

## metadata

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `name` | string | MUST | 使用 dotted namespace（如 `order.load`、`payment.create`） |
| `version` | integer | MUST | 正整數，單調遞增 |
| `description` | string | MAY | 人類可讀說明 |
| `labels` | map | MAY | 分類鍵值對 |

`name` 即為 workflow step 中 `action` 欄位所參照的識別字。

---

## Input / Output Schema

與 workflow 的 schema 語法相同（JSON Schema 子集），用於驗證 task 被呼叫時的輸入與輸出。

```yaml
input_schema:
  type: object
  properties:
    order_id:
      type: string
    amount:
      type: number
  required: [order_id]

output_schema:
  type: object
  properties:
    id:
      type: string
    status:
      type: string
```

- 引擎在 step 執行前 SHOULD 驗證 input 是否符合 task 的 `input_schema`
- 引擎在 step 執行後 SHOULD 驗證 output 是否符合 task 的 `output_schema`
- 驗證失敗 → step FAILED（錯誤碼 `schema_validation_error`）

---

## Backend Types

### http

呼叫 HTTP/HTTPS endpoint。

```yaml
backend:
  type: http
  url: string                 # MUST — 完整 URL（支援 ${ } 表達式）
  method: string              # MAY — HTTP method，預設 POST
  headers: map                # MAY — HTTP headers
  body_mapping: string        # MAY — input 到 body 的映射模式
  timeout: duration           # MAY — HTTP 請求 timeout
  retry_on_status: array      # MAY — 哪些 HTTP status code 觸發重試
  response_mapping:           # MAY — response 到 output 的映射
    body: string              #   從 response body 取值的 CEL 表達式
    status_code: string       #   output 中接收 status code 的欄位名
```

#### 行為

- 引擎發送 HTTP 請求至 `url`
- Task input 預設以 JSON body 傳送（`Content-Type: application/json`）
- Response body MUST 為 JSON，解析後作為 task output
- HTTP 2xx = 成功，其他 = 失敗
- `retry_on_status` 中的 status code（如 `[429, 502, 503]`）視為可重試的暫時性失敗，會觸發 workflow step 的 `retry` 機制（見 [06-step-task](06-step-task.md)）。不在此列表中的非 2xx status code 視為永久性失敗，不觸發重試

#### Redirect 處理

- 引擎 MUST NOT 自動跟隨 3xx redirect
- 3xx status code 視為失敗（同非 2xx 處理）
- 若需 redirect，task definition 的 `url` SHOULD 直接指向最終目標

#### Response 限制

| 限制 | 預設值 | 說明 |
|------|--------|------|
| Response body 大小上限 | 10 MB | 超過 → task 失敗 |
| Response timeout | 使用 step `timeout` 或 `backend.timeout` | 取較小值 |

引擎 MAY 允許透過引擎設定調整 response body 大小上限。

#### Artifact 限制

HTTP backend **不支援** artifact resource binding（`resources` 欄位）。原因：HTTP backend 以 JSON body 傳輸資料，無法將 artifact 綁定至本地檔案系統路徑。若需傳遞檔案，task handler 應自行透過 `input` 取得 artifact URI 並處理下載 / 上傳。驗證階段（DRAFT → VALIDATED）MUST 檢查此限制。

#### Content-Type 處理

| Response Content-Type | 行為 |
|----------------------|------|
| `application/json` | 正常解析 JSON 作為 output |
| 其他（如 `text/html`、`text/plain`） | 嘗試解析為 JSON。成功 → 正常處理；失敗 → task 失敗（error code = `task_failed`） |
| 無 Content-Type header | 同上，嘗試解析為 JSON |

本規格僅支援 JSON response。非 JSON 格式的 response（如 XML、CSV）MUST 透過中間服務轉換為 JSON，或使用 stdio backend 處理。

#### body_mapping

| 值 | 說明 |
|----|------|
| `full`（預設） | 整個 input map 作為 request body |
| `none` | 不傳送 body |

#### response_mapping

自訂如何將 HTTP response 轉換為 task output：

```yaml
response_mapping:
  body: ${ response.body.data }      # 從 response body 中取子欄位
  status_code: "http_status"          # 將 status code 存入 output.http_status
```

若未設定 `response_mapping`，response body 整體作為 output。

#### 範例

```yaml
apiVersion: task/v2
kind: Task

metadata:
  name: payment.create
  version: 2
  description: "建立付款請求"

input_schema:
  type: object
  properties:
    order_id:
      type: string
    amount:
      type: number
    currency:
      type: string
      default: "TWD"
  required: [order_id, amount]

output_schema:
  type: object
  properties:
    payment_id:
      type: string
    status:
      type: string

backend:
  type: http
  url: "https://api.payment.example.com/v1/charges"
  method: POST
  headers:
    Authorization: "Bearer ${ secret.PAYMENT_API_KEY }"
    Idempotency-Key: "${ env.IDEMPOTENCY_KEY }"
  timeout: 10s
  retry_on_status: [429, 502, 503]
  response_mapping:
    body: ${ response.body }
```

---

### builtin

呼叫引擎內建的 action。

```yaml
backend:
  type: builtin
  handler: string             # MUST — 內建 handler 識別字
  config: map                 # MAY — handler 專屬設定
```

#### 行為

- 引擎內部直接呼叫已註冊的 handler function
- 不經過任何外部 I/O
- 適用於引擎自帶的系統功能（如 logging、metrics、internal state 操作）

#### 範例

```yaml
apiVersion: task/v2
kind: Task

metadata:
  name: system.echo
  version: 1
  description: "回傳輸入（用於測試）"

input_schema:
  type: object

output_schema:
  type: object

backend:
  type: builtin
  handler: echo
```

---

### stdio

Spawn 子 process，透過 stdin/stdout 交換結構化 JSON 訊息。Tool 可以用任何程式語言實作，**不依賴引擎的實作語言**。

```yaml
backend:
  type: stdio
  command: string             # MUST — 啟動 tool process 的命令
  shell: string               # MAY — shell 路徑，預設 /bin/sh
  env: map                    # MAY — 額外環境變數（值支援 ${ } CEL 表達式）
  working_dir: string         # MAY — 工作目錄
  config: map                 # MAY — 傳入 tool 的靜態設定（透過 request JSON）
  timeout_kill_after: duration # MAY — SIGTERM 後等待多久發 SIGKILL，預設 5s
```

#### 行為

- 引擎啟動 shell process（使用 `backend.shell`，預設 `/bin/sh`）執行 `command`
- 引擎將 **request JSON** 寫入子 process 的 **stdin**，隨後關閉 stdin
- Tool 將 **response JSON** 寫入 **stdout**
- **stderr** 作為 log 記錄（寫入 execution_log，不影響成敗判定）
- Process exit code 0 且 stdout 為有效 response JSON → 依 `success` 欄位判定
- Process exit code ≠ 0 → 失敗（stderr 前 4KB 作為 error message）
- Process timeout → 先發 SIGTERM，等待 `timeout_kill_after`，再發 SIGKILL
- `env` 中的值支援 `${ }` CEL 表達式（在 task 執行時求值）

#### Request / Response 格式

**Request**（引擎 → tool）：

```json
{
  "input": { },
  "config": { },
  "context": {
    "idempotency_key": "string | null",
    "instance_id": "string",
    "step_id": "string",
    "attempt": 1
  },
  "artifacts": {
    "order_file": {
      "path": "/tmp/slogan/abc123/step1/order.csv",
      "content_type": "text/csv",
      "size": 4096,
      "access": "read"
    }
  }
}
```

- `artifacts`：若 step 有 `resources` 綁定，引擎將 artifact 準備為本地檔案，並在此傳入路徑與存取權限。Tool 透過 `path` 直接以檔案系統讀寫。詳見 [13-artifacts](13-artifacts.md)。

**Response**（tool → 引擎）：

```json
{
  "success": true,
  "output": { },
  "error": null
}
```

失敗時：

```json
{
  "success": false,
  "output": null,
  "error": {
    "message": "string",
    "code": "string"
  }
}
```

#### Request / Response 大小限制

| 限制 | 預設值 | 說明 |
|------|--------|------|
| Request JSON（stdin）大小上限 | 10 MB | 超過 → 不啟動 process，step 失敗 |
| Response JSON（stdout）大小上限 | 10 MB | 超過 → 視為無效 response，step 失敗 |
| stderr 記錄上限 | 64 KB | 超過部分截斷 |

引擎 MAY 允許透過引擎設定調整大小上限。

#### Process 信號處理

Process timeout 的信號序列：

```
Timeout 到達
  ↓
發送 SIGTERM（讓 tool 有機會 graceful shutdown）
  ↓
等待 timeout_kill_after（預設 5s）
  ↓
若 process 仍存在 → 發送 SIGKILL（強制終止）
```

- SIGTERM 後 tool SHOULD 盡快完成清理並結束
- tool 可攔截 SIGTERM 進行 cleanup，但 MUST 在 `timeout_kill_after` 內結束
- SIGKILL 無法被攔截，process 立即終止
- 引擎 MUST 同時終止 tool 啟動的所有子 process（使用 process group / session）

#### Process Resource Limit

引擎 MAY 對 stdio tool process 設定 resource limit：

| 資源 | 建議限制 | 超過時行為 |
|------|---------|----------|
| 記憶體（RSS） | 512 MB | OS 終止 process（SIGKILL / OOM），step 失敗 |
| CPU 時間 | 由 step timeout 控制 | Timeout 處理 |
| Open file descriptors | 1024 | Process 無法開啟新檔案 |

具體的 resource limit 值由引擎設定決定。引擎 SHOULD 使用 OS 機制（如 `ulimit`、cgroups）限制 process 資源。

#### 特殊字元處理

- Request / Response JSON MUST 為 UTF-8 編碼
- JSON 中的字串 MAY 包含任意 Unicode 字元（含 null byte `\u0000`，以 JSON escape 表示）
- stdin 寫入 request JSON 後 MUST 關閉 stdin（EOF），避免 tool 無限等待

#### 語言無關性

Slogan 專案 MAY 提供各語言的 helper library（如 Node.js、Python、Go）簡化 stdin/stdout 的序列化處理，但這並非必要 — 任何能讀 stdin 寫 stdout 的程式皆可作為 stdio tool。

#### 範例

```yaml
apiVersion: task/v2
kind: Task

metadata:
  name: order.load
  version: 3
  description: "從資料庫載入訂單"

input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

output_schema:
  type: object
  properties:
    id:
      type: string
    amount:
      type: number
    status:
      type: string
    customer_email:
      type: string
    shipping_address:
      type: object

backend:
  type: stdio
  command: "node ./tools/order-tasks/load-order.js"
  config:
    db_pool: "primary"
```

---

## Task Definition Lifecycle

與 workflow definition 相同的生命週期：

```
DRAFT → VALIDATED → PUBLISHED → DEPRECATED → ARCHIVED
```

- 僅 PUBLISHED 狀態的 task 可被 workflow step 呼叫
- Workflow 中參照的 `action` 在 validation 時 SHOULD 檢查對應的 task definition 是否存在

---

## Execution Policy 與 Task Definition 的關係

Execution policy（`replayable` / `idempotent` / `non_repeatable`）定義在 **workflow step** 中，而非 task definition 中。

原因：同一個 task definition 在不同 workflow context 下可能有不同的 execution policy 需求。例如 `payment.create` 在主流程中是 `non_repeatable`，但在測試用的 mock workflow 中可以是 `replayable`。

不過，task definition MAY 在 metadata 中以 `labels` 標註建議的 policy：

```yaml
metadata:
  name: payment.create
  version: 2
  labels:
    suggested_policy: non_repeatable
```

此標註僅供參考，不具強制性。引擎 MAY 在 workflow validation 時產生警告。
