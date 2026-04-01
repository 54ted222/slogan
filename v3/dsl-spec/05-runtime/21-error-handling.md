# 21 — 錯誤處理

本文件定義統一的錯誤處理與 timeout 處理模型。

---

## 錯誤處理層級

錯誤處理分為三個層級，依優先順序由高至低：

| 層級 | 位置 | 捕捉範圍 |
|------|------|----------|
| **Step-level** | step 上的 `catch` | 該 step 本身的錯誤 |
| **Block-level** | 控制流程 step 上的 `catch`（if / switch / foreach / parallel / saga） | 該 block 內未被 step-level 捕捉的錯誤 |
| **Workflow-level** | `config.catch` | 所有未被上層捕捉的錯誤 |

### 優先順序

Step 失敗時，引擎按以下順序尋找 handler：

1. 該 step 自身的 `catch` → 若有，執行
2. 包含該 step 的最近控制流程 step 的 `catch` → 若有，執行
3. 外層控制流程 step 的 `catch`（向上遞迴）
4. `config.catch` → 若有，執行
5. 都沒有 → workflow instance FAILED

第一個匹配的 handler 處理錯誤，不再向上傳播（除非 handler 內使用 `fail` 重新拋出）。

---

## catch 結構

`catch` 的值為 step 陣列（不需要 `steps` wrapper）：

```yaml
catch:
  - type: emit
    event: workflow.step_failed
    data:
      step: ${ error.step_id }
      message: ${ error.message }
```

### 適用的 step 類型

以下 step 類型支援 `catch` 區塊：

