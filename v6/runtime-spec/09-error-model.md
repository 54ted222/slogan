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
| `invalid_items` | foreach.items 求值結果為 null |
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
| `incomplete_protocol` | tool 未發出 result 即結束 |
| `exit_code` | exit code 不在 success 列表 |
| `http_error` | HTTP 4xx/5xx 在 error_on_status 列表、或 retry_on_status 耗盡 attempts |
| `http_body_malformed_json` | Content-Type 為 JSON 但 body 語法錯誤 |
| `http_body_decode_failed` | body 以宣告 charset 解碼失敗 |
| `connection_error` | 網路 / TLS 連線失敗 |
| `schema_violation` | input/output 不符 JSON Schema |
| `backend_crashed` | tool process / extension 在 RUNNING 中崩潰 |
| `lifecycle_init_failed` | tool lifecycle init backend 失敗 |
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
| `definition_in_use` | 嘗試刪除仍有 active instance 引用的 definition version |
| `max_recursion_depth_exceeded` | 運行時 function 遞迴深度超限 |
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
| `registry.invalid_lifecycle_backend` | tool `lifecycle.init.backend.type` 為 `extension` 或非 `exec` / `http` |
| `registry.invalid_action_version` | `builtin.*` action 帶 `@version` 宣告（builtin 不支援版本） |
| `registry.invalid_version` | `metadata.version` 非正整數 |
### Workflow / Trigger

| code | 觸發 |
|------|------|
| `workflow.input_schema_violation` | trigger 階段輸入驗證失敗 |
| `trigger.when_eval_failed` | event trigger 的 `when` CEL 求值異常（視為過濾未通過） |
| `trigger.input_mapping_error` | event trigger 的 `input_mapping` 中某欄位 CEL 求值異常 |
| `output_schema_violation` | return 輸出驗證失敗 |
| `input_too_large` | step input snapshot 超過 size limit |
| `output_too_large` | step / instance output 超過 size limit |
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

## Workflow 級 catch

`config.catch` 在 instance FAILED 即將定案前最後求值；可：

- `type: emit` 發失敗事件
- `type: fail` 重寫錯誤訊息（仍 FAILED）
- `type: return` 將 FAILED 轉為 SUCCEEDED 並寫入 output（建議謹慎）

`config.catch` 僅可 namespace `error.*` / `input` / `vars` / `steps` / `secret`（已凍結）。**允許的 step types**（白名單，載入驗證器 MUST 拒絕白名單外）：

| type | 允許 | 說明 |
|------|------|------|
| `emit` | ✅ | 通知、發失敗事件 |
| `fail` | ✅ | 重寫 / 包裝錯誤訊息後保持 FAILED |
| `return` | ✅ | 將 FAILED 轉為 SUCCEEDED（謹慎） |
| `assign` | ✅ | 更新 vars（僅觀測用；instance 即將終結） |
| `if` / `switch` | ✅ | 分支控制（body 內仍受此白名單約束） |
| `task` | ❌ | 禁止；catch 不得發起新的 tool / function 呼叫 |
| `wait` | ❌ | 禁止；instance 即將終結，不得阻塞 |
| `foreach` / `parallel` | ❌ | 禁止；同上 |
| `saga` | ❌ | 禁止；config.catch 不展開補償 |
| `callback` | ❌ | 僅 function 內部可用，不適用 workflow catch |

若 YAML 中 config.catch 含任一禁止 type → 載入失敗，`registry.invalid_catch_step`（新增錯誤碼）。Step 級 `catch`（task / wait 等）仍保留**全部** step types 可用性，不受此限制。

---

## 觀測

execution_log 中每則 ErrorObject MUST 完整保留（含 cause）；redaction 規則對 message 套用（過濾 secret 明文）。

Engine 提供 metric：`error_total{type, kind}` 區分 user_fail vs system error。
