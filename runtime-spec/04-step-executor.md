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

1. 建立 execution context（與該 step 的一般表達式求值相同的 namespace：`input`、`steps`、`vars`、`loop`、`env`、`secret`、`artifacts`）
2. 以 Expression Evaluator 求值（見 [06-expression-evaluator](06-expression-evaluator.md)）
3. `false` → step 狀態 READY → SKIPPED，結束
4. `true` → 繼續
5. 求值失敗 → step 狀態 → FAILED（error code: `expression_error`）

### 2. 狀態轉換

Step 狀態 READY → RUNNING。

### 3. Max Step Executions 檢查

每次 step 進入 RUNNING，引擎 MUST 遞增 instance 的 step execution counter。若超過 `config.max_step_executions` → instance FAILED（error code: `max_step_executions_exceeded`）。

---

## 各 Step Type 的執行邏輯

### task

1. 解析 `action` → 找到對應的 task definition（見下方版本解析規則）
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

#### 版本解析

`action` 格式為 `"<task_name>"` 或 `"<task_name>@<version>"`。版本解析規則：

| 值 | 行為 |
|----|------|
| 未指定版本（預設） | 查找最新 PUBLISHED 版本的 task definition |
| 指定整數版本 | 查找該版本的 task definition（MUST 為 PUBLISHED 或 DEPRECATED） |

找不到匹配的 task definition → step FAILED（error code: `definition_not_found`）。

#### Artifact Resources 綁定

若 step 有 `resources` 欄位：

1. 解析每個 resource 的 `ref` → 對應 workflow 的 `artifacts` 區塊中的 key
2. 驗證 artifact record 存在
3. 將 artifact 中繼資料（path、content_type 等）注入 task 的 execution context
4. Task Executor 可透過 `artifacts` namespace 存取綁定的 artifact

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
3. 根據 `concurrency` 排程迭代：
   - `concurrency = 1`（預設）：依序執行，前一迭代完成後才啟動下一個
   - `concurrency > 1`：以 sliding window 方式，同時執行最多 N 個迭代；當一個迭代完成後，啟動下一個待處理迭代
4. 每個迭代：
   a. 設定 `loop.item` = items[i]，`loop.index` = i
   b. 為 `do` 中的 steps 建立 step instances（`parent_step_id` = foreach step，`iteration_index` = i）
   c. 執行 do 中的 steps
5. 迭代完成 → 按 `failure_policy` 判斷最終狀態：
   - `fail_fast`：任一迭代失敗 → 取消所有進行中迭代，foreach → FAILED
   - `continue`：等待所有迭代完成，若有任一失敗 → foreach → FAILED
   - `ignore`：忽略失敗，foreach → SUCCEEDED
6. 收集 output array：
   - `continue` / `ignore` 模式：array 長度 = items 長度，索引一一對應。成功迭代為其最後一個 step 的 output，失敗迭代為 `null`
   - `fail_fast` 模式：array 為已完成迭代的 compact 結果（不含被取消的迭代），長度可能小於 items 長度。output[i] 對應 items[i]（按原始索引順序，跳過未完成的）

### parallel

1. 為每個 branch 建立 step instances（`parent_step_id` = parallel step，`branch_index` = 分支索引）
2. 同時啟動所有 branches（各 branch 第一個 step → READY）
3. 按 `failure_policy` 管理分支完成：
   - `fail_fast`：任一 branch 失敗 → 取消所有其他 branches 的 RUNNING/WAITING/READY/PENDING steps，parallel → FAILED
   - `wait_all`：等待所有 branches 完成，若有任一失敗 → parallel → FAILED
4. 收集 output array（索引對應 branches 順序）：
   - 成功的 branch：該 branch 最後一個 step 的 output
   - 失敗的 branch（`wait_all` 且 on_error 已處理）：`null`

### emit

1. 求值 `data` 中的 CEL 表達式
2. 組合事件信封（type、data、自動產生 id、timestamp）
3. 持久化事件（at-least-once 保證）
4. 將事件交由 Event Router
5. Step → SUCCEEDED

Emit 不等待事件被消費。

### wait_event

1. 建立 wait_subscription record（event_type、match_expression、expires_at）
2. 若有 `timeout` → 建立 timeout_schedule
3. Step 狀態 RUNNING → WAITING
4. Instance 狀態 → WAITING
5. 後續由 Event Router 或 Timeout Manager 觸發恢復：
   - Event Router 匹配成功 → step WAITING → RUNNING → SUCCEEDED，output = 匹配事件的 `event.data`
   - Timeout Manager 觸發 → step WAITING → TIMED_OUT，進入 on_timeout 處理

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

1. 解析 `workflow` → 找到對應的 workflow definition（版本解析規則同 task step，預設 "latest" → 最新 PUBLISHED 版本）
2. 檢查巢狀深度：沿 `parent_instance_id` 鏈計算深度，超過上限（建議 10 層）→ step FAILED（`max_depth_exceeded`）
3. 求值 `input` 中的 CEL 表達式
4. 建立 child workflow instance：
   - `parent_instance_id` = 當前 instance
   - `parent_step_id` = 當前 step
