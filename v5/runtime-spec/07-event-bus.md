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
  },
  data:       any,           # 業務 payload；internal 事件可為 null
  timestamp:  ISO8601,
  delivery:   "broadcast" | "unicast",   # broadcast: 所有訂閱者；unicast: 指定接收者
  target:     string | null, # unicast 時的 instance_id
  trace_id:   string,        # 跨 instance 追蹤
}
```

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
- **順序**：同 `(scope, source.instance_id)` 內以 timestamp 升序投遞；跨 source 不保證。
- **延遲**：`emit.delay > 0` 時事件先入延遲 queue；deadline 到才實際投遞。
- **死訊**：訂閱者持續處理失敗（如連續超過 `max_redelivery`，建議 5 次）→ 進入 dead-letter；引擎發 `bus.dead_letter` internal 事件可被 ops 監控。

---

## Wait subscription

`type: wait` 的 event / events 模式建立 subscription：

```
WaitSubscription {
  subscription_id: uuid,
  instance_id:     string,
  step_id:         string,
  patterns: [
    { event_type: string, match_expr: CEL | null }
  ],
  deadline:        timestamp,    # 由 wait.timeout 計算
  any_of:          bool,         # events 陣列模式為 true
  created_at:      timestamp,
}
```

訂閱寫入 Instance Store 的 wait_subscriptions 表（持久化），同步於 Event Bus 的訂閱者註冊（in-memory）。

匹配流程：

```
def deliver(event, subscription):
    for pattern in subscription.patterns:
        if event.type != pattern.event_type:
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
2. 求值 input_mapping → 構造 instance input
3. 驗證 input 符合 workflow.input_schema → 失敗則 reject 並發 alert
4. 建立新 workflow instance（PENDING）
5. 發 instance.created 事件
```

對 `manual` trigger：由 API / CLI 明確呼叫；不訂閱 Event Bus。

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
| `instance.created` | Trigger Resolver | Engine Loop | 開始執行 |
| `instance.completed` | Engine Loop | Engine Loop / 父 instance | 子 instance 終結通知 |
| `instance.failed` | Engine Loop | 同上 | |
| `instance.cancelled` | Engine Loop / API | 同上 | |
| `step.completed` | Engine Loop | Engine Loop | 推進下一步 / 喚醒 wait step 模式 |
| `step.async_started` | Engine Loop | Engine Loop | foreach/parallel async 模式 |
| `event.matched` | Event Bus | Engine Loop | 喚醒 wait step |
| `wait.timeout` | Engine Loop（timer） | Engine Loop | 觸發 wait step FAILED |
| `tool.callback` | Backend Driver | Engine Loop | 路由至 caller handler |
| `tool.stream` | Backend Driver | Engine Loop / 訂閱 wait | 串流投遞 |
| `engine.lease_expired` | Lease Manager | Engine Loop | 觸發 instance 接管 |
| `bus.dead_letter` | Event Bus | Ops 監控 | 訊息無法投遞 |

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
