# 22 — Artifacts

本文件定義 artifact 系統的宣告、存取與生命週期。

---

## Artifact 定義

Artifacts 在 workflow 頂層的 `artifacts` 區塊中宣告：

```yaml
artifacts:
  order_file:
    kind: file
    lifecycle: input
    required: true
    description: "訂單原始資料檔"

  result_report:
    kind: file
    lifecycle: output
    description: "處理結果報告"

  temp_data:
    kind: data
    lifecycle: intermediate
```

### 欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `kind` | string | MUST | `file` 或 `data` |
| `lifecycle` | string | MUST | `input`、`output`、`intermediate` |
| `required` | boolean | MAY | 預設 `false` |
| `description` | string | MAY | 人類可讀說明 |

---

## kind

| Kind | 說明 |
|------|------|
| `file` | 檔案型態（有 URI、content_type） |
| `data` | 結構化資料（JSON 相容） |

---

## lifecycle

| Lifecycle | 說明 |
|-----------|------|
| `input` | 必須在 instance 建立時提供；workflow 執行期間為唯讀 |
| `output` | 在 workflow 執行期間產生；workflow 完成後可取回 |
| `intermediate` | 在 workflow 執行期間建立與消費；workflow 完成後 MAY 被清理 |

### required

- 僅對 `input` lifecycle 有意義
- `required: true` → instance 建立時 MUST 提供該 artifact，否則建立失敗
- `output` 和 `intermediate` 的 `required` 被忽略

---

## URI 存取模型

Artifact 統一透過 URI 提供給 task 和 agent 使用。Task / agent 需要自行負責下載與上傳。

### CEL 中的 artifact 存取

在 CEL 表達式中可讀取 artifact 的中繼資料與 URI：

| 路徑 | 型別 | 說明 |
|------|------|------|
| `artifacts.<name>.uri` | string | artifact 的存取 URI（含簽名，有時效性） |
| `artifacts.<name>.upload_uri` | string | artifact 的上傳 URI（僅 `output` / `intermediate` lifecycle） |
| `artifacts.<name>.size` | int | artifact 的大小（bytes） |
| `artifacts.<name>.content_type` | string | MIME type |
| `artifacts.<name>.content` | string | artifact 的文字內容（僅 `kind: data` 或文字型 file） |

### Task 存取 artifact

Task 透過 `input` 接收 artifact 的 URI，自行下載與上傳：

```yaml
- id: load_order
  type: task
  action: order.load
  input:
    order_file_uri: ${ artifacts.order_file.uri }

- id: generate_report
  type: task
  action: report.generate
  input:
    order_file_uri: ${ artifacts.order_file.uri }
    upload_uri: ${ artifacts.result_report.upload_uri }
```

Task handler 需自行實作：
- 從 `uri` 下載 artifact 內容（HTTP GET）
- 將結果上傳至 `upload_uri`（HTTP PUT）
- 上傳完成後引擎自動更新 artifact 的中繼資料

### Agent 存取 artifact

Agent 同樣透過 URI 存取 artifact。URI 可透過以下方式提供：

1. **在 prompt 中注入**：

```yaml
- id: analyze_order
  type: agent
  agent: order.analyzer
  prompt: |
    請分析以下訂單資料並產生風險報告：

    訂單檔案 URI：${ artifacts.order_file.uri }
    報告上傳 URI：${ artifacts.result_report.upload_uri }
  input:
    order_id: ${ input.order_id }
```

2. **在 input 中傳遞**：

```yaml
- id: analyze_order
  type: agent
  agent: order.analyzer
  input:
    order_file_uri: ${ artifacts.order_file.uri }
    upload_uri: ${ artifacts.result_report.upload_uri }
```

3. **在 prompt 中注入 `kind: data` 的內容**：

```yaml
- id: analyze_order
  type: agent
  agent: order.analyzer
  prompt: |
    請分析以下訂單資料：

    ${ artifacts.order_file.content }
```

> **注意**：將 artifact 內容直接注入 prompt 時，需注意 prompt injection 風險。若 artifact 內容來自不受信任的來源，建議透過 tool 下載 URI 內容，讓 LLM 在 tool result 中處理，而非直接嵌入 system/user prompt。

### 存取規則

- `input` lifecycle 的 artifact：僅提供 `uri`（唯讀），不提供 `upload_uri`
- `output` / `intermediate` lifecycle 的 artifact：提供 `uri`（若已有內容）與 `upload_uri`
- URI 包含簽名與時效（如 S3 presigned URL），過期需重新取得
- 引擎 MUST 確保 URI 的有效期涵蓋 step 的合理執行時間

### 跨 Sub-workflow 傳遞

Artifact 不能直接跨 workflow 邊界存取。若需在 parent 和 child 之間共用 artifact，透過 input 傳遞 URI：

```yaml
# Parent workflow
- id: process_file
  type: sub_workflow
  workflow: file_processor
  input:
    file_uri: ${ artifacts.order_file.uri }
```

---

## 儲存後端

Artifact 的實際儲存機制（S3、local filesystem、database 等）由 runtime 實作決定。本規格僅定義：

- artifact 的宣告語法
- lifecycle 語意
- CEL 可讀取的中繼資料欄位（含 URI）
- task / agent 透過 URI 自行下載與上傳 artifact 內容

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
2. 從 storage backend 刪除實際檔案 / 資料
3. 更新 artifact_record（標記為已清理或刪除記錄）

若步驟 2 失敗（如 storage 不可用），引擎 SHOULD 在背景排程重試，不影響 instance 狀態。

> Terminal 狀態的完整清單詳見 [23-lifecycle](23-lifecycle.md)。

### 保留策略

引擎 SHOULD 支援可設定的 artifact 保留期限：

| 設定 | 預設 | 說明 |
|------|------|------|
| `artifacts.retention.input` | 永久 | Input artifact 保留期限 |
| `artifacts.retention.output` | 永久 | Output artifact 保留期限 |
| `artifacts.retention.intermediate` | 0（立即清理） | Intermediate artifact 保留期限 |

超過保留期限的 artifact，引擎以背景排程清理。

---

## URI 過期處理

- 引擎產生的 URI 包含簽名與時效（如 S3 presigned URL）
- 若 task / agent 在 URI 過期後嘗試存取 → 下載或上傳失敗，由 task / agent 自行處理
- 引擎 SHOULD 確保 URI 的有效期涵蓋 step 的合理執行時間
- 若 step 有 `timeout` 設定，URI 有效期 SHOULD >= `timeout` 值
