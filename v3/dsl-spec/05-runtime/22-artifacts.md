# 22 — Artifacts

本文件定義 artifact 系統的宣告、workspace 實體化、存取與生命週期。

---

## Artifact 定義

Artifacts 在 workflow 頂層的 `artifacts` 區塊中宣告：

```yaml
artifacts:
  order_file:
    kind: file
    lifecycle: input
    required: true
    workspace_path: input/order_file.csv
    description: "訂單原始資料檔"
    source:
      type: s3
      bucket: my-bucket
      key: orders/${ input.order_id }.csv

  local_config:
    kind: directory
    lifecycle: input
    workspace_path: input/config
    description: "共用設定目錄"
    source:
      type: local
      path: /data/shared/config/

  result_report:
    kind: file
    lifecycle: output
    workspace_path: output/result_report.pdf
    description: "處理結果報告"
    source:
      type: s3
      bucket: order-results
      key: reports/${ input.order_id }.pdf
    sync:
      strategy: on_success
      conflict: overwrite

  temp_data:
    kind: data
    lifecycle: intermediate
    concurrency: last_write_wins
    description: "中間處理資料"
```

### 欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `kind` | string | MUST | `file`、`data`、`directory` |
| `lifecycle` | string | MUST | `input`、`output`、`intermediate` |
| `required` | boolean | MAY | 預設 `false`（僅對 `input` 有意義） |
| `description` | string | MAY | 人類可讀說明 |
| `workspace_path` | string | MAY | 在 workspace 中的相對路徑（支援子目錄）；未指定時以 artifact 名稱放在根目錄 |
| `access` | string | MAY | `read` 或 `read_write`（預設依 lifecycle 決定） |
| `concurrency` | string | MAY | 並行寫入策略：`lock`（預設）或 `last_write_wins` |
| `source` | object | MAY | 來源設定 |
| `sync` | object | MAY | 同步策略設定 |

---

## kind

| Kind | 說明 |
|------|------|
| `file` | 單一檔案（有 content_type） |
| `data` | 結構化資料（JSON 相容） |
| `directory` | 目錄（遞迴複製或 mount） |

---

## lifecycle

| Lifecycle | 說明 |
|-----------|------|
| `input` | 必須在 instance 建立時提供；workflow 執行期間預設唯讀 |
| `output` | 在 workflow 執行期間產生；workflow 完成後可取回 |
| `intermediate` | 在 workflow 執行期間建立與消費；workflow 完成後 MAY 被清理 |

### required

- 僅對 `input` lifecycle 有意義
- `required: true` → instance 建立時 MUST 提供該 artifact 的 source 或內容，否則建立失敗
- `output` 和 `intermediate` 的 `required` 被忽略

---

## Workspace 目錄

每個 workflow instance 啟動時，引擎在暫存資料夾下建立一個 **workspace 目錄**，並將所有 artifact **立即**實體化到其中（不支援 lazy 下載）。

Artifact 可透過 `workspace_path` 指定在 workspace 中的子目錄位置：

```
/tmp/slogan/instances/<instance_id>/artifacts/
  ├── input/
  │   ├── order_file.csv        # input: 從 S3 下載的副本（read-only）
  │   └── config/               # input: 本機目錄的唯讀副本
  │       ├── app.json
  │       └── rules.yaml
  ├── output/
  │   └── result_report.pdf     # output: task 產生的檔案（read-write）
  └── temp_data.json            # intermediate: 無指定 workspace_path，放在根目錄
```

> 若未指定 `workspace_path`，artifact 以其名稱作為檔名放在 workspace 根目錄。

---

## 來源類型（source）

| Source Type | 說明 |
|-------------|------|
| `s3` | 遠端 S3 相容儲存（需設定 bucket、key） |
| `local` | 本機檔案或目錄路徑 |

> 若未指定 `source`，引擎在 workspace 中建立空檔案或空目錄（適用於 `output` / `intermediate`）。

### `type: s3`

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | string | MUST | `s3` |
| `bucket` | string | MUST | S3 bucket 名稱 |
| `key` | string | MUST | S3 object key（支援 `${}` 表達式） |
| `endpoint` | string | MAY | 自訂 S3 endpoint（如 MinIO） |
| `region` | string | MAY | AWS region |

### `type: local`

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | string | MUST | `local` |
| `path` | string | MUST | 本機絕對路徑（支援 `${}` 表達式）；可指向檔案或目錄 |

