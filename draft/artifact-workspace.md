# Artifact Workspace 模型

本文件描述 artifact 系統從 URI 模型調整為 workspace 目錄模型的設計草案。

---

## 設計動機

現行 URI 模型要求每個 task/agent 自行透過 HTTP 下載與上傳 artifact，增加了工具開發者的負擔。新的 workspace 模型讓引擎負責檔案的實體化與同步，task/agent 直接以檔案系統操作 artifact。

---

## 核心概念

### Workspace 目錄

每個 workflow instance 啟動時，引擎在暫存資料夾下建立一個 **workspace 目錄**。引擎根據 artifact 宣告，將所有 artifact 的原始檔案**立即**下載或連結到此目錄中。

Artifact 可透過 `workspace_path` 指定在 workspace 中的子目錄位置，支援自訂目錄結構：

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

### 來源類型（source）

Artifact 可來自多種來源：

```yaml
artifacts:
  order_file:
    kind: file
    lifecycle: input
    required: true
    workspace_path: input/order_file.csv
    source:
      type: s3
      bucket: my-bucket
      key: orders/${input.order_id}.csv

  local_config:
    kind: directory
    lifecycle: input
    workspace_path: input/config
    source:
      type: local
      path: /data/shared/config/

  result_report:
    kind: file
    lifecycle: output
    workspace_path: output/result_report.pdf
    source:
      type: s3
      bucket: my-bucket
      key: results/${input.order_id}/report.pdf

  temp_data:
    kind: data
    lifecycle: intermediate
```

| Source Type | 說明 |
|-------------|------|
| `s3` | 遠端 S3 相容儲存（需設定 bucket、key） |
| `local` | 本機檔案或目錄路徑 |

> 若未指定 `source`，引擎在 workspace 中建立空檔案或空目錄（適用於 `output` / `intermediate`）。
>
> 若未指定 `workspace_path`，artifact 以其名稱作為檔名放在 workspace 根目錄。

### Artifact 欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `kind` | string | MUST | `file`、`data`、`directory` |
| `lifecycle` | string | MUST | `input`、`output`、`intermediate` |
| `required` | boolean | MAY | 預設 `false`（僅對 `input` 有意義） |
| `description` | string | MAY | 人類可讀說明 |
| `workspace_path` | string | MAY | 在 workspace 中的相對路徑（支援子目錄）；未指定時以 artifact 名稱放在根目錄 |
| `access` | string | MAY | `read` 或 `read_write`（預設依 lifecycle 決定） |
| `concurrency` | string | MAY | 並行寫入策略：`lock`（預設）或 `last_write_wins` |
| `source` | object | MAY | 來源設定（見下方） |
| `sync` | object | MAY | 同步策略設定（見下方） |

### kind

| Kind | 說明 |
|------|------|
| `file` | 單一檔案 |
| `data` | 結構化資料（JSON 相容） |
| `directory` | 目錄（遞迴複製或 mount） |

### source 欄位

#### `type: s3`

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | string | MUST | `s3` |
| `bucket` | string | MUST | S3 bucket 名稱 |
| `key` | string | MUST | S3 object key（支援 `${}` 表達式） |
| `endpoint` | string | MAY | 自訂 S3 endpoint（如 MinIO） |
| `region` | string | MAY | AWS region |

#### `type: local`

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | string | MUST | `local` |
| `path` | string | MUST | 本機絕對路徑（支援 `${}` 表達式）；可指向檔案或目錄 |

> 當 `kind: directory` 且 `source.type: local` 時，引擎遞迴複製整個目錄到 workspace。若為 `access: read`，引擎 MAY 使用 bind mount 或 symlink 優化。

---

## 存取權限（access）

每個 artifact 根據 lifecycle 決定預設存取權限，也可透過 `access` 欄位覆寫：

```yaml
artifacts:
  order_file:
    kind: file
    lifecycle: input
    access: read          # 預設：input → read
    source:
      type: s3
      bucket: my-bucket
      key: orders/data.csv

  shared_config:
    kind: file
    lifecycle: input
    access: read_write    # 覆寫：允許修改並同步回原始位置
    source:
      type: local
      path: /data/shared/config.json
```

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

---

## 同步策略（sync）

對於 `access: read_write` 的 artifact，可設定同步策略控制何時將 workspace 中的變更同步回原始位置：

```yaml
artifacts:
  result_report:
    kind: file
    lifecycle: output
    source:
      type: s3
      bucket: my-bucket
      key: results/report.pdf
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

```yaml
artifacts:
  shared_state:
    kind: data
    lifecycle: intermediate
    concurrency: lock           # 預設：互斥鎖

  event_log:
    kind: file
    lifecycle: output
    concurrency: last_write_wins  # 允許並行寫入，最後寫入者勝出
