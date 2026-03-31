# 99 — 完整範例集

本文件提供完整範例，示範 DSL v3 的各項功能。所有範例使用 v3 語法（`apiVersion: */v3`、`condition:` 取代 `expr:`、兩段式 tool 命名）。

---

## 範例 1：Task Definition（三種 backend）

### stdio — 訂單載入

```yaml
apiVersion: task/v3
kind: Task

metadata:
  name: order.load
  version: 3

input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

output_schema:
  type: object
  properties:
    id:
      type: string
    amount:
      type: number
    status:
      type: string

backend:
  type: stdio
  command: "node ./tools/order-tasks/load-order.js"
  config:
    db_pool: "primary"
```

### http — 付款建立

```yaml
apiVersion: task/v3
kind: Task

metadata:
  name: payment.create
  version: 2

input_schema:
  type: object
  properties:
    order_id:
      type: string
    amount:
      type: number
    currency:
      type: string
      default: "TWD"
  required: [order_id, amount]

output_schema:
  type: object
  properties:
    payment_id:
      type: string
    status:
      type: string

backend:
  type: http
  url: "https://api.payment.example.com/v1/charges"
  method: POST
  headers:
    Authorization: "Bearer ${ secret.PAYMENT_API_KEY }"
  timeout: 10s
  retry_on_status: [429, 502, 503]
```

### builtin — Echo（測試用）

```yaml
apiVersion: task/v3
kind: Task

metadata:
  name: system.echo
  version: 1

backend:
  type: builtin
  handler: echo
```

---

## 範例 2：基本 Workflow — 訂單處理

最簡單的 workflow：載入訂單、付款、回傳結果。

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: simple_order
  version: 1

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

steps:
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }
    retry:
      max_attempts: 3
      delay: 2s
      backoff: exponential
    timeout: 10s

  - type: assign
    vars:
      is_high_value: ${ steps.load_order.output.amount > 1000 }

  - type: if
    condition: ${ steps.load_order.output.status == "cancelled" }
    then:
      - type: fail
        message: "order has been cancelled"
        code: "order_cancelled"

  - id: create_payment
    type: task
    action: payment.create
    input:
      order_id: ${ steps.load_order.output.id }
      amount: ${ steps.load_order.output.amount }
    timeout: 30s

  - type: return
    output:
      order_id: ${ steps.load_order.output.id }
      status: "completed"
      payment_id: ${ steps.create_payment.output.payment_id }
```

---

## 範例 3：多層錯誤處理

展示 step-level、block-level、workflow-level 三層錯誤處理。

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: resilient_pipeline
  version: 1

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    job_id:
      type: string
  required: [job_id]

config:
  timeout: 1h
  # Workflow-level: 最後防線
  on_error:
    - type: task
      action: alert.send
      input:
        channel: "ops"
        message: ${ "pipeline failed: " + error.message }
      execution:
        policy: replayable

steps:
  - id: fetch_data
    type: task
    action: data.fetch
    input:
      job_id: ${ input.job_id }
    retry:
      max_attempts: 3
      delay: 5s
      backoff: exponential
    timeout: 60s
    # Step-level: 重試用盡後，記錄錯誤但繼續
    on_error:
      - type: assign
        vars:
          fetch_failed: true
          fetch_error: ${ error.message }

  - type: parallel
    failure_policy: wait_all
    # Block-level: 捕捉 parallel 內的錯誤
    on_error:
      - type: emit
        event: pipeline.partial_failure
        data:
          job_id: ${ input.job_id }
          error: ${ error.message }
    branches:
      - steps:
          - type: task
            action: transform.type_a
            input:
              job_id: ${ input.job_id }
            timeout: 5m
      - steps:
          - type: task
            action: transform.type_b
            input:
              job_id: ${ input.job_id }
            timeout: 5m

  - type: return
    output:
      job_id: ${ input.job_id }
      status: ${ default(vars.fetch_failed, false) ? "partial" : "complete" }
```

