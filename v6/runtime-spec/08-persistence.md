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
| `substate` | string \| null | RUNNING 細分（`executing` / `waiting_signal` / `suspended_callback` / `compensating` / `cancelling`） |
| `input` | jsonb | 初始 input |
| `output` | jsonb \| null | 終態 output |
| `error` | jsonb \| null | 終態 error 物件 |
| `vars` | jsonb | 當前 vars namespace |
| `created_at` / `started_at` / `ended_at` | timestamp | |
| `lease_owner` / `lease_expires_at` | string / timestamp | Lease |
| `trace_id` | string | 跨 instance 追蹤 |
| `retention_until` | timestamp | 保存期截止 |
| `action_pins` | jsonb | map `{canonical_action_name: version}`；首次解析該 action 時寫入，鎖定此 instance 後續同名解析（見 `05-task-registry.md`） |
| `cancel_requested` | bool | 外部 cancel 訊號標記；由不持有 lease 的進程以無鎖 UPDATE 寫入，lease 持有進程於 event loop tick 檢查並進入 CANCELLED 流程（見 `10-concurrency.md`）。預設 `false`；寫入後不再回寫 `false`（終態本身已代表取消完成） |
| `cancel_reason` | string \| null | cancel_requested 被置為 true 時隨附的 reason；供 observability 使用 |

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
| `deadline` | timestamp \| null | wait.timeout 對應 deadline；`null` 表示無限等待（對應 `wait.timeout` 缺省，見 `07-event-bus.md`）。GC job MUST 以 `deadline IS NOT NULL AND deadline < now()` 為掃描條件，避免誤刪無限等待的 subscription |
| `created_at` | timestamp | |

### delayed_events

| 欄位 | 型別 | 說明 |
|------|------|------|
| `event_id` | uuid | 原事件 `event.id`；投遞時**原樣**送出以維持訂閱端去重語意（不重新生成） |
| `event_type` | string | |
| `scope` | enum | |
| `data` | jsonb | |
| `due_at` | timestamp | |
| `source_instance` | uuid | 同 `Event.source.instance_id` |
| `source_step_id` | string \| null | 同 `Event.source.step_id`（emit 所屬 step；無則 null） |
| `source_sequence` | int64 | 同 `Event.source.sequence`；於 emit step 寫入 delayed_events 時一併分配並持久化，避免投遞時重新編號導致同源順序破壞 |
| `trace_id` | string | 繼承自 emit 所在 instance；投遞時保留以維持跨 instance 追蹤鏈（見 `07-event-bus.md` trace_id 傳播） |
| `delivery` | enum | `broadcast` / `unicast`；使用者 emit 恆為 broadcast，internal 延遲事件可為 unicast |
| `target` | string \| null | unicast 目標 instance_id；broadcast 時為 null |
| `claimed_by` | string \| null | 搶取該 event 的 engine_id；null 表示可被搶取 |
| `claimed_until` | timestamp \| null | claim 的 TTL；過期視為未 claim |

投遞時 publisher 以上述欄位還原 `Event` 物件；`timestamp` 採投遞時刻（非原 emit 時刻，與 `07-event-bus.md` 行為一致）。`event_id` / `source.*` / `trace_id` 保留原值，確保訂閱端去重與 trace 關聯正確。

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

### trigger_dedup

Event trigger 建立 instance 的去重記錄（見 `07-event-bus.md` 的 Idempotency 章節）：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `workflow_name` | string | 被觸發的 workflow `metadata.name`（含 project 前綴） |
| `workflow_version` | int | 被觸發的 workflow version |
| `event_id` | uuid | 原事件 `event.id` |
| `instance_id` | uuid \| null | 成功建立的 instance id；供觀測 / 重投遞時回查（manual trigger idempotency 亦寫入此表以 `event_id = idempotency_key` 形式統一） |
| `trigger_index` | int | 被選中的 `triggers[]` 索引；多 trigger 同時匹配時用於追蹤（見 `dsl-spec/02-workflow.md`） |
| `created_at` | timestamp | INSERT 時刻 |
| `expires_at` | timestamp | 由 engine config `registry.trigger_dedup_ttl`（預設 24h）決定 |

