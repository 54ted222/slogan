# 02 — Instance Lifecycle

本文件定義 workflow / function instance 的狀態機、轉換規則與持久化時機。

---

## Instance 種類

| 種類 | 由誰建立 | 終結方式 |
|------|----------|----------|
| **Workflow Instance** | Trigger Resolver | `return` step、`fail` step、unhandled error、`config.timeout` |
| **Function Instance** | `type: task` 呼叫 function | `return` step、unhandled error、外層 timeout |

Function instance 由父 instance「擁有」；父 instance 若被取消，子 instance 同步取消。

---

## 狀態機

```
   created            ┌──────────┐
       │              │ PENDING  │ ◄─ instance 已建立但未開始
       ▼              └────┬─────┘
   ┌───────┐               │
   │ start │               ▼
   └───┬───┘          ┌──────────┐
       └─────────────►│ RUNNING   │
                      └────┬─────┘
            ┌──────────────┼────────────────┐
            ▼              ▼                ▼
       ┌──────────┐   ┌──────────┐    ┌──────────┐
       │SUCCEEDED │   │  FAILED  │    │CANCELLED  │
       └──────────┘   └──────────┘    └──────────┘
            ▲              ▲                ▲
            │              │                │
        return step    fail step / 未捕     父 instance 取消 /
                      捉錯誤 / timeout       手動 API
```

中間態（**RUNNING** 細分）：

| 子態 | 條件 |
|------|------|
| `RUNNING.executing` | 當前 step 處於 RUNNING |
| `RUNNING.waiting_signal` | 當前 step 為 `wait` signals 模式，已建立訂閱（事件訊號 / step 訊號） |
| `RUNNING.suspended_callback` | function instance 處於 `type: callback` step，等待 caller handler |
| `RUNNING.compensating` | saga 進入補償階段 |
| `RUNNING.cancelling` | 收到 cancel 訊號、已發 cancel 至子節點、等待 grace period 或子節點終結；到期 / 全數終結後 transition 至 CANCELLED（見 `10-concurrency.md` Cancel propagation） |

---

## 狀態轉換

### PENDING → RUNNING
- Trigger Resolver 寫入 instance 後立即發 `instance.created` 事件。
- Engine Loop 接到事件，acquire lease，將狀態改為 `RUNNING.executing`，從 `steps[0]` 開始。
- 若 `input_schema` 在 PENDING 階段已驗過，此處不重驗。

### RUNNING → SUCCEEDED
- 觸發點：`type: return` step 或 `steps[]` 自然走完（fall-through）。
- fall-through 時最終 output **恆為 `null`**（不繼承最後一個 step 的 output；若需以最後 step output 作為 instance output，MUST 顯式以 `type: return` 宣告）。若 workflow / function 定義了 `output_schema` 且 schema 不接受 `null`，則 instance FAILED，`error.type == "output_schema_violation"`。
- 終結前（以下步驟於**單一 checkpoint transaction** 內 atomic 執行）：
  1. 求值 `output_schema`（若 instance 為 workflow / function）
     - 驗證失敗 → instance 狀態改為 FAILED，`instance.output` 保持 `null`（**不**寫入驗證前的候選值），`error.type == "output_schema_violation"`
     - 驗證成功 → 候選 output 成為最終 `instance.output`
  2. 寫入最終狀態（SUCCEEDED 或 FAILED）、output、終結時間戳
  3. 釋放 Resource Pool 中此 instance 持有的資源（觸發 lifecycle destroy，見下）
  4. **Checkpoint commit 後**才發出 `instance.completed` / `instance.failed` 事件（不可在 commit 前發，以免父 instance 讀到尚未持久化的狀態）
  5. 若有父 instance 等待此實例，wake 父 instance（父讀到的 output 即 checkpoint 中的值）

**Destroy hook 失敗處理**：

- Lifecycle destroy hook 於 checkpoint commit **之後** 執行；視為 best-effort，不改變已定案的 instance 終態。
- destroy 失敗 → 記錄 `lifecycle.destroy_failed` observability 事件（含 `instance_id` / `resource_key` / error），並排入清理重試佇列（依 resource pool 策略）。
- destroy 成功與否**不影響** `instance.completed` / `instance.failed` 事件發送順序；父 instance 的 catch / failure_policy 只看 instance 終態。

