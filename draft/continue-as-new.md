# Continue-as-New

> **狀態**：草稿
> **範圍**：長時間 workflow 重置機制，帶新 state 重啟並截斷歷史，避免無限增長

---

## 設計動機

長時間執行的 workflow（如定期輪詢、持續監控）會累積大量的 step output、event 歷史與對話紀錄，導致：

- **Storage 膨脹**：instance 資料不斷增長
- **效能下降**：expression 求值時 `steps` namespace 越來越大
- **Recovery 風險**：crash recovery 需要重播大量歷史

Continue-as-new 允許 workflow 在執行過程中「重啟自己」，建立新 instance 並帶入精簡的 state，舊 instance 標記完成。

---

## 語法設計

### 方案 A：專用 Step Type

```yaml
- type: continue_as_new
  input:
    # 帶入新 instance 的 input（精簡後的 state）
    cursor: ${ vars.last_cursor }
    processed_count: ${ vars.processed_count + size(steps.batch.output.items) }
    config: ${ input.config }
```

### 方案 B：`return` 的變體

```yaml
- type: return
  continue_as_new: true
  output:
    cursor: ${ vars.last_cursor }
    processed_count: ${ vars.processed_count + size(steps.batch.output.items) }
```

---

## 執行語意

1. 引擎遇到 `continue_as_new` step
2. 建立新 workflow instance（相同 workflow name + version）
3. 新 instance 的 `input` = `continue_as_new` 的 input/output
4. 舊 instance 狀態設為 `CONTINUED`（新終態）
5. 舊 instance 記錄新 instance ID（`continued_as` 欄位）
6. 新 instance 記錄舊 instance ID（`continued_from` 欄位）

```
Instance A (CONTINUED) ──continued_as──→ Instance B (RUNNING)
                        ←continued_from──
```

### Instance 狀態變更

新增 `CONTINUED` 作為終態：

```
RUNNING → CONTINUED（觸發 continue_as_new 時）
```

`CONTINUED` 與 `COMPLETED` 類似，但語意為「已由新 instance 接續」。

---

## 典型場景

### 定期輪詢

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: data_sync
  version: 1

input_schema:
  type: object
  properties:
    cursor: { type: string, default: "" }
    total_synced: { type: integer, default: 0 }

steps:
  - id: batch
    type: task
    action: data.fetch_batch
    input:
      cursor: ${ input.cursor }
      batch_size: 100

  - id: process
    type: foreach
    items: ${ steps.batch.output.items }
    steps:
      - type: task
        action: data.sync_item
        input:
          item: ${ item }

  # 還有更多資料 → continue-as-new
  - type: if
    expr: ${ steps.batch.output.has_more }
    then:
      - type: continue_as_new
        input:
          cursor: ${ steps.batch.output.next_cursor }
          total_synced: ${ input.total_synced + size(steps.batch.output.items) }

  # 無更多資料 → 正常結束
  - type: return
    output:
      total_synced: ${ input.total_synced + size(steps.batch.output.items) }
      status: "completed"
```

### 長期監控

```yaml
steps:
  - id: check
    type: task
    action: monitoring.check_status
    input: { target: ${ input.target } }

  - type: if
    expr: ${ steps.check.output.alert }
    then:
      - type: task
        action: notification.send
        input:
          message: ${ steps.check.output.alert_message }

  # 每輪結束後 continue-as-new，避免歷史無限增長
  - type: continue_as_new
    input:
      target: ${ input.target }
      check_count: ${ input.check_count + 1 }
```

---

## 查詢 API

### 追蹤 Continue-as-New 鏈

```
GET /instances/{id}/chain    # 回傳整條 continue-as-new 鏈
```

```json
{
  "chain": [
    { "id": "inst_001", "status": "CONTINUED", "continued_as": "inst_002" },
    { "id": "inst_002", "status": "CONTINUED", "continued_as": "inst_003" },
    { "id": "inst_003", "status": "RUNNING", "continued_as": null }
  ]
}
```

### Storage Schema 變更

```sql
ALTER TABLE workflow_instances
  ADD COLUMN continued_from TEXT REFERENCES workflow_instances(id),
  ADD COLUMN continued_as TEXT REFERENCES workflow_instances(id);
```

---

## 與 Labels 的整合

Continue-as-new 時，label 預設繼承至新 instance：

```yaml
- type: continue_as_new
  input: { ... }
  inherit_labels: true       # 預設 true
  labels:                    # 額外新增或覆寫的 label
    batch_number: ${ string(input.batch_number + 1) }
```

---

## 待決定

- Step type 命名：`continue_as_new` vs `restart` vs `renew`
- 方案 A（專用 step）vs 方案 B（return 變體）
- 是否允許 continue 到不同版本的 workflow
- 舊 instance 的資料保留策略（自動清理 vs 手動）
- `CONTINUED` 是否計入 completed 的統計
- 深度限制（避免無限 continue-as-new 迴圈）
