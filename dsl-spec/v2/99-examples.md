# 99 — Examples

本文件提供完整範例，示範 v2 DSL 的所有功能。

---

## 範例 1：Task Definition（三種 backend）

### stdio — 訂單載入

```yaml
apiVersion: task/v2
kind: Task

metadata:
  name: order.load
  version: 3

input:
  schema:
    type: object
    properties:
      order_id:
        type: string
    required: [order_id]

output:
  schema:
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

### stdio — 檔案壓縮（shell script）

```yaml
apiVersion: task/v2
kind: Task

metadata:
  name: file.compress
  version: 1

input:
  schema:
    type: object
    properties:
      source_path:
        type: string
      target_path:
        type: string
    required: [source_path, target_path]

output:
  schema:
    type: object
    properties:
      size_bytes:
        type: integer

backend:
  type: stdio
  command: "./tools/file-compress.sh"
```

### http — 付款建立

```yaml
apiVersion: task/v2
kind: Task

metadata:
  name: payment.create
  version: 2

input:
  schema:
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

output:
  schema:
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
apiVersion: task/v2
kind: Task

metadata:
  name: system.echo
  version: 1

backend:
  type: builtin
  handler: echo
```

---

## 範例 2：最小 Workflow

最簡單的 workflow：一個 task 加一個 return。

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: hello_world
  version: 1

triggers:
  - type: manual

steps:
  - id: greet
    type: task
    action: system.echo
    input:
      message: "hello world"

  - type: return
    output:
      message: ${ steps.greet.output.result }
```

---

## 範例 3：完整訂單流程

示範全部 11 種 step types，包含 sub_workflow。

### Parent Workflow: order_fulfillment

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: order_fulfillment
  version: 1
  description: "訂單履行完整流程"
  labels:
    team: commerce
    domain: order

triggers:
  - type: manual
  - type: event
    event: order.created
    when: ${ event.data.source == "api" }
    input_mapping:
      order_id: ${ event.data.order_id }
      action: ${ event.data.action }
      notify_customer: ${ has(event.data.notify) ? event.data.notify : true }
      items: ${ event.data.line_items }

input:
  schema:
    type: object
    properties:
      order_id:
        type: string
      action:
        type: string
        enum: [pay, ship]
      notify_customer:
        type: boolean
        default: true
      items:
        type: array
        items:
          type: object
          properties:
            sku:
              type: string
            qty:
              type: integer
              minimum: 1
          required: [sku, qty]
    required: [order_id, action]

output:
  schema:
    type: object
    properties:
      order_id:
        type: string
      status:
        type: string
      payment_id:
        type: string
    required: [order_id, status]

artifacts:
  order_file:
    kind: file
    lifecycle: input
    required: true

  result_report:
    kind: file
    lifecycle: output

config:
  timeout: 2h
  max_step_executions: 500
  on_error:
    - type: emit
      event: workflow.error
      data:
        workflow: "order_fulfillment"
        order_id: ${ input.order_id }
        error: ${ error.message }

steps:
  # ── 1. 載入訂單（task）──
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }
    resources:
      - name: order_file
        type: artifact
        ref: order_file
        access: read
    retry:
      max_attempts: 3
      delay: 2s
      backoff: exponential
    timeout: 10s
    execution:
      policy: replayable

  # ── 2. 指派變數（assign）──
  - type: assign
    vars:
      payment_input:
        order_id: ${ steps.load_order.output.id }
        amount: ${ steps.load_order.output.amount }
      is_high_value: ${ steps.load_order.output.amount > 1000 }

  # ── 3. 檢查是否已取消（if）──
  - type: if
    expr: ${ steps.load_order.output.status == "cancelled" }
    then:
      - type: emit
        event: order.rejected
        data:
          order_id: ${ steps.load_order.output.id }
          reason: "order already cancelled"
      - type: fail
        message: "order has been cancelled"
        code: "order_cancelled"
    else:
      - type: emit
        event: order.processing
        data:
          order_id: ${ steps.load_order.output.id }

  # ── 4. 依 action 分流（switch）──
  - type: switch
    expr: ${ input.action }
    cases:
      - value: "pay"
        then:
          # ── 4a. 呼叫付款子流程（sub_workflow）──
          - id: process_payment
            type: sub_workflow
            workflow: payment_processing
            version: 3
            input:
              order_id: ${ steps.load_order.output.id }
              amount: ${ steps.load_order.output.amount }
            timeout: 35m
            execution:
              policy: non_repeatable
            on_error:
              - type: emit
                event: payment.failed
                data:
                  order_id: ${ steps.load_order.output.id }
                  error: ${ error.message }

      - value: "ship"
        then:
          - id: create_shipment
            type: task
            action: shipment.create
            input:
              order_id: ${ steps.load_order.output.id }
              address: ${ steps.load_order.output.shipping_address }
            execution:
              policy: idempotent
            timeout: 30s

    default:
      - type: fail
        message: ${ "unknown action: " + input.action }
        code: "invalid_action"

  # ── 5. 庫存預留（foreach）──
  - id: reserve_items
    type: foreach
    items: ${ input.items }
    as: item
    concurrency: 3
    failure_policy: fail_fast
    do:
      - type: task
        action: inventory.reserve
        input:
          order_id: ${ steps.load_order.output.id }
          sku: ${ loop.item.sku }
          qty: ${ loop.item.qty }
        execution:
          policy: idempotent

  # ── 6. 平行處理（parallel）──
  - type: parallel
    failure_policy: wait_all
    on_error:
      - type: emit
        event: order.finalize_partial
        data:
          order_id: ${ steps.load_order.output.id }
          error: ${ error.message }
    branches:
      - - type: if
          expr: ${ input.notify_customer == true }
          then:
            - type: task
              action: notify.customer
              input:
                order_id: ${ steps.load_order.output.id }
                email: ${ steps.load_order.output.customer_email }

      - - id: generate_report
          type: task
          action: report.generate
          input:
            order_id: ${ steps.load_order.output.id }
            items_count: ${ input.items.size() }
          resources:
            - name: result_report
              type: artifact
              ref: result_report
              access: write

  # ── 7. 等待確認事件（wait_event）──
  - type: wait_event
    event: fulfillment.confirmed
    match: ${ event.data.order_id == steps.load_order.output.id }
    timeout: 1h
    on_timeout:
      - type: emit
        event: order.confirmation_timeout
        data:
          order_id: ${ steps.load_order.output.id }
      - type: fail
        message: "fulfillment confirmation timeout"

  # ── 8. 完成事件（emit）──
  - type: emit
    event: order.completed
    data:
      order_id: ${ steps.load_order.output.id }
      completed_at: ${ now() }

  # ── 9. 回傳結果（return）──
  - type: return
    output:
      order_id: ${ steps.load_order.output.id }
      status: "completed"
      payment_id: ${ default(steps.process_payment.output.payment_id, "") }