---

## 範例 4：Agent Definition — 訂單風險分析

定義一個訂單風險分析 agent，使用 model alias、toolset 引用 skills、兩段式 tool 命名。

```yaml
apiVersion: agent/v3
kind: Agent

metadata:
  name: order.analyzer
  version: 1
  description: "訂單風險分析專家"

model: smart                          # 引用 model alias

system_prompt: |
  你是訂單風險分析專家。根據訂單資料與客戶歷史，
  評估風險等級並產生 JSON 報告。

input_schema:
  type: object
  properties:
    order_id: { type: string }
  required: [order_id]

output_schema:
  type: object
  properties:
    risk_level: { type: string, enum: [low, medium, high] }
    summary: { type: string }
  required: [risk_level, summary]

tools:
  - type: task
    action: order.load
  - type: task
    action: customer.get_history
  - type: tool
    name: agent.ask_human             # 兩段式命名
  - type: tool
    name: artifact.read
  - type: tool
    name: artifact.write
  - type: toolset
    name: order-processing            # 透過 toolset 引用 skills

config:
  max_iterations: 10
  output_format: json
  stream: iteration

hooks:
  on_before_tool_call:
    - type: task
      action: audit.log
      input:
        session_id: ${ session.id }
        tool: ${ hook.tool_name }
```

---

## 範例 5：Workflow with Agent Step

### 5a. 批次審核（自動流程，tools_exclude 排除 ask_human）

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: order_batch_review
  version: 1

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id: { type: string }
  required: [order_id]

steps:
  # 1. 載入訂單
  - id: load
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  # 2. Agent 自動分析 — 排除 ask_human（全自動流程不需人工介入）
  - id: analyze
    type: agent
    agent: order.analyzer
    prompt: "分析訂單 ${ input.order_id } 的風險等級"
    input:
      order_id: ${ input.order_id }
    tools:
      - type: agent
        agent: fraud.detector           # 額外合併 agent-as-tool
    tools_exclude: [agent.ask_human]    # 排除人工介入
    config:
      persist_history: none
    timeout: 2m

  # 3. 根據風險等級分流
  - type: switch
    condition: ${ steps.analyze.output.risk_level }
    cases:
      - value: "high"
        then:
          - type: emit
            event: order.high_risk
            data:
              order_id: ${ input.order_id }
              risk_level: "high"
              summary: ${ steps.analyze.output.summary }

  - type: return
    output:
      order_id: ${ input.order_id }
      risk_level: ${ steps.analyze.output.risk_level }
      summary: ${ steps.analyze.output.summary }
```

### 5b. 人工審核（保留 ask_human、MCP、skills via toolset）

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: order_manual_review
  version: 1

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id: { type: string }
  required: [order_id]

steps:
  - id: load
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  # Agent 人工審核 — 保留 ask_human，加入 MCP tool、使用 prompt template
  - id: manual_review
    type: agent
    agent: order.analyzer
    prompt:
      use_template: review-prompt
      input:
        order_id: ${ input.order_id }
        tier: ${ steps.load.output.customer_tier }
    input:
      order_id: ${ input.order_id }
    tools:
      - type: mcp
        server: compliance-mcp
        tools: [check_sanctions_list]
    system_prompt_append: "審核模式：高風險案例，務必透過 agent.ask_human 向審核人員確認。"
    config:
      persist_history: full
      stream: token                   # 即時串流給審核 UI
    timeout: 10m

  - type: return
    output:
      order_id: ${ input.order_id }
      review: ${ steps.manual_review.output }
```

---

## 範例 6：Resources Definition

宣告 MCP servers、string templates、model aliases 等共用資源。

