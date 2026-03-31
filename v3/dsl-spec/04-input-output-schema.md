# 04 — Input / Output Schema

本文件定義 definition 的輸入與輸出契約。適用於 `kind: Workflow`、`kind: Task`、`kind: Agent`。

---

## Input Schema

`input_schema` 定義 instance 建立時 MUST 提供的資料結構。使用 JSON Schema 子集。

### 支援的型別

| 型別 | 說明 |
|------|------|
| `string` | 字串 |
| `number` | 浮點數 |
| `integer` | 整數 |
| `boolean` | 布林值 |
| `array` | 陣列，可用 `items` 定義元素型別 |
| `object` | 物件，可用 `properties` 定義子欄位 |

### 支援的約束

| 約束 | 適用型別 | 說明 |
|------|----------|------|
| `required` | object | 必填欄位列表 |
| `enum` | string, number, integer | 列舉允許值 |
| `default` | 所有型別 | 預設值 |
| `minLength` / `maxLength` | string | 字串長度限制 |
| `minimum` / `maximum` | number, integer | 數值範圍 |
| `minItems` / `maxItems` | array | 陣列長度限制 |

### 範例

```yaml
input_schema:
  type: object
  properties:
    order_id:
      type: string
    action:
      type: string
      enum: [pay, ship, cancel]
    priority:
      type: integer
      minimum: 1
      maximum: 5
      default: 3
    notify_customer:
      type: boolean
      default: true
    items:
      type: array
      items:
        type: object
        properties:
          sku:
            type: string
          qty:
            type: integer
            minimum: 1
        required: [sku, qty]
  required: [order_id, action]
```

### 存取方式

Input 資料在 CEL 中透過 `input` namespace 存取：

```yaml
input:
  order_id: ${ input.order_id }
  is_urgent: ${ input.priority >= 4 }
```

### 驗證行為

- 驗證在 instance 建立時執行
- 缺少 `required` 欄位 → instance 建立失敗
- 型別不符 → instance 建立失敗
- 有 `default` 的欄位，未提供時自動填入預設值
- 驗證失敗時，instance 不會被建立，回傳 `schema_validation_error`

---

## Output Schema

`output_schema` 定義完成時回傳的資料結構。語法與 input schema 相同。

### 範例

```yaml
output_schema:
  type: object
  properties:
    order_id:
      type: string
    status:
      type: string
      enum: [completed, partial]
    payment_id:
      type: string
  required: [order_id, status]
```

### 驗證行為

**Workflow**：
- 驗證在 `return` step 執行時進行
- 驗證失敗 → workflow instance 狀態變為 FAILED，錯誤碼 `schema_validation_error`

**Agent**：
- 驗證在 agent 產生 final response 後進行
- 驗證失敗 → agent step FAILED，錯誤碼 `agent_output_invalid`

**Task**：
- 驗證在 task backend 回傳結果後進行
- 驗證失敗 → task step FAILED，錯誤碼 `schema_validation_error`

若沒有定義 `output_schema`，output 不做驗證。

---

## Schema 為可選

`input_schema` 和 `output_schema` 皆為可選：

- 沒有 `input_schema` → 不做 input schema 驗證
  - Manual trigger：呼叫端提供的資料直接成為 `input` namespace（若無提供則為空 map）
  - Event trigger 有 `input_mapping`：映射結果成為 `input` namespace
  - Event trigger 無 `input_mapping`：`event.data` 整體成為 `input` namespace（passthrough）
- 沒有 `output_schema` → 完成時不驗證 output
