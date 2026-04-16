# 07 — Event Bus

本文件定義 Event Bus 的訊息模型、scope、投遞語意、wait 解析與 emit 路由。對應 [dsl-spec/03-steps.md](../dsl-spec/03-steps.md) 的 `wait` / `emit` 與 trigger event。

---

## 角色

Event Bus 是引擎與外部、引擎內部多 instance 間的非同步通訊匯流排：

- 業務事件：由 `emit` step 發送，可觸發 trigger 或喚醒 wait
- 內部事件：`instance.created` / `step.completed` / `tool.stream` / `tool.callback` / `wait.timeout` 等
- 控制事件：`instance.cancel` / `engine.lease_expired`

實作可選：in-memory bus（單機）、基於 message queue（NATS / Kafka / Redis Streams）、基於資料庫 polling。本規格不限定，僅約束行為。

---

## 事件物件

```
Event {
  id:         string,        # UUID v4
  type:       string,        # 業務或內部 type，dotted snake_case
  scope:      "workflow" | "project" | "global" | "internal",
  source:     {
    instance_id: string,
    project:     string,
    step_id:     string | null,
    sequence:    int64,      # 單調遞增序列號，per source.instance_id；保證同源全序
  },
  data:       any,           # 業務 payload；internal 事件可為 null
  timestamp:  ISO8601,       # μs 精度（RFC 3339 Nano）；僅作觀測，排序用 sequence
  delivery:   "broadcast" | "unicast",   # broadcast: 所有訂閱者；unicast: 指定接收者
  target:     string | null, # unicast 時的 instance_id
  trace_id:   string,        # 跨 instance 追蹤；見「trace_id 傳播」
}
```

- `source.sequence` 由 engine 在 emit 時分配（從 instance 的 vars 或專用計數器讀取 + 1）；持久化於 instance state，重啟後不重複
- 跨 instance 排序無保證；bus 消費者依 `(source.instance_id, source.sequence)` 做同源全序，依 `timestamp` 做觀測性展示

### source 於特殊上下文的歸屬

| 發生位置 | `source.instance_id` | `source.step_id` |
|---------|----------------------|------------------|
| Workflow / Function 一般 step（`emit`） | 所在 instance | emit step 自身 id |
| Callback handler 內的 `emit`（caller 側 `type: task` 的 `callback:`） | **caller workflow instance**（非 function instance） | 呼叫該 tool/function 的 task step id（handler 本身非正式 step） |
| Saga compensate 觸發的 tool 若 emit | 所在 saga instance | 原 origin step 的 id（compensate step 本身不暴露 id） |
| Tool stream / tool.callback internal 事件 | 呼叫 tool 的 instance | 呼叫 tool 的 task step id |
| Trigger 建立 instance（內部 `instance.created`） | 新建立的 instance | `null`（無觸發 step） |

此規則確保事件追蹤鏈能以 `(instance_id, step_id)` 精準定位業務來源，不因跨 context（callback / compensate）而模糊化。`source.sequence` 於 callback / compensate 期間仍由 caller instance 的計數器分配（共用單一序列）。

---

## Scope 與訂閱範圍

| Scope | 可被誰接收 |
|-------|-----------|
| `workflow` | **僅發送它的同一 instance** 內的 wait（包含 self-emit 場景）；不出實例 |
| `project` | 同一 project 內任何 workflow 的 trigger 或 wait（含發送者自己） |
| `global` | 任意 project 的 trigger 或 wait（含發送者自己） |
| `internal` | 引擎內部訂閱者；不可被 trigger 引用 |

`emit` step 的 scope 屬性決定事件路由範圍；`scope: workflow` 的事件**不會**離開該 instance 的虛擬隔離域，不會送至其他 instance 的 wait。