### RUNNING → FAILED
- 觸發點：未被任何 `catch` / saga 捕捉的錯誤、`type: fail` step、`config.timeout` 等。
- 終結前同上（仍跑 destroy hooks）；output 為 `null`，`error` 物件包含 `type` / `message` / `code` / `step_id` / `cause?`。
- 父 instance 收到 `instance.failed` 事件後，依其 `catch` 或 `failure_policy` 決策。

### RUNNING → CANCELLED
- 觸發：父 instance cancellation、外部 API 顯式 cancel、workflow 級 timeout 在 `cancellation_policy: graceful` 下的軟取消。
- 引擎發送 cancel 訊號至所有未終結子節點，等待至 `cancel_grace_period`（預設 30s）；超時則強制終結。
- 已執行的 step 不回滾（除非整體在 saga 內，由 saga 補償）。
- **`config.catch` 是否執行**：
  - 外部 API / 父 instance 傳播 cancel → **不執行** `config.catch`（cancel 非「錯誤」語意；instance 直接轉 CANCELLED）
  - Workflow `config.timeout` 到期 → **執行** `config.catch`（timeout 視為錯誤，與 RUNNING → FAILED 同路徑，`error.type == "timeout"`）；若 catch 消化為 SUCCEEDED / FAILED，最終 state 依 catch 結果，**不**再轉 CANCELLED
  - 兩種情境以 `cancel_reason` 欄位區分（`api` / `parent` / `timeout` / `ops`，見 `instance.cancel` 事件的 `requested_by`）
- **延遲事件清理**：cancel 同時 DELETE 該 instance 的所有未投遞 `delayed_events`（`WHERE source_instance = $id AND claimed_by IS NULL`）；已被其他 engine claim 中的 event 仍會投遞（避免破壞 claim 所有者的執行），但接收方若對應 wait 已不存在則丟棄。
- **終結流程**：寫終結 checkpoint（CANCELLED、ended_at）→ 發 `instance.cancelled` 事件 → 執行 destroy hooks（best-effort，同 SUCCEEDED / FAILED；失敗僅記錄於 `lifecycle.destroy_failed`）→ 清理 Resource Pool。

#### cancellation_policy

於 `workflow.config.cancellation_policy`（或 timeout 情境）可配置：

| 值 | 行為 |
|----|------|
| `graceful`（預設） | 發 cancel 訊號 → 等待 `cancel_grace_period`（預設 30s）→ 仍未終結者強制終結（exec SIGKILL、http connection force close、sub-instance hard cancel） |
| `hard` | 不等寬限期；直接強制終結所有未終結子節點 |

`cancel_grace_period` 可由 `workflow.config.cancel_grace_period` 覆寫（限 graceful 模式）。`hard` 模式下此欄位忽略。

`cancellation_policy` 與 `cancel_grace_period` 的 CEL 求值在 workflow 載入時一次性完成（常量），不可使用 instance-time namespace。

---

## Step state（per-step）

每個 step 各自的微狀態機：

```
WAITING ─► RUNNING ─► SUCCEEDED
                    └► FAILED ─► (捕捉) ─► SUCCEEDED
                    │                  └► (未捕捉) ─► 向上傳播
                    └► SKIPPED （when == false）
```

| State | 條件 | 寫入持久化 |
|-------|------|------------|
| `WAITING` | 兩種情境：(A) 首次尚未進入 / `when` 未求值；(B) 上一 attempt FAILED 且進入 retry backoff（`scheduled_at = next_attempt_at`） | (A) 否；(B) 是 — 寫 attempt=N+1 row 以記錄 backoff deadline（見 `03-step-execution.md` retry 狀態機） |
| `RUNNING` | `when` 真值通過、進入執行；外部 I/O 已啟動 | 是（檢查點） |
| `SUCCEEDED` | step handler 回傳成功；output 已寫入 | 是 |
| `FAILED` | step handler 拋錯或外部回報失敗 | 是（含 error） |
| `SKIPPED` | `when` 求值為 false | 是（output: null） |

---

## 持久化時機（Checkpoint）

下列轉換 MUST 在「事件被視為已處理」之前 atomic 寫入 Instance Store：

