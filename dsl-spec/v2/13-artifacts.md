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

## Tool 如何存取 Artifact 內容

Artifact 的內容存取透過**本地檔案系統**進行。引擎在呼叫 tool 前，負責將 artifact 準備為本地檔案；tool 完成後，引擎負責將修改過的檔案上傳回 storage backend。

```
引擎（執行前）                Tool                    引擎（執行後）
  │                            │                        │
  │  從 storage 下載至暫存目錄   │                        │
  │──────────────────────────▶│                        │
  │                            │  讀寫本地檔案            │
  │                            │──────────────────────▶│
  │                            │                        │  上傳修改的檔案至 storage
```

### stdio request 中的 artifacts

引擎將綁定的 artifact 以 `artifacts` map 傳入 tool 的 request JSON：

```json
{
  "input": { ... },
  "artifacts": {
    "order_file": {
      "path": "/tmp/slogan/abc123/step1/order.csv",
      "content_type": "text/csv",
      "size": 4096,
      "access": "read"
    },
    "result_report": {
      "path": "/tmp/slogan/abc123/step1/result.json",
      "content_type": null,
      "size": null,
      "access": "write"
    }
  }
}
```

Tool 透過 `path` 欄位直接以檔案系統操作，不需知道 storage backend 的實作。

### data kind 的處理

`kind: data` 的 artifact 在傳入 tool 前，引擎將其序列化為 JSON 檔案。Tool 寫回時，引擎讀取 JSON 檔案並更新 artifact record。

---

## 儲存後端

Artifact 的實際儲存機制（S3、local filesystem、database 等）由 runtime 實作決定。本規格僅定義：

- artifact 的宣告語法
- 存取控制語意
- lifecycle 語意
- CEL 可讀取的中繼資料欄位
- tool 透過本地檔案路徑存取 artifact 內容（引擎負責 storage ↔ 本地檔案的轉換）