**`scope: workflow` 於 function instance 的嚴格隔離**：function instance 內 emit `scope: workflow` 的事件**僅限該 function instance 本身**，**不會**傳播至父 workflow instance 或兄弟 function instance（即每個 instance 為獨立 workflow-scope 虛擬域）。同理，父 workflow instance 的 `scope: workflow` 事件也不會下傳至子 function instance。跨 instance 通訊請使用 `scope: project` 或 `global`，並以 correlation key（如 `parent_id` / 業務 id）於 `match` 中辨識。此設計避免語意歧義，並保留「workflow = 單一 instance 的隔離域」的簡潔心智模型。

**根目錄 definition 的 project 匹配**：匹配「同 project」時以 `project.name` 字串比較（根目錄 definition `project.name == ""`）；因此根目錄 definition emit 的 `scope: project` 事件只匹配其他根目錄 definition 的訂閱者，不會匹配具名 project 內的訂閱者，反之亦然（詳見 `dsl-spec/03-steps.md` emit 節）。

**Self-emit（同 instance 自發自收）**：

- `scope: workflow` 的 self-emit 走 engine 內的 short-circuit path（不進 Event Bus 外部隊列），直接路由至該 instance 的 wait subscription
- 仍保持「先 checkpoint 再喚醒」的順序：emit step SUCCEEDED 寫入 checkpoint 後，engine loop 才投遞至 wait
- `scope: project` / `global` 的 self-emit 經完整 bus 路徑；投遞時序可能晚於同 step 的後續 step，使用者不應依賴 self-emit 的即時性

`scope: project` 與 `global` 事件可同時喚醒多個訂閱者；每個訂閱者收到該事件的獨立副本。

---

## 投遞語意

- **At-least-once**：每個訂閱者至少收到一次該事件；可能因網路或 engine 進程重啟而重複。
- **訂閱者責任**：以 `event.id` 做去重（trigger / wait subscription 維護「已處理 event id」表，TTL = 訂閱 deadline）。
- **順序**：同 `source.instance_id` 內以 `source.sequence` 升序投遞；跨 source 不保證。實作 bus（NATS/Kafka/Redis）MAY 依 key=source.instance_id 做 partition 以維持同源順序。
- **延遲**：`emit.delay > 0` 時事件先入延遲 queue；deadline 到才實際投遞。
- **死訊**：訂閱者持續處理失敗（如連續超過 `max_redelivery`，建議 5 次）→ 進入 dead-letter；引擎發 `bus.dead_letter` internal 事件可被 ops 監控。

#### delivery / target 的使用規則

| 情境 | `delivery` | `target` | 備註 |
|------|-----------|---------|------|
| Business emit（預設） | `broadcast` | null | 所有匹配 trigger / wait 的訂閱者皆收到 |
| Internal `step.completed` / `wait.timeout` / `event.matched` | `unicast` | 目標 instance_id | Engine Loop 只路由至該 instance；避免廣播雜訊 |
| Internal `tool.callback` | `unicast` | caller instance_id | 走 driver 直通；不發至 bus broadcast |
| Internal `tool.stream` | `broadcast`（內部） | null | 供 workflow 的 wait 訂閱 |
| Internal `instance.cancel` | `unicast` | target instance_id | |

- `emit` step 目前**僅支援 broadcast**（v6）；DSL 無 unicast 欄位。unicast 是 engine 內部路由機制，不暴露給使用者業務邏輯
- Internal unicast event 若 target instance 不存在（已 GC）→ 事件進 `bus.dead_letter`，`failure_reason: "instance_not_found"`

#### bus.dead_letter 事件結構

```
{
  type: "bus.dead_letter",
  scope: "internal",
  data: {
    original_event: Event,          # 完整原事件物件
    subscription_id: string | null, # 觸發死訊的 subscription（若有）
    target_instance_id: string | null,
    failure_reason: string,          # "handler_timeout" / "instance_not_found" / "invoke_failed" / ...
    last_error: ErrorObject | null,  # 最後一次投遞失敗的 error
    attempts: int,                   # 已嘗試投遞次數
    first_attempted_at: ISO8601,
    last_attempted_at: ISO8601,
  }
}
```

