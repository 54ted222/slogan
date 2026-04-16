# 09 — Error Model

本文件定義引擎中所有錯誤的結構、傳播路徑與標準錯誤碼。對應 [dsl-spec/03-steps.md](../dsl-spec/03-steps.md) 的「錯誤處理模型」。

---

## Error 物件結構

統一格式（出現於 `error.*` namespace、step state、execution log）：

```
ErrorObject {
  type:        string,        # 標準錯誤碼，dotted snake_case
  message:     string,        # 人類可讀
  code:        string | null, # 業務 / 外部系統錯誤碼（如 HTTP status）
  step_id:     string | null, # 發生錯誤的 step id（saga 補償等可用）
  fragment:    string | null, # CEL fragment（expression_error 才有）
  cause:       ErrorObject | null, # 巢狀原因（wait async_step_failed 等）
  retryable:   bool,          # 提示性，由 step type 決定預設值
  details:     map | null,    # 額外結構化資訊，如 compensation_failures
}
```

`error.message` 與 `error.code` 可被使用者透過 `type: fail` 自訂。

---

## 標準錯誤碼分類

### 表達式 / 求值

| code | 觸發 |
|------|------|
| `expression_error` | CEL 求值任何異常（聚合） |
| `expression_error.identifier_not_found` | 引用不存在的 namespace 路徑 |
| `expression_error.type_error` | 型別不符 |
| `expression_error.overflow` | 數值溢位 |
| `expression_error.too_deep` | AST 深度超限 |
| `expression_error.too_large` | 結果大小超限 |
| `expression_error.template_eval` | template 字串組裝失敗 |
| `expression_error.mapping` | stdout.mapping 求值失敗 |
| `expression_error.division_by_zero` | 整數 / 浮點除法除以 0；或取模除以 0 |
| `expression_error.out_of_bounds` | 字串 / list 索引 / 切片越界 |

### Step 級

| code | 觸發 |
|------|------|
| `timeout` | step / wait / callback / workflow timeout |
| `step_error` | step handler 拋出未分類異常 |
| `lease_lost_unsafe_resume` | 非 idempotent step 在 RUNNING 中失去 lease |
| `cancelled` | 接收到 cancel 訊號中止 |

### 條件 / 控制流

| code | 觸發 |
|------|------|
| `invalid_count` | foreach.count 為負或非整數 |
| `invalid_items` | foreach.items 求值結果非 list（含 null / string / int / map / bool） |
| `async_step_failed` | wait signals 中 step 訊號所等待的 async step FAILED |
| `branch_failed` | parallel branch 失敗（fail_fast 政策下） |

### Saga

| code | 觸發 |
|------|------|
| `saga_failed` | saga 內 step 失敗、補償完成（含失敗）後向上拋 |
| `compensate_failed` | 個別補償 action 失敗（記入 details.compensation_failures） |

### Tool / Backend

