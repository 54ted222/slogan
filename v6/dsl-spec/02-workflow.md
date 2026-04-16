# 02 — Workflow Definition

本文件定義 `kind: Workflow` 的頂層 YAML 結構、觸發機制與 workflow 級設定。

---

## 頂層欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `apiVersion` | string | MUST | 固定值 `workflow/v6` |
| `kind` | string | MUST | 固定值 `Workflow` |
| `metadata` | object | MUST | 識別與描述資訊 |
| `triggers` | array | MUST | 啟動機制，至少一個 |
| `input_schema` | object | MAY | 輸入 JSON Schema |
| `output_schema` | object | MAY | 輸出 JSON Schema |
| `config` | object | MAY | workflow 級設定 |
| `steps` | array | MUST | 主要步驟序列 |

---

## triggers

Trigger 的職責是**建立新的 workflow instance**。一個 workflow MAY 定義多個 triggers。

### manual — 手動觸發

```yaml
triggers:
  - type: manual
```

透過 API 或 CLI 手動啟動，呼叫端提供 input 資料。

### event — 事件觸發

```yaml
triggers:
  - type: event
    event: order.created
    when: ${ event.data.source == "api" }
    input_mapping:
      order_id: ${ event.data.order_id }
      action: ${ event.data.type }
```

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `event` | string | MUST | 要訂閱的事件類型 |
| `when` | CEL expression | MAY | 過濾條件，回傳 boolean |
| `input_mapping` | map | MAY | 事件資料到 workflow input 的映射 |

---

## input_schema / output_schema

以 JSON Schema 定義 workflow 的輸入輸出結構。驗證時機：

| 階段 | 時機 | 失敗處理 |
|------|------|----------|
| `input_schema` | trigger 欲建立 instance 時、入庫前 | 拒絕建立 instance；trigger 回報錯誤 `workflow.input_schema_violation`，詳情含違反欄位路徑 |
| `output_schema` | `type: return` 求值其 `output` 後、寫回前 | workflow 以 FAILED 結束，`error.type == "output_schema_violation"` |

驗證器 MUST 使用嚴格模式（未定義欄位視為違反 `additionalProperties: false`，除非 schema 顯式允許）。

```yaml
input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

output_schema:
  type: object
  properties:
    status:
      type: string
```

---

## config

Workflow 級設定，所有欄位皆為可選。

| 欄位 | 型別 | 說明 |
|------|------|------|
| `timeout` | duration | workflow instance 的最長執行時間 |
| `max_step_executions` | integer | step 執行次數上限（防止無限迴圈） |
| `cancellation_policy` | enum | `graceful`（預設）/ `hard`；見 `runtime-spec/02-instance-lifecycle.md` |
| `cancel_grace_period` | duration | graceful 模式的寬限期，預設 30s |
| `catch` | catch 陣列 | workflow 級錯誤處理（含 timeout） |
| `secrets` | string 陣列 | 依賴的 secret 名稱列表；載入期驗證所有名稱存在於當前 project scope，缺失 → 載入失敗 `registry.missing_secret`；運行時不額外檢查（CEL 引用未宣告的 secret 仍可解析，但建議開發者盡量宣告以便 CI/CD 檢查） |

```yaml
config:
  timeout: 2h
  max_step_executions: 1000
  catch:
    - type: fail
      when: ${ error.type == "timeout" }
      message: "workflow timeout exceeded"
    - type: emit
      event: workflow.failed
      data:
        error: ${ error.message }
```

---

## 完整結構範例

### 最小結構

```yaml
apiVersion: workflow/v6
kind: Workflow

metadata:
  name: hello
  version: 1

triggers:
  - type: manual

steps:
  - type: task
    action: system.echo
    input:
      message: "hello world"
```

### 完整結構

```yaml
apiVersion: workflow/v6
kind: Workflow

metadata:
  name: order_fulfillment
  version: 3
  description: "訂單履行流程"
  labels:
    team: commerce

triggers:
  - type: manual
  - type: event
    event: order.created
    when: ${ event.data.source == "api" }

input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

output_schema:
  type: object
  properties:
    status:
      type: string

config:
  timeout: 2h

steps:
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  - type: return
    output:
      status: "completed"
```