```yaml
apiVersion: resource/v3
kind: Resources

metadata:
  name: order-resources
  version: 1

model_aliases:
  fast:
    provider: anthropic
    name: claude-haiku-4-5-20251001
    config: { temperature: 0, max_tokens: 2048 }
  smart:
    provider: anthropic
    name: claude-sonnet-4-20250514
    config: { temperature: 0, max_tokens: 4096 }

mcp_servers:
  compliance-mcp:
    transport: stdio
    command: "python3 ./mcp-servers/compliance/main.py"
  github-mcp:
    transport: stdio
    command: "npx -y @modelcontextprotocol/server-github"
    env:
      GITHUB_TOKEN: ${ secret.GITHUB_TOKEN }

string_templates:
  review-prompt:
    template: |
      請審核訂單 ${ order_id }，客戶等級為 ${ tier }。
      重點關注金額異常與歷史模式。
    input_schema:
      type: object
      properties:
        order_id: { type: string }
        tier: { type: string }
      required: [order_id]

skills:
  risk-analysis:
    path: ./skills/risk-analysis
    description: "風險分析知識包"
```

---

## 範例 7：Toolset Definition

將 tools 與 skills 組合為具名的能力集合，供 agent definition 引用。

```yaml
apiVersion: toolset/v3
kind: Toolset

metadata:
  name: order-processing
  version: 1
  description: "訂單處理相關工具與技能集合"

tools:
  - type: task
    action: order.load
  - type: task
    action: order.update
  - type: task
    action: customer.get_history
  - type: mcp
    server: compliance-mcp
    tools: [check_sanctions_list]
  - type: tool
    name: agent.ask_human

skills:
  - name: risk-analysis

includes:
  - customer-management             # 組合另一個 toolset
```

### 完整訂單套件（透過 includes 組合多個 toolset）

```yaml
apiVersion: toolset/v3
kind: Toolset

metadata:
  name: full-order-suite
  version: 1
  description: "完整訂單處理套件，包含所有相關 toolset"

tools:
  - type: agent
    agent: fraud.detector             # agent-as-tool
  - type: tool
    name: artifact.read
  - type: tool
    name: artifact.write

includes:
  - order-processing
```

---

## 範例 8：Saga Compensation

付款與訂單建立的交易性補償。每個 step 宣告自己的 `compensate` 區塊，由 `saga` 統一協調。

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: order_with_payment
  version: 1

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id: { type: string }
    amount: { type: number }
  required: [order_id, amount]

steps:
  - type: saga
    id: payment_saga
    config:
      on_failure: continue            # Best-effort 補償
      recovery: backward              # 反向補償
    steps:
      # Step 1: 扣款
      - id: debit
        type: task
        action: payment.debit
        input:
          order_id: ${ input.order_id }
          amount: ${ input.amount }
        compensate:
          - type: task
            action: payment.refund
            input:
              amount: ${ input.amount }
              reference: ${ steps.debit.output.transaction_id }
            retry:
              max_attempts: 3
              delay: 5s
              backoff: exponential

      # Step 2: 建立訂單
      - id: create_order
        type: task
        action: order.create
        input:
          order_id: ${ input.order_id }
          payment_id: ${ steps.debit.output.transaction_id }
        compensate:
          - type: task
            action: order.cancel
            input:
              order_id: ${ input.order_id }
              reason: "payment saga rolled back"

      # Step 3: 發送確認通知（若此 step 失敗，觸發 Step 2 → Step 1 反向補償）
      - id: notify
        type: task
        action: notification.send
        input:
          order_id: ${ input.order_id }
          template: "order_confirmed"

    on_compensation_complete:
      - type: emit
        event: saga.compensated
        data:
          order_id: ${ input.order_id }
      - type: fail
        message: "訂單交易失敗，已完成補償回滾"
        code: "saga_compensated"

  - type: return
    output:
      order_id: ${ input.order_id }
      payment_id: ${ steps.debit.output.transaction_id }
      status: "confirmed"