> 當 `kind: directory` 且 `source.type: local` 時，引擎遞迴複製整個目錄到 workspace。若為 `access: read`，引擎 MAY 使用 bind mount 或 symlink 優化。

---

## 存取權限（access）

| Access | 說明 | Workspace 行為 |
|--------|------|---------------|
| `read` | 唯讀 | 檔案在 workspace 中為唯讀副本或 symlink；不同步回原始位置 |
| `read_write` | 可讀寫 | 檔案在 workspace 中為可寫副本；根據同步策略同步回原始位置 |

### 預設存取權限

| Lifecycle | 預設 Access |
|-----------|------------|
| `input` | `read` |
| `output` | `read_write` |
| `intermediate` | `read_write` |

> `access` 可覆寫預設值。例如 `lifecycle: input` + `access: read_write` 允許修改 input artifact 並同步回原始位置。

---

## 同步策略（sync）

對於 `access: read_write` 的 artifact，可設定同步策略控制何時將 workspace 中的變更同步回原始位置：

```yaml
sync:
  strategy: on_success
  conflict: overwrite
```

### sync 欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `strategy` | string | MAY | 同步時機（預設 `on_success`） |
| `conflict` | string | MAY | 衝突處理策略（預設 `overwrite`） |

### 同步時機（strategy）

| Strategy | 說明 |
|----------|------|
| `on_success` | Workflow instance 成功完成後同步（預設） |
| `on_step_complete` | 每個使用此 artifact 的 step 完成後立即同步 |
| `eager` | 檔案變更後盡快同步（引擎偵測檔案變更） |
| `manual` | 不自動同步；需透過 API 手動觸發同步 |

### 衝突處理（conflict）

| Conflict | 說明 |
|----------|------|
| `overwrite` | 直接覆寫原始位置（預設） |
| `fail` | 若原始位置自下載後已被修改，同步失敗並報錯 |
| `version` | 以版本號保留新舊版本（需儲存後端支援） |

---

## 並行寫入控制（concurrency）

當多個 step 可能同時寫入同一 artifact 時，透過 `concurrency` 欄位控制行為：

| Concurrency | 說明 |
|-------------|------|
| `lock` | 預設。引擎以互斥鎖防止同一 artifact 同時被多個 step 寫入。若 step 嘗試寫入已鎖定的 artifact，該 step 等待鎖釋放。引擎 MUST 以 artifact 名稱字母序取鎖以防止死鎖。 |
| `last_write_wins` | 允許並行寫入，不加鎖。最後完成寫入的 step 結果為最終值。適用於 append-only 或冪等寫入場景。 |

> `access: read` 的 artifact 不受並行控制影響（無寫入操作）。

---

## Workspace 實體化策略

引擎根據來源類型、kind 和存取權限決定如何在 workspace 中建立檔案或目錄：

| Source | Kind | Access | 實體化方式 |
|--------|------|--------|-----------|
| `s3` | `file` | `read` | 下載到 workspace，設為唯讀 |
| `s3` | `file` | `read_write` | 下載到 workspace，設為可寫 |
| `s3` | `directory` | `read` | 下載前綴下所有物件到 workspace 目錄，設為唯讀 |
| `s3` | `directory` | `read_write` | 下載前綴下所有物件到 workspace 目錄，設為可寫 |
| `local` | `file` | `read` | 建立 symlink 或唯讀副本 |
| `local` | `file` | `read_write` | 建立可寫副本（copy-on-write） |
| `local` | `directory` | `read` | bind mount 或遞迴唯讀複製 |
| `local` | `directory` | `read_write` | 遞迴複製整個目錄 |

> 引擎 MAY 使用 symlink / bind mount 優化唯讀本機檔案與目錄的存取。使用時，引擎 MUST 確保 workspace 內的權限為唯讀。

---

## CEL 中的 artifact 存取

在 CEL 表達式中可讀取 artifact 的中繼資料與路徑：

| 路徑 | 型別 | 說明 |
|------|------|------|
| `artifacts._workspace_path` | string | workspace 目錄絕對路徑 |
| `artifacts.<name>.path` | string | artifact 在 workspace 中的絕對路徑 |
| `artifacts.<name>.size` | int | artifact 大小（bytes） |
| `artifacts.<name>.content_type` | string | MIME type |
| `artifacts.<name>.content` | string | artifact 的文字內容（僅 `kind: data` 或文字型 file） |
| `artifacts.<name>.access` | string | 存取權限（`read` / `read_write`） |
| `artifacts.<name>.exists` | boolean | 檔案是否已存在於 workspace |

