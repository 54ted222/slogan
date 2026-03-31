# 本地測試模式

> **狀態**：草稿
> **範圍**：`slogan test` 命令，支援本地執行 workflow、mock task backend、replay 歷史紀錄驗證

---

## 設計動機

目前 workflow 開發缺乏快速驗證手段。開發者需要部署引擎、註冊 definition、建立 instance 才能測試。本地測試模式的目標：

- **快速回饋**：撰寫 YAML 後立即在本地驗證邏輯
- **無需基礎設施**：不需要引擎、資料庫、MCP server
- **Mock 支援**：模擬 task 回傳值，測試分支邏輯
- **Replay 驗證**：以歷史紀錄重播，確保 workflow 變更不影響既有行為

---

## CLI 命令

### 基本用法

```bash
# 執行 workflow，傳入 input
slogan test workflow.yaml --input '{"order_id": "ORD-001"}'

# 從檔案讀取 input
slogan test workflow.yaml --input-file input.json

# 指定 mock 檔案
slogan test workflow.yaml --input '{"order_id": "ORD-001"}' --mock mocks.yaml

# Replay 模式
slogan test workflow.yaml --replay recording.json
```

### 輸出

```bash
# 預設輸出：最終 output
$ slogan test workflow.yaml --input '{"order_id": "ORD-001"}' --mock mocks.yaml
✓ Step: load_order (task) — SUCCEEDED (mocked)
✓ Step: analyze (agent) — SUCCEEDED (mocked)
✓ Step: check_risk (if) — took then branch
✓ Step: flag_review (task) — SUCCEEDED (mocked)

Output:
{
  "status": "flagged",
  "risk_level": "high"
}

# 詳細模式
slogan test workflow.yaml --input '...' --mock mocks.yaml --verbose
```

---

## Mock 定義

### Mock 檔案格式

```yaml
# mocks.yaml
tasks:
  order.load:
    output:
      order_id: "ORD-001"
      amount: 50000
      status: "pending"
      customer_id: "CUST-789"

  customer.get_history:
    output:
      total_orders: 3
      avg_amount: 5000
      risk_flags: ["high_value_first_order"]

  order.flag_for_review:
    output:
      review_id: "REV-001"

# Agent mock — 直接回傳 output，不實際呼叫 LLM
agents:
  order.analyzer:
    output:
      risk_level: "high"
      summary: "金額異常，首次大額訂單"
      flags: ["high_value", "new_customer"]

# Workflow mock（sub_workflow）
workflows:
  payment_processing:
    output:
      transaction_id: "TXN-001"
      status: "completed"
```

### 條件式 Mock

根據 input 回傳不同結果：

```yaml
tasks:
  order.load:
    cases:
      - when: ${ input.order_id == "ORD-001" }
        output:
          order_id: "ORD-001"
          amount: 50000
          status: "pending"
      - when: ${ input.order_id == "ORD-002" }
        output:
          order_id: "ORD-002"
          amount: 100
          status: "completed"
    default:
      error:
        code: "not_found"
        message: "Order not found"
```

### Mock 失敗

```yaml
tasks:
  notification.send:
    error:
      code: "timeout"
      message: "Notification service unavailable"
```

---

## Replay 模式

### 錄製

從引擎匯出 instance 的執行紀錄：

```bash
# 匯出 instance 的執行紀錄
slogan instance export <instance_id> --format recording > recording.json
```

### 錄製檔案格式

```json
{
  "workflow": "order_processing",
  "version": 1,
  "input": { "order_id": "ORD-001" },
  "steps": [
    {
      "id": "load_order",
      "type": "task",
      "action": "order.load",
      "input": { "order_id": "ORD-001" },
      "output": { "order_id": "ORD-001", "amount": 50000 },
      "status": "SUCCEEDED"
    }
  ],
  "output": { "status": "flagged" }
}
```

### Replay 驗證

```bash
# 以錄製檔案重播，驗證 workflow 邏輯一致
slogan test workflow.yaml --replay recording.json

# 驗證結果
✓ Step: load_order — output matches recording
✓ Step: analyze — output matches recording
✗ Step: check_risk — branch diverged (expected: then, actual: else)
  → Workflow change affected branching logic
```

Replay 模式使用錄製檔案中的 task output 作為 mock，僅驗證控制流邏輯（分支選擇、step 執行順序、最終 output）。

---

## 本地執行引擎

`slogan test` 內嵌一個輕量級執行引擎：

| 元件 | 本地模式 | 完整引擎 |
|------|---------|---------|
| Step Executor | 內嵌（in-process） | 獨立 service |
| Expression Evaluator | 內嵌 CEL 引擎 | 相同 |
| Storage | In-memory | PostgreSQL |
| Task Executor | Mock 回傳 | 實際執行 task handler |
| MCP Server | 不啟動（mock） | 實際連線 |
| Event Router | In-memory | 完整 event system |

---

## 待決定

- Mock 檔案格式細節（YAML vs JSON）
- CEL 表達式在 mock `when` 中的支援範圍
- Replay 模式的比對策略（strict vs loose）
- 是否支援 step-by-step 互動式執行（debug 模式）
- Agent step 的 mock 粒度（整個 agent vs 個別 tool call）
- 是否支援 `--watch` 模式（檔案變更自動重跑）
