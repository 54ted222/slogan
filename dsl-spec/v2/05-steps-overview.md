# 05 — Steps Overview

本文件定義 step 的共通屬性、執行順序與生命週期。

---

## Step 類型

| 類型 | 說明 |
|------|------|
| `task` | 呼叫 task definition |
| `assign` | 設定 vars 變數 |
| `if` | 條件分支 |
| `switch` | 多值分支 |
| `foreach` | 迴圈迭代 |
| `parallel` | 平行分支 |
| `emit` | 發送事件 |
| `wait_event` | 等待外部事件 |
| `fail` | 中止 workflow 並報錯 |
| `return` | 回傳結果並完成 workflow |
| `sub_workflow` | 呼叫另一個 workflow definition |

---

## 共通屬性

| 屬性 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `id` | string | MAY | 唯一識別字，`snake_case`（見下方 ID 規則） |
| `type` | string | MUST | step 類型（見上表） |
| `description` | string | MAY | 人類可讀說明 |
| `condition` | CEL expression | MAY | 前置條件，為 `false` 時 step 被 SKIPPED |

部分 step 類型額外支援：

| 屬性 | 適用類型 | 說明 |
|------|----------|------|
| `timeout` | task, sub_workflow, wait_event, foreach, parallel | 執行時間上限 |
| `retry` | task, sub_workflow | 重試設定 |
| `on_error` | task, sub_workflow, if, switch, foreach, parallel | 錯誤處理（值為 step 陣列） |
| `on_timeout` | task, sub_workflow, wait_event | timeout 處理（值為 step 陣列） |

### condition 範例

```yaml
- type: task
  action: notify.customer
  condition: ${ input.notify_customer == true }
  input:
    order_id: ${ input.order_id }
```

當 `condition` 求值為 `false`，該 step 直接進入 SKIPPED 狀態，不執行任何操作。

---

## Step ID 規則

`id` 為 **非必填**。規則如下：

### 何時需要 id

- 當後續 step 需要透過 `steps.<id>.output` 參照該 step 的輸出時，MUST 提供 `id`
- 提供 `id` 可提升可讀性，建議在重要的 step 上加上 `id`

### 何時可以省略 id

- 不需要被參照輸出的 step MAY 省略 `id`
- 典型場景：`emit`、`fail`、`return`、簡單的 `if` 分支內的 steps

### 自動產生 ID

- 引擎 MUST 為省略 `id` 的 step 自動產生內部 ID，用於持久化、logging、error reporting
- 自動產生的 ID 格式為 `_<type>_<index>`（如 `_emit_0`、`_fail_1`）
- 自動產生的 ID 以 `_` 開頭
- 使用者定義的 ID MUST NOT 以 `_` 開頭（避免衝突）

### 唯一性

- 所有使用者定義的 `id` MUST 在整個 workflow definition 中全域唯一（包含巢狀 steps）
- 這確保 `steps.<id>.output` 參照不會產生歧義
- id MUST 為 `snake_case`，僅含小寫字母、數字、底線

### 範例

```yaml
steps:
  # 有 id：後續需要參照 output
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  # 無 id：只是發事件，不需要被參照
  - type: emit
    event: order.processing
    data:
      order_id: ${ steps.load_order.output.id }

  # 無 id：終止 workflow
  - type: return
    output:
      status: "completed"
```

---

## 巢狀 Step 陣列的簡化語法

控制流程 step（if、switch、parallel）與 error handler 中的 step 陣列，直接使用陣列，不需要額外的 `steps` wrapper：

```yaml
# ✅ v2 語法：直接陣列
- type: if
  expr: ${ input.ready }
  then:
    - type: task
      action: order.process
    - type: emit
      event: order.done
  on_error:
    - type: fail
      message: "process failed"

# ❌ 不再使用 steps wrapper
- type: if
  then:
    steps:      # 不需要
      - ...
```

適用的欄位：`then`、`else`、`default`、`on_error`、`on_timeout`、`branches`（每個 branch）、`foreach` 的 `do`。

---

## 執行順序

- `steps` 陣列中的 steps 按順序執行（sequential）
- 平行執行僅發生在 `parallel` step 的 branches 內，以及 `foreach` 設定 `concurrency > 1` 時
- 每個 step 完成後，引擎才會啟動下一個 step

---

## Step 生命週期

```
PENDING ──→ READY ──→ RUNNING ──→ SUCCEEDED
   │            │        │   │
   │            │        │   ├──→ FAILED
   │            │        │   ├──→ WAITING    （僅 wait_event）
   │            │        │   └──→ TIMED_OUT
   │            │
   │            └──→ SKIPPED       （condition 為 false）
   │
   ├──→ CANCELLED   （外部取消：workflow cancel / parent timeout）
   │
READY ──→ CANCELLED
RUNNING ──→ CANCELLED
WAITING ──→ CANCELLED
```

### 狀態說明

| 狀態 | 說明 |
|------|------|
| PENDING | 尚未到達（前面的 step 還在執行） |
| READY | 前置條件滿足，等待排程執行 |
| RUNNING | 正在執行中 |
| SUCCEEDED | 執行成功完成 |
| FAILED | 執行失敗（包含 retry 用盡後仍失敗） |
| WAITING | 等待外部事件（僅 `wait_event`） |
| TIMED_OUT | 執行超時。若有 `on_timeout` handler 且正常完成，錯誤視為已處理，workflow 繼續執行下一個 step；step 狀態仍為 TIMED_OUT，output 為 `null` |
| SKIPPED | `condition` 為 false，或所在分支未被選中。output 為 `null` |
| CANCELLED | 被外部取消（workflow cancel 請求或 parent timeout 觸發） |

### 狀態轉換規則

- PENDING → READY：前一個 step 完成（SUCCEEDED、SKIPPED、或 TIMED_OUT 且 on_timeout 已處理）
- READY → RUNNING：排程器開始執行
- READY → SKIPPED：`condition` 求值為 `false`
- RUNNING → SUCCEEDED：執行完成
- RUNNING → FAILED：執行失敗（retry 用盡）
- RUNNING → WAITING：`wait_event` 進入等待
- RUNNING → TIMED_OUT：超過 `timeout`
- WAITING → RUNNING：收到匹配事件，恢復執行
- PENDING → CANCELLED：外部取消時，尚未到達的 step
- READY → CANCELLED：外部取消時，等待排程的 step
- RUNNING → CANCELLED：外部取消時，正在執行的 step
- WAITING → CANCELLED：外部取消時，等待事件的 step

所有 terminal 狀態（SUCCEEDED、FAILED、TIMED_OUT、SKIPPED、CANCELLED）MUST NOT 再轉換。

### Non-terminal 狀態的 output 語意

| 狀態 | `steps.<id>.output` |
|------|---------------------|
| SUCCEEDED | task/step 的回傳值 |
| SKIPPED | `null` |
| TIMED_OUT | `null`（即使 on_timeout handler 已處理） |
| FAILED | `null`（即使 on_error handler 已處理） |
| CANCELLED | `null` |
