# 05 — Steps Overview

本文件定義 step 的共通屬性、執行順序與生命週期。

---

## Step 類型

| 類型           | 說明                                            |
| -------------- | ----------------------------------------------- |
| `task`         | 呼叫 task definition                            |
| `assign`       | 設定 vars 變數                                  |
| `if`           | 條件分支                                        |
| `switch`       | 多值分支                                        |
| `foreach`      | 迴圈迭代                                        |
| `parallel`     | 平行分支                                        |
| `emit`         | 發送事件                                        |
| `wait_event`   | 等待外部事件                                    |
| `fail`         | 中止 workflow 並報錯                            |
| `return`       | 回傳結果並完成 workflow（支援 continue-as-new） |
| `sub_workflow` | 呼叫另一個 workflow definition                  |
| `agent`        | 呼叫 agent definition                           |
| `saga`         | 補償區塊，包裹一組需要交易性補償的 steps        |

---

## 共通屬性

| 屬性          | 型別           | 必填 | 說明                                                                            |
| ------------- | -------------- | ---- | ------------------------------------------------------------------------------- |
| `id`          | string         | MAY  | 唯一識別字，`snake_case`（見下方 ID 規則）                                      |
| `type`        | string         | MUST | step 類型（見上表）                                                             |
| `description` | string         | MAY  | 人類可讀說明                                                                    |
| `when`        | CEL expression | MAY  | 前置條件，為 `false` 時 step 被 SKIPPED                                         |
| `compensate`  | step 陣列      | MAY  | 補償邏輯，saga 範圍內 step 失敗時反向執行（見 [12-step-saga](12-step-saga.md)） |

部分 step 類型額外支援：

| 屬性         | 適用類型                                                       | 說明                           |
| ------------ | -------------------------------------------------------------- | ------------------------------ |
| `timeout`    | task, sub_workflow, wait_event, agent                          | 執行時間上限                   |
| `retry`      | task, sub_workflow, agent                                      | 重試設定                       |
| `on_error`   | task, sub_workflow, agent, if, switch, foreach, parallel, saga | 錯誤處理（值為 step 陣列）     |
| `on_timeout` | task, sub_workflow, wait_event, agent                          | timeout 處理（值為 step 陣列） |

### when 範例

```yaml
- type: task
  action: notify.customer
  when: ${ input.notify_customer == true }
  input:
    order_id: ${ input.order_id }
```

當 `when` 求值為 `false`，該 step 直接進入 SKIPPED 狀態，不執行任何操作。

### compensate 範例

```yaml
- id: debit
  type: task
  action: payment.debit
  input:
    amount: ${ input.amount }
  compensate:
    - type: task
      action: payment.refund
      input:
        amount: ${ input.amount }
        reference: ${ steps.debit.output.transaction_id }
```

`compensate` 區塊僅在 `saga` step 範圍內或 workflow 觸發補償時執行。詳見 [12-step-saga](12-step-saga.md)。

---

## Step ID 規則

`id` 為 **非必填**。規則如下：

### 何時需要 id

- 當**非緊鄰的後續 step** 需要透過 `steps.<id>.output` 參照該 step 的輸出時，MUST 提供 `id`
- 提供 `id` 可提升可讀性，建議在重要的 step 上加上 `id`

### 何時可以省略 id

- 緊接的下一個 step 可透過 `prev` 存取前一步的輸出，不需要 `id`
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
  # 有 id：被非緊鄰的後續 step 參照
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  # 無 id：用 prev 參照前一步的輸出
  - type: emit
    event: order.processing
    data:
      order_id: ${ prev.output.id }

  # 無 id：用 steps.<id> 參照非緊鄰的 step
  - type: task
    action: order.process
    input:
      order_id: ${ steps.load_order.output.id }

  # 無 id：終止 workflow
  - type: return
    output:
      status: "completed"
```

---

## 巢狀 Step 陣列的簡化語法

控制流程 step（if、switch、parallel、saga）與 error handler 中的 step 陣列，直接使用陣列，不需要額外的 `steps` wrapper：

```yaml
# ✅ 直接陣列
- type: if
  when: ${ input.ready }
  then:
    - type: task
      action: order.process
    - type: emit
      event: order.done
  on_error:
    - type: fail
      message: "process failed"
```

適用的欄位：`then`、`else`、`default`、`on_error`、`on_timeout`、`compensate`、`branches`（每個 branch）、`foreach` 的 `do`、`saga` 的 `steps`。

---

## 執行順序

- `steps` 陣列中的 steps 按順序執行（sequential）
- 平行執行僅發生在 `parallel` step 的 branches 內，以及 `foreach` 設定 `concurrency > 1` 時
- 每個 step 完成後，引擎才會啟動下一個 step

---

## Step 生命週期

```
PENDING ──→ READY ──→ RUNNING ──→ SUCCEEDED
                │        │
                │        ├──→ FAILED
                │        ├──→ SKIPPED    （if/switch 無匹配分支）
                │        ├──→ WAITING ──⇄── RUNNING（收到事件）
                │        │      ├──→ TIMED_OUT （等待逾時）
                │        │      └──→ CANCELLED
                │        ├──→ TIMED_OUT
                │        └──→ CANCELLED  （外部取消）
                │
                └──→ SKIPPED       （when 為 false）
```

### 狀態說明

| 狀態      | 說明                                                                |
| --------- | ------------------------------------------------------------------- |
| PENDING   | 尚未到達（前面的 step 還在執行）                                    |
| READY     | 前置條件滿足，等待排程執行                                          |
| RUNNING   | 正在執行中                                                          |
| SUCCEEDED | 執行成功完成                                                        |
| FAILED    | 執行失敗（包含 retry 用盡後仍失敗）                                 |
| WAITING   | 等待外部事件（`wait_event`）或人類回覆（agent `ask_human`）         |
| TIMED_OUT | 執行超時                                                            |
| CANCELLED | 被外部取消（workflow timeout、parent 取消、API 取消）               |
| SKIPPED   | `when` 為 false、所在分支未被選中、或控制流程 step 無匹配的執行分支 |

### 狀態轉換規則

- PENDING → READY：前一個 step 完成（SUCCEEDED、SKIPPED、或 FAILED / TIMED_OUT 但錯誤已被 handler 處理）
- READY → RUNNING：排程器開始執行
- READY → SKIPPED：`when` 求值為 `false`
- RUNNING → SKIPPED：控制流程 step 無匹配的執行分支
- RUNNING → SUCCEEDED：執行完成
- RUNNING → FAILED：執行失敗（retry 用盡）
- RUNNING → WAITING：`wait_event` 進入等待、或 agent `ask_human` 等待人類回覆
- RUNNING → TIMED_OUT：超過 `timeout`
- RUNNING → CANCELLED：外部取消（workflow timeout 或取消請求）
- WAITING → RUNNING：收到匹配事件或人類回覆，恢復執行
- WAITING → TIMED_OUT：等待逾時
- WAITING → CANCELLED：外部取消

所有 terminal 狀態（SUCCEEDED、FAILED、TIMED_OUT、CANCELLED、SKIPPED）MUST NOT 再轉換。
