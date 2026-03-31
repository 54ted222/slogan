# Instance Labels + 搜尋 API

> **狀態**：草稿
> **範圍**：instance 自訂標籤（key-value）與進階查詢（按標籤、時間範圍、workflow name 過濾）

---

## 設計動機

目前 workflow instance 僅有基本屬性（ID、workflow name、status、created_at），缺乏使用者自訂的分類與查詢機制。實際場景中需要：

- 按業務維度分類 instance（如 `customer_id`、`region`、`priority`）
- 在大量 instance 中快速找到目標（如「某客戶的所有訂單處理 workflow」）
- 支援 Dashboard 與監控的多維度篩選

---

## Label 定義

### 在 Workflow Definition 中宣告

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: order_processing
  version: 1

# 定義此 workflow 支援的 label keys（可選，用於驗證）
label_schema:
  customer_id:
    type: string
    required: true
  region:
    type: string
    enum: [tw, jp, us, eu]
  priority:
    type: string
    enum: [low, medium, high]
    default: medium
```

### 在建立 Instance 時指定

```yaml
# API: POST /instances
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

### 在 Workflow 執行中動態設定

透過 `assign` step 或專用機制更新 label：

```yaml
# 方案 A：專用 step
- type: set_labels
  labels:
    priority: ${ steps.analyze.output.risk_level == "high" ? "high" : "medium" }

# 方案 B：內建 task
- type: task
  action: instance.set_labels
  input:
    labels:
      priority: ${ steps.analyze.output.risk_level }
```

---

## Label 限制

| 限制 | 值 |
|------|-----|
| Key 最大長度 | 63 字元 |
| Value 最大長度 | 256 字元 |
| 每個 instance 最多 label 數 | 32 |
| Key 命名規則 | `[a-z][a-z0-9_-]*`（小寫字母開頭） |
| Value 型別 | 僅 string |

---

## 搜尋 API

### 基本查詢

```
GET /instances?workflow=order_processing&status=running
```

### Label 篩選

```
GET /instances?label.customer_id=CUST-789&label.region=tw
```

### 進階查詢語法

```
GET /instances?filter=workflow:order_* AND label.region:tw AND status:running AND created_at>2025-01-01
```

| 運算子 | 說明 | 範例 |
|--------|------|------|
| `:` | 相等 | `label.region:tw` |
| `>` / `<` / `>=` / `<=` | 比較（時間、數值） | `created_at>2025-01-01` |
| `*` | 通配符 | `workflow:order_*` |
| `AND` / `OR` | 邏輯組合 | `status:running AND label.priority:high` |
| `NOT` | 否定 | `NOT status:completed` |

### 排序與分頁

```
GET /instances?filter=...&sort=created_at:desc&limit=50&cursor=xxx
```

---

## Storage Schema 變更

```sql
-- 新增 labels 表
CREATE TABLE instance_labels (
  instance_id TEXT NOT NULL REFERENCES workflow_instances(id),
  key TEXT NOT NULL,
  value TEXT NOT NULL,
  PRIMARY KEY (instance_id, key)
);

-- 索引（加速查詢）
CREATE INDEX idx_instance_labels_key_value ON instance_labels(key, value);
```

或在 `workflow_instances` 表中新增 JSONB 欄位：

```sql
ALTER TABLE workflow_instances ADD COLUMN labels JSONB DEFAULT '{}';
CREATE INDEX idx_instances_labels ON workflow_instances USING GIN (labels);
```

---

## CLI 支援

```bash
# 查詢帶 label 的 instances
slogan instance list --label customer_id=CUST-789 --label region=tw

# 設定 label
slogan instance label <id> priority=high

# 移除 label
slogan instance label <id> priority-
```

---

## 待決定

- Label 是否支援非 string 型別（integer、boolean）
- `label_schema` 是否強制（缺少 required label 時拒絕建立 instance）
- 搜尋語法採用簡單 query parameter 還是進階 filter DSL
- Storage 方案：獨立表 vs JSONB 欄位
- 是否支援 label 的歷史追蹤（何時被修改）
