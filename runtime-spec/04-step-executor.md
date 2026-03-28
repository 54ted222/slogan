# 04 — Step Executor

本文件定義 Step Executor 的行為：根據 step type 分派執行邏輯，管理 step 的完整生命週期。

---

## 職責

Step Executor 接收一個 READY 狀態的 step instance，執行對應的邏輯，回傳結果。

```
Scheduler 分派 step (READY)
     ↓
Step Executor:
  1. 求值 condition（若有）
  2. READY → RUNNING（或 SKIPPED）
  3. 求值 input 中的 CEL 表達式
  4. 依 step type 分派處理
  5. 回傳結果給 Scheduler
```

---

## 通用流程

所有 step types 共享以下前置流程：

### 1. Condition 求值

若 step 有 `condition` 欄位：

1. 以 Expression Evaluator 求值（見 [06-expression-evaluator](06-expression-evaluator.md)）
2. `false` → step 狀態 READY → SKIPPED，結束
3. `true` → 繼續
4. 求值失敗 → step 狀態 → FAILED（error code: `expression_error`）

### 2. 狀態轉換

Step 狀態 READY → RUNNING。

### 3. Max Step Executions 檢查

每次 step 進入 RUNNING，引擎 MUST 遞增 instance 的 step execution counter。若超過 `config.max_step_executions` → instance FAILED（error code: `max_step_executions_exceeded`）。

---

## 各 Step Type 的執行邏輯

### task

1. 解析 `action` → 找到對應的 task definition（見版本解析規則）
2. 求值 `input` 中的 CEL 表達式
3. 若 task definition 有 `input.schema` → 驗證 input
4. 交由 Task Executor 執行（見 [05-task-executor](05-task-executor.md)）
5. 若 task definition 有 `output.schema` → 驗證 output
6. SUCCEEDED：記錄 output
7. FAILED：進入 retry 判斷

#### Retry 流程

```
Task 失敗
  ↓
attempt < max_attempts？
  ├── 是 → attempt += 1，等待 delay，重跑 task
  └── 否 → step FAILED，進入 error handling
```

- Retry 間隔：`fixed` = 固定 delay；`exponential` = delay × 2^(attempt - 1)
- Retry 僅適用於 FAILED（非 TIMED_OUT）
- 每次 retry 遞增 step_instance 的 `attempt` 欄位

### assign

1. 求值 `vars` 中每個 value 的 CEL 表達式
2. 將結果寫入 instance 的 vars namespace
3. 持久化 vars 至 storage
4. Step → SUCCEEDED

Assign 不產生 output（`steps.<id>.output` 不適用）。

### if

1. 求值 `expr`
2. `true` → 選擇 `then` 分支
3. `false` → 選擇 `else` 分支（若有），無 else → step SKIPPED
4. 為選中分支的 steps 建立 step instances
5. 未選中分支的 steps 建立 step instances（state = SKIPPED）
6. 執行選中分支
7. 分支完成 → 父 step 的 output = 最後一個 step 的 output

### switch

1. 求值 `expr`
2. 依序比對 `cases[].value`，找到第一個相等的 case
3. 匹配 → 執行該 case 的 `then`
4. 無匹配 → 執行 `default`（若有），無 default → step SKIPPED
5. 分支完成 → 父 step 的 output = 最後一個 step 的 output

### foreach

1. 求值 `items` → list
2. 空 list → step SUCCEEDED，output = `[]`
3. 根據 `concurrency` 分批啟動迭代
4. 每個迭代：
   a. 設定 `loop.item` = items[i]，`loop.index` = i
   b. 為 `do` 中的 steps 建立 step instances
   c. 執行 do 中的 steps
5. 迭代完成 → 按 `failure_policy` 判斷最終狀態
6. 收集 output array

### parallel

1. 為每個 branch 建立 step instances
2. 同時啟動所有 branches
3. 等待所有 branches 完成（或依 `failure_policy` 提前終止）
4. 按 `failure_policy` 判斷最終狀態
5. 收集 output array（索引對應 branches 順序）

### emit

1. 求值 `data` 中的 CEL 表達式
2. 組合事件信封（type、data、自動產生 id、timestamp）
3. 持久化事件（at-least-once 保證）
4. 將事件交由 Event Router
5. Step → SUCCEEDED

Emit 不等待事件被消費。

### wait_event

1. 建立 wait_subscription record
2. 若有 `timeout` → 建立 timeout_schedule
3. Step 狀態 → WAITING
4. Instance 狀態 → WAITING
5. 後續由 Event Router 或 Timeout Manager 觸發恢復

### fail

1. 求值 `message`（若為 CEL 表達式）
2. 判斷 context：
   - 在 on_error / on_timeout handler 中 → 重新拋出錯誤至上層
   - 在一般 step 序列中 → instance → FAILED
3. 記錄 error（message、code）

### return

1. 求值 `output` 中的 CEL 表達式
2. 若有 `output.schema` → 驗證 output
   - 驗證失敗 → instance → FAILED（`schema_validation_error`）
3. Instance → SUCCEEDED，記錄 output

### sub_workflow

1. 解析 `workflow` → 找到對應的 workflow definition
2. 求值 `input` 中的 CEL 表達式
3. 建立 child workflow instance：
   - `parent_instance_id` = 當前 instance
   - `parent_step_id` = 當前 step
4. 通知 Scheduler 排程 child instance
5. Parent step 保持 RUNNING，等待 child 完成
6. Child SUCCEEDED → parent step SUCCEEDED，output = child return output
7. Child FAILED → parent step FAILED（可被 on_error 捕捉）

---

## Error Handling 整合

Step Executor 在 step 失敗時，按照 [dsl-spec/v2/12-error-handling](../dsl-spec/v2/12-error-handling.md) 定義的三層錯誤處理模型處理：

1. 檢查 step-level `on_error`
2. 檢查 block-level `on_error`（向上遞迴）
3. 檢查 workflow-level `config.on_error`
4. 都沒有 → instance FAILED

Handler 的 step instances 動態建立並執行，遵循相同的 Step Executor 流程。

---

## Execution Context

Step Executor 為每個 step 建立 execution context，供 Expression Evaluator 使用：

| Namespace | 來源 |
|-----------|------|
| `input` | instance input |
| `steps` | 已完成 steps 的 output map |
| `vars` | assign 設定的變數 |
| `loop` | foreach 迭代中的 item/index |
| `env` | 環境變數 |
| `secret` | 加密機密值 |
| `artifacts` | artifact 中繼資料 |
| `error` | 錯誤資訊（僅 on_error handler 內） |
| `timeout` | timeout 資訊（僅 on_timeout handler 內） |

Context 在 step 進入 RUNNING 時建立，包含該時間點的 snapshot。