Ops 監控 SHOULD 對 `bus.dead_letter` 做告警；實作 MAY 提供「重新投遞 dead letter」API。

### Dead Letter 重投遞規則

若實作提供重投遞 API（如 `POST /dead-letters/<id>/redeliver`）：

- 重投遞 MUST 產生**新的 `event.id`**（以新 UUID v4），不沿用原 `original_event.id`
- 重投遞事件的 `data` / `type` / `scope` 可與原事件相同；但 `event.id` 不同，因此：
  - 不與已投遞至訂閱者的去重表衝突（原 event.id 可能已記錄）
  - 訂閱者視為新事件處理，業務層若需冪等需自行在 `data` 攜帶 correlation key
- `source.instance_id` 若對應的 source instance **已清理**（超過 retention）→ 重投遞 API 回傳 `409 source_instance_gone`，拒絕重投（避免 orphan 事件）
- 重投遞事件的 `source` 保留原值（便於追蹤）；`timestamp` 為重投遞時刻（非原時刻）
- 重投遞歷次寫入 `bus.redelivery_log` 表（`original_event_id`、`new_event_id`、`redelivered_at`、`operator`）供 audit

---

## Wait subscription

`type: wait` 的 `signals` 中事件訊號建立 subscription（step 訊號透過 `step.completed` 事件訂閱處理，不經此路徑）：

```
WaitSubscription {
  subscription_id: uuid,
  instance_id:     string,
  step_id:         string,
  patterns: [
    {
      event_type: string,
      scope:      "workflow" | "project" | "global" | null,  # null = 不限 scope
      match_expr: CEL | null,
    }
  ],
  deadline:        timestamp,    # 由 wait.timeout 計算；一次性決定、寫入 checkpoint
  any_of:          bool,         # signals 模式下恆為 true（任一匹配即喚醒）
  created_at:      timestamp,
}
```

訂閱寫入 Instance Store 的 wait_subscriptions 表（持久化），同步於 Event Bus 的訂閱者註冊（in-memory）。

**Deadline 決定時機**：

- `wait.timeout` 求值發生在 step 進入 RUNNING 前、與其他欄位（`signals[].match`）同一批次；若求值錯誤 → step 直接 FAILED，`error.type == "expression_error"`（不建立 subscription）
- `deadline = <subscription 寫入 store 的 transaction commit 時刻> + <timeout 求值結果>`；**一次性決定**，寫入 checkpoint 後不再重算
- Engine 重啟 / failover 後 MUST 沿用 checkpoint 中的 `deadline`，即使 `wait.timeout` 是 CEL 表達式也不重新求值（避免 replay 與線上執行的偏差）
- 若缺省 `wait.timeout` → `deadline = null`，表示無限等待（僅 instance / workflow timeout 可終止）
- 若 `timeout` 求值為 `0` 或負值 → step 在訂閱寫入前 FAILED，`error.type == "invalid_duration"`（與 `wait.duration` / `retry.delay` 等其他 duration 欄位一致）

**Subscription 建立與 step signal 的競賽**：

- 若 `signals` 內含 step 訊號且目標 async step **已終態**：走 fast path（見 `03-step-execution.md` 的 `type: wait`），不建立 subscription
- 若目標 async step **尚未啟動**（PENDING，例如 wait 在 parallel 的另一 branch 中先執行）：`step.completed` 事件仍以 event bus 訂閱方式等待，subscription 於其 `deadline` 到期前保留
- 若目標 step **不存在於 workflow definition**（載入驗證已拒絕）：此情境不會進入執行期；但若 definition 重載替換後 step 消失，engine 以 `error.type == "invalid_signal_target"` 讓 wait step 立即 FAILED

**Subscription 清理規則**：

