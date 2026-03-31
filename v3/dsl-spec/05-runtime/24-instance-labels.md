# 24 — Instance Labels

本文件定義 workflow instance 上的自訂標籤（key-value labels）機制，涵蓋定義、設定、查詢與儲存。

---

## 設計動機

Workflow instance 的基本屬性（ID、workflow name、status、created_at）不足以滿足實際運營需求。Instance labels 允許使用者以業務維度分類與查詢 instance：

- 按 `customer_id`、`region`、`priority` 等維度分類
- 在大量 instance 中快速篩選目標
- 支援 Dashboard 與監控的多維度篩選

---

## Label 限制

| 限制 | 值 |
|------|-----|
| Key 最大長度 | 63 字元 |
| Value 最大長度 | 256 字元 |
| 每個 instance 最多 label 數 | 32 |
| Key 命名規則 | `[a-z][a-z0-9_-]*`（小寫字母開頭） |
| Value 型別 | 僅 `string` |

不支援非 string 型別（integer、boolean 等）。若需以數值篩選，請將數值轉為 string 後儲存。

---

## label_schema — 建議性 Schema

Workflow definition MAY 在頂層宣告 `label_schema`，描述此 workflow 預期使用的 label keys：

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: order_processing
  version: 1

label_schema:
  customer_id:
    description: "客戶 ID"
  region:
    description: "部署區域"
    enum: [tw, jp, us, eu]
  priority:
    description: "優先級"
    enum: [low, medium, high]
    default: medium
```

### 行為

- `label_schema` 為**建議性**（advisory only），引擎 MUST NOT 以此拒絕 instance 建立
- 即使 instance 缺少 `label_schema` 中標記的 key，引擎仍 MUST 允許建立
- 即使 instance 攜帶 `label_schema` 中未列出的 key，引擎仍 MUST 允許
- `label_schema` 的主要用途為文件化與 IDE 提示

---

## 設定 Label

### 在建立 Instance 時指定

透過 API 建立 instance 時，MAY 在 request body 中提供 `labels`：

```json
POST /instances
{
  "workflow": "order_processing",
  "version": 1,
  "input": { "order_id": "ORD-001" },
  "labels": {
    "customer_id": "CUST-789",
    "region": "tw",
    "priority": "high"
  }
}
```

引擎 MUST 驗證 labels 符合[限制](#label-限制)（key 格式、長度、數量），不符合時拒絕建立並回傳錯誤。

### 在 Workflow 執行中動態設定

透過 builtin task `instance.set_labels` 在 step 中動態新增或更新 label：

```yaml
- id: update_priority
  type: task
  action: instance.set_labels
  input:
    labels:
      priority: ${ steps.analyze.output.risk_level == "high" ? "high" : "medium" }
      processed_by: "auto"
```

#### instance.set_labels 行為

- `input.labels` 為 `map<string, string>`，MUST 至少包含一組 key-value
- 已存在的 key → 更新 value
- 不存在的 key → 新增
- 未出現在 `input.labels` 中的既有 key → 保持不變（不會被移除）
- 超出限制（數量、長度）→ step 失敗，labels 不做任何變更（原子性）

#### 移除 Label

透過 builtin task `instance.remove_labels`：

```yaml
- id: cleanup_labels
  type: task
  action: instance.remove_labels
  input:
    keys:
      - temporary_flag
      - debug_info
```

---

## Label 繼承（continue-as-new）

當 workflow 透過 `renew` 執行 continue-as-new（詳見 [10-step-terminal](10-step-terminal.md)）時，預設繼承原 instance 的所有 labels：

```yaml
- id: restart
  type: return
  renew: true
  input:
    cursor: ${ steps.process.output.next_cursor }
  # inherit_labels: true  ← 預設值
