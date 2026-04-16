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
| `spawn_failed` | exec backend process spawn 失敗 |
| `incomplete_protocol` | tool 未發出 result 即結束 |
| `exit_code` | exit code 不在 success 列表 |
| `http_error` | HTTP 4xx/5xx 在 error_on_status 列表 |
| `connection_error` | 網路 / TLS 連線失敗 |
| `schema_violation` | input/output 不符 JSON Schema |
| `backend_crashed` | tool process / extension 在 RUNNING 中崩潰 |
| `lifecycle_init_failed` | tool lifecycle init backend 失敗 |

### Registry / 載入

| code | 觸發 |
|------|------|
| `registry.action_not_found` | resolve 失敗 |
| `registry.invalid_action_name` | 命名混用 `-` / `_` 等不合規 |
| `registry.duplicate_action` | 重複註冊 |
| `registry.name_conflict` | 同類型同名衝突 |
| `registry.dependency_cycle` | Function 間循環依賴（載入期偵測） |
| `max_recursion_depth_exceeded` | 運行時 function 遞迴深度超限 |
### Workflow / Trigger

| code | 觸發 |
|------|------|
| `workflow.input_schema_violation` | trigger 階段輸入驗證失敗 |
| `output_schema_violation` | return 輸出驗證失敗 |

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
     step_id: <current_step_id>
   }
3. 依當前位置決定傳播：
   - 一般 step 序列中：直接 raise（向所在控制結構傳播）
   - catch handler 中：rethrow，跳出 catch
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

`config.catch` 內只能用 `error.*` / `input` / `vars` / `steps` 等已凍結的 namespace；不可再執行 `wait` 等需要長時間 I/O 的 step（實作可拒絕；建議僅允許 emit/fail/return/assign）。

---

## 觀測

execution_log 中每則 ErrorObject MUST 完整保留（含 cause）；redaction 規則對 message 套用（過濾 secret 明文）。

Engine 提供 metric：`error_total{type, kind}` 區分 user_fail vs system error。