```

---

## 範例 9：Continue-as-New（renew）

資料同步場景：以 cursor 分頁方式持續同步資料。每次處理一批後，透過 `renew` 建立新 instance 繼續處理，避免單一 instance 累積過多 step history。

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: data_sync
  version: 1
  description: "以 cursor 分頁方式持續同步外部資料"

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    cursor:
      type: string
      default: ""
    total_synced:
      type: integer
      default: 0
    batch_size:
      type: integer
      default: 100

steps:
  # 1. 以 cursor 取得一批資料
  - id: fetch_batch
    type: task
    action: datasource.fetch
    input:
      cursor: ${ input.cursor }
      limit: ${ input.batch_size }
    timeout: 60s

  # 2. 逐筆處理
  - id: process_items
    type: foreach
    items: ${ steps.fetch_batch.output.items }
    as: item
    concurrency: 5
    do:
      - type: task
        action: datasource.sync_item
        input:
          item: ${ loop.item }
        execution:
          policy: idempotent

  # 3. 計算同步數量
  - type: assign
    vars:
      new_total: ${ input.total_synced + size(steps.fetch_batch.output.items) }

  # 4. 判斷是否還有下一頁
  - type: if
    condition: ${ steps.fetch_batch.output.has_more == true }
    then:
      # 還有資料 → continue-as-new，帶 cursor 繼續
      - type: return
        renew: true
        output:
          cursor: ${ steps.fetch_batch.output.next_cursor }
          total_synced: ${ vars.new_total }
          batch_size: ${ input.batch_size }
    else:
      # 同步完成
      - type: emit
        event: data_sync.completed
        data:
          total_synced: ${ vars.new_total }
      - type: return
        output:
          total_synced: ${ vars.new_total }
          status: "completed"
```

---

## 範例 10：Instance Labels

展示 `label_schema` 宣告、建立 instance 時附帶 labels、以及在 workflow 執行中動態更新 labels。

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: order_processing
  version: 1
  description: "訂單處理流程（含 instance labels）"

label_schema:
  customer_id:
    description: "客戶 ID"
  region:
    description: "部署區域"
    enum: [tw, jp, us, eu]
  priority:
    description: "優先級"
    enum: [low, medium, high]
    default: medium

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

steps:
  # 1. 載入訂單
  - id: load
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  # 2. 分析風險
  - id: analyze
    type: task
    action: order.analyze_risk
    input:
      order_id: ${ input.order_id }

  # 3. 動態設定 priority label（根據分析結果）
  - id: set_priority
    type: task
    action: instance.set_labels
    input:
      labels:
        priority: ${ steps.analyze.output.risk_level == "high" ? "high" : "medium" }
        processed_by: "auto"

  # 4. 處理訂單
  - id: process
    type: task
    action: order.process
    input:
      order_id: ${ input.order_id }

  - type: return
    output:
      status: "completed"
      risk_level: ${ steps.analyze.output.risk_level }
```

建立 instance 時附帶初始 labels：

```bash
curl -X POST https://engine.example.com/instances \
  -H "Content-Type: application/json" \
  -d '{
    "workflow": "order_processing",
    "version": 1,
    "input": { "order_id": "ORD-001" },
    "labels": {
      "customer_id": "CUST-789",
      "region": "tw"
    }
  }'
```

查詢帶 label 的 instances：

```bash
# 查詢台灣區域、高優先級的 running instances
slogan instance list --label region=tw --label priority=high --status running
```

---

## 相關文件

- [01-kind-definitions](01-kind-definitions.md) — Kind 一覽與 apiVersion 格式
- [05-steps-overview](05-steps-overview.md) — Step 共通屬性
- [08-step-control-flow](08-step-control-flow.md) — if / switch（`condition:` 語法）
- [10-step-terminal](10-step-terminal.md) — return / fail / renew
- [12-step-saga](12-step-saga.md) — saga 與 step-level compensate
- [13-agent-definition](13-agent-definition.md) — Agent Definition
- [18-toolset-definition](18-toolset-definition.md) — Toolset Definition
- [19-resources-definition](19-resources-definition.md) — Resources Definition
- [24-instance-labels](24-instance-labels.md) — Instance Labels
