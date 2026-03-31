# 10 — Steps: fail / return（含 continue-as-new）

本文件定義兩種終止 workflow 的 step 類型，以及 `return` step 的 continue-as-new（renew）機制。

---

## fail

立即終止 workflow instance，標記為 FAILED。

### Schema

```yaml
- type: fail
  message: string | CEL expression    # MUST — 錯誤訊息
  code: string                        # MAY — 機器可讀的錯誤碼
```

### 行為

- 在一般 step 序列中（包含巢狀 if、switch、foreach、parallel 內）：立即終止 workflow instance，Instance 狀態變為 FAILED
- 在 `on_error` / `on_timeout` handler 中：視為錯誤重新拋出，繼續向上層尋找 handler（見 [21-error-handling](21-error-handling.md)）
- `message` 與 `code` 記錄在 instance 的錯誤資訊中
- 後續 steps 不會執行

### 範例

```yaml
- type: fail
  message: "order has been cancelled"
  code: "order_cancelled"

- type: fail
  message: ${ "unknown action: " + input.action }
  code: "invalid_action"
```

---

## return

結束 workflow instance，標記為 SUCCEEDED，並回傳輸出資料。亦可搭配 `renew: true` 觸發 continue-as-new 機制。

### Schema

```yaml
- type: return
  output: map | CEL expression    # MAY — 回傳的資料
  renew: bool                     # MAY — 預設 false；true 時觸發 continue-as-new
  version: string                 # MAY — 僅 renew: true 時有效；指定新 instance 的 workflow version
  inherit_labels: bool            # MAY — 僅 renew: true 時有效；預設 true，繼承當前 instance 的 labels
  labels: map<string, string>     # MAY — 僅 renew: true 時有效；額外新增或覆寫的 labels
```

### 行為（一般 return）

- 立即終止 workflow instance，狀態變為 SUCCEEDED
- `output` 會依照 `output_schema` 驗證（若有定義）
- 驗證失敗 → instance 狀態變為 FAILED（錯誤碼 `schema_validation_error`）
- 後續 steps 不會執行
- 若 `output` 未指定，回傳 `null`

### 行為（renew: true — continue-as-new）

當 `renew: true` 時，引擎執行以下流程：

1. 以 `output` 作為新 instance 的 `input`，建立新 workflow instance
2. 若指定 `version`，新 instance 使用該版本的 workflow definition；否則使用相同版本
3. 舊 instance 狀態設為 `CONTINUED`（終態，計入 completed 統計）
4. 舊 instance 記錄新 instance ID（`continued_as` 欄位）
5. 新 instance 記錄舊 instance ID（`continued_from` 欄位）
6. 若 `inherit_labels` 為 `true`（預設），舊 instance 的 labels 會複製到新 instance
7. 若指定 `labels`，在繼承的基礎上新增或覆寫

```
Instance A (CONTINUED) ──continued_as──→ Instance B (RUNNING)
                        ←continued_from──
```

> `CONTINUED` 是終態，語意為「已由新 instance 接續」，視為成功完成的一種。詳見 [23-lifecycle](23-lifecycle.md)。

### 範例

```yaml
# 一般 return
- type: return
  output:
    order_id: ${ steps.load_order.output.id }
    status: "completed"
    payment_id: ${ steps.create_payment.output.payment_id }

# continue-as-new：帶 cursor 繼續
- type: return
  renew: true
  output:
    cursor: ${ steps.batch.output.next_cursor }
    total_synced: ${ input.total_synced + size(steps.batch.output.items) }

# continue-as-new：指定新版本 + labels
- type: return
  renew: true
  version: "2"
  inherit_labels: true
  labels:
    batch_number: ${ string(input.batch_number + 1) }
  output:
    cursor: ${ vars.last_cursor }
    config: ${ input.config }
```

---

## Continue-as-New 詳細說明

### 設計動機

長時間執行的 workflow（如定期輪詢、持續監控）會累積大量的 step output、event 歷史與對話紀錄，導致：