| Step 類型 | 說明 |
|-----------|------|
| `task` | 捕捉 task handler 執行錯誤 |
| `sub_workflow` | 捕捉子 workflow 失敗 |
| `agent` | 捕捉 agent session 錯誤（見下方 [Agent 錯誤碼](#agent-錯誤碼)） |
| `saga` | 捕捉 saga 補償後的最終錯誤（見 [12-step-saga](../02-steps/12-step-saga.md)） |
| `if` / `switch` / `foreach` / `parallel` | Block-level 捕捉 |
| `wait` | 捕捉等待逾時降級的錯誤 |

### Handler 內可用的 namespace

| 變數 | 型別 | 說明 |
|------|------|------|
| `error.message` | string | 錯誤訊息 |
| `error.code` | string | 錯誤碼 |
| `error.step_id` | string | 發生錯誤的 step id |

此外，handler 也可存取 `input`、`steps`、`vars` 等一般 namespace。

### Handler 完成行為

| Handler 結果 | 後續行為 |
|-------------|----------|
| 正常完成（無 `fail`） | 錯誤視為已處理，workflow 繼續執行（見下方「繼續點」） |
| 使用 `fail` step | 錯誤重新拋出，繼續向上層尋找 handler |
| Handler 自身失敗 | 視為未處理錯誤，繼續向上層尋找 handler |

### 繼續點（handler 正常完成後）

Handler 正常完成時，workflow 的繼續執行位置取決於 handler 所在的層級：

| Handler 層級 | 繼續點 |
|-------------|--------|
| **Step-level** | 失敗 step 的同一序列中，下一個 step |
| **Block-level** | 控制流程 step（if / switch / foreach / parallel / saga）的同一序列中，下一個 step |
| **Workflow-level** | 失敗 step 在頂層 `steps` 中對應的下一個 step |

### 與 foreach / parallel failure_policy 的交互

- 若 `foreach` / `parallel` 內某個 step 失敗，且該 step 的 `catch` 正常完成（未使用 `fail`）→ 該 step 視為已處理，迭代/分支繼續執行，不觸發 `failure_policy`
- 若 step 的 `catch` 不存在或重新拋出錯誤 → 該迭代/分支視為失敗，`failure_policy` 生效

---

## catch 與 saga compensate 的關係

`catch` 與 saga `compensate` 是兩種互補的錯誤處理機制，執行順序明確：

1. Step 失敗 → 先執行該 step 的 `catch`（若有）
2. `catch` 正常完成 → 錯誤已處理，step 不再為 FAILED → **不觸發** saga 補償，workflow 繼續
3. `catch` 不存在、重新拋出錯誤、或自身失敗 → step 仍為 FAILED → **觸發** saga 補償（若 step 位於 saga 範圍內）

| 機制 | 觸發範圍 | 用途 |
|------|----------|------|
| `catch` | 單一 step 失敗 | 錯誤處理（降級、記錄、重新拋出） |
| `compensate` | Saga 範圍內 step 失敗 | 反向回復已完成的副作用 |

兩者互補而非互斥：`catch` 處理「如何應對當下的錯誤」，`compensate` 處理「如何回滾已完成的操作」。典型模式是在 `catch` 中進行告警或降級嘗試，若無法恢復則讓 saga 補償接手回滾。

> 詳見 [12-step-saga](../02-steps/12-step-saga.md) 的「與 catch 的關係」章節。

---

## on_timeout 結構

`on_timeout` 的值同樣為 step 陣列：

```yaml
on_timeout:
  - type: emit
    event: workflow.step_timeout
    data:
      step: ${ timeout.step_id }
      duration: ${ string(timeout.duration) }
```

### Handler 內可用的 namespace

| 變數 | 型別 | 說明 |
|------|------|------|
| `timeout.step_id` | string | 超時的 step id |
| `timeout.duration` | duration | 設定的 timeout 值 |

### Handler 完成行為

與 `catch` 相同。

---

## Retry 與錯誤處理的交互

```
Step 執行 → 失敗
  ↓
有 retry？→ 是 → 重試（直到 max_attempts 用盡）
  ↓                    ↓
  否               仍然失敗
  ↓                    ↓
  └────────────────────┘
  ↓
有 catch？→ 是 → 執行 handler
  ↓                    ↓
  否             handler 結果
  ↓                    ↓
  └────────────────────┘
  ↓
向上層尋找 handler（block → workflow）
  ↓
都沒有 → 位於 saga 內？→ 是 → 觸發 saga 補償
  ↓                              ↓
  否                        補償結果
  ↓                              ↓
  └──────────────────────────────┘
  ↓
workflow FAILED
```

重點：retry 會在 `catch` 之前全部用盡。`catch` 在 saga 補償之前執行。

---

## Timeout 級聯

### Step timeout

- 適用於 `task`、`sub_workflow`、`wait`、`agent`
- 超時 → step TIMED_OUT → 觸發該 step 的 `on_timeout`
- `on_timeout` 不存在 → 視為 step FAILED，進入錯誤處理流程

### Workflow timeout

- 定義在 `config.timeout`
- 是 instance 從 CREATED 到完成的絕對時間上限
- 觸發時：
  1. 所有 RUNNING、WAITING 與 READY 的 steps 被取消（CANCELLED）
  2. 不觸發任何 step-level handler（`catch`、`on_timeout`）
  3. 觸發 `config.on_timeout`（若有）→ handler 完成後 instance 仍為 FAILED（handler 用途為清理與通知，不改變結果）
  4. 無 `config.on_timeout` → instance FAILED

`config.on_timeout` handler 中的 `timeout` namespace：`timeout.step_id` 為 `null`（workflow 級 timeout 無對應 step），`timeout.duration` 為 `config.timeout` 的值。

### Sub-workflow timeout

- Parent step 的 `timeout` 限制 child instance 的總執行時間
- Parent step timeout 觸發時：
  1. Child instance 被 CANCELLED
  2. Child 的 `config.on_timeout` **不**觸發（外部取消，非自身超時）
  3. Parent step 的 `on_timeout` 觸發

### 優先順序

Step timeout < workflow timeout。若 workflow timeout 先到，覆蓋所有 step-level 行為。

---

## 標準錯誤碼

### 錯誤碼結構

錯誤碼為 `snake_case` 字串。標準錯誤碼由引擎定義，自訂錯誤碼由 task handler 定義。

### Namespace 規則

| Namespace | 格式 | 來源 |
|-----------|------|------|
| 標準 | 無前綴（如 `task_failed`） | 引擎產生 |
| 自訂 | 建議使用 dotted namespace（如 `payment.insufficient_funds`） | Task handler 回報 |

Task handler 回傳的 `error.code` 由引擎原樣傳遞至 `catch` handler 的 `error.code` namespace。引擎不會改寫 handler 回傳的 code。

### 標準錯誤碼清單

#### Task 相關

| 錯誤碼 | 觸發條件 | 說明 |
|--------|---------|------|
| `task_failed` | Task handler 回報失敗（未指定 code 時） | Task 執行失敗的預設錯誤碼 |
| `task_not_found` | `action` 對應的 task definition 不存在或非 PUBLISHED | Task definition 找不到 |
| `task_version_archived` | 固定版本號的 task 已被 ARCHIVED | Task 版本已封存 |

#### Agent 相關

| 錯誤碼 | 觸發條件 | 是否終止 agent | 說明 |
|--------|---------|---------------|------|
| `agent_max_iterations` | 達到 `config.max_iterations` 上限但未完成 | 是 — step FAILED | 迭代次數超過上限 |
| `agent_max_tool_calls` | 達到 `config.max_tool_calls` 上限 | 是 — step FAILED | Tool 呼叫次數超過上限 |
| `agent_output_invalid` | Agent 回傳的 output 不符合 `output_schema` | 是 — step FAILED | 輸出格式驗證失敗 |
| `agent_llm_error` | LLM API 呼叫失敗（網路錯誤、rate limit、provider 錯誤） | 是 — step FAILED | LLM 呼叫失敗 |
| `agent_tool_failed` | Agent 呼叫的 tool 執行失敗 | **否** — 結果送回 LLM | Tool 失敗不終止 agent（見下方說明） |
| `agent_not_found` | 指定的 agent definition 不存在或非 PUBLISHED | 是 — step FAILED | Agent definition 找不到 |

> **重要**：`agent_tool_failed` 不會直接導致 agent step 失敗。Tool 失敗的結果會作為 tool result 送回 LLM，由 LLM 決定下一步（重試、使用其他 tool、或放棄）。只有當 LLM 明確表示無法完成任務，或達到迭代 / tool call 上限時，agent step 才會 FAILED。此錯誤碼記錄於 `agent_tool_calls` 表中，而非直接觸發 `catch`。
>
> Agent 錯誤碼的完整定義詳見 [13-agent-definition](../03-agent/13-agent-definition.md)。

#### Timeout 相關

| 錯誤碼 | 觸發條件 | 說明 |
|--------|---------|------|
| `step_timeout` | Step 執行超過 `timeout` 設定 | Step 級 timeout |
| `workflow_timeout` | Instance 超過 `config.timeout` 設定 | Workflow 級 timeout |
| `wait_timeout` | `wait`（event 模式）等待超過 `timeout` 設定 | Wait event 專用 timeout |

注意：`on_timeout` handler 透過 `timeout` namespace（而非 `error` namespace）取得 timeout 資訊。上述錯誤碼用於 timeout 未被 `on_timeout` 處理而降級為 `catch` 時。

#### Schema 相關

| 錯誤碼 | 觸發條件 | 說明 |
|--------|---------|------|
| `schema_validaticatch` | Input / output schema 驗證失敗 | 資料不符合 JSON Schema |
| `input_required` | 必填 input 欄位缺失 | Instance 建立或 task 呼叫時 |

#### 表達式相關

| 錯誤碼 | 觸發條件 | 說明 |
|--------|---------|------|
| `expressicatch` | CEL 表達式求值失敗 | 型別錯誤、null reference 等 |

#### 控制流程相關

| 錯誤碼 | 觸發條件 | 說明 |
|--------|---------|------|
| `sub_workflow_failed` | Child workflow instance FAILED | 子 workflow 失敗 |
| `sub_workflow_not_found` | `workflow` 對應的 workflow definition 不存在或非 PUBLISHED | Workflow definition 找不到 |
| `max_depth_exceeded` | sub_workflow 巢狀超過深度限制 | 巢狀深度超過上限 |
| `max_step_executions_exceeded` | 超過 `config.max_step_executions` | 防止無限迴圈 |

#### Saga 相關

| 錯誤碼 | 觸發條件 | 說明 |
|--------|---------|------|
| `compensation_failed` | Saga 補償流程中有 compensate step 失敗 | 見 [12-step-saga](../02-steps/12-step-saga.md) |

#### Trigger 相關

| 錯誤碼 | 觸發條件 | 說明 |
|--------|---------|------|
| `trigger_input_invalid` | Trigger 產生的 input 不符合 input_schema | Scheduled / HTTP / event trigger |
| `trigger_mapping_error` | input_mapping CEL 表達式求值失敗 | Trigger 的映射表達式錯誤 |

#### Artifact 相關

| 錯誤碼 | 觸發條件 | 說明 |
|--------|---------|------|
| `artifact_required` | 必填 input artifact 未提供 | `required: true` 的 artifact 缺失 |
| `artifact_not_found` | Artifact URI 指向不存在的資源 | URI 無效或已過期 |

#### 系統相關

| 錯誤碼 | 觸發條件 | 說明 |
|--------|---------|------|
| `payload_too_large` | Event data 或 step output 超過大小上限 | 見 runtime-spec 設定 |
| `cancelled` | Instance 被外部取消 | API 取消或 parent timeout |
| `unknown` | 未分類的錯誤 | 兜底錯誤碼 |

### 自訂錯誤碼

Task handler 的 response JSON 中 `error.code` 欄位可使用自訂錯誤碼：

```json
{
  "success": false,
  "output": null,
  "error": {
    "message": "餘額不足",
    "code": "payment.insufficient_funds"
  }
}
```

自訂錯誤碼在 `catch` handler 中可透過 `error.code` 存取：

```yaml
catch:
  - type: if
    when: ${ error.code == "payment.insufficient_funds" }
    then:
      - type: emit
        event: payment.balance_low
        data:
          order_id: ${ input.order_id }
    else:
      - type: fail
        message: ${ error.message }
        code: ${ error.code }
```

### `fail` step 的 code 欄位

`fail` step MAY 指定 `code`，用於向上層傳遞錯誤碼：

```yaml
- type: fail
  message: "payment timeout"
  code: "payment.timeout"
```

若未指定 `code`，預設為 `unknown`。

---

## 範例：多層級錯誤處理

```yaml
config:
  timeout: 2h
  catch:
    - type: emit
      event: workflow.failed
      data:
        error: ${ error.message }
  on_timeout:
    - type: emit
      event: workflow.timeout
      data:
        workflow: "order_fulfillment"

steps:
  - id: process_order
    type: task
    action: order.process
    input:
      order_id: ${ input.order_id }
    retry:
      max_attempts: 3
      delay: 2s
      backoff: exponential
    timeout: 30s
    # Step-level: 捕捉此 task 的錯誤
    catch:
      - type: emit
        event: order.process_failed
        data:
          order_id: ${ input.order_id }
          error: ${ error.message }
      # 不使用 fail → 錯誤已處理，繼續後續 steps

  - id: analyze
    type: agent
    agent: order.analyzer
    prompt: "分析訂單 ${ input.order_id } 的風險"
    input:
      order_id: ${ input.order_id }
    # Step-level: 捕捉 agent 錯誤（如 agent_max_iterations、agent_llm_error）
    catch:
      - type: if
        when: ${ error.code == "agent_llm_error" }
        then:
          - type: emit
            event: agent.llm_unavailable
            data:
              order_id: ${ input.order_id }
          # 降級處理，不使用 fail → 繼續後續 steps
        else:
          - type: fail
            message: ${ error.message }
            code: ${ error.code }

  - id: finalize
    type: parallel
    failure_policy: wait_all
    # Block-level: 捕捉 parallel 內任一 branch 的未處理錯誤
    catch:
      - type: emit
        event: order.finalize_partial
        data:
          error: ${ error.message }
    branches:
      - steps:
          - type: task
            action: notify.customer
            input:
              order_id: ${ input.order_id }
      - steps:
          - type: task
            action: report.generate
            input:
              order_id: ${ input.order_id }
```

### 範例：catch 與 saga 補償的交互

```yaml
steps:
  - type: saga
    catch:
      - type: emit
        event: saga.error
        data:
          error: ${ error.message }
      - type: fail
        message: ${ error.message }
    steps:
      - id: debit
        type: task
        action: payment.debit
        input:
          amount: ${ input.amount }
        # Step-level catch：嘗試降級處理
        catch:
          - type: if
            when: ${ error.code == "payment.insufficient_funds" }
            then:
              # 降級成功 → step 不再為 FAILED → 不觸發 saga 補償
              - type: assign
                vars:
                  payment_status: "deferred"
            else:
              # 無法處理 → 重新拋出 → step 仍為 FAILED → 觸發 saga 補償
              - type: fail
                message: ${ error.message }
                code: ${ error.code }
        compensate:
          - type: task
            action: payment.refund
            input:
              amount: ${ input.amount }

      - id: create_order
        type: task
        action: order.create
        input:
          items: ${ input.items }
        compensate:
          - type: task
            action: order.cancel
            input:
              order_id: ${ steps.create_order.output.order_id }
```
