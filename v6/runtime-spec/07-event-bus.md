# 07 — Event Bus

本文件定義 Event Bus 的訊息模型、scope、投遞語意、wait 解析與 emit 路由。對應 [dsl-spec/03-steps.md](../dsl-spec/03-steps.md) 的 `wait` / `emit` 與 trigger event。

---

## 角色

Event Bus 是引擎與外部、引擎內部多 instance 間的非同步通訊匯流排：

- 業務事件：由 `emit` step 發送，可觸發 trigger 或喚醒 wait
- 內部事件：`instance.created` / `step.completed` / `tool.stream` / `tool.callback` / `wait.timeout` 等
- 控制事件：`instance.cancel` / `engine.lease_lost`

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

---

## Scope 與訂閱範圍

| Scope | 可被誰接收 |
|-------|-----------|
| `workflow` | 同一 workflow instance 內的 wait / 內部處理 |
| `project` | 同一 project 內任何 workflow 的 trigger 或 wait |
| `global` | 任意 project 的 trigger 或 wait |
| `internal` | 引擎內部訂閱者；不可被 trigger 引用 |

`emit` step 的 scope 屬性決定事件路由範圍；`scope: workflow` 的事件**不會**離開該 instance 的虛擬隔離域，不會送至其他 instance 的 wait。

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
  deadline:        timestamp,    # 由 wait.timeout 計算
  any_of:          bool,         # signals 模式下恆為 true（任一匹配即喚醒）
  created_at:      timestamp,
}
```

訂閱寫入 Instance Store 的 wait_subscriptions 表（持久化），同步於 Event Bus 的訂閱者註冊（in-memory）。

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
| `step.completed` | Engine Loop | Engine Loop | 推進下一步 / 喚醒 wait signals 中的 step 訊號 |
| `step.async_started` | Engine Loop | Engine Loop | foreach/parallel async 模式 |
| `event.matched` | Event Bus | Engine Loop | 喚醒 wait step |
| `wait.timeout` | Engine Loop（timer） | Engine Loop | 觸發 wait step FAILED |
| `tool.callback` | Backend Driver | Engine Loop | 路由至 caller handler |
| `tool.stream` | Backend Driver | Engine Loop / 訂閱 wait | 串流投遞 |
| `engine.lease_expired` | Lease Manager | Engine Loop | 觸發 instance 接管 |
| `bus.dead_letter` | Event Bus | Ops 監控 | 訊息無法投遞 |

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
