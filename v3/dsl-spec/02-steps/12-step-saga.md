# 12 — Step: saga 與 step-level compensate

本文件定義兩種補償機制：任意 step 上的 `compensate` 區塊，以及 `type: saga` step 包裹一組需要交易性補償的 steps。

---

## Overview

Workflow 中多個 step 可能涉及跨服務操作（如扣款、建立訂單、發通知），當後續 step 失敗時，需要反向補償已完成的操作。本規格提供兩種互補的補償機制：

| 機制 | 適用場景 | 失敗行為 | Recovery 方向 |
|------|----------|----------|---------------|
| Step-level `compensate` | 任意 step 宣告個別補償邏輯 | 固定 abort | 僅 backward |
| `type: saga` step | 包裹一組 steps，統一管理補償策略 | 可配置（abort / continue） | 可配置（backward / forward） |

兩者可同時使用：`saga` step 內的子 steps 各自宣告 `compensate` 區塊，由 saga 統一協調補償執行。

---

## Step-level compensate

任意 step 皆可宣告 `compensate` 區塊（step 陣列）。當該 step 已 SUCCEEDED 但後續失敗觸發補償時，引擎執行此區塊。

### Schema

```yaml
- id: string
  type: task | sub_workflow | agent | ...
  # ... step 本體屬性
  compensate:                            # MAY — step 陣列
    - type: task
      action: string
      input: { ... }
      retry: { ... }                     # MAY — 補償 step 支援 retry
```

### 行為規則

| 規則 | 說明 |
|------|------|
| 觸發條件 | step 狀態為 SUCCEEDED，且後續 step 失敗觸發補償流程 |
| 執行順序 | 反向（backward only）：最後完成的 step 最先補償 |
| 補償範圍 | 僅 SUCCEEDED 的 step 會執行其 compensate 區塊 |
| 失敗行為 | 固定 abort — compensate step 失敗時立即中止補償流程 |
| Retry | compensate 區塊內的 step 支援標準 retry 設定 |

> **注意**：Step-level `compensate` 區塊僅在 `saga` step 範圍內，或 workflow 層級觸發補償時才會執行。單獨宣告 `compensate` 而未被 saga 包裹時，引擎不會自動觸發補償。

---

## type: saga Step

`saga` step 包裹一組需要交易性補償的 steps，提供可配置的補償策略。

### Schema

```yaml
- type: saga                             # MUST
  id: string                             # MAY
  steps: [...]                           # MUST — 可補償的 step 序列
  config:
    on_failure: abort | continue         # MAY, 預設 abort
    recovery: backward | forward         # MAY, 預設 backward
  on_compensation_complete: [...]        # MAY — 補償完成後執行的 step 陣列
  catch: [...]                        # MAY — 錯誤處理（step 陣列）
```

### config 說明

#### on_failure

控制補償 step 本身失敗時的行為。

| 值 | 說明 |
|----|------|
| `abort`（預設） | 補償 step 失敗時立即中止，不再執行剩餘補償 |
| `continue` | Best-effort 模式，補償 step 失敗時記錄錯誤，繼續執行剩餘補償 |

#### recovery

控制補償的方向。

| 值 | 說明 |
|----|------|
| `backward`（預設） | 反向補償：從最後一個 SUCCEEDED step 開始，依反向順序執行各 step 的 compensate 區塊 |
| `forward` | 正向恢復：重試失敗的 step（retry-based recovery），嘗試讓整個 saga 完成而非回滾 |

### on_compensation_complete

補償流程完成後（無論成功或 best-effort 完成）執行的 step 陣列。典型用途為記錄補償結果或發送通知。

```yaml
on_compensation_complete:
  - type: emit
    event: saga.compensated
    data:
      saga_id: ${ steps.my_saga.id }
  - type: fail
    message: "Saga 補償完成，交易已回滾"
```

---

## 補償執行語意

### 觸發條件

Saga 範圍內任一 step 狀態為 FAILED（且未被該 step 自身的 `catch` 恢復），即觸發補償流程。

### 執行順序（backward recovery）

反向執行：最後一個 SUCCEEDED step 的 compensate 最先執行，依序回溯至第一個。

```
steps 執行順序:  A → B → C（失敗）
補償執行順序:    B.compensate → A.compensate
```

### 補償範圍

| Step 狀態 | 是否補償 |
|-----------|---------|
| SUCCEEDED | 是 — 執行其 compensate 區塊 |
| FAILED | 否 — 觸發補償的 step，無需補償 |
| SKIPPED | 否 — 未執行，無需補償 |
| PENDING | 否 — 尚未到達 |

### Forward Recovery

當 `recovery: forward` 時，引擎不執行 compensate 區塊，而是重試失敗的 step：

1. 失敗的 step 依據其 `retry` 設定重試
2. 若重試成功，saga 繼續執行後續 steps
3. 若重試仍失敗（retry 用盡），saga 最終狀態為 FAILED

Forward recovery 適用於暫時性錯誤（如網路超時），且所有操作皆為 idempotent 的場景。

### Retry on Compensate Steps

Compensate 區塊內的 step 支援標準 retry 設定（決策 A3）：

