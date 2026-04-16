# 08 — Persistence

本文件定義 Instance Store 的邏輯資料模型、checkpoint 寫入規則、保存期、以及 replay / debug 對 execution log 的要求。本規格描述行為，不規範儲存後端（PostgreSQL / RocksDB / 檔案系統皆可）。

---

## 邏輯資料表

### instances

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | uuid | 主鍵 |
| `parent_id` | uuid \| null | 父 instance（function instance 才有） |
| `kind` | enum | `workflow` / `function_instance` |
| `definition_name` | string | 對應 definition 的完整 name |
| `definition_version` | int | |
| `project` | string \| null | |
| `state` | enum | `PENDING` / `RUNNING` / `SUCCEEDED` / `FAILED` / `CANCELLED` |
| `substate` | string \| null | RUNNING 細分（`executing` / `waiting_signal` / `suspended_callback` / `compensating`） |
| `input` | jsonb | 初始 input |
| `output` | jsonb \| null | 終態 output |
| `error` | jsonb \| null | 終態 error 物件 |
| `vars` | jsonb | 當前 vars namespace |
| `created_at` / `started_at` / `ended_at` | timestamp | |
| `lease_owner` / `lease_expires_at` | string / timestamp | Lease |
| `trace_id` | string | 跨 instance 追蹤 |
| `retention_until` | timestamp | 保存期截止 |

### steps

| 欄位 | 型別 | 說明 |
|------|------|------|
| `instance_id` | uuid | FK |
| `step_id` | string | 使用者定義或自動生成 |
| `step_path` | string | 含巢狀路徑（`saga.0.if.then.0` 等） |
| `type` | string | step type |
| `state` | enum | `WAITING` / `RUNNING` / `SUCCEEDED` / `FAILED` / `SKIPPED` |
| `attempt` | int | 第幾次嘗試（retry） |
| `input_snapshot` | jsonb | 求值後的 input（snapshot） |
| `output` | jsonb \| null | |
| `error` | jsonb \| null | |
| `started_at` / `ended_at` | timestamp | |
| `signature` | string | 對 input + action 的 hash，用於 idempotent retry 比對 |
| `compensate_state` | enum \| null | saga 補償用：`pending` / `started` / `done` / `failed`（見 `03-step-execution.md` saga 補償狀態機） |

主鍵：`(instance_id, step_id, attempt)`

### wait_subscriptions

訂閱的 patterns 以 JSONB 欄位儲存（signals 可含多個 pattern）：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `subscription_id` | uuid | |
| `instance_id` / `step_id` | | |
| `patterns` | jsonb | `[{event_type, scope?, match_expr?}]` 陣列 |
| `any_of` | bool | signals 模式恆為 true |
| `deadline` | timestamp | wait.timeout |
| `created_at` | timestamp | |

### delayed_events

| 欄位 | 型別 | 說明 |
|------|------|------|
| `event_id` | uuid | |
| `event_type` | string | |
| `scope` | enum | |
| `data` | jsonb | |
| `due_at` | timestamp | |
| `source_instance` | uuid | |
| `claimed_by` | string \| null | 搶取該 event 的 engine_id；null 表示可被搶取 |
| `claimed_until` | timestamp \| null | claim 的 TTL；過期視為未 claim |

多 engine 搶取 SQL 模式：

```sql
UPDATE delayed_events
SET    claimed_by = $engine_id, claimed_until = now() + 30s
WHERE  event_id = (
         SELECT event_id FROM delayed_events
         WHERE due_at <= now()
           AND (claimed_by IS NULL OR claimed_until < now())
         ORDER BY due_at ASC
         FOR UPDATE SKIP LOCKED
         LIMIT 1
       )
RETURNING *;
```

投遞成功後 DELETE；投遞失敗則清空 `claimed_by`（下次 retry）。避免時鐘漂移造成重複投遞。

### resource_pool

每個 (tool_name, version) 在每個 instance 的 lifecycle init 結果與 destroy 狀態：

| 欄位 | 說明 |
|------|------|
| `instance_id` | |
| `tool_name` / `tool_version` | |
| `init_output` | jsonb \| null |
| `init_error` | jsonb \| null |
| `init_at` | timestamp |
| `destroy_at` | timestamp \| null |

### execution_log

每個重要事件的 append-only 流水：

| 欄位 | 說明 |
|------|------|
| `seq` | int64（per-instance auto-increment） |
| `instance_id` | |
| `step_id` | nullable |
| `event_type` | string（`step.transition` / `expr.eval` / `tool.invoke` / `bus.publish` / ...） |
| `payload` | jsonb |
| `at` | timestamp |

---

## Checkpoint 規則

每次 step transition 都是一個 checkpoint，必須以 atomic transaction 寫入：

```
BEGIN
  UPDATE steps SET state, output, error, ended_at WHERE ...
  UPDATE instances SET state, substate, vars, lease_expires_at WHERE id = ...
  INSERT INTO execution_log (event_type='step.transition', ...)
  -- 若該 transition 觸發 wait subscription 變動：
  INSERT/DELETE wait_subscriptions ...
  -- 若 emit：
  INSERT delayed_events 或直接 publish
COMMIT
```

事件投遞（外部 bus）發生在 commit 之後；使用 outbox pattern 確保「持久化先於投遞」。

---

## Lease 機制

lease 透過樂觀鎖：

