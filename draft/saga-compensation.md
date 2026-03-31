# Saga / Compensation 模式

> **狀態**：草稿
> **範圍**：宣告式補償區塊設計，用於長交易的失敗回復

---

## 設計動機

Workflow 中多個 step 可能涉及跨服務操作（如扣款、建立訂單、發通知），當後續 step 失敗時，需要反向補償已完成的操作。目前缺乏語言層面的宣告式補償機制，使用者只能手動在 `on_error` 中撰寫補償邏輯，容易遺漏且不易維護。

**越晚加入越難**：補償模式影響 step 的語意與執行語意，若等核心穩定後再加，可能需要大幅調整 step 執行邏輯與 storage schema。

---

## 初步構想

### 方案 A：Step 層級 `compensate` 區塊

在每個 step 上宣告其補償邏輯，引擎在失敗時按反向順序執行已完成 step 的 compensate：

```yaml
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

  - id: create_order
    type: task
    action: order.create
    input:
      items: ${ input.items }
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

當 `notify` 失敗時，引擎反向執行：`create_order.compensate` → `debit.compensate`。

### 方案 B：`saga` 區塊包裹

以 `saga` step 包裹一組需要補償的操作：

```yaml
- type: saga
  steps:
    - id: debit
      type: task
      action: payment.debit
      input: { ... }
      compensate:
        - type: task
          action: payment.refund
          input: { ... }

    - id: create_order
      type: task
      action: order.create
      input: { ... }
      compensate:
        - type: task
          action: order.cancel
          input: { ... }

  on_compensation_complete:
    - type: fail
      message: "Saga 補償完成，交易已回滾"
```

### 方案 C：全域 compensation 策略

在 workflow 層級定義 compensation 策略，不在每個 step 上宣告：

```yaml
compensation:
  strategy: backward   # backward | forward | manual
  steps:
    debit:
      - type: task
        action: payment.refund
        input: { ... }
    create_order:
      - type: task
        action: order.cancel
        input: { ... }
```

---

## 補償執行語意

| 維度 | 說明 |
|------|------|
| 觸發條件 | Saga 範圍內任一 step FAILED（且非 `on_error` 已處理） |
| 執行順序 | 反向（最後完成的 step 最先補償） |
| 補償範圍 | 僅已成功完成（SUCCEEDED）的 step |
| 補償失敗 | 補償 step 本身失敗 → 記錄錯誤，繼續執行其餘補償（best-effort） |
| 狀態 | 新增 instance 狀態：`COMPENSATING`、`COMPENSATED`、`COMPENSATION_FAILED` |

---

## 與現有 `on_error` 的關係

| 機制 | 觸發範圍 | 用途 |
|------|---------|------|
| `on_error` | 單一 step 失敗 | 錯誤處理（重試、降級、記錄） |
| `compensate` | Saga 範圍內 step 失敗 | 反向回復已完成的副作用 |

兩者互補：`on_error` 先執行，若未能恢復（step 仍為 FAILED），則觸發 saga 補償。

---

## 待決定

- 方案 A（step 層級）vs 方案 B（saga 區塊）vs 方案 C（全域）
- 補償失敗時的行為：best-effort 還是中止
- 補償是否支援重試
- 巢狀 saga 的語意
- `sub_workflow` 和 `agent` step 的補償行為
- 是否需要 forward recovery（重試前進而非回滾）
- Storage schema 需新增哪些欄位追蹤補償狀態