- 匹配喚醒後：subscription 立即刪除（any_of 模式喚醒時同時刪除同 wait step 的其他 subscriptions）
- Instance 終結（SUCCEEDED / FAILED / CANCELLED）：關聯的 wait_subscriptions 在 instance checkpoint transaction 內同步刪除，避免孤立訂閱
- Deadline 過期：由背景 GC job 掃描並刪除（實作 SHOULD 每分鐘掃一次；deadline 觸發 wait.timeout 時也順便刪）
- 事件投遞至已不存在的 instance（因 GC race）：bus 記錄 warning 並丟棄，不發 `event.matched`

匹配流程：

```
def deliver(event, subscription):
    for pattern in subscription.patterns:
        if event.type != pattern.event_type:
            continue
        if pattern.scope is not None and event.scope != pattern.scope:
            continue
        if pattern.match_expr:
            ctx = build_event_match_ctx(event, subscription.instance)
            try:
                if not evaluate(pattern.match_expr, ctx):
                    continue
            except ExpressionError:
                log_warn(...)
                continue
        # match
        publish_internal_event("event.matched", {
            instance_id: subscription.instance_id,
            step_id: subscription.step_id,
            subscription_id: subscription.id,
            event: event,
        })
        if subscription.any_of:
            cancel_subscription(subscription)
        return
```

`build_event_match_ctx(event, instance)` 提供 `event.*`、`input`、`vars`、`steps` 等 namespace（read-only）。

### 多訂閱者投遞（foreach / parallel 並行 wait）

Event bus 對每個 `wait_subscription` 是**獨立投遞**：

- 同一事件 `e` 若同時匹配 N 個 subscription（如 foreach 10 個迭代，每迭代內有 `wait`，全部訂閱同一 `event_type`），**N 個 subscription 皆各自收到一次**；不會「第一個搶走」
- 「**any-of 同 wait step 內其他 signal 取消**」規則（上述 `cancel_subscription(subscription)`）**僅限於同一 subscription 的兄弟 pattern**（即同一 wait step 的 signals 陣列）；不影響其他 wait step（即使其屬於同一 instance 的不同 step，或其他 instance / 其他迭代）
- 各 subscription 的 `match` 表達式各自求值；匹配結果互不影響；即使 match 為空（無過濾），N 個並行迭代的 wait 仍各收到一份並各自決定 SUCCEEDED
- 推薦做法：foreach 內的 wait 以 `match` 綁定該迭代的關鍵識別（如 `${ event.data.order_id == loop.item.order_id }`），讓每迭代只對「屬於它的」事件喚醒；否則多迭代同時被同一事件喚醒是合法但可能非預期的行為

**去重**：同一事件重投遞至同一 subscription 以 `(subscription_id, event.id)` 去重（見「Idempotency」）；不同 subscription 間無去重（屬各自獨立訂閱者）。

---

## Trigger

Trigger Resolver 維護 trigger subscription（不持久化於 instance level，由 workflow definition 決定）：

```
TriggerSubscription {
  workflow_name:     string,
  workflow_version:  int,
  trigger_index:     int,
  event_type:        string,
  match_expr:        CEL | null,
  input_mapping:     map<string, CEL>,
}
```

事件匹配時：

```
1. 求值 match_expr
   - 異常（identifier / type 錯誤）→ 事件視為不匹配；記錄 warning metric
2. 求值 input_mapping → 構造 instance input
   - 個別欄位求值異常 → 該欄位值為 null（由 schema validation 決定是否接受）
3. 驗證 input 符合 workflow.input_schema
   - 失敗 → trigger reject；**不建立 instance**；發內部事件 `trigger.rejected` 含 {workflow_name, trigger_index, event_id, violation_path}
   - 此錯誤不向業務事件來源回報（event 已進 bus）；由 ops 監控 `trigger.rejected` 事件
4. 建立新 workflow instance（PENDING）
5. 發 instance.created 事件
```

對 `manual` trigger：由 API / CLI 明確呼叫；不訂閱 Event Bus。input_schema 違反時 API **同步**回傳 4xx + `workflow.input_schema_violation`，呼叫端知錯；不建立 instance。