```
UPDATE instances
SET    lease_owner = $engine_id, lease_expires_at = now() + 30s
WHERE  id = $instance_id
  AND  (lease_owner IS NULL OR lease_expires_at < now() OR lease_owner = $engine_id)
RETURNING ...
```

只有更新成功的 engine 進程持有 lease。每次 checkpoint 順便延展 lease。Engine 進程 crash → lease 自然過期；其他 engine 進程在下次 polling 時搶到。

---

## 並發隔離

- 同一 instance 的 step 寫入由 lease（跨進程）+ 單線程 event loop（同進程）保證序列化。
- 不同 instance 之間獨立。
- `wait_subscriptions` 與 `event.matched` 路徑：subscription 寫入屬於 instance 的 transaction；事件投遞由 bus 推送；engine 收到 `event.matched` 時以 `(instance_id, step_id)` 取得 lease 後處理。

讀取一致性：

- 同 instance 讀「自己最新狀態」MUST 在 transaction 內讀（避免讀到舊版）。
- 跨 instance 讀（如 wait step 模式讀目標 step 狀態）採 snapshot；若目標 step 仍 RUNNING 則訂閱 step.completed 事件。

---

## Idempotency

### Step 級

`steps.signature` = hash(`{action_name, input_snapshot, attempt}`)；用於：

- 若 step `idempotent: true` 且 retry 觸發前發現上一次 attempt 已 SUCCEEDED 但 lease 切換至其他進程 → 直接讀取 cached output（避免重跑）
- 若 `idempotent: false` → 每 attempt 重新執行，signature 僅作觀測

### Tool 級

- exec/http backend 將 `idempotency_key` 注入 `context`；tool 可用此值對外部系統去重。
- `idempotency_key = hash(instance_id + step_id + attempt + input_snapshot)`；attempt 變動時重新生成。
- `input_snapshot` 定義為「step 進入 RUNNING 時寫入 checkpoint 的 input 物件」；不含 `when` / `retry.delay` / `timeout` 等控制欄位的求值結果。
- Tool 若宣告 `idempotent: true`，同一 `idempotency_key` 在引擎重試時 MUST 視為同一邏輯操作；backend driver 在 retry 前會先查 cached output。

---

## 保存期與清理

| 終態 | 預設保存 | 清理對象 |
|------|----------|----------|
| `SUCCEEDED` workflow | 7 天 | 全部關聯 steps / log / sub-instance |
| `FAILED` workflow | 30 天 | 同上 |
| `CANCELLED` workflow | 7 天 | 同上 |
| Sub-instance | 跟父走 | |
| `wait_subscriptions` | instance 終結時立即刪除；未終結但已超過 deadline 者依 GC job 刪除 | |
| `delayed_events` | 投遞後立即 | |
| `execution_log` | 跟所屬 instance 走 | |
| `resource_pool` | instance 結束時 |

清理任務以背景 job 執行；建議分批刪除避免長 transaction。

每個 project 可以在 project.yaml 設定覆寫保存期（v6 目前未指定 yaml schema；保留為實作擴充）。

---

## 加密

- `secret.*` 解密後的明文不寫入任何持久化欄位（input_snapshot / log / output）。
- 寫入前對 input_snapshot / output / log 做 redaction：以 SecretAccessor 追蹤被讀過的明文 → log writer 將其替換為 `"***"`。
- TLS / disk encryption 由實作層提供。

---

## Size Limits（存入前檢查）

| 欄位 | 預設上限 | 超限行為 |
|------|----------|----------|
| `steps.input_snapshot` | 16 MB | step FAILED，`error.type == "input_too_large"`，error.details.size 保留原始大小 |
| `steps.output` | 16 MB | step FAILED，`error.type == "output_too_large"`；原 output 不入庫，error.details.preview 保留前 4 KB |
| `instance.output` | 16 MB | instance FAILED，`error.type == "output_too_large"` |
| 單筆 `execution_log.payload` | 1 MB | 超限則截斷尾部，log 記錄 truncation 標記 |
| 單一 event.data（Event Bus） | 1 MB | emit step FAILED，`error.type == "event_too_large"` |

上限可由 engine config 覆寫（`engine.max_step_output_bytes` 等）。超大資料建議以 artifact 儲存並在 output 中傳 path 引用。

---

## Replay

```
def replay(instance_id, until: timestamp | step_id | None):
    1. 載入 instance state
    2. 重建 vars / steps namespaces 至 until 點
    3. 讀取 execution_log，對所有 expr.eval 與 now()/uuid() 結果可重建
    4. 重新求值 expression（檢查結果一致性）
    5. 回報 diff（哪些 expression 結果改變、哪些 step output 不同）
```

Replay 不重新呼叫外部 tool；用於 audit、debug、CEL 升級驗證。

---

## 觀測指標

實作 SHOULD 暴露：

- `instance_count{state, kind}`
- `step_duration_ms{type, state}` histogram
- `lease_acquisitions_total{engine_id}`
- `lease_lost_total{instance_kind}`
- `event_bus_lag_seconds{type}`
- `wait_subscriptions_count{state}`
- `tool_invocations_total{action_name, success}`
- `expression_eval_failures_total{type}`
- `saga_compensate_total{action_name, result}` — result: `done` / `failed`
- `callback_handler_duration_ms{function_name, callback_name}` histogram
- `backend_spawn_duration_ms{backend_type}` histogram
- `error_total{type, kind}` — kind: `user_fail` / `system_error`（與 09-error-model.md 對齊）
