# 99 — 完整範例集

使用 v5 語法的完整範例。

---

## 範例 1：Tool Definition

```yaml
apiVersion: tool/v5
kind: Tool

metadata:
  name: order.load
  version: 1
  description: "從資料庫載入訂單"

input_schema:
  type: object
  properties:
    order_id: { type: string }
  required: [order_id]

output_schema:
  type: object
  properties:
    id: { type: string }
    amount: { type: number }
    status: { type: string }

backend:
  type: exec
  mode: protocol
  command: "node ./tools/order/load.js"
```

---

## 範例 2：基本 Workflow — 訂單處理

```yaml
apiVersion: workflow/v5
kind: Workflow

metadata:
  name: order_fulfillment
  version: 1
  description: "訂單履行流程"

triggers:
  - type: manual
  - type: event
    event: order.created
    when: ${ event.data.source == "api" }
    input_mapping:
      order_id: ${ event.data.order_id }

input_schema:
  type: object
  properties:
    order_id: { type: string }
  required: [order_id]

config:
  timeout: 2h

steps:
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  - type: if
    when: ${ steps.load_order.output.status == "cancelled" }
    then:
      - type: fail
        message: "order already cancelled"

  - id: wait_payment
    type: wait
    event: payment.confirmed
    match: ${ event.data.order_id == steps.load_order.output.id }
    timeout: 30m
    catch:
      - type: fail
        when: ${ error.type == "timeout" }
        message: "payment timeout"

  - type: emit
    event: order.completed
    data:
      order_id: ${ steps.load_order.output.id }

  - type: return
    output:
      status: "completed"
      order_id: ${ steps.load_order.output.id }
```

---

## 範例 3：呼叫 Function

```yaml
apiVersion: workflow/v5
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
  required: [order_id]

steps:
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  - id: process_payment
    type: task
    action: payment.process       # 呼叫 kind: Function
    input:
      order_id: ${ steps.load_order.output.id }
      amount: ${ steps.load_order.output.amount }
    timeout: 5m
    catch:
      - type: fail
        message: ${ "payment failed: " + error.message }

  - type: return
    output:
      payment_id: ${ steps.process_payment.output.payment_id }
      status: "completed"
```

---

## 範例 4：Saga 補償

```yaml
apiVersion: workflow/v5
kind: Workflow

metadata:
  name: order_saga
  version: 1

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id: { type: string }
    amount: { type: number }
    items: { type: array }
  required: [order_id, amount, items]

steps:
  # inventory.reserve 的 Tool definition 已定義 compensate → inventory.release
  # payment.charge 的 Tool definition 已定義 compensate → payment.refund
  # shipping.create 無 compensate（通知類操作）
  - type: saga
    steps:
      - id: reserve_stock
        type: task
        action: inventory.reserve
        input: { items: ${ input.items } }

      - id: charge
        type: task
        action: payment.charge
        input: { amount: ${ input.amount } }

      - type: task
        action: shipping.create
        input: { order_id: ${ input.order_id } }

    catch:
      - type: emit
        event: order.rolled_back
        data: { order_id: ${ input.order_id } }
      - type: fail
        message: "訂單處理失敗，已回滾"

  - type: return
    output: { status: "completed" }
```

---

## 範例 5：Agent — 訂單風險分析

### Agent Definition

```yaml
apiVersion: agent/v5
kind: Agent

metadata:
  name: order.analyzer
  version: 1
  description: "訂單風險分析 agent"

model: smart

system: |
  你是訂單風險分析專家。分析訂單資料，評估風險等級（low/medium/high），
  並提供建議。如有疑慮，可透過 ask_human 詢問人工審核。

input_schema:
  type: object
  properties:
    order: { type: object }
output_schema:
  type: object
  properties:
    risk_level: { type: string, enum: [low, medium, high] }
    reason: { type: string }

tools:
  - order.load
  - customer.get_history
  - agent.ask_human

config:
  max_iterations: 15
  output_format: json
```

### 在 Workflow 中使用

```yaml
apiVersion: workflow/v5
kind: Workflow

metadata:
  name: order_risk_check
  version: 1

triggers:
  - type: event
    event: order.created
    input_mapping:
      order_id: ${ event.data.order_id }

steps:
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  - id: analyze
    type: agent
    agent: order.analyzer
    prompt: |
      分析以下訂單的風險：
      訂單 ID: ${ steps.load_order.output.id }
      金額: ${ steps.load_order.output.amount }
    input:
      order: ${ steps.load_order.output }
    tools:
      - customer.get_history
    timeout: 3m

  - type: if
    when: ${ steps.analyze.output.risk_level == "high" }
    then:
      - type: emit
        event: order.high_risk
        data:
          order_id: ${ steps.load_order.output.id }
          reason: ${ steps.analyze.output.reason }

  - type: return
    output:
      risk_level: ${ steps.analyze.output.risk_level }
```

---

## 範例 6：Resources 定義

```yaml
apiVersion: resource/v5
kind: Resources

metadata:
  name: order-resources
  version: 1

mcp_servers:
  compliance-mcp:
    transport: stdio
    command: "python -m compliance_server"
    env:
      DB_URL: ${ secret.COMPLIANCE_DB_URL }

model_aliases:
  fast:
    provider: anthropic
    name: claude-haiku-4-5-20251001
    config: { max_tokens: 4096 }
  smart:
    provider: anthropic
    name: claude-sonnet-4-20250514
    config: { max_tokens: 8192 }

toolsets:
  order-tools:
    tools:
      - order.load
      - order.update
      - "compliance-mcp.*"
    skills:
      - name: risk-analysis
```

---

## 範例 7：Parallel + foreach

```yaml
apiVersion: workflow/v5
kind: Workflow

metadata:
  name: order_finalize
  version: 1

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id: { type: string }
    items: { type: array }
  required: [order_id, items]

steps:
  - id: reserve_items
    type: foreach
    items: ${ input.items }
    concurrency: 3
    failure_policy: fail_fast
    do:
      - type: task
        action: inventory.reserve
        input:
          sku: ${ loop.item.sku }
          qty: ${ loop.item.qty }

  - type: parallel
    branches:
      - steps:
          - type: task
            action: notify.customer
            input: { order_id: ${ input.order_id } }
      - steps:
          - type: task
            action: report.generate
            input: { order_id: ${ input.order_id } }

  - type: return
    output: { status: "finalized" }
```