#### Manual trigger idempotency

API 觸發 workflow 時 MAY 帶 `Idempotency-Key` header（建議值：UUID 或業務唯一 key）：

- Engine 以 `(workflow_name, workflow_version, idempotency_key)` 為複合鍵查找既有 instance
- 命中（TTL 內，預設 24h）：回傳既有 instance（不建立新的），HTTP 200 + instance state；同 key 重複呼叫冪等
- 未命中：建立新 instance，同時記錄 idempotency_key → instance_id 映射，TTL 24h

**無 `Idempotency-Key` 時**：每次呼叫皆建立新 instance；呼叫端自行承擔重複風險（例如網路超時重試會產生多個 instance）

實作 SHOULD 暴露 `manual_trigger_idempotency_ttl` config 覆寫 24h 預設。

#### Trigger 效能與索引

高事件吞吐場景（10K+ events/s × 數千 trigger subscription）不能對每事件 O(N) 全掃 CEL：

- Trigger Resolver MUST 以 `event_type` 建立 index（hash map），事件到達時 O(1) 取回候選 triggers
- 候選 triggers 的 `when` CEL MUST 在載入期預編譯（parse + type check）；執行期只做 evaluate
- `when` 求值異常（identifier / type error）→ 該 trigger 視為不匹配（log warning，不拒絕事件；避免一個壞 trigger 癱瘓整個事件流）
- 建議實作再對 `when` 內的常量條件（如 `event.data.source == "api"`）做 partial evaluation，在 index 層額外分桶
- 監控指標：`trigger_eval_duration_ms{workflow}` histogram

---

## emit step

```
1. 求值 event 名稱、scope、data、delay
2. 構造 Event 物件，trace_id 繼承自 instance（若無則新生）
3. 若 delay > 0 → 寫入延遲 queue（持久化）；否則 publish to Event Bus
4. step SUCCEEDED, output: { event_id }
```

emit 的順序保證：

- 同一 step 順序執行 emit → 在 bus 中 timestamp 保證升序
- 不同 step 並行 emit → 順序由 bus 與 wall-clock 決定，無強保證

---

## tool.stream / tool.callback 事件

- `tool.stream`：由 Tool Backend Driver 在 `mode: stream` 下發出；scope: `internal`；data: stream payload。
- workflow 可訂閱 `tool.stream` 事件透過 `wait` 取得資料（將 event_type 設為 `tool.stream` 並 match）。
- `tool.callback`：driver 內部使用，不暴露給使用者 wait。

---

## 內部事件目錄

| Type | 發送者 | 訂閱者 | 用途 |
|------|--------|--------|------|
| `trigger.rejected` | Trigger Resolver | Ops 監控 | input_schema 驗證失敗，instance 未建立 |
| `instance.created` | Trigger Resolver | Engine Loop | 開始執行 |
| `instance.completed` | Engine Loop | Engine Loop / 父 instance | 子 instance 終結通知 |
| `instance.failed` | Engine Loop | 同上 | |
| `instance.cancelled` | Engine Loop / API | 同上 | |
| `instance.cancel` | 外部 API / 父 instance / Engine Loop | Lease 持有 Engine Loop | 唯一控制型 internal event（unicast，見 scope: internal 路由）；通知 lease 持有者讀取 `instance.cancel_requested` 並進入 CANCELLED 流程 |
| `step.completed` | Engine Loop | Engine Loop | 推進下一步 / 喚醒 wait signals 中的 step 訊號 |
| `step.async_started` | Engine Loop | Engine Loop | foreach/parallel async 模式 |
| `event.matched` | Event Bus | Engine Loop | 喚醒 wait step |
| `wait.timeout` | Engine Loop（timer） | Engine Loop | 觸發 wait step FAILED |
| `tool.callback` | Backend Driver | Engine Loop | 路由至 caller handler |
| `tool.stream` | Backend Driver | Engine Loop / 訂閱 wait | 串流投遞 |
| `engine.lease_expired` | Lease Manager | Engine Loop | 觸發 instance 接管 |
| `bus.dead_letter` | Event Bus | Ops 監控 | 訊息無法投遞 |
| `trigger.skipped` | Trigger Resolver | Ops 監控 | event 通過 dedup 但未匹配任一 trigger 的 scope / when；或 trigger.when_eval_failed（event ack 但不建立 instance） |
| `lifecycle.destroy_failed` | Resource Pool | Ops 監控 | destroy hook 單次失敗；含 `{instance_id, resource_key, error, attempt}`；排入重試佇列 |
| `lifecycle.destroy_abandoned` | Resource Pool | Ops 監控 | destroy hook 重試耗盡後放棄；含 `{instance_id, resource_key, error, attempts}` |
| `tool.protocol_anomaly` | Backend Driver | Ops 監控 | 協議違反但 step 不失敗（unknown call_id / 遲到訊息等）；含 `{instance_id, step_id, reason, ...}` |
| `callback.late_result` | Engine Loop | Ops 監控 | callback_result 於 step 已終態後送達；丟棄並記錄 |
| `subscription.match_eval_failed` | Event Bus | Ops 監控 | wait subscription 的 match CEL 求值異常；視為不匹配、繼續等待 |