| code | 觸發 |
|------|------|
| `spawn_failed` | exec backend process spawn 失敗（聚合碼） |
| `spawn_failed.not_found` | command 檔案不存在（`ENOENT`） |
| `spawn_failed.permission_denied` | command 無執行權限（`EACCES` / `EPERM`） |
| `spawn_failed.resource_exhausted` | fork 失敗，通常因 PID 上限或 RLIMIT（`EAGAIN`） |
| `spawn_failed.oom` | OS OOM killer 於 spawn 階段觸發 |
| `spawn_failed.working_dir` | `working_dir` 不存在或無權限 |
| `spawn_failed.working_dir_escape` | `working_dir` canonical 化後不在 `<workspace_root>/<instance_id>` 前綴下；`error.details.reason` ∈ `{prefix_mismatch, symlink_resolve_failed, broken_symlink}` |
| `incomplete_protocol` | tool 未發出 result 即結束 |
| `exit_code` | exit code 不在 success 列表 |
| `http_error` | HTTP 4xx/5xx 在 error_on_status 列表、或 retry_on_status 耗盡 attempts |
| `http_body_malformed_json` | Content-Type 為 JSON 但 body 語法錯誤 |
| `http_body_decode_failed` | body 以宣告 charset 解碼失敗 |
| `http_redirect_blocked` | HTTP 3xx 未被 `error_on_status` / `retry_on_status` 明示處理；`error.details.location` 保留 Location header，`error.code` 為原 3xx status |
| `http_request_body_too_large` | request body 序列化後超過 `engine.http_request_body_limit`（預設 16 MB）|
| `connection_error` | 網路 / TLS 連線失敗 |
| `schema_violation` | input/output 不符 JSON Schema |
| `backend_crashed` | tool process / extension 在 RUNNING 中崩潰 |
| `lifecycle_init_failed` | tool lifecycle init backend 失敗 |
| `extension_handler_panic` | extension handler 內部未捕捉例外 / panic / WASM trap |
| `invalid_args` | tool exec backend `args` 陣列某元素 CEL 求值為 null 或非 string |
| `invalid_env` | tool exec backend `env` 某 value CEL 求值為 null 或非 string |
| `stdout_too_large` | tool 原始 stdout bytes 超過 `engine.tool_stdout_raw_limit` |
| `http_body_too_large` | tool http backend 回應 body 超過 `engine.http_body_limit` |

### Registry / 載入

| code | 觸發 |
|------|------|
| `registry.action_not_found` | resolve 失敗 |
| `registry.invalid_action_name` | 命名混用 `-` / `_` 等不合規 |
| `registry.duplicate_action` | 重複註冊 |
| `registry.name_conflict` | 同類型同名衝突 |
| `registry.dependency_cycle` | Function 間循環依賴（載入期偵測） |
| `registry.missing_secret` | workflow.config.secrets 宣告的 secret 不存在於 project scope |
| `workflow_version_deleted` | instance 綁定的 workflow version 被強制刪除 |
| `workflow_version_deleted.hash_mismatch` | replay 時對應 `(canonical, version)` 於 registry 的 `definition_hash` 與 instance pin 時的 hash 不符（表明 version 內容被違規覆寫）；見 `05-task-registry.md` hot reload 規則 |
| `definition_in_use` | 嘗試刪除仍有 active instance 引用的 definition version |
| `max_recursion_depth_exceeded` | 運行時 function 遞迴深度超限 |
| `max_step_executions_exceeded` | instance 執行的 step 次數超過 `config.max_step_executions`（見 `02-instance-lifecycle.md`）；走 config.catch |
| `registry.unsupported_schema_keyword` | input_schema / output_schema 使用 v6 不支援的 JSON Schema keyword（如 `$ref` / `allOf` / `oneOf`）；載入期拒絕，`details.keyword` 為觸發的 keyword |
| `schema_incomplete` | Function callback 只宣告 input_schema 或 output_schema 之一 |
| `invalid_var_name` | `type: assign` 的 vars key 含 path separator 或保留前綴 |
| `invalid_duration_format` | duration 字面值或 CEL 結果不符 `<N><s/m/h>` 格式 |
| `invalid_duration` | duration 求值為 0 或負值 |
| `invalid_wait_config` | wait step 同時宣告 `duration` 與 `timeout` |
| `registry.invalid_catch_step` | workflow.config.catch 含白名單外的 step type |
| `registry.version_not_specified` | 全域 default_version_policy=require_explicit 下，action 未帶 @version |
| `registry.secret_access_denied` | 跨 project 引用未標記 `metadata.shared: true` 的 secret |
| `registry.invalid_trigger_scope` | event trigger 宣告 `scope: workflow`（不合法） |
| `invalid_retry_config` | `retry.max_attempts` 字面值或 CEL 結果 < 1 或非整數 |
| `registry.invalid_lifecycle_backend` | tool `lifecycle.init.backend.type` 或 `lifecycle.destroy.backend.type` 為 `extension` 或非 `exec` / `http`；`error.details.hook` ∈ `{init, destroy}` |
| `registry.invalid_action_version` | `builtin.*` action 帶 `@version` 宣告（builtin 不支援版本） |
| `registry.invalid_version` | `metadata.version` 非正整數 |
| `registry.invalid_http_method` | tool http backend `method` 不在支援清單 |
| `registry.invalid_label_value` | `metadata.labels.<key>` value 非 string 或含換行 / 超長 |
| `registry.invalid_metadata` | `metadata.description` 超過 1024 chars 或其他 metadata 欄位違反基本限制 |
| `registry.invalid_event_name` | `emit.event` / `wait.signals.event` / `trigger.event` 名稱格式錯或使用保留前綴 |
| `registry.invalid_workflow_definition` | Workflow `steps[]` 為空或其他結構性錯誤（包含巢狀 step 結構違規如 `parallel.branches: []` / `saga.steps: []` / `callback.<name>: []` 等，發生於 Workflow 定義內時） |
| `registry.invalid_function_definition` | Function `steps[]` 為空或其他結構性錯誤（同上，發生於 Function 定義內時） |

