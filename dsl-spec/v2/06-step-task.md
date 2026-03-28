# 06 — Step: task

`task` step 呼叫一個已定義的 task definition 來執行工作。Task 的定義（怎麼執行）與使用（在 workflow 中呼叫）是分離的。

Task definition 的完整規格見 [17-task-definition](17-task-definition.md)。

---

## Schema

```yaml
- id: string                          # MAY — 需要參照 output 時才必填
  type: task                          # MUST
  action: string                      # MUST — task definition 的 name
  version: integer | "latest"         # MAY, 預設 "latest"
  input: map | CEL expression         # MAY — 傳給 task 的輸入
  resources: array                    # MAY — artifact 綁定
  execution:
    policy: replayable | idempotent | non_repeatable  # MAY, 預設 replayable
  retry:
    max_attempts: integer             # MAY, 預設 1（不重試）
    delay: duration                   # MAY, 預設 1s
    backoff: fixed | exponential      # MAY, 預設 fixed
  timeout: duration                   # MAY
  on_error: [...]                     # MAY, step 陣列
  on_timeout: [...]                   # MAY, step 陣列
```

---

## action 與 Task Definition 的關係

`action` 欄位參照一個 task definition 的 `metadata.name`：

```yaml
# Workflow step（使用端）
- id: load_order
  type: task
  action: order.load          # ← 參照 task definition name
  input:
    order_id: ${ input.order_id }

# Task definition（定義端，獨立 YAML 檔）
# apiVersion: task/v2
# kind: Task
# metadata:
#   name: order.load          # ← 被參照的名稱
#   version: 3
# backend:
#   type: sdk
#   command: "node ./tools/order-tasks/load-order.js"
```

### 版本解析

| 值 | 說明 |
|----|------|
| `"latest"`（預設） | 執行時解析為最新的 PUBLISHED 版本 |
| `integer` | 固定版本號 |

在正式環境中 SHOULD 使用固定版本號以確保可重現性。

---

## input 傳遞

`input` 為傳給 task definition 的輸入資料，會在執行前依 task definition 的 `input.schema` 驗證。

### 字面值 map

```yaml
input:
  order_id: ${ input.order_id }
  amount: 100
  currency: "USD"
```

### 單一 CEL 表達式（回傳 map）

```yaml
input: ${ vars.payment_input }
```

### 混合

```yaml
input:
  order_id: ${ steps.load_order.output.id }
  label: "manual"
```

---

## output 模型

Task handler 回傳的值即為 step output，後續 steps 透過 `steps.<id>.output` 存取：

```yaml
# 假設 order.load 回傳 { id: "ORD-001", amount: 500, status: "active" }

- id: check
  type: if
  expr: ${ steps.load_order.output.status == "active" }
```

Output 會依 task definition 的 `output.schema` 驗證（若有定義）。

---

## resources 綁定

將 artifact 綁定到 task，指定存取權限：

```yaml
resources:
  - name: order_file        # handler 內部使用的名稱
    type: artifact
    ref: order_file          # 對應 artifacts 區塊中的 key
    access: read             # read | write | read_write
```

詳見 [13-artifacts](13-artifacts.md)。

---

## Execution Policy

定義 task 在 crash recovery 時的重跑策略。Policy 定義在 **workflow step** 中，而非 task definition 中。

| Policy | 說明 | Recovery 行為 |
|--------|------|---------------|
| `replayable` | 可安全重跑，無副作用或有補償邏輯 | 重跑 |
| `idempotent` | 可重跑，使用 idempotency key 確保結果一致 | 帶 key 重跑 |
| `non_repeatable` | 不可重跑（如扣款、發送通知） | 直接標記 FAILED |

### 預設值

未指定 `execution.policy` 時，預設為 `replayable`。

### idempotent 的 key

Runtime SHOULD 使用 `workflow_instance_id + step_id + attempt_number` 作為 idempotency key，傳遞給 task handler。

### 為何不定義在 task definition 中

同一個 task definition 在不同 workflow 情境下可能需要不同的 policy。例如 `payment.create` 在主流程中是 `non_repeatable`，但在測試 workflow 中可以是 `replayable`。

---

## Retry 機制

| 欄位 | 型別 | 預設 | 說明 |
|------|------|------|------|
| `max_attempts` | integer | 1 | 總嘗試次數（含首次），1 = 不重試 |
| `delay` | duration | 1s | 重試間隔 |
| `backoff` | string | fixed | `fixed`: 固定間隔；`exponential`: 指數遞增 |

### 行為

- 僅在 step FAILED 時重試，TIMED_OUT 不觸發重試
- 重試次數用盡仍失敗 → 觸發 `on_error`（若有），否則 step 最終狀態為 FAILED
- `exponential` backoff：delay × 2^(attempt - 1)，例如 delay=1s 則重試間隔為 1s, 2s, 4s, ...

---

## 範例

### 基本呼叫

```yaml
- id: load_order
  type: task
  action: order.load
  input:
    order_id: ${ input.order_id }
```

### 帶版本固定與 retry

```yaml
- id: load_order
  type: task
  action: order.load
  version: 3
  input:
    order_id: ${ input.order_id }
  retry:
    max_attempts: 3
    delay: 2s
    backoff: exponential
  timeout: 10s
  execution:
    policy: replayable
```

### Non-repeatable task

```yaml
- id: create_payment
  type: task
  action: payment.create
  input:
    order_id: ${ steps.load_order.output.id }
    amount: ${ steps.load_order.output.amount }
  execution:
    policy: non_repeatable
  timeout: 30s
  on_error:
    - type: emit
      event: payment.failed
      data:
        error: ${ error.message }
```
