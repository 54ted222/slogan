# 08 — Steps: Control Flow

本文件定義四種控制流程 step：`if`、`switch`、`foreach`、`parallel`。

---

## if

條件分支。根據 CEL 表達式的結果選擇執行 `then` 或 `else` 分支。

### Schema

```yaml
- type: if
  when: CEL expression        # MUST, 回傳 boolean
  then: [...]                      # MUST, step 陣列
  else: [...]                      # MAY, step 陣列
  on_error: [...]                  # MAY
```

### 行為

- `when` 求值為 `true` → 執行 `then` 陣列
- `when` 求值為 `false` → 執行 `else` 陣列（若有），否則 SKIP
- `when` 求值失敗 → step FAILED
- 分支內的 steps 按順序執行
- 未被選中的分支中的 steps 狀態為 SKIPPED

### output

若 `if` step 有 `id`，`steps.<id>.output` 為被選中分支中最後一個 step 的 output。若被選中分支為空或所有 steps 皆 SKIPPED，output 為 `null`。

### 範例

```yaml
- type: if
  when: ${ steps.load_order.output.status == "cancelled" }
  then:
    - type: emit
      event: order.rejected
      data:
        order_id: ${ steps.load_order.output.id }
    - type: fail
      message: "order cancelled"
  else:
    - type: emit
      event: order.processing
      data:
        order_id: ${ steps.load_order.output.id }
```

---

## switch

多值分支。根據 CEL 表達式的結果匹配對應的 case。

### Schema

```yaml
- type: switch
  when: CEL expression        # MUST
  cases:                           # MUST, 至少一個
    - value: any                   #   比對值
      then: [...]                  #   step 陣列
  default: [...]                   # MAY, step 陣列
  on_error: [...]                  # MAY
```

### 行為

- `when` 求值一次，結果與每個 `case.value` 做相等比較
- 匹配第一個符合的 case → 執行其 `then` 陣列
- 無匹配且有 `default` → 執行 default 陣列
- 無匹配且無 `default` → step SKIPPED
- `when` 求值失敗 → step FAILED
- `value` 可以是 string、number、boolean

### output

若 `switch` step 有 `id`，`steps.<id>.output` 為被選中 case（或 default）中最後一個 step 的 output。若無匹配且無 default（step SKIPPED），output 為 `null`。

### 範例

```yaml
- type: switch
  when: ${ input.action }
  cases:
    - value: "pay"
      then:
        - id: process_payment
          type: sub_workflow
          workflow: payment_processing
          input:
            order_id: ${ steps.load_order.output.id }
            amount: ${ steps.load_order.output.amount }

    - value: "ship"
      then:
        - id: create_shipment
          type: task
          action: shipment.create
          input:
            order_id: ${ steps.load_order.output.id }

  default:
    - type: fail
      message: ${ "unknown action: " + input.action }
```

---

## foreach

迴圈迭代。對一個 list 中的每個元素執行一組 steps。

### Schema

```yaml
- type: foreach
  items: CEL expression          # MUST, 回傳 list
  as: string                     # MAY, 預設 "item"
  concurrency: integer           # MAY, 預設 1（sequential）
  failure_policy: string         # MAY, 預設 "fail_fast"
  do: [...]                      # MUST, step 陣列
  on_error: [...]                # MAY
```

### 迭代變數

| 變數 | 說明 |
|------|------|
| `loop.item` | 當前迭代的元素 |
| `loop.index` | 當前迭代的索引（從 0 開始） |

`as` 欄位僅作為文件用途（提升可讀性），runtime 一律使用 `loop.item` 與 `loop.index`。

### concurrency

- `1`（預設）：sequential，逐一執行
- `> 1`：最多同時執行 N 個迭代
- 每個迭代內的 steps 仍按順序執行

### failure_policy

| 值 | 說明 |
|----|------|
| `fail_fast` | 任一迭代失敗 → 立即中止所有迭代，foreach FAILED |
| `continue` | 所有迭代都執行完畢，若有任一失敗 → foreach FAILED |
| `ignore` | 忽略個別迭代的失敗，foreach 永遠 SUCCEEDED |

### output

`steps.<foreach_id>.output` 為一個 array，長度 = `items` 長度，索引與 items 一一對應（所有 `failure_policy` 模式皆同）：

- 成功的迭代：該迭代中最後一個 step 的 output
- 失敗的迭代：`null`
- 被取消的迭代（`fail_fast` 模式）：`null`

### 範例

```yaml
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
```

---

## parallel

平行分支。同時執行多個獨立的 step 序列。

### Schema

```yaml
- type: parallel
  branches:                      # MUST, 至少兩個 branch
    - steps: [...]
    - steps: [...]
  failure_policy: string         # MAY, 預設 "fail_fast"
  on_error: [...]                # MAY
```

`branches` 為 branch 物件的陣列，每個 branch 包含 `steps` 欄位（step 陣列）。

### failure_policy

| 值 | 說明 |
|----|------|
| `fail_fast` | 任一 branch 失敗 → 取消其他 branches，parallel FAILED |
| `wait_all` | 等待所有 branches 完成，若有任一失敗 → parallel FAILED |

### output

`steps.<parallel_id>.output` 為一個 array，索引對應 `branches` 的順序：

- 成功的 branch：該 branch 最後一個 step 的 output
- 失敗的 branch（`failure_policy` 為 `wait_all` 且 `on_error` 已處理）：`null`

### 範例

```yaml
- id: finalize
  type: parallel
  failure_policy: wait_all
  branches:
    - steps:
        - type: task
          action: notify.customer
          when: ${ input.notify_customer == true }
          input:
            order_id: ${ steps.load_order.output.id }

    - steps:
        - id: generate_report
          type: task
          action: report.generate
          input:
            order_id: ${ steps.load_order.output.id }
```

---

## 巢狀限制

- 控制流程 steps MAY 任意巢狀（如 parallel 內含 foreach、foreach 內含 if）
- 引擎 SHOULD 限制最大巢狀深度（建議上限 10 層），超過時在 validation 階段報錯
- 巢狀 steps 中有指定 `id` 的 step，其 id 仍 MUST 全域唯一