5. 通知 Scheduler 排程 child instance
6. Parent step 保持 RUNNING，等待 child 完成
7. Child SUCCEEDED → parent step SUCCEEDED，output = child return output
8. Child FAILED → parent step FAILED（error code: `sub_workflow_failed`，可被 on_error 捕捉）

---

## Error Handling 整合

Step Executor 在 step 失敗時，按照 [dsl-spec/v2/12-error-handling](../dsl-spec/v2/12-error-handling.md) 定義的三層錯誤處理模型處理：

1. 檢查 step-level `on_error`
2. 檢查 block-level `on_error`（向上遞迴）
3. 檢查 workflow-level `config.on_error`
4. 都沒有 → instance FAILED

### Handler Step Instance 建立與執行

找到匹配的 handler 後：

1. 為 handler 的 step 陣列動態建立 step instances（標記為 handler steps，不影響主序列）
2. 在 execution context 中注入 `error` 或 `timeout` namespace（見下方 namespace 建構規則）
3. 依序執行 handler steps（遵循相同的 Step Executor 流程）
4. 根據 handler 結果決定後續行為：

| Handler 結果 | 後續行為 |
|-------------|---------|
| 正常完成（無 `fail`） | 錯誤視為已處理，按繼續點推進 |
| 使用 `fail` step | 錯誤重新拋出，繼續向上層尋找 handler |
| Handler 自身失敗 | 視為未處理錯誤，繼續向上層尋找 handler |

### 繼續點（handler 正常完成後）

| Handler 層級 | 繼續點 |
|-------------|--------|
| Step-level | 失敗 step 的同一序列中，下一個 step |
| Block-level | 控制流程 step 的同一序列中，下一個 step |
| Workflow-level | 失敗 step 在頂層 `steps` 中對應的下一個 step |

### error namespace 建構

on_error handler 中的 `error` namespace MUST 包含以下欄位：

| 欄位 | 型別 | 說明 | 來源 |
|------|------|------|------|
| `error.message` | string | 錯誤訊息 | step_instance.error.message，MUST 非 null |
| `error.code` | string | 錯誤碼 | step_instance.error.code，見 [dsl-spec/v2/12-error-handling](../dsl-spec/v2/12-error-handling.md) 標準錯誤碼 |
| `error.step_id` | string | 失敗 step ID | step_instance.step_id |

`error.code` MUST 為以下標準錯誤碼之一：`task_failed`、`timeout`、`schema_validation_error`、`expression_error`、`sub_workflow_failed`、`max_depth_exceeded`、`max_step_executions_exceeded`、`unknown`。若 task backend 回報自訂 error code，以 `task_failed` 為 `error.code`，自訂 code 記錄在 `error.message` 中。

### timeout namespace 建構

on_timeout handler 中的 `timeout` namespace MUST 包含：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `timeout.step_id` | string \| null | 超時的 step ID（workflow timeout 時為 `null`） |
| `timeout.duration` | duration | 設定的 timeout 值 |

### on_timeout handler 特殊規則

- Step-level `on_timeout`：與 `on_error` 相同的繼續點規則
- Workflow-level `config.on_timeout`：handler 完成後 instance 仍為 FAILED（handler 用途為清理與通知）
- 若 `on_timeout` handler 自身失敗或超時 → 視為未處理錯誤，向上層尋找 handler

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

---

## Schema 驗證語意

引擎在多個時機點執行 JSON Schema 驗證。以下定義驗證的範圍與行為。

### 支援的 JSON Schema 約束

| 約束 | 說明 |
|------|------|
| `type` | 型別檢查（string、integer、number、boolean、object、array） |
| `required` | 必填欄位 |
| `enum` | 列舉合法值 |
| `default` | 預設值（未提供時自動填入） |
| `minimum` / `maximum` | 數值範圍 |
| `minLength` / `maxLength` | 字串長度範圍 |
| `minItems` / `maxItems` | 陣列長度範圍 |
| `properties` | 物件屬性定義 |
| `items` | 陣列元素定義 |

### 各驗證點的行為

| 驗證點 | 觸發時機 | 驗證範圍 | 失敗結果 |
|--------|---------|---------|---------|
| Workflow input | CreateInstance / event trigger | 所有約束 + default 填入 | Instance 不建立（`schema_validation_error`） |
| Task input | task step 執行前 | 所有約束 | Step FAILED（`schema_validation_error`） |
| Task output | task step 執行後 | 所有約束 | Step FAILED（`schema_validation_error`） |
| Workflow output（explicit return） | return step 執行時 | 所有約束 | Instance FAILED（`schema_validation_error`） |
| Workflow output（implicit completion） | 最後一個 step 完成 | 僅 `required` 欄位 | Instance FAILED（`schema_validation_error`） |

隱式完成僅檢查 `required` 是因為 output 為 `null`，無法對其執行值約束。若有 required 欄位則代表 workflow 設計上需要 explicit return。