可用時機：所有 steps。

---

## Task 存取 artifact

本機執行的 task 直接存取 workspace 目錄中的檔案：

```yaml
- id: load_order
  type: task
  action: order.load
  input:
    order_file: ${ artifacts.order_file.path }

- id: generate_report
  type: task
  action: report.generate
  input:
    order_file: ${ artifacts.order_file.path }
    output_file: ${ artifacts.result_report.path }
```

Task handler 直接對 workspace 中的檔案進行讀寫操作，無需自行下載或上傳。

---

## Agent 存取 artifact

Agent 可透過以下方式操作 artifact：

### 1. 直接檔案操作

本機 agent 的 tool 可直接讀寫 workspace 目錄中的檔案。

### 2. 在 prompt 中注入路徑

```yaml
- id: analyze_order
  type: agent
  agent: order.analyzer
  prompt: |
    請分析 ${ artifacts.order_file.path } 的訂單資料，
    結果寫入 ${ artifacts.result_report.path }。
  input:
    order_id: ${ input.order_id }
```

### 3. 在 input 中傳遞路徑

```yaml
- id: analyze_order
  type: agent
  agent: order.analyzer
  input:
    order_file: ${ artifacts.order_file.path }
    output_file: ${ artifacts.result_report.path }
```

### 4. 在 prompt 中注入 `kind: data` 的內容

```yaml
- id: analyze_order
  type: agent
  agent: order.analyzer
  prompt: |
    請分析以下訂單資料：
    ${ artifacts.order_file.content }
```

> **注意**：將 artifact 內容直接注入 prompt 時，需注意 prompt injection 風險。若 artifact 內容來自不受信任的來源，建議透過 tool 讀取檔案，讓 LLM 在 tool result 中處理，而非直接嵌入 system/user prompt。

---

## 遠端 Task 的 Artifact CRUD API

對於遠端執行的 task（如 HTTP backend），引擎提供一組 OpenAPI 端點供操作 artifact：

### API 端點

```
Base URL: {engine_base_url}/api/v1/instances/{instance_id}/artifacts
```

#### 列出所有 artifact

```
GET /api/v1/instances/{instance_id}/artifacts
```

回應：

```json
{
  "artifacts": [
    {
      "name": "order_file",
      "kind": "file",
      "lifecycle": "input",
      "access": "read",
      "content_type": "text/csv",
      "size": 1024,
      "exists": true
    }
  ]
}
```

#### 讀取 artifact 內容

```
GET /api/v1/instances/{instance_id}/artifacts/{name}
```

- 回應 `Content-Type` 對應 artifact 的 MIME type
- 回應 body 為 artifact 的原始內容
- `kind: data` 的 artifact 回應 `application/json`
- `kind: directory` 回應目錄下所有檔案的 tar.gz 壓縮包

#### 建立 / 更新 artifact 內容

```
PUT /api/v1/instances/{instance_id}/artifacts/{name}
```

- Request `Content-Type` 指定 artifact 的 MIME type
- Request body 為 artifact 的完整內容
- 僅 `access: read_write` 的 artifact 可寫入；否則回傳 `403 Forbidden`
- 寫入成功回傳 `204 No Content`

#### 刪除 artifact 內容

```
DELETE /api/v1/instances/{instance_id}/artifacts/{name}
```

- 僅 `lifecycle: intermediate` 且 `access: read_write` 的 artifact 可刪除
- 其他情況回傳 `403 Forbidden`
- 刪除成功回傳 `204 No Content`

#### 取得 artifact 中繼資料

```
GET /api/v1/instances/{instance_id}/artifacts/{name}/metadata
```

回應：

```json
{
  "name": "order_file",
  "kind": "file",
  "lifecycle": "input",
  "access": "read",
  "content_type": "text/csv",
  "size": 1024,
  "exists": true,
  "checksum": "sha256:abcdef...",
  "last_modified": "2026-04-02T10:00:00Z"
}
```

### 認證

引擎同時支援兩種 token 注入方式，遠端 task 可擇一或同時使用：