- **Storage 膨脹**：instance 資料不斷增長
- **效能下降**：expression 求值時 `steps` namespace 越來越大
- **Recovery 風險**：crash recovery 需要重播大量歷史

Continue-as-new 允許 workflow 在執行過程中「重啟自己」，建立新 instance 並帶入精簡的 state，舊 instance 標記完成。

### Storage Schema 變更

```sql
ALTER TABLE workflow_instances
  ADD COLUMN continued_from TEXT REFERENCES workflow_instances(id),
  ADD COLUMN continued_as TEXT REFERENCES workflow_instances(id);
```

### 舊 Instance 資料保留策略

引擎 MUST 同時支援兩種策略：

- **自動清理**：引擎依配置的保留期限（如 7 天）自動清除 `CONTINUED` 狀態的舊 instance 資料
- **手動清理**：管理者透過 API 手動刪除舊 instance

### 查詢 API：追蹤 Continue-as-New 鏈

```
GET /instances/{id}/chain
```

回傳整條 continue-as-new 鏈：

```json
{
  "chain": [
    { "id": "inst_001", "status": "CONTINUED", "continued_as": "inst_002" },
    { "id": "inst_002", "status": "CONTINUED", "continued_as": "inst_003" },
    { "id": "inst_003", "status": "RUNNING", "continued_as": null }
  ]
}
```

### Label 繼承

Continue-as-new 時，label 預設繼承至新 instance。可透過 `labels` 欄位額外新增或覆寫：

```yaml
- type: return
  renew: true
  inherit_labels: true       # 預設 true
  labels:
    batch_number: ${ string(input.batch_number + 1) }
  output:
    cursor: ${ vars.last_cursor }
```

若設定 `inherit_labels: false`，新 instance 僅包含 `labels` 中明確指定的 labels。

### 完整範例：資料同步（cursor 分頁）

```yaml
apiVersion: workflow/v3
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
    condition: ${ steps.batch.output.has_more }
    then:
      - type: return
        renew: true
        output:
          cursor: ${ steps.batch.output.next_cursor }
          total_synced: ${ input.total_synced + size(steps.batch.output.items) }

  # 無更多資料 → 正常結束
  - type: return
    output:
      total_synced: ${ input.total_synced + size(steps.batch.output.items) }
      status: "completed"
```

### 完整範例：長期監控

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: long_term_monitor
  version: 1

input_schema:
  type: object
  properties:
    target: { type: string }
    check_count: { type: integer, default: 0 }

steps:
  - id: check
    type: task
    action: monitoring.check_status
    input: { target: ${ input.target } }

  - type: if
    condition: ${ steps.check.output.alert }
    then:
      - type: task
        action: notification.send
        input:
          message: ${ steps.check.output.alert_message }

  # 每輪結束後 continue-as-new，避免歷史無限增長
  - type: return
    renew: true
    output:
      target: ${ input.target }
      check_count: ${ input.check_count + 1 }
```

---

## 提前結束語意

`fail` 和 `return` 都是 **立即終止** 指令：

- 在一般 step 序列中：無論出現在哪個巢狀層級（if branch、switch case、foreach iteration、parallel branch），都會終止整個 workflow instance
- **例外**：在 `on_error` / `on_timeout` handler 中使用 `fail` 時，不會直接終止 workflow，而是將錯誤重新拋出至上層 handler（見 [21-error-handling](21-error-handling.md)）
- 不會自動執行任何 cleanup 或 compensation 邏輯
- 若需要在失敗時進行清理，SHOULD 使用 `on_error` handler 或 `saga` 區塊（見 [12-step-saga](12-step-saga.md)）

### 隱式完成

若 workflow 的 `steps` 陣列全部執行完畢，且沒有遇到 `return` 或 `fail`：

- Instance 狀態變為 SUCCEEDED
- Output 為 `null`
- 若有定義 `output_schema` 且 schema 有 `required` 欄位 → FAILED（`schema_validation_error`）
