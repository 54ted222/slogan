# 02 — Document Structure

本文件定義 workflow definition（`kind: Workflow`）的頂層 YAML 結構。

---

## 頂層欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `apiVersion` | string | MUST | 固定值 `workflow/v3` |
| `kind` | string | MUST | 固定值 `Workflow` |
| `metadata` | object | MUST | workflow 的識別與描述資訊 |
| `triggers` | array | MUST | 啟動機制，至少一個 |
| `input_schema` | object | MAY | 輸入 schema 定義 |
| `output_schema` | object | MAY | 輸出 schema 定義 |
| `label_schema` | object | MAY | instance label 的 schema 定義（建議性，不強制驗證） |
| `artifacts` | map | MAY | artifact 宣告 |
| `config` | object | MAY | workflow 級設定 |
| `steps` | array | MUST | 主要步驟序列 |

---

## metadata

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `name` | string | MUST | workflow 唯一識別名稱，`snake_case` |
| `version` | integer | MUST | 版本號，正整數，單調遞增 |
| `description` | string | MAY | 人類可讀的說明 |
| `labels` | map\<string, string\> | MAY | 分類用的鍵值對 |

```yaml
metadata:
  name: order_fulfillment
  version: 3
  description: "處理訂單從建立到完成的完整流程"
  labels:
    team: commerce
    priority: high
```

---

## label_schema

定義此 workflow 的 instance 支援哪些 label key。為**建議性**，不驗證——缺少 label 不會阻擋 instance 建立。

```yaml
label_schema:
  customer_id:
    type: string
  region:
    type: string
    enum: [tw, jp, us, eu]
  priority:
    type: string
    enum: [low, medium, high]
    default: medium
```

詳見 [24-instance-labels](24-instance-labels.md)。

---

## config

Workflow 級設定。所有欄位皆為可選。

| 欄位 | 型別 | 說明 |
|------|------|------|
| `timeout` | duration | workflow instance 的最長執行時間 |
| `max_step_executions` | integer | step 執行次數上限（防止無限迴圈），超過時 instance FAILED（錯誤碼 `max_step_executions_exceeded`） |
| `on_error` | step 陣列 | workflow 級錯誤處理（見 [21-error-handling](21-error-handling.md)） |
| `on_timeout` | step 陣列 | workflow 級 timeout 處理 |
| `secrets` | string 陣列 | 依賴的 secret 名稱列表（見 [25-secrets-and-env](25-secrets-and-env.md)） |

```yaml
config:
  timeout: 2h
  max_step_executions: 1000
  on_error:
    - type: emit
      event: workflow.failed
      data:
        workflow: "order_fulfillment"
        error: ${ error.message }
  on_timeout:
    - type: fail
      message: "workflow timeout exceeded"
```

---

## 完整結構範例

### 最小結構

```yaml
apiVersion: workflow/v3
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
apiVersion: workflow/v3
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

label_schema:
  customer_id:
    type: string
  region:
    type: string
    enum: [tw, jp, us, eu]

artifacts:
  order_file:
    kind: file
    lifecycle: input
    required: true

config:
  timeout: 2h
  max_step_executions: 1000

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
