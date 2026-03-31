# 30 — 本地測試模式（`slogan test`）

本文件定義 `slogan test` CLI 命令，支援本地執行 workflow、mock task / agent / sub_workflow 回傳值，以及 replay 歷史紀錄驗證。

---

## 設計動機

目前 workflow 開發缺乏快速驗證手段。開發者需要部署引擎、註冊 definition、建立 instance 才能測試。本地測試模式的目標：

- **快速回饋** — 撰寫 YAML 後立即在本地驗證邏輯
- **無需基礎設施** — 不需要引擎、資料庫、MCP server
- **Mock 支援** — 模擬 task、agent、sub_workflow 回傳值，測試分支邏輯
- **Replay 驗證** — 以歷史紀錄重播，確保 workflow 變更不影響既有行為

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

### 參數說明

| 參數 | 型別 | 說明 |
|------|------|------|
| `<file>` | positional | Workflow YAML 檔案路徑 |
| `--input` | string | JSON 格式的 workflow input |
| `--input-file` | path | 從 JSON 檔案讀取 input |
| `--mock` | path | Mock 檔案路徑（YAML 格式，決策 D1） |
| `--replay` | path | Replay 錄製檔案路徑（JSON 格式） |
| `--verbose` | flag | 顯示每個 step 的詳細 input / output |

### 限制（v1）

- **不支援 `--watch`**（決策 D4）— 不提供檔案變更自動重跑功能
- **不支援互動式 debug 模式**（決策 D5）— 不提供 step-by-step 暫停 / 繼續功能

---

## Mock 定義

Mock 檔案格式為 YAML（決策 D1），包含三個頂層區塊：`tasks`、`agents`、`workflows`。

### 整體結構

```yaml
# mocks.yaml
tasks:
  <task.action>:
    output: { ... }          # 直接回傳 output
    # 或
    cases: [...]             # 條件式 mock
    default: { ... }         # 條件皆不符時的 fallback
    # 或
    error: { ... }           # 模擬失敗

agents:
  <agent.name>:
    output: { ... }          # 整個 agent 直接回傳 output（決策 D3）

workflows:
  <workflow.name>:
    output: { ... }          # sub_workflow 直接回傳 output
```

### Task Mock

#### 固定 output

```yaml
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
```

#### 條件式 Mock

根據 input 回傳不同結果。`when` 使用 CEL 表達式，`input` 為該 task 實際收到的呼叫參數：

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

引擎依序評估 `cases`，匹配第一個 `when` 為 `true` 的 case。若皆不符則使用 `default`。

#### Mock 失敗

模擬 task 呼叫失敗，用於測試 `on_error`、`retry` 等錯誤處理邏輯：

```yaml
tasks:
  notification.send:
    error:
      code: "timeout"
      message: "Notification service unavailable"
```

### Agent Mock（決策 D3）

Agent mock 在整個 agent 層級運作，直接回傳 output，不實際呼叫 LLM 也不執行 tool call loop：

```yaml
agents:
  order.analyzer:
    output:
      risk_level: "high"
      summary: "金額異常，首次大額訂單"
      flags: ["high_value", "new_customer"]
```

> **v1 不支援細粒度 mock**（如個別 tool call 的 mock）。Agent mock 僅支援整個 agent 的 output。

### Workflow Mock（sub_workflow）

模擬 sub_workflow 的回傳值：

```yaml
workflows:
  payment_processing:
    output:
      transaction_id: "TXN-001"
      status: "completed"
```

---

## Replay 模式

Replay 模式使用引擎匯出的歷史執行紀錄重播 workflow，驗證 workflow 變更是否影響既有行為。

### 錄製匯出

從引擎匯出 instance 的完整執行紀錄：

```bash
slogan instance export <instance_id> --format recording > recording.json
```

### 錄製檔案格式

錄製檔案為 JSON 格式，包含 workflow input、每個 step 的 input / output、及最終結果：

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
    },
    {
      "id": "analyze",
      "type": "agent",
      "agent": "order.analyzer",
      "input": { "order_id": "ORD-001" },
      "output": { "risk_level": "high", "summary": "金額異常" },
      "status": "SUCCEEDED"
    },
    {
      "id": "check_risk",
      "type": "if",
      "branch": "then",
      "status": "SUCCEEDED"
    }
  ],
  "output": { "status": "flagged", "risk_level": "high" }
}
```

### Replay 驗證流程

1. 讀取錄製檔案中的 `input` 作為 workflow input
2. 錄製檔案中各 step 的 `output` 自動作為 mock 回傳值
3. 執行 workflow，逐步比對每個 step 的 output 與分支選擇

### 比對策略（決策 D2：Strict）

所有 step output MUST 完全一致：

| 比對項目 | 規則 |
|----------|------|
| Step output | JSON deep equal（key 順序無關） |
| 分支選擇 | if → `then` / `else` 必須一致；switch → 選中的 case 必須一致 |
| Step 執行順序 | 必須一致 |
| 最終 output | JSON deep equal |

### Replay 驗證輸出

```
$ slogan test workflow.yaml --replay recording.json