- 主鍵：`(workflow_name, workflow_version, event_id)`；INSERT 衝突 → 視為重複事件（丟棄、ack bus）
- INSERT 必須發生在 `input_mapping` 求值**前**（詳見 `dsl-spec/02-workflow.md` Trigger 處理順序）；確保未通過 `when` 的 trigger 不占用 dedup slot
- GC：背景 job 以 `expires_at < now()` 清理；`expires_at` 不支援 API 延長
- Manual trigger 的 `Idempotency-Key` 查找以 `(workflow_name, workflow_version, idempotency_key)` 為 key，**邏輯上**可與此表共用儲存（將 `event_id` 欄位一般化為 `dedup_key`）；實作 MAY 分表以避免語意混淆

### outbox

「checkpoint 持久化先於 bus 投遞」的中介表；engine loop 於 checkpoint transaction 內 INSERT 事件，背景 publisher 於 DB commit 後讀取並投遞至 Event Bus：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `outbox_id` | bigserial / uuid | 主鍵；publisher 以此排序投遞 |
| `event_id` | uuid | 對應 `Event.id`（訂閱端以此去重） |
| `event_type` | string | 同 Event |
| `scope` | enum | `workflow` / `project` / `global` / `internal` |
| `delivery` | enum | `broadcast` / `unicast` |
| `target` | string \| null | unicast 時的 instance_id |
| `data` | jsonb | |
| `source_instance` | uuid | 同 `Event.source.instance_id` |
| `source_step_id` | string \| null | |
| `source_sequence` | int64 | |
| `trace_id` | string | |
| `committed_at` | timestamp | DB commit 完成時間（publisher 可用以等待 durability ACK；見 Checkpoint Durability 章節） |
| `published_at` | timestamp \| null | publisher 投遞成功時間；NULL 代表尚未投遞 |
| `attempts` | int | 投遞嘗試次數；超過實作上限（建議 5）→ 進 `bus.dead_letter` 並保留此 row 供 audit |
| `last_error` | jsonb \| null | 最後一次投遞失敗的 error |

- Publisher 以 `WHERE published_at IS NULL ORDER BY outbox_id ASC` 取得待投遞；投遞成功 UPDATE `published_at = now()`
- 已投遞的 row MAY 由背景 job 於 `published_at < now() - <retention>`（建議 24h）後清理
- 同一 `event_id` 不會被 INSERT 兩次（engine loop 於 emit 求值階段分配唯一 id）；publisher retry 以 `event_id` 對 bus 去重
- Delayed event（`emit.delay > 0`）**不**經 outbox；直接寫入 `delayed_events`（見上表），到期由搶取機制投遞至 bus

### bus_redelivery_log

Dead letter 重投遞的 audit trail（見 `07-event-bus.md` Dead Letter 重投遞規則）：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `log_id` | bigserial / uuid | 主鍵 |
| `original_event_id` | uuid | 被重投遞的原 `event.id` |
| `new_event_id` | uuid | 重投遞產生的新 `event.id` |
| `redelivered_at` | timestamp | 重投遞時刻 |
| `operator` | string | 觸發重投遞的 ops 操作者 identity（API token / user id） |
| `reason` | string \| null | 人類可讀備註（MAY） |

- 不做自動清理（audit 用途）；實作 MAY 依保存期政策歸檔
- 寫入時機：重投遞 API 於產生新 event 並入 outbox 的同一 transaction 內 INSERT

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
| `reusable_across_restart` | bool | tool 定義的 `lifecycle.init.reusable_across_restart`；engine crash 後是否復用 |
| `init_engine_id` | string | 建立此 init_output 的 engine 進程 ID（restart detect 用） |

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

### Checkpoint Durability 要求

Outbox publisher **MUST 僅於 checkpoint transaction 的 durability ACK 到達後**才投遞事件：

| Deployment | Durability ACK 定義 |
|-----------|--------------------|
| 單機 SQLite / PostgreSQL | `fsync` 完成（WAL + data file） |
| PostgreSQL 主從（同步複寫） | 主確認 + 至少一個同步 replica 確認；由 `synchronous_commit` 決定 |
| PostgreSQL 多主 / Aurora | 依該平台對「commit 成功」的語意（通常 quorum write） |
| 雲端 managed DB（Cloud SQL / RDS） | API 2xx return == durability ACK |
| 嵌入式 / in-memory（僅測試） | 視為即時 durable（僅供測試；生產不允許） |

此規則保證：訂閱 `step.completed` / `instance.completed` / 其他 internal event 的消費者（另一 engine 進程、另一 instance 的 wait）讀到事件時，**對應 checkpoint 已可被任意副本讀取**，不會發生「事件先到、checkpoint 未可見」的 race。