> 結構性錯誤的 `error.type` 依**擁有該結構的頂層 kind** 決定：DSL spec 中如 [03-steps.md](../dsl-spec/03-steps.md) / [05b-function.md](../dsl-spec/05b-function.md) 所列的 `registry.invalid_workflow_definition` 為預設文案；實際發生於 Function definition 載入時，引擎 MUST 以 `registry.invalid_function_definition` 回報。`details.reason` 欄位（如 `empty_parallel_branches` / `empty_saga_steps` / `empty_callback_handler`）於兩者皆一致，便於監控規則通用化。
| `registry.extension_handler_not_found` | tool `backend.type: extension` 的 handler 未註冊於 engine extension registry |
| `invalid_fail_config` | `type: fail` 的 `message` 為空或 `code` 格式違反規則 |
| `registry.duplicate_manual_trigger` | 同一 workflow 的 `triggers[]` 宣告多個 `type: manual` |
| `registry.invalid_signal_target` | `wait.signals[].step` 指向不存在、非 async 或不可達作用域的 step（載入期）；或 hot reload 後目標 step 消失時，既有 subscription 以 `error.type == "invalid_signal_target"` 令 wait step FAILED（運行期） |
| `registry.missing_callback_handler` | caller（workflow / function）的 `type: task` step 的 `callback:` map 缺漏 function 宣告的 callback handler；或含有 function 未宣告的 callback 名（`details.reason: "unknown_callback_name"`）；caller 載入期拒絕 |
| `registry.invalid_env_key` | tool `backend.env` 使用 `SLOGAN_*` 保留前綴 key（`details.reason: "reserved_prefix"`）；或 key 不符合 env var 命名規則（`^[A-Za-z_][A-Za-z0-9_]*$`） |
| `registry.version_content_mismatch` | hot reload 時同 `(canonical, version)` 已存在但 `definition_hash` 不同；既有 version 不可覆寫，須分配新 version 號 |
| `action_pin_version_not_found` | instance 的 `action_pins` 指向的版本已從 registry 刪除，step 嘗試解析該 action 時失敗；`error.details` 含 `pinned_version` 與 `available_versions` |
### Workflow / Trigger

| code | 觸發 |
|------|------|
| `workflow.input_schema_violation` | trigger 階段輸入驗證失敗 |
| `trigger.when_eval_failed` | event trigger 的 `when` CEL 求值異常（視為過濾未通過） |
| `trigger.input_mapping_error` | event trigger 的 `input_mapping` 中某欄位 CEL 求值異常 |
| `workflow.instance_create_failed` | trigger 通過驗證但 instance 建立（寫入 store）失敗（size limit / persistence error 等） |
| `output_schema_violation` | return 輸出驗證失敗 |
| `input_too_large` | step input snapshot 或 instance.input 超過 size limit |
| `output_too_large` | step / instance output 超過 size limit |
| `vars_too_large` | instance.vars 整體序列化超過 size limit（assign 寫入前檢查） |
| `event_too_large` | emit event.data 超過 size limit |