### 內部事件訂閱限制

所有 `scope: internal` 的事件 **僅 engine 與 ops 監控可訂閱**；使用者 workflow 的：

- `wait.signals[].event`：載入期拒絕保留前綴（見 `dsl-spec/03-steps.md` 的 event 名稱規則）；**唯一例外** 為 `tool.stream`，允許 workflow 訂閱以接收本 instance tool 的 stream 訊息
- Event trigger：不含例外；`tool.stream` 為 internal scope，trigger 不能訂閱 internal 事件
- `emit.event`：同上保留規則（使用者不能 emit 保留 type，含 `tool.stream`）

**`tool.stream` 的訂閱作用域**：wait 訂閱 `tool.stream` 時以 `match` 表達式辨識來源（如 `match: ${ event.source.step_id == "my_task" }`）；`tool.stream` 事件的 `source.instance_id` 恆為呼叫 tool 的 instance 自身（見「source 於特殊上下文的歸屬」），因此跨 instance 不會相互干擾。

業務層若需要「instance 完成」事件，建議 workflow 主動 `emit workflow.completed`（使用者自訂 type）。

### 主要 internal event 的 data payload

| Event type | `data` 欄位 |
|-----------|-------------|
| `instance.created` | `{instance_id, workflow_name, workflow_version, triggered_by_trigger_index, input_summary (前 4 KB)}` |
| `instance.completed` | `{instance_id, kind, output_summary (前 4 KB), duration_ms}` |
| `instance.failed` | `{instance_id, kind, error: <完整 ErrorObject>, duration_ms, owner}` |
| `instance.cancelled` | `{instance_id, kind, reason, requested_by ("api" \| "parent" \| "timeout" \| "ops")}`（與 `instance.cancel.requested_by` / `instances.cancel_reason` 欄位語意一致） |
| `instance.cancel` | `{target_instance_id, reason, requested_by ("api" \| "parent" \| "timeout" \| "ops"), requested_at}` — unicast 至目標 instance 的 lease 持有者；不由 ops 常態訂閱 |
| `step.completed` | `{instance_id, step_id, step_path, status, attempt, duration_ms}`（不含 output，讀 checkpoint） |
| `step.async_started` | `{instance_id, step_id, step_path, total_iterations (foreach 才有)}` |
| `trigger.rejected` | `{workflow_name, trigger_index, event_id, failure_reason, violation_path?}` |
| `bus.dead_letter` | 見上「bus.dead_letter 事件結構」 |

實作 SHOULD 將 data 限制在 4 KB 以內（超過請以 `*_summary` 前綴的摘要欄位傳遞）；完整狀態請訂閱者依 `instance_id` 讀取 Instance Store。