Outbox publisher 若因 DB 重啟 / 網路延遲導致 ACK 晚於 commit：

- publisher 沿用 outbox 表內 `committed = true, published = false` 的紀錄重投（冪等，訂閱端以 `event.id` dedup）
- Commit 成功但 ACK 未到達（例如 DB 故障切換）→ publisher 重啟後 **仍可讀取 outbox row**（commit 已確認）；投遞恢復
- 故 engine loop 的「發 `step.completed` 事件」以「INSERT outbox + DB commit」為邊界；實際 bus publish 由背景 publisher 接手

### Checkpoint Transaction 邊界

單一 transaction 內 MUST 包含（依 step transition 實際涉及者）：

| 表 | 是否在 checkpoint transaction 內 | 備註 |
|----|--------------------------------|------|
| `instances` | MUST | state / substate / vars / lease_expires_at |
| `steps` | MUST | 當前 step 的 state / output / error |
| `execution_log` | MUST | `step.transition` 紀錄 |
| `wait_subscriptions` | MUST | 該 step 若為 wait，新增或刪除 subscription |
| `delayed_events` | MUST | 該 step 若為 emit with delay，INSERT |
| `resource_pool`（init output） | MAY（異步寫） | init 完成後寫入；不阻塞 step transition |
| `outbox`（待投遞至 Event Bus 的即時事件） | MUST INSERT | commit 後由背景 publisher 投遞；schema 見上 |
| `trigger_dedup` | MUST INSERT | Event trigger 建立 instance 前、instance INSERT 前 |

**Cascading 規則**：

- Instance 終結（CANCELLED / FAILED / SUCCEEDED）的 transaction 內 MUST 同步 DELETE 該 instance 的所有未終結 `wait_subscriptions` 與 `delayed_events`（claimed_by IS NULL 者）
- `resource_pool` 的 rows **不在** 終結 transaction 內刪除；destroy hook 於 checkpoint commit 之後 best-effort 執行（見 `02-instance-lifecycle.md`），執行結束（成功或失敗）由 resource pool 背景 job DELETE 該 row。未成功 destroy 的 row MAY 保留於清理重試佇列，直至重試上限後刪除並記錄 `lifecycle.destroy_abandoned` observability 事件
- Isolation level SHOULD 為 `READ COMMITTED` 或以上；Lease UPDATE 本身以 atomic 條件 WHERE 達成互斥，不依賴 SERIALIZABLE
- 跨 instance 的操作（如一個 instance 的 emit 喚醒另一 instance 的 wait）**不在同一 transaction**；投遞由 outbox publisher 異步完成，訂閱者以 `event.id` 去重

---

## Lease 機制

lease 透過樂觀鎖：

```sql
UPDATE instances
SET    lease_owner = $engine_id, lease_expires_at = now() + 30s
WHERE  id = $instance_id
  AND  (lease_owner IS NULL OR lease_expires_at < now() OR lease_owner = $engine_id)
RETURNING id, lease_owner, lease_expires_at, cancel_requested, cancel_reason, state, substate;
```

- 更新成功（`RETURNING` 有結果）的 engine 進程持有 lease；其他進程回 0 rows，視為搶取失敗
- PENDING instance 剛建立時 `lease_owner IS NULL`，第一個執行 UPDATE 成功的 engine 取得 lease
- `RETURNING cancel_requested, cancel_reason` 用途：lease 首次搶取與每次 renew 皆於同一 UPDATE 中讀回這兩欄位，避免「搶到 lease 後才發現已被 cancel」需額外 query 的 race。若 `cancel_requested = true` → engine 直接進入 CANCELLED 流程，不執行原計劃的 step transition（與 `10-concurrency.md` Lease vs cancel 章節一致）
- 每次 checkpoint 於同一 transaction 內延展 lease；lease 延展與 state 變動原子化
- Engine 進程 crash → lease 自然過期（不需額外釋放動作）；其他 engine 進程在下次 polling / `instance.created` 事件到達時搶到

### engine_id 規範

`lease_owner` / `delayed_events.claimed_by` / `resource_pool.init_engine_id` 皆儲存 `engine_id` 字串，代表 engine 進程身分：