✓ Step: load_order — output matches recording
✓ Step: analyze — output matches recording
✗ Step: check_risk — branch diverged (expected: then, actual: else)
  → Workflow change affected branching logic

Result: DIVERGED (1 step diverged)
```

| 符號 | 狀態 | 說明 |
|------|------|------|
| `✓` | match | Step output 與分支選擇完全一致 |
| `✗` | diverge | Output 不一致或分支選擇不同 |

最終結果為 `PASSED`（所有 step 皆 match）或 `DIVERGED`（任一 step 不一致）。

---

## 輸出格式

### 預設輸出

```
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
```

### 詳細模式（`--verbose`）

```
$ slogan test workflow.yaml --input '{"order_id": "ORD-001"}' --mock mocks.yaml --verbose

✓ Step: load_order (task) — SUCCEEDED (mocked)
  Input:  {"order_id": "ORD-001"}
  Output: {"order_id": "ORD-001", "amount": 50000, "status": "pending"}

✓ Step: analyze (agent) — SUCCEEDED (mocked)
  Input:  {"order_id": "ORD-001"}
  Output: {"risk_level": "high", "summary": "金額異常，首次大額訂單"}

✓ Step: check_risk (if) — took then branch
  Condition: steps.load_order.output.amount > 10000 → true

✓ Step: flag_review (task) — SUCCEEDED (mocked)
  Input:  {"order_id": "ORD-001", "risk_level": "high"}
  Output: {"review_id": "REV-001"}

Output:
{
  "status": "flagged",
  "risk_level": "high"
}
```

---

## 本地執行引擎

`slogan test` 內嵌一個輕量級執行引擎，支援 workflow 邏輯驗證但不包含完整的基礎設施：

| 元件 | 本地模式（`slogan test`） | 完整引擎 |
|------|--------------------------|----------|
| Step Executor | 內嵌（in-process） | 獨立 service |
| Expression Evaluator | 內嵌 CEL 引擎 | 相同 |
| Storage | In-memory | PostgreSQL |
| Task Executor | Mock 回傳 | 實際執行 task handler |
| Agent Executor | Mock 回傳（整個 agent output） | 實際 LLM agentic loop |
| MCP Server | 不啟動（mock） | 實際連線 |
| Event Router | In-memory | 完整 event system |
| Timer / Timeout | 不等待（立即觸發） | 實際計時 |

---

## 完整範例

### 範例 1：基本 Mock 用法

**workflow.yaml**

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: order_risk_check
  version: 1

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

steps:
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  - id: analyze
    type: agent
    agent: order.analyzer
    prompt: "分析訂單風險"
    input:
      order_id: ${ input.order_id }

  - type: if
    when: ${ steps.analyze.output.risk_level == "high" }
    then:
      - id: flag
        type: task
        action: order.flag_for_review
        input:
          order_id: ${ input.order_id }
          reason: ${ steps.analyze.output.summary }

  - type: return
    output:
      risk_level: ${ steps.analyze.output.risk_level }
      summary: ${ steps.analyze.output.summary }
```

**mocks.yaml**

```yaml
tasks:
  order.load:
    output:
      order_id: "ORD-001"
      amount: 50000
      status: "pending"
      customer_id: "CUST-789"

  order.flag_for_review:
    output:
      review_id: "REV-001"

agents:
  order.analyzer:
    output:
      risk_level: "high"
      summary: "金額異常，首次大額訂單"
      flags: ["high_value", "new_customer"]
```

**執行**

```bash
slogan test workflow.yaml --input '{"order_id": "ORD-001"}' --mock mocks.yaml
```

### 範例 2：條件式 Mock

**mocks.yaml**

```yaml
tasks:
  order.load:
    cases:
      - when: ${ input.order_id == "ORD-001" }
        output:
          order_id: "ORD-001"
          amount: 50000
          status: "pending"
      - when: ${ input.order_id == "ORD-999" }
        output:
          order_id: "ORD-999"
          amount: 100
          status: "completed"
    default:
      error:
        code: "not_found"
        message: "Order not found"
```

**執行不同 input 驗證分支邏輯**

```bash
# 高金額 → 進入 flag 分支
slogan test workflow.yaml --input '{"order_id": "ORD-001"}' --mock mocks.yaml

# 低金額 → 跳過 flag
slogan test workflow.yaml --input '{"order_id": "ORD-999"}' --mock mocks.yaml

# 找不到 → on_error 處理
slogan test workflow.yaml --input '{"order_id": "ORD-XXX"}' --mock mocks.yaml
```

### 範例 3：Replay 驗證

```bash
# 從正式環境匯出錄製檔案
slogan instance export inst_abc123 --format recording > recording.json

# 以新版 workflow 重播驗證
slogan test workflow-v2.yaml --replay recording.json
```

---

## 相關文件

- [05-steps-overview](../02-steps/05-steps-overview.md) — Step 共通屬性與類型
- [13-agent-definition](../03-agent/13-agent-definition.md) — Agent Definition
- [21-error-handling](../05-runtime/21-error-handling.md) — 錯誤處理機制