```yaml
compensate:
  - type: task
    action: payment.refund
    input:
      payment_id: ${ steps.debit.output.payment_id }
    retry:
      max_attempts: 3
      delay: 2s
      backoff: exponential
```

---

## 新增 Instance 狀態

補償機制引入三個新的 workflow instance 狀態：

| 狀態 | 說明 |
|------|------|
| `COMPENSATING` | 正在執行補償流程 |
| `COMPENSATED` | 補償流程已成功完成（所有 compensate 皆 SUCCEEDED） |
| `COMPENSATION_FAILED` | 補償流程中有 compensate step 失敗（`on_failure: abort` 時中止；`on_failure: continue` 時部分失敗） |

### 狀態轉換

```
RUNNING ──→ COMPENSATING ──→ COMPENSATED
                  │
                  └──→ COMPENSATION_FAILED
```

- RUNNING → COMPENSATING：saga 內 step 失敗，觸發補償
- COMPENSATING → COMPENSATED：所有補償 step 成功完成
- COMPENSATING → COMPENSATION_FAILED：補償 step 失敗（abort 模式中止 / continue 模式部分失敗）

---

## 與 catch 的關係

| 機制 | 觸發範圍 | 用途 |
|------|----------|------|
| `catch` | 單一 step 失敗 | 錯誤處理（重試、降級、記錄） |
| `compensate` | Saga 範圍內 step 失敗 | 反向回復已完成的副作用 |

兩者互補，執行順序如下：

1. Step 失敗 → 先執行該 step 的 `catch`（若有）
2. `catch` 執行後，若 step 仍為 FAILED → 觸發 saga 補償
3. `catch` 成功恢復（step 不再為 FAILED）→ 不觸發補償，saga 繼續執行

---

## 範例

### Step-level compensate（個別 step 宣告補償）

```yaml
steps:
  - type: saga
    steps:
      - id: debit
        type: task
        action: payment.debit
        input:
          account: ${ input.account_id }
          amount: ${ input.amount }
        compensate:
          - type: task
            action: payment.refund
            input:
              account: ${ input.account_id }
              amount: ${ input.amount }
              reference: ${ steps.debit.output.transaction_id }
            retry:
              max_attempts: 3
              delay: 2s
              backoff: exponential

      - id: create_order
        type: task
        action: order.create
        input:
          items: ${ input.items }
          payment_ref: ${ steps.debit.output.transaction_id }
        compensate:
          - type: task
            action: order.cancel
            input:
              order_id: ${ steps.create_order.output.order_id }

      - id: notify
        type: task
        action: notification.send
        input:
          message: "訂單已建立"
```

當 `notify` 失敗時，引擎反向執行補償：`create_order.compensate` → `debit.compensate`。`notify` 本身無 compensate 區塊（僅發通知，無需回滾），且為觸發失敗的 step，不進行補償。

### Saga block 搭配 on_compensation_complete

```yaml
- id: order_saga
  type: saga
  config:
    on_failure: continue
    recovery: backward
  steps:
    - id: reserve_stock
      type: task
      action: inventory.reserve
      input:
        items: ${ input.items }
      compensate:
        - type: task
          action: inventory.release
          input:
            reservation_id: ${ steps.reserve_stock.output.reservation_id }

    - id: charge
      type: task
      action: payment.charge
      input:
        amount: ${ input.total }
      compensate:
        - type: task
          action: payment.refund
          input:
            charge_id: ${ steps.charge.output.charge_id }

    - id: ship
      type: task
      action: shipping.create
      input:
        order_id: ${ input.order_id }

  on_compensation_complete:
    - type: emit
      event: order.rolled_back
      data:
        order_id: ${ input.order_id }
    - type: fail
      message: "訂單處理失敗，已完成補償"
```

使用 `on_failure: continue` 確保即使部分補償失敗，仍盡力執行所有補償。

### Saga with forward recovery

```yaml
- type: saga
  config:
    recovery: forward
  steps:
    - id: sync_crm
      type: task
      action: crm.sync_customer
      input:
        customer_id: ${ input.customer_id }
      retry:
        max_attempts: 5
        delay: 3s
        backoff: exponential

    - id: sync_erp
      type: task
      action: erp.sync_order
      input:
        order_id: ${ input.order_id }
      retry:
        max_attempts: 5
        delay: 3s
        backoff: exponential
```

當 `sync_erp` 失敗時，引擎不回滾已完成的 `sync_crm`，而是重試 `sync_erp`（依其 retry 設定）。適用於所有操作皆為 idempotent 且希望最終完成的場景。

---

## 限制

| 限制 | 說明 |
|------|------|
| 巢狀 saga | **不支援** — saga step 內不可再包含 saga step |
| sub_workflow 內的 compensate | sub_workflow step 可宣告 compensate 區塊，遵循相同規則 |
| agent 內的 compensate | agent step 可宣告 compensate 區塊，遵循相同規則 |
| compensate 區塊內的 step 類型 | 支援所有 step 類型，但建議以 task 為主（補償應為簡單、確定性的操作） |
| forward recovery + compensate | `recovery: forward` 時不執行 compensate 區塊，僅重試失敗的 step |
