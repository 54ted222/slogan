# 99 — 完整範例集

使用 v6 語法的完整範例。

---

## 範例 1：Tool Definition

```yaml
apiVersion: tool/v6
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
apiVersion: workflow/v6
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
    signals:
      - event: payment.confirmed
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
apiVersion: workflow/v6
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
apiVersion: workflow/v6
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

## 範例 5：Parallel + foreach

```yaml
apiVersion: workflow/v6
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