1. `PENDING` → `RUNNING.executing`（instance 開始）
2. step `WAITING` → `RUNNING`（含當下的 namespace snapshot 摘要）
3. step `RUNNING` → 終態（含 output / error）
4. wait subscription 建立 / 取消
5. saga 進入 / 退出補償階段
6. `RUNNING` → `SUCCEEDED` / `FAILED` / `CANCELLED`（含最終 output / error）

非 checkpoint 的中間狀態（如 tool 的中間 stdout 訊息、step 內部計數器）SHOULD 寫入 execution log 但不影響重啟後的 step 終態判定。

---

## Lease 與單一擁有者

- Engine 進程透過 lease 擁有 instance：lease 內容為 `(engine_id, expires_at)`。
- Lease TTL 預設 30s；lease 持有進程 MUST 在 step transition 之間定期 renew。
- Lease 過期後，其他 engine 進程可在下一個 transition 點接管；接管時讀取最新 checkpoint，從中復原。
- 同一 engine 進程為單線程 event loop，進程內不需額外 lock。

復原語意：

| 失去 lease 時的步驟狀態 | 接管後行為 |
|--------------------------|------------|
| `WAITING` | 重新求值 `when`、進入 RUNNING |
| `RUNNING` 的 `type: task` 呼叫 Tool 且 Tool 宣告 `idempotent: true` | 重跑此 step；engine 依 `signature` 嘗試讀取 cached output（見 `03-step-execution.md` Attempt 與 signature） |
| `RUNNING` 的 `type: task` 呼叫 Tool 且 Tool 為 `idempotent: false`（預設） | step → FAILED，`error.type == "lease_lost_unsafe_resume"`；走 `retry` / `catch` 路徑。**In-flight callback 清理**：若失去 lease 時 tool 已發出 callback 請求但尚未收到 `callback_result`，接管者 MUST 以 `(instance_id, step_path, attempt)` 讀取 `call_id_dedup`，對所有尚未回應的 call_id 觸發 orphan cleanup（見 `06-tool-backend.md` 的「Lease lost 後的 callback orphan 處置」）；tool process 本身由前一 engine 的 driver 負責回收（SIGTERM → SIGKILL），但新 engine 無法直接通知舊 tool process，依 process timeout 兜底 |
| `RUNNING` 的 `type: task` 呼叫 Function | 父 step 無重執行風險（子 function instance 自有 lease 與 checkpoint）；直接續讀子 instance 狀態，子 instance 終結後以正常路徑喚醒父 step |
| `RUNNING` 的 `type: assign` / `if` / `switch` / `return` / `fail` | 無副作用；直接重跑（純求值；結果由 deterministic 求值保證一致，`now()` / `uuid()` 從 execution_log 讀回，見 `08-persistence.md` Replay） |
| `RUNNING` 的 `type: emit` | 依 checkpoint 位置：若 outbox 已 INSERT（同 transaction 內）→ publisher 自行 publish，step 視為 SUCCEEDED；若 checkpoint 尚未 commit → 重跑本 step（新 event.id 生成；原未持久的 emit 被丟棄） |
| `RUNNING` 的 `type: wait` | subscription 已寫入；接管者讀取 subscription 與 deadline 繼續等待（見 `07-event-bus.md` wait subscription） |
| `RUNNING` 的 `type: foreach` / `parallel` / `saga` | 依 checkpoint 還原 iteration / branch / compensation 狀態；各子 step 依上述規則各自復原 |
| `RUNNING` 的 `type: callback` | 依 checkpoint `suspended_callback` 還原；若 callback_result 已於丟失 lease 前收到並持久化 → 繼續；若尚未收到 → 重新等待（caller handler 以 call_id 去重避免重複執行） |
| `SUCCEEDED` / `FAILED` / `SKIPPED` | 直接讀取 checkpoint，不重執行 |

**術語**：表中「step `idempotent: true/false`」為 DSL / registry 層面的 Tool definition 屬性（見 `dsl-spec/05-tool.md`）；step 本身無此欄位。同一 Tool 被多 step 引用時共用同一 `idempotent` 旗標。

---

## 巢狀關係

```
Workflow Instance
 ├─ Step (type: task action: <function>) → Function Instance
 │    └─ Step (type: callback)  ← 暫停，等待 caller handler
 │
 └─ Step (type: task action: <tool>) → Tool execution（不算 instance）
```