### 系統 / 內部

| code | 觸發 |
|------|------|
| `internal_error` | 引擎 bug；不應正常出現 |
| `persistence_error` | Instance Store 寫入失敗 |
| `bus.dead_letter` | 事件投遞反覆失敗 |

---

## 傳播路徑

```
Step Handler 拋錯
  │
  ▼
Step.retry 觸發（若 attempts < max_attempts）─► 重試
  │ (用盡)
  ▼
Step.catch 處理
  │ (有 catch[] 且不重新拋)
  ▼ SUCCEEDED（fall through 到下一 step）
  │ (catch 中 type: fail 或 catch 自身 FAILED)
  ▼
所在控制結構（if / switch / foreach / parallel）的失敗策略
  │ 例：foreach failure_policy=continue → 計入 failed_indices；fail_fast → 中止
  ▼
父 step（saga / function 呼叫端 task）
  │ saga → 進入補償
  │ 一般 → 視為該 step FAILED 向上
  ▼
Workflow.config.catch
  │
  ▼
Instance FAILED → 父 instance（若有）以一個 step FAILED 呈現
```

---

## Catch 行為精確化

```
def handle_step_failure(step, error):
    if step.retry and step.attempt < step.retry.max_attempts:
        sleep(compute_backoff(step))
        retry_step(step)
        return

    if step.catch:
        ctx = build_namespaces(...) | { error: error }
        catch_outcome = execute_steps(step.catch, ctx)
        if catch_outcome.status == "SUCCEEDED":
            step.status = "SUCCEEDED"
            step.output = catch_outcome.output         # catch 最後一個 step 的 output
            return
        else:
            propagate(catch_outcome.error)             # catch 自身錯誤向上
            return

    propagate(error)
```

`catch` 內的 step 本身 **不再進入 catch**（避免無限）；catch 內的 step FAILED 直接向 catch 區塊外拋。

---

## Fail Step

```
- type: fail
  message: "..."
  code: "..."
```

執行語意：

```
1. 求值 message / code（支援 ${ }）
2. 構造 ErrorObject {
     type: "user_fail",
     message,
     code,
     step_id: <current_step_id>,
     cause: <parent_error> | null       # 見下方「catch 中 rethrow」
   }
3. 依當前位置決定傳播：
   - 一般 step 序列中：直接 raise（向所在控制結構傳播）；cause = null
   - catch handler 中：rethrow，跳出 catch；cause = catch 觸發的原 error（即 `error.*` namespace 內容）
```

### catch 中 rethrow 的 error 結構

當 `type: fail` 出現在 catch handler 內：

- 新 ErrorObject 的 `type` 仍為 `"user_fail"`
- `cause` 欄位指向觸發 catch 的**原 error**（完整保留 `type` / `code` / `cause` 鏈）
- 下游 catch / observability 可透過 `error.cause.type` 取得原因類型（如 `timeout` / `schema_violation` 等）

範例：

```yaml
catch:
  - type: fail
    message: ${ "wrapped: " + error.message }
    code: "wrapped_failure"
# 傳播後：
#   error.type == "user_fail"
#   error.code == "wrapped_failure"
#   error.cause.type == "timeout"   # 原錯誤類型
```

`type: "user_fail"` 是頂層 `error.type`；使用者可在 catch 中分流（例如 `when: ${ error.type == "user_fail" && error.code == "order_cancelled" }`）。

---

## Saga 補償錯誤聚合