---

## trace_id 傳播

所有 instance 與 event 皆帶 `trace_id`，以維持跨元件追蹤鏈：

| 產生/繼承時機 | trace_id 來源 |
|----------------|----------------|
| Manual trigger | API caller 提供 W3C traceparent header → 解析其 trace-id；無則生成 UUID v4 |
| Event trigger | 繼承來源 event 的 `trace_id`（若無則新生） |
| Function instance / sub-instance | 繼承父 instance 的 `trace_id` |
| `emit` 產生的 event | 繼承當前 instance 的 `trace_id` |
| Internal event（step.completed 等） | 繼承當前 instance 的 `trace_id` |
| Tool backend HTTP 請求 | engine MAY 注入 `traceparent` header（基於 instance.trace_id + 新 span_id） |

確保單一觸發到所有衍生 instance / event 共享同一 `trace_id`，observability 系統可以其串接完整執行鏈。

---

## 順序與一致性

- **單 instance 內**：所有事件處理串行化（同進程：event loop 天然串行；跨進程：lease 保證）；step.completed 與其後的 step 啟動原子化。
- **多 instance 之間**：完全 concurrent，無順序保證；instance.completed 的接收順序與發送順序可能不同。
- **Trigger 競爭**：同一事件可能匹配多個 workflow 的 trigger；每個 trigger 各自建立 instance（不互斥）。

---

## Idempotency

- **Trigger**：以 `(workflow_name, workflow_version, event.id)` 去重；同一事件不重複建立 instance。
- **Wait 投遞**：以 `(subscription_id, event.id)` 去重；同一事件不重複喚醒同一訂閱。
- **emit 重試**：若 publish 失敗，引擎 retry；以 `event.id` 去重，bus 只接受第一次。

### 去重紀錄的持久化

所有去重 key MUST 持久化（而非僅 in-memory），避免 engine crash 後重複投遞：

| 粒度 | 存放表 | TTL | crash 後行為 |
|------|--------|-----|---------------|
| Trigger 去重 | `trigger_dedup`（獨立表）key = `(workflow_name, workflow_version, event.id)` | 24h（可由 engine config 覆寫） | 重啟後沿用；同 event.id 不再建立 instance |
| Wait 投遞去重 | 與 `wait_subscriptions` 同表的 `delivered_event_ids` 陣列欄位（或子表） | 跟 subscription deadline；deadline 到或 subscription 喚醒則整批刪除 | 重啟後沿用；事件重投遞不喚醒已處理 subscription |
| emit 去重 | Bus 實作層內部（如 NATS dedupe window / Kafka key compaction） | 實作決定；建議 ≥ publish retry 上限的時窗 | Bus 層責任 |

### 多訂閱者一事件的處理

同一事件（同 `event.id`）可能同時匹配：
1. 多個 workflow 的 trigger
2. 多個 instance 的 wait subscription（如 project-scope broadcast）
3. 自身的 trigger + 他 instance 的 wait

規則：

- 每個訂閱者 **獨立去重**；trigger 表與 wait 表互不共用去重記錄
- 若 bus 以 `event.id` 做 sender-side dedup（emit 重試），只是抑制 publish，不影響接收端獨立去重
- 一個訂閱者一次接收同一 event 若失敗（handler FAILED） → 不計入該訂閱者去重記錄（下次重試仍需處理）；僅**成功處理**後才寫入去重表
- 去重寫入 MUST 與「訂閱者處理結果 commit」同一 transaction；避免「去重已寫但處理未 commit」導致事件永久遺失

---

## 關閉 / 排空

引擎關閉時（graceful shutdown）：

```
1. 停止接受新 instance（trigger resolver 暫停）
2. 等待當前 step 達 checkpoint 邊界
3. 釋放 lease；其他 engine 進程接管或保留至下一次 reload
4. 排空延遲 queue（已 due 的事件投遞完畢）
5. 寫入 shutdown checkpoint 並關閉
```