- 父 instance 在子 instance 終結前處於 `RUNNING.executing`（單一 step 等待中）；子 instance 終結事件喚醒父 instance。
- Cancellation 從父向子單向傳播；子的失敗向父呈現為一個 step FAILED（依 catch 處理）。

---

## Timeout 計時

| 範圍 | 起算點 | 結束點 | 超時行為 |
|------|--------|--------|----------|
| `config.timeout`（workflow / function） | instance `RUNNING` 開始 | `SUCCEEDED` / `FAILED` | 整 instance FAILED，`error.type == "timeout"`；先走 `config.catch` |
| `timeout`（step 級） | step `RUNNING` 開始 | step 終態 | step FAILED，`error.type == "timeout"`；先走 step `catch` |
| `wait.timeout` | wait subscription 建立 | event 到達 / step 終態 | wait FAILED，`error.type == "timeout"` |
| `callback timeout` | 發出 callback 訊息 | handler `return` | callback step FAILED，`error.type == "timeout"` |

### max_step_executions 限制

`workflow.config.max_step_executions` / `function.config.max_step_executions`（見 DSL spec）於運行時以 instance-level counter 追蹤：

- Counter 初值為 0；每次 step 由 `WAITING` / retry-sleep 進入 `RUNNING` checkpoint 時 +1（即每次 attempt 計 1 次，retry 也計入；`when:false` 的 SKIPPED **不**計入，因為未進入 RUNNING）
- 每次 checkpoint commit 後比對：若 counter > `config.max_step_executions` → instance 立即進入終結流程，`error.type == "max_step_executions_exceeded"`、`error.details.limit` 為該欄位值、`error.details.observed` 為實際 counter
- 該錯誤走 `config.catch`（與 timeout 同路徑）；catch 消化後 instance 可 SUCCEEDED；否則 FAILED
- Counter 持久化於 `instances.step_executions_count` 欄位；engine 重啟沿用，不重置
- 無 `config.max_step_executions` 宣告 → 不做檢查（無限制）；實作 MAY 於 engine config 設置全域 `engine.default_max_step_executions` 作為保底（預設不設）
- 嵌套 foreach / parallel：每個子 step 執行各自計 1 次（包括 foreach 內每次迭代的每個 step）；保護意圖即「防止 runaway loops」

### Callback timeout 規則

Callback 的 timeout 來源（由先到先生效：取最小 deadline）：

- **`type: callback` step 的 `timeout`**（DSL 允許；見 `dsl-spec/05b-function.md`）：handler 執行的整體時長；覆蓋發出 callback 事件至 handler `return` 全程
- **Tool callback** 為 tool 在 task step 執行中對外觸發的 callback（非獨立 step）：不存在「callback 自己的 timeout 欄位」，繼承當前 task step 的 `timeout` 剩餘時間；即 `step.timeout` 時鐘在 callback 發出後不暫停。若發出 callback 時 step 已剩 5s，tool callback handler MUST 在 5s 內完成並回覆
- 兩情境皆再向上受 function `config.timeout` / workflow `config.timeout` 限制；皆無則可無限等待（不建議）
- Callback 逾時處置：
  1. 引擎標記 step FAILED，`error.type == "timeout"`
  2. Tool 端：引擎 MUST 對同一 `call_id` 回覆 `{"type":"callback_result","call_id":"...","error":{"type":"timeout","message":"caller timeout"}}`，讓 tool 可清理；之後引擎關閉 stdin / SSE 連線
  3. Function callback：caller 撤銷 handler 的 pending state；function instance 收到 callback_result error 自行決定是否 FAILED
  4. 若 callback 遲到（step 已終態後才送達 callback_result）→ 引擎丟棄該訊息並記錄 `callback.late_result` 觀測事件
- Tool 協議可附 `deadline` 欄位：引擎首次 callback 請求時 MAY 於 `input` 之外帶 `deadline` (Unix 毫秒) 告知 tool 當前 callback 必須在何時前回覆；tool 側未實作者 MAY 忽略。

Timeout 計時器 SHOULD 持久化（記錄 deadline 而非剩餘秒數），確保 engine 重啟後仍能正確觸發。

### 巢狀 timeout 的先到先生效

Workflow → Function task step → Function instance 三層皆可有 timeout：

| 層級 | 欄位 | 生效範圍 |
|------|------|----------|
| Workflow config | `workflow.config.timeout` | workflow instance 整體 |
| Task step | `timeout` on `type: task` | 從呼叫 function 起到其終結 |
| Function config | 呼叫的 Function 自身 `config.timeout`（若有） | function instance 整體 |

