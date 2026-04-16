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
| `RUNNING.waiting_event` | 當前 step 為 `wait` event 模式，已寫入 wait subscription |
| `RUNNING.waiting_step` | 當前 step 為 `wait` step 模式，等待 async step 終態 |
| `RUNNING.suspended_callback` | function instance 處於 `type: callback` step，等待 caller handler |
| `RUNNING.compensating` | saga 進入補償階段 |

---

## 狀態轉換

### PENDING → RUNNING
- Trigger Resolver 寫入 instance 後立即發 `instance.created` 事件。
- Engine Loop 接到事件，acquire lease，將狀態改為 `RUNNING.executing`，從 `steps[0]` 開始。
- 若 `input_schema` 在 PENDING 階段已驗過，此處不重驗。

### RUNNING → SUCCEEDED
- 觸發點：`type: return` step 或 `steps[]` 自然走完（fall-through）。
- 終結前：
  1. 求值 `output_schema`（若 instance 為 workflow / function）
  2. 寫入最終 output 與終結時間戳
  3. 釋放 Resource Pool 中此 instance 持有的資源（觸發 lifecycle destroy）
  4. 發出 `instance.completed` 事件
  5. 若有父 instance 等待此實例，wake 父 instance

### RUNNING → FAILED
- 觸發點：未被任何 `catch` / saga 捕捉的錯誤、`type: fail` step、`config.timeout` 等。
- 終結前同上（仍跑 destroy hooks）；output 為 `null`，`error` 物件包含 `type` / `message` / `code` / `step_id` / `cause?`。
- 父 instance 收到 `instance.failed` 事件後，依其 `catch` 或 `failure_policy` 決策。

### RUNNING → CANCELLED
- 觸發：父 instance cancellation、外部 API 顯式 cancel、workflow 級 timeout 在 `cancellation_policy: graceful` 下的軟取消。
- 引擎發送 cancel 訊號至所有未終結子節點，等待至 `cancel_grace_period`（預設 30s）；超時則強制終結。
- 已執行的 step 不回滾（除非整體在 saga 內，由 saga 補償）。
- 終結前同上。

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
| `WAITING` | 尚未進入；`when` 未求值 | 否 |
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

非 checkpoint 的中間狀態（如 LLM token 串流的部分輸出）SHOULD 寫入 execution log 但不影響重啟後的 step 終態判定。

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
| `RUNNING` 且 step `idempotent: true` | 重跑此 step |
| `RUNNING` 且 step `idempotent: false`（預設） | step → FAILED，`error.type == "lease_lost_unsafe_resume"`；走 `retry` / `catch` 路徑 |
| `SUCCEEDED` / `FAILED` / `SKIPPED` | 直接讀取 checkpoint，不重執行 |

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

Timeout 計時器 SHOULD 持久化（記錄 deadline 而非剩餘秒數），確保 engine 重啟後仍能正確觸發。

---

## 結束後的保存期

| 終態 | 預設保存 |
|------|----------|
| `SUCCEEDED` workflow instance | 7 天（含 output 與 log；可由 project 設定覆寫） |
| `FAILED` workflow instance | 30 天 |
| `CANCELLED` workflow instance | 7 天 |
| Sub-instance（function instance） | 隨父 instance 一同清除 |
| `tool.stream` 中間訊息 | 不持久化（即時投遞至訂閱者後丟棄） |