```

### Child Workflow: payment_processing

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: payment_processing
  version: 3
  description: "付款處理子流程"

triggers:
  - type: manual

input:
  schema:
    type: object
    properties:
      order_id:
        type: string
      amount:
        type: number
    required: [order_id, amount]

output:
  schema:
    type: object
    properties:
      payment_id:
        type: string
      status:
        type: string
    required: [payment_id, status]

config:
  timeout: 35m

steps:
  - id: create_payment
    type: task
    action: payment.create
    input:
      order_id: ${ input.order_id }
      amount: ${ input.amount }
      idempotency_key: ${ uuid() }
    execution:
      policy: non_repeatable
    timeout: 30s
    on_error:
      - type: emit
        event: payment.create_failed
        data:
          order_id: ${ input.order_id }
          error: ${ error.message }
      - type: fail
        message: ${ "payment creation failed: " + error.message }
        code: "payment_create_failed"

  - id: wait_confirmed
    type: wait_event
    event: payment.confirmed
    match: ${ event.data.payment_id == steps.create_payment.output.payment_id }
    timeout: 30m
    on_timeout:
      - type: emit
        event: payment.timeout
        data:
          order_id: ${ input.order_id }
          payment_id: ${ steps.create_payment.output.payment_id }
      - type: fail
        message: "payment confirmation timeout"
        code: "payment_timeout"

  - type: return
    output:
      payment_id: ${ steps.create_payment.output.payment_id }
      status: "confirmed"
```

---

## 範例 4：事件驅動鏈

純事件驅動的 workflow：由事件觸發，處理後發送新事件。

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: inventory_alert
  version: 1

triggers:
  - type: event
    event: inventory.low_stock
    when: ${ event.data.qty < 10 }

input:
  schema:
    type: object
    properties:
      sku:
        type: string
      qty:
        type: integer
      warehouse:
        type: string
    required: [sku, qty]

steps:
  - type: if
    expr: ${ input.qty < 3 }
    then:
      - type: task
        action: purchasing.create_order
        input:
          sku: ${ input.sku }
          qty: 100
          priority: "urgent"
        execution:
          policy: idempotent
      - type: emit
        event: inventory.urgent_reorder
        data:
          sku: ${ input.sku }
          ordered_qty: 100
    else:
      - type: task
        action: purchasing.create_order
        input:
          sku: ${ input.sku }
          qty: 50
          priority: "normal"
        execution:
          policy: idempotent

  - type: task
    action: notify.warehouse
    input:
      warehouse: ${ default(input.warehouse, "main") }
      sku: ${ input.sku }
      message: "reorder placed"
```

---

## 範例 5：多層錯誤處理

展示 step-level、block-level、workflow-level 三層錯誤處理。

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: resilient_pipeline
  version: 1

triggers:
  - type: manual

input:
  schema:
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
      - - type: task
          action: transform.type_a
          input:
            job_id: ${ input.job_id }
          timeout: 5m
      - - type: task
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

## 範例 6：Condition 使用

展示 step 的 `condition` 屬性，條件性跳過 steps。

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: conditional_steps
  version: 1

triggers:
  - type: manual

input:
  schema:
    type: object
    properties:
      skip_validation:
        type: boolean
        default: false
      environment:
        type: string
        enum: [dev, staging, prod]
    required: [environment]

steps:
  - type: task
    action: data.validate
    condition: ${ input.skip_validation != true }
    input:
      strict: ${ input.environment == "prod" }

  - type: task
    action: deploy.run
    input:
      env: ${ input.environment }

  - type: task
    action: test.smoke
    condition: ${ input.environment == "prod" }
    input:
      env: ${ input.environment }

  - type: return
    output:
      deployed: true
      validated: ${ input.skip_validation != true }
```