#### 方式一：透過 task input 注入

引擎自動將 API 端點與 token 注入 task 的 input：

```json
{
  "artifact_api_base": "https://engine.example.com/api/v1/instances/inst_123/artifacts",
  "artifact_api_token": "eyJhbGciOi..."
}
```

#### 方式二：透過 HTTP header 傳遞

引擎在呼叫 HTTP backend task 時自動附加 header：

```
X-Artifact-API-Base: https://engine.example.com/api/v1/instances/inst_123/artifacts
X-Artifact-Token: eyJhbGciOi...
```

Task 在呼叫 artifact API 時，將 token 放入 `Authorization` header：

```
Authorization: Bearer eyJhbGciOi...
```

#### Token 規格

- Token 為 JWT，包含 instance_id、step_id、允許存取的 artifact 列表與權限
- Token 有效期與 step timeout 一致
- 引擎 MUST 驗證 token 中的權限與 artifact 的 access 設定一致

---

## 跨 Sub-workflow 傳遞

Artifact 不能直接跨 workflow 邊界存取。若需在 parent 和 child 之間共用 artifact，透過 input 傳遞 workspace 中的路徑：

```yaml
# Parent workflow
- id: process_file
  type: sub_workflow
  workflow: file_processor
  input:
    file_path: ${ artifacts.order_file.path }
```

Child workflow 可透過 `input.file_path` 直接存取 parent workspace 中的檔案（共用同一 workspace 目錄）。

Child workflow 完成後若需將結果傳回 parent，MUST 透過 `return` step 的 output 傳遞路徑或內容。

---

## Workflow 生命週期中的 Artifact 行為

### 1. Instance 建立

1. 建立 workspace 目錄（含子目錄結構）
2. 驗證所有 `required: true` 的 `input` artifact 來源可達
3. **立即**下載 / 連結所有 `input` artifact 到 workspace
4. 為 `output` / `intermediate` artifact 預留空檔案或空目錄

### 2. Step 執行

1. 引擎將 workspace 路徑（本機 task/agent）或 API 端點資訊（遠端 task）傳給執行者
2. 本機 task / agent 直接操作 workspace 中的檔案
3. 遠端 task 透過 CRUD API 操作 artifact
4. 若 sync strategy 為 `on_step_complete`，step 完成後觸發同步

### 3. Instance 完成

1. 若 sync strategy 為 `on_success` 且 instance 成功 → 同步所有 `read_write` artifact 到原始位置
2. 若 instance 失敗 → 不同步（除非 strategy 為 `eager` 或 `on_step_complete` 已同步的部分）
3. 清理 `intermediate` artifact（依保留策略）

### 4. Instance 清理

1. 清理 workspace 目錄
2. `intermediate` artifact 依保留策略清理
3. `input` / `output` artifact 依保留策略保留

---

## Artifact 清理

### 清理時機

| Lifecycle | 清理時機 | 說明 |
|-----------|---------|------|
| `input` | 不主動清理 | Instance 到達 terminal 狀態後保留，供稽核查詢 |
| `output` | 不主動清理 | 同上 |
| `intermediate` | Instance 到達 terminal 狀態後 | 引擎 MAY 立即清理或以背景排程清理 |

### 清理流程

引擎清理 artifact 時 MUST 依以下順序：

1. 確認 workflow instance 為 terminal 狀態（SUCCEEDED / FAILED / CANCELLED / COMPENSATED / COMPENSATION_FAILED / CONTINUED）
2. 從 workspace 刪除實際檔案 / 資料
3. 從 storage backend 刪除原始檔案（若適用）
4. 更新 artifact_record（標記為已清理或刪除記錄）

若步驟 2–3 失敗（如 storage 不可用），引擎 SHOULD 在背景排程重試，不影響 instance 狀態。

> Terminal 狀態的完整清單詳見 [23-lifecycle](23-lifecycle.md)。

### 保留策略

引擎 SHOULD 支援可設定的 artifact 保留期限：

| 設定 | 預設 | 說明 |
|------|------|------|
| `artifacts.retention.input` | 永久 | Input artifact 保留期限 |
| `artifacts.retention.output` | 永久 | Output artifact 保留期限 |
| `artifacts.retention.intermediate` | 0（立即清理） | Intermediate artifact 保留期限 |

超過保留期限的 artifact，引擎以背景排程清理。