- 格式：`^[a-zA-Z0-9][a-zA-Z0-9._-]{0,127}$`，長度 ≤ 128 chars
- 唯一性：engine 進程啟動時 MUST 產生**全域唯一**的 engine_id；推薦格式 `<hostname>-<pid>-<boot_uuid_short>`，其中 `boot_uuid_short` 為進程啟動時產生的 uuid v4 前 8 碼，避免同主機同 pid 重用導致 lease 誤認
- 不得重用：進程重啟 MUST 產生新 engine_id，即使 pod name / hostname 相同（否則重啟後可能續用前 crash 進程遺留的 lease 而誤認為「同一持有者」）
- 寫入此欄位的 SQL MUST 於 engine config 初始化時一次性決定並緩存，避免每次 UPDATE 重算
- **絕不使用** SELECT 後 UPDATE 的兩步驟模式；MUST 以單一 atomic UPDATE ... WHERE 條件確保互斥

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
- `idempotency_key` = **SHA256** hex of `instance_id || "\0" || step_id || "\0" || canonical_json(input_snapshot) || "\0" || attempt_as_decimal_string`（`\0` 為 NUL byte 分隔；`canonical_json` 為鍵遞增排序後的 JSON serialization，與 signature 同規格，見 `03-step-execution.md`）；**含 attempt**，每次 retry 變動
- `input_snapshot` 定義為「step 首次進入 RUNNING 時寫入 checkpoint 的 input 物件」；後續 retry 沿用同一 snapshot（見 `03-step-execution.md` 的 attempt 與 signature 語意）；不含 `when` / `retry.delay` / `timeout` 等控制欄位的求值結果。

### idempotency_key vs signature：用途區分

| 識別子 | 包含欄位 | 用途 | 接收者 |
|--------|---------|------|--------|
| `signature`（engine 內部） | action_name + input_snapshot + attempt | Engine 做 retry cache lookup（lease 切換後避免重跑已 SUCCEEDED 的 attempt） | Engine 內部 |
| `idempotency_key`（tool context） | instance_id + step_id + attempt + input_snapshot | Tool 對**外部系統**去重（如同 Stripe API 的 Idempotency-Key） | 傳至 tool，由 tool 自行使用 |

- 兩者**皆含 attempt**；重試時 key 變動，tool 視為新請求；外部系統若希望跨 retry 共用結果，需自行以 (instance_id + step_id) 部分為 key
- Tool 若宣告 `idempotent: true`：engine 於 retry 前查 `signature` cache；**tool 仍每次收到不同的 `idempotency_key`**，因為 tool 通常需要對外部系統去重而非本地（此設計刻意讓 tool 能觀測到 attempt 變化）
- 若使用者需要「跨 attempt 穩定的 idempotency key」（如冪等付款請求），請於 CEL 中顯式生成：`idempotency_hint: ${ instance_id + ":" + step_id }` 作為 input 一部分，由 tool 讀取此欄位而非 context.idempotency_key

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
| `instance.input` | 16 MB | trigger 階段拒絕建立 instance，`error.type == "input_too_large"`；事件 ack 消費並進 dead letter |
| `steps.input_snapshot` | 16 MB | step FAILED，`error.type == "input_too_large"`，error.details.size 保留原始大小 |
| `steps.output` | 16 MB | step FAILED，`error.type == "output_too_large"`；原 output 不入庫，error.details.preview 保留前 4 KB |
| `instance.output` | 16 MB | instance FAILED，`error.type == "output_too_large"` |
| `instances.vars`（整體序列化） | 16 MB | assign step FAILED，`error.type == "vars_too_large"`；寫入前檢查 |
| 單筆 `execution_log.payload` | 1 MB | 超限則截斷尾部，log 記錄 truncation 標記 |
| 單一 event.data（Event Bus） | 1 MB | emit step FAILED，`error.type == "event_too_large"` |

上限可由 engine config 覆寫（`engine.max_step_output_bytes` / `engine.max_instance_input_bytes` / `engine.max_vars_bytes` 等）。超大資料建議以 artifact 儲存並在 output 中傳 path 引用。

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

### Non-deterministic 函式記錄

`now()` / `uuid()` 的結果寫入 execution_log，key 為：

```
(instance_id, step_path, attempt, call_index)
```

- `attempt`：每次 retry attempt 獨立；**retry 重新求值會產生新值**（`now()` 更新、`uuid()` 重新生成）
- `call_index`：同一 expression 內多次呼叫（如 `${ uuid() + "-" + uuid() }`）的序號
- Replay 時按 key 讀回對應值；若某 attempt 在 log 中不存在（例如未被持久化的 attempt）則回報 `replay_missing_entry`
- 實務建議：`now()` / `uuid()` 在 catch handler 內使用時視為新 attempt 的一部分；handler 內重新求值取得新值

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