計時 **獨立**，三個 deadline 並存；先到的先生效：

- Function timeout 先到 → function instance FAILED（`error.type: timeout`），父 task step 收到 function FAILED（可被父 catch 處理）
- Task step timeout 先到 → task step FAILED，function instance 被 cancel
- Workflow timeout 先到 → workflow 進入終結流程（含 config.catch），所有子 function instance 被 cancel

建議應用層設計為 **function timeout < task step timeout < workflow timeout**，避免上層 timeout 優先觸發導致難以定位問題。

---

## 版本鎖定（Definition pinning）

Workflow / Function instance 建立時 MUST 記錄所用 definition 的 `(name, version)`；instance 生存期內：

- 所有 step 執行、子 function 呼叫、action 解析都**使用該版本快照**；即使 definition 在運行中被更新/重載，instance 的行為不變
- Registry 為每個 active version 保留完整 definition 記憶體快照；僅當**沒有 active instance** 引用舊 version 時才可 GC
- 若 definition 被明確刪除（CLI `slogan delete`）而仍有 active instance → 拒絕刪除，error `definition_in_use`
- 強制刪除（`--force`）→ 所有使用該 version 的 instance 被標記為 FAILED，`error.type: "workflow_version_deleted"`

Trigger 建立新 instance 時：

- Manual / event trigger 皆取 registry 中該 workflow 的**最大 version**（除非 API 帶明確 `version=N`）
- 新 instance 一旦建立即鎖定該 version，不再隨 definition 更新改變

### 版本選擇 API

- Manual trigger：API 支援 `?version=N` query / `{version: N}` body 欄位顯式選版；省略則取 registry 最大
- Event trigger：每個 workflow 的 `triggers[]` 針對當時 registry 的最大 version 訂閱；若同 workflow 有多版本共存，舊版 trigger 亦維持訂閱（behavior：所有 active 版本的 trigger 獨立接收事件，各自建立 instance）
- 指定不存在的 version → `registry.action_not_found`

### 版本回滾（Rollback）

v6 不支援原地「降版」；回滾透過**發佈新版本**完成：

1. 假設 v2 有 bug，運維決定回退至 v1 的行為
2. **不可**將 v2 definition 改回 v1 內容（違反「同 `(name, version)` 內容不可變」規則；見 `dsl-spec/01-overview.md`）
3. 正確做法：以 v1 的內容為基礎，bump 至 v3（或 v4 等）發佈新版；registry 最大 version 即為 v3
4. 已運行的 v2 instance 在 action_pin 保護下繼續以 v2 執行至完成；新 instance 使用 v3
5. 管理員若需**立即中止**所有 v2 instance：`slogan cancel --workflow=<name> --version=2 --all`（由實作提供）

此設計確保：
- Instance 行為可追溯（version 即為 immutable snapshot id）
- 回滾不會破壞正在執行的業務（緊急中止需顯式）
- Git 式的版本歷史（新版覆蓋舊版，但舊版記錄保留）

---

## 結束後的保存期

| 終態 | 預設保存 |
|------|----------|
| `SUCCEEDED` workflow instance | 7 天（含 output 與 log；可由 project 設定覆寫） |
| `FAILED` workflow instance | 30 天 |
| `CANCELLED` workflow instance | 7 天 |
| Sub-instance（function instance） | 隨父 instance 一同清除 |

### Retention 覆寫規則

- Project / engine config 可設定預設保存期倍率或絕對值（`project.yaml` 的 `retention_policy: {succeeded: 30d, failed: 90d}`）
- **Runtime API 不支援延長單一 instance 的 retention**：`retention_until` 於 instance 建立時即依該 version 的 project 設定決定並寫入 checkpoint；後續不可變更
- 需要長期保留（如 audit trail）：請於 workflow 內以 `type: task` 呼叫專用 tool（如 `audit.archive`）將關鍵資料匯出至外部系統（S3 / 資料倉儲）；engine 本身不作為長期歸檔
- GC 行為：背景 job 依 `retention_until <= now()` 硬刪除；無「軟刪除」階段；對已刪除的 instance_id 做 API 查詢 → `404 instance_not_found`（不區分「從未存在」與「已回收」）