```

| 設定 | 行為 |
|------|------|
| `inherit_labels: true`（預設） | 新 instance 複製原 instance 的所有 labels |
| `inherit_labels: false` | 新 instance 不帶任何 label |

---

## 搜尋 API

### 基本查詢

```
GET /instances?workflow=order_processing&status=running
```

### Label 篩選

以 `label.` 前綴的 query parameter 篩選 label：

```
GET /instances?label.customer_id=CUST-789&label.region=tw
```

多個 label 條件之間為 AND 關係。

### 結合基本屬性與 label

```
GET /instances?workflow=order_processing&status=running&label.region=tw&label.priority=high
```

### 排序與分頁

引擎 MUST 支援 cursor-based 分頁：

```
GET /instances?label.region=tw&sort=created_at:desc&limit=50&cursor=eyJpZCI6MTAwfQ
```

| 參數 | 型別 | 說明 |
|------|------|------|
| `sort` | string | 排序欄位與方向，格式 `field:asc\|desc`，預設 `created_at:desc` |
| `limit` | integer | 每頁筆數，預設 50，最大 200 |
| `cursor` | string | 分頁游標（由前一次回應提供） |

#### 回應格式

```json
{
  "items": [
    {
      "instance_id": "inst_abc123",
      "workflow": "order_processing",
      "version": 1,
      "status": "running",
      "labels": {
        "customer_id": "CUST-789",
        "region": "tw",
        "priority": "high"
      },
      "created_at": "2026-03-31T10:00:00Z"
    }
  ],
  "next_cursor": "eyJpZCI6MTUwfQ",
  "has_more": true
}
```

---

## Storage

Labels 儲存於 `workflow_instances` 表的 JSONB 欄位，搭配 GIN index 加速查詢：

```sql
ALTER TABLE workflow_instances ADD COLUMN labels JSONB NOT NULL DEFAULT '{}';
CREATE INDEX idx_instances_labels ON workflow_instances USING GIN (labels);
```

### 查詢範例

```sql
-- 單一 label 查詢
SELECT * FROM workflow_instances
WHERE labels @> '{"region": "tw"}';

-- 多 label 查詢
SELECT * FROM workflow_instances
WHERE labels @> '{"region": "tw", "priority": "high"}';
```

---

## CLI 支援

### 查詢帶 label 的 instances

```bash
slogan instance list --label customer_id=CUST-789 --label region=tw
slogan instance list --label region=tw --status running --sort created_at:desc
```

### 設定 label

```bash
slogan instance label set <instance_id> priority=high region=tw
```

### 移除 label

```bash
slogan instance label remove <instance_id> priority temporary_flag
```

---

## 完整範例

### Workflow Definition（含 label_schema）

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: order_processing
  version: 1
  description: "訂單處理流程"

label_schema:
  customer_id:
    description: "客戶 ID"
  region:
    description: "部署區域"
    enum: [tw, jp, us, eu]
  priority:
    description: "優先級"
    enum: [low, medium, high]
    default: medium

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

steps:
  - id: analyze
    type: task
    action: order.analyze_risk
    input:
      order_id: ${ input.order_id }

  - id: set_priority
    type: task
    action: instance.set_labels
    input:
      labels:
        priority: ${ steps.analyze.output.risk_level }

  - id: process
    type: task
    action: order.process
    input:
      order_id: ${ input.order_id }

  - id: done
    type: return
    output:
      status: "completed"
```

### API 建立 Instance（附 labels）

```bash
curl -X POST https://engine.example.com/instances \
  -H "Content-Type: application/json" \
  -d '{
    "workflow": "order_processing",
    "version": 1,
    "input": { "order_id": "ORD-001" },
    "labels": {
      "customer_id": "CUST-789",
      "region": "tw"
    }
  }'
```

### 動態更新 label（builtin task）

```yaml
- id: escalate
  type: task
  action: instance.set_labels
  input:
    labels:
      priority: "high"
      escalated: "true"
      escalated_reason: ${ steps.check.output.reason }
```

---

## 相關文件

- [06-step-task](06-step-task.md) — builtin task `instance.set_labels`、`instance.remove_labels`
- [10-step-terminal](10-step-terminal.md) — continue-as-new 與 `inherit_labels`
- [02-document-structure](02-document-structure.md) — `label_schema` 在 Workflow 頂層結構中的位置
