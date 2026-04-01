# 13 — Artifacts

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

## 存取控制

在 `task` step 的 `resources` 中綁定 artifact 並指定存取權限：

```yaml
- id: load_order
  type: task
  action: order.load
  resources:
    - name: order_file          # handler 內部使用的名稱
      type: artifact
      ref: order_file            # artifacts 區塊中的 key
      access: read               # read | write | read_write
```

### 欄位

| 欄位 | 說明 |
|------|------|
| `name` | handler 內部存取此 artifact 時使用的名稱 |
| `type` | 固定值 `artifact` |
| `ref` | 對應 `artifacts` 區塊中的 key |
| `access` | `read`、`write`、`read_write` |

### 存取規則

- `read`：handler 可讀取，不可修改
- `write`：handler 可建立或覆寫
- `read_write`：handler 可讀取並修改
- 多個 step 可同時以 `read` 存取同一 artifact
- 同一時間僅一個 step 可以 `write` 或 `read_write` 存取 artifact（引擎 MUST 確保互斥）
- 若 step 嘗試以 `write` 存取 `input` lifecycle 的 artifact → step FAILED

---

## CEL 中的 artifact 存取

在 CEL 表達式中可讀取 artifact 的中繼資料：

| 路徑 | 型別 | 說明 |
|------|------|------|
| `artifacts.<name>.uri` | string | artifact 的儲存位置 |
| `artifacts.<name>.size` | int | artifact 的大小（bytes） |
| `artifacts.<name>.content_type` | string | MIME type |

```yaml
- type: if
  expr: ${ artifacts.order_file.size > 0 }
  then:
    - type: task
      action: file.process
      input:
        uri: ${ artifacts.order_file.uri }
```

注意：CEL 只能讀取中繼資料，不能讀取 artifact 的內容。內容存取由 task handler 透過 resources 綁定進行。

---

## 跨 Sub-workflow 傳遞

Artifact 不能直接跨 workflow 邊界存取。若需在 parent 和 child 之間共用 artifact，MUST 透過 input 傳遞 URI：

```yaml
# Parent workflow
- id: process_file
  type: sub_workflow
  workflow: file_processor
  input:
    file_uri: ${ artifacts.order_file.uri }

# Child workflow 需在自己的 artifacts 區塊重新宣告
```

---

## 儲存後端

Artifact 的實際儲存機制（S3、local filesystem、database 等）由 runtime 實作決定。本規格僅定義：

- artifact 的宣告語法
- 存取控制語意
- lifecycle 語意
- CEL 可讀取的中繼資料欄位

---

## TODO: Task Handler 存取協議

> 以下為待設計的規格區塊，目前尚未定案。

目前規格定義了 artifact 的宣告與 `resources` 綁定，但尚未定義 **task handler 實際如何讀寫 artifact 內容**的協議。

### 設計方向

統一採用 **URI** 方式：runtime 將 artifact 的存取資訊（含 URI 與必要的認證資訊）注入到 task input 中，handler 透過 URI 自行存取內容。

- Runtime 在 task 執行前解析 `resources` 綁定，將 artifact URI 與 credentials 注入到 handler 可見的 context
- Handler 使用標準協議（HTTP GET/PUT、S3 SDK 等）透過 URI 存取內容
- 適用所有 backend type（bash、http、sdk、builtin）

### 待定義項目

- 注入的資料結構（URI、credentials、metadata 的 schema）
- 認證機制（presigned URL、bearer token、或其他）
- Write artifact 的上傳確認與原子性保證
- 大檔案的分片上傳支援

### 另：Task HTTP Backend 預設 OpenAPI 格式

Task definition 的 `http` backend 應支援以 OpenAPI 格式定義 API 介面，作為預設的 HTTP task 描述方式。待定義項目：

- OpenAPI spec 的引用方式（inline 或外部檔案）
- 從 OpenAPI 自動推導 input/output schema
- 與現有 `url`、`method`、`headers` 欄位的關係
