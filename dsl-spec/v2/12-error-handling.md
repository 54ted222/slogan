# 12 — Error Handling

本文件定義統一的錯誤處理與 timeout 處理模型。

---

## 錯誤處理層級

錯誤處理分為三個層級，依優先順序由高至低：

| 層級 | 位置 | 捕捉範圍 |
|------|------|----------|
| **Step-level** | step 上的 `on_error` | 該 step 本身的錯誤 |
| **Block-level** | 控制流程 step 上的 `on_error`（if / switch / foreach / parallel） | 該 block 內未被 step-level 捕捉的錯誤 |
| **Workflow-level** | `config.on_error` | 所有未被上層捕捉的錯誤 |

### 優先順序

Step 失敗時，引擎按以下順序尋找 handler：

1. 該 step 自身的 `on_error` → 若有，執行
2. 包含該 step 的最近控制流程 step 的 `on_error` → 若有，執行
3. 外層控制流程 step 的 `on_error`（向上遞迴）
4. `config.on_error` → 若有，執行
5. 都沒有 → workflow instance FAILED

第一個匹配的 handler 處理錯誤，不再向上傳播（除非 handler 內使用 `fail` 重新拋出）。

---

## on_error 結構

`on_error` 的值為 step 陣列（不需要 `steps` wrapper）：

```yaml
on_error:
  - type: emit
    event: workflow.step_failed
    data:
      step: ${ error.step_id }
      message: ${ error.message }
```

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
| 正常完成（無 `fail`） | 錯誤視為已處理，workflow 從失敗 step 的下一個 step 繼續 |
| 使用 `fail` step | 錯誤重新拋出，繼續向上層尋找 handler |
| Handler 自身失敗 | 視為未處理錯誤，繼續向上層尋找 handler |

### 與 foreach / parallel failure_policy 的交互

- 若 `foreach` / `parallel` 內某個 step 失敗，且該 step 的 `on_error` 正常完成（未使用 `fail`）→ 該 step 視為已處理，迭代/分支繼續執行，不觸發 `failure_policy`
- 若 step 的 `on_error` 不存在或重新拋出錯誤 → 該迭代/分支視為失敗，`failure_policy` 生效

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

與 `on_error` 相同。

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
有 on_error？→ 是 → 執行 handler
  ↓                    ↓
  否             handler 結果
  ↓                    ↓
  └────────────────────┘
  ↓
向上層尋找 handler（block → workflow）
  ↓
都沒有 → workflow FAILED
```

重點：retry 會在 `on_error` 之前全部用盡。

---

## Timeout 級聯

### Step timeout

- 適用於 `task`、`sub_workflow`、`wait_event`
- 超時 → step TIMED_OUT → 觸發該 step 的 `on_timeout`
- `on_timeout` 不存在 → 視為 step FAILED，進入錯誤處理流程

### Workflow timeout

- 定義在 `config.timeout`
- 是 instance 從 CREATED 到完成的絕對時間上限
- 觸發時：
  1. 所有 RUNNING 與 WAITING 的 steps 被取消（CANCELLED）
  2. 不觸發任何 step-level handler（`on_error`、`on_timeout`）
  3. 觸發 `config.on_timeout`（若有）
  4. 無 `config.on_timeout` → instance FAILED

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

| 錯誤碼 | 說明 |
|--------|------|
| `task_failed` | task handler 回報失敗 |
| `timeout` | 執行超時 |
| `schema_validation_error` | input / output schema 驗證失敗 |
| `expression_error` | CEL 表達式求值失敗 |
| `sub_workflow_failed` | 子 workflow 失敗 |
| `max_depth_exceeded` | sub_workflow 巢狀超過深度限制 |
| `max_step_executions_exceeded` | 超過 `config.max_step_executions` 上限 |
| `unknown` | 未分類的錯誤 |

---

## 範例：多層級錯誤處理

```yaml
config:
  timeout: 2h
  on_error:
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
    on_error:
      - type: emit
        event: order.process_failed
        data:
          order_id: ${ input.order_id }
          error: ${ error.message }
      # 不使用 fail → 錯誤已處理，繼續後續 steps

  - id: finalize
    type: parallel
    failure_policy: wait_all
    # Block-level: 捕捉 parallel 內任一 branch 的未處理錯誤
    on_error:
      - type: emit
        event: order.finalize_partial
        data:
          error: ${ error.message }
    branches:
      - - type: task
          action: notify.customer
          input:
            order_id: ${ input.order_id }
      - - type: task
          action: report.generate
          input:
            order_id: ${ input.order_id }
```
