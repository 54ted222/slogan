# 07 — Step: assign

`assign` 步驟用於設定 `vars` namespace 中的變數，供後續 steps 使用。

---

## Schema

```yaml
- type: assign       # MUST
  vars:              # MUST — 要設定的變數
    var_name: value
```

---

## 語意

- `vars` 中的每個 key 成為 `vars.<key>` 可存取的變數
- 值可以是字面值、CEL 表達式、或包含表達式的巢狀結構
- 同名變數會被後續的 `assign` 覆寫
- `assign` step 永遠成功（除非 CEL 表達式求值失敗）

---

## 值類型

### 字面值

```yaml
- type: assign
  vars:
    max_retries: 3
    default_currency: "USD"
    is_test: false
```

### CEL 表達式

```yaml
- type: assign
  vars:
    is_pay_action: ${ input.action == "pay" }
    order_total: ${ steps.load_order.output.amount * 1.1 }
```

### 巢狀結構

```yaml
- type: assign
  vars:
    payment_input:
      order_id: ${ steps.load_order.output.id }
      amount: ${ steps.load_order.output.amount }
      currency: "USD"
```

上例中 `vars.payment_input` 為一個 map，可在後續 step 中作為整體傳遞：

```yaml
- id: create_payment
  type: task
  action: payment.create
  input: ${ vars.payment_input }
```

---

## 作用域

- `vars` 為 workflow instance 級的全域命名空間
- 一旦設定，所有後續 steps 皆可存取（含巢狀 steps）
- 在 `foreach` 內的 `assign` 設定的變數，迴圈外也可存取（最後一次迭代的值）
- 在 `parallel` 的不同 branches 中同時 `assign` 同名變數，結果為未定義行為（SHOULD 避免）