```
ErrorObject for saga_failed = {
  type: "saga_failed",
  message: "saga aborted: <triggering step>",
  step_id: <triggering step id>,
  cause: <original ErrorObject>,
  details: {
    compensation_failures: [
      { step_id, error: ErrorObject },
      ...
    ]
  }
}
```

`error.cause` 是觸發補償的原 step 錯誤；`details.compensation_failures` 列出補償過程中的失敗清單。

---

## Workflow / Function 級 catch

`config.catch` 在 workflow **或** function instance FAILED 即將定案前最後求值；可：

- `type: emit` 發失敗事件
- `type: fail` 重寫錯誤訊息（仍 FAILED）
- `type: return` 將 FAILED 轉為 SUCCEEDED 並寫入 output（建議謹慎）

`config.catch` 可用 namespace（皆已凍結 / read-only）：`error.*` / `input` / `vars` / `steps` / `secret` / `env` / `project` / `artifacts` / `context`（受限子集：僅 `instance_id` / `trace_id`，因無「當前 step」語意）。**不可用** `prev`（instance 層級無前一 step 概念）、`loop`（不在迭代內）、`event` / `callback`（非 trigger / callback 脈絡）。**允許的 step types**（白名單，載入驗證器 MUST 拒絕白名單外；workflow 與 function 共用同一表）：

| type | 允許 | 說明 |
|------|------|------|
| `emit` | ✅ | 通知、發失敗事件 |
| `fail` | ✅ | 重寫 / 包裝錯誤訊息後保持 FAILED |
| `return` | ✅ | 將 FAILED 轉為 SUCCEEDED（謹慎）；output 求值後 MUST 通過 instance 的 `output_schema`（workflow / function 同規則，見下） |
| `assign` | ✅ | 更新 vars（僅觀測用；instance 即將終結） |
| `if` / `switch` | ✅ | 分支控制（body 內仍受此白名單約束） |
| `task` | ❌ | 禁止；catch 不得發起新的 tool / function 呼叫 |
| `wait` | ❌ | 禁止；instance 即將終結，不得阻塞 |
| `foreach` / `parallel` | ❌ | 禁止；同上 |
| `saga` | ❌ | 禁止；config.catch 不展開補償 |
| `callback` | ❌ | config.catch 內不可觸發 callback（function 即將終結，caller 的 handler 無法安全路由） |

若 YAML 中 config.catch 含任一禁止 type → 載入失敗，`registry.invalid_catch_step`（新增錯誤碼）。Step 級 `catch`（task / wait 等）仍保留**全部** step types 可用性，不受此限制。

**`config.catch` 內 `type: return` 的終態規則**（workflow / function 共用）：

1. `return.output` CEL 求值；求值失敗 → instance 仍 FAILED，`error.type == "expression_error"`；先前的原 error 保留於 `error.cause`
2. 對 instance 的 `output_schema`（若有）驗證：
   - 驗證通過 → instance SUCCEEDED，output 為 return 求值結果；原 error 被吞沒（不進 execution_log 的終態 error；但 catch 執行過程仍於 log 保留）
   - 驗證失敗 → instance **仍 FAILED**，`error.type == "output_schema_violation"`（此為新錯誤；原 catch-觸發 error 保留於 `error.cause`，形成 `cause` 鏈）；候選 output 不寫入 `instances.output`
3. 若 instance 無 `output_schema` 宣告 → 跳過步驟 2；return 的 output 直接成為 instance output（SUCCEEDED）
4. 對 function instance：SUCCEEDED 時父 task step 收到成功 output；FAILED 時父 step 以 FAILED 呈現，可被父層 catch 捕捉
5. 對 workflow instance：SUCCEEDED 時走完整 instance 終結流程（lifecycle destroy / retention）；FAILED 時同

---

## 觀測

execution_log 中每則 ErrorObject MUST 完整保留（含 cause）；redaction 規則對 message 套用（過濾 secret 明文）。

Engine 提供 metric：`error_total{type, kind}` 區分 user_fail vs system error。