```

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
>
> 所有 artifact 在 instance 建立時**立即**實體化（不支援 lazy 下載），確保 step 執行時所有檔案已就緒。

---

## Task / Agent 存取 Artifact

### 本機 Task 與 Agent

本機執行的 task 和 agent 直接存取 workspace 目錄中的檔案：

```yaml
- id: process_order
  type: task
  action: order.process
  input:
    workspace_dir: ${ artifacts._workspace_path }
    order_file: ${ artifacts.order_file.path }
    output_file: ${ artifacts.result_report.path }
```

### CEL 中的 artifact 存取

| 路徑 | 型別 | 說明 |
|------|------|------|
| `artifacts._workspace_path` | string | workspace 目錄絕對路徑 |
| `artifacts.<name>.path` | string | artifact 在 workspace 中的絕對路徑 |
| `artifacts.<name>.size` | int | artifact 大小（bytes） |
| `artifacts.<name>.content_type` | string | MIME type |
| `artifacts.<name>.content` | string | 文字內容（僅 `kind: data` 或文字型 file） |
| `artifacts.<name>.access` | string | 存取權限（`read` / `read_write`） |
| `artifacts.<name>.exists` | boolean | 檔案是否已存在於 workspace |

### Agent 存取

Agent 可透過以下方式操作 artifact：

1. **直接檔案操作**（本機 agent）— agent 的 tool 可直接讀寫 workspace 目錄中的檔案
2. **透過 prompt 注入路徑**：

```yaml
- id: analyze
  type: agent
  agent: analyzer
  prompt: |
    請分析 ${ artifacts.order_file.path } 的內容，
    結果寫入 ${ artifacts.result_report.path }。
```

3. **透過 `kind: data` 注入內容**：

```yaml
- id: analyze
  type: agent
  agent: analyzer
  prompt: |
    請分析以下訂單資料：
    ${ artifacts.order_file.content }
```

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

## Workflow 生命週期中的 Artifact 行為

### 1. Instance 建立

1. 建立 workspace 目錄
2. 驗證所有 `required: true` 的 `input` artifact 來源可達
3. 下載 / 連結所有 `input` artifact 到 workspace
4. 為 `output` / `intermediate` artifact 預留空檔案或目錄

### 2. Step 執行

1. 引擎將 workspace 路徑或 API 端點資訊傳給 task / agent
2. Task / Agent 直接操作 workspace 中的檔案（本機）或透過 API 操作（遠端）
3. 若 sync strategy 為 `on_step_complete`，step 完成後觸發同步

### 3. Instance 完成

1. 若 sync strategy 為 `on_success` 且 instance 成功 → 同步所有 `read_write` artifact 到原始位置
2. 若 instance 失敗 → 不同步（除非 strategy 為 `eager` 或 `on_step_complete` 已同步的部分）
3. 清理 `intermediate` artifact（依保留策略）

### 4. Instance 清理

1. 清理 workspace 目錄
2. `intermediate` artifact 依保留策略清理
3. `input` / `output` artifact 依保留策略保留

---

## 完整範例

```yaml
kind: Workflow
name: order-processing
version: "1.0"

input:
  order_id: { type: string, required: true }

artifacts:
  order_file:
    kind: file
    lifecycle: input
    required: true
    workspace_path: input/order_file.csv
    description: "訂單原始資料檔"
    source:
      type: s3
      bucket: order-data
      key: orders/${ input.order_id }.csv

  shared_config:
    kind: directory
    lifecycle: input
    workspace_path: input/config
    description: "共用設定目錄"
    source:
      type: local
      path: /etc/app/config/

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

steps:
  - id: validate_order
    type: task
    action: order.validate
    input:
      order_file: ${ artifacts.order_file.path }
      config_dir: ${ artifacts.shared_config.path }

  - id: process_order
    type: task
    action: order.process
    input:
      order_file: ${ artifacts.order_file.path }
      output_file: ${ artifacts.result_report.path }
      temp_file: ${ artifacts.temp_data.path }

  # 遠端 task — 透過 API 操作 artifact（token 自動注入 input + header）
  - id: remote_enrich
    type: task
    action: enrichment.run
    backend: http
    input:
      order_id: ${ input.order_id }

  - id: analyze
    type: agent
    agent: order.analyzer
    prompt: |
      請分析 ${ artifacts.order_file.path } 的訂單資料，
      結果寫入 ${ artifacts.result_report.path }。
```

---

## 設計決策紀錄

| 編號 | 議題 | 決策 |
|------|------|------|
| N1 | Workspace 目錄結構 | **B** — 支援子目錄，artifact 可指定 `workspace_path` |
| N2 | 大檔案 lazy 下載 | **A** — 所有 artifact 在 instance 建立時全部下載 |
| N3 | 本機 source 支援目錄 | **B** — 支援目錄（`kind: directory`，遞迴複製或 mount） |
| N4 | 遠端 API token 注入 | **C** — 兩者皆支援（task input + HTTP header） |
| N5 | 並行寫入處理 | **C** — 由 artifact `concurrency` 欄位決定（`lock` / `last_write_wins`） |
