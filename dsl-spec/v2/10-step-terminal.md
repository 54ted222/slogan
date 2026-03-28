# 10 — Steps: fail / return

本文件定義兩種終止 workflow 的 step 類型。

---

## fail

立即終止 workflow instance，標記為 FAILED。

### Schema

```yaml
- type: fail
  message: string | CEL expression    # MUST — 錯誤訊息
  code: string                        # MAY — 機器可讀的錯誤碼
```

### 行為

- 立即終止 workflow instance，無論 `fail` 出現在哪個層級（巢狀 if、switch、on_error 等）
- Instance 狀態變為 FAILED
- `message` 與 `code` 記錄在 instance 的錯誤資訊中
- 後續 steps 不會執行

### 範例

```yaml
- type: fail
  message: "order has been cancelled"
  code: "order_cancelled"

- type: fail
  message: ${ "unknown action: " + input.action }
  code: "invalid_action"
```

---

## return

結束 workflow instance，標記為 SUCCEEDED，並回傳輸出資料。

### Schema

```yaml
- type: return
  output: map | CEL expression    # MAY — 回傳的資料
```

### 行為

- 立即終止 workflow instance，狀態變為 SUCCEEDED
- `output` 會依照 `output.schema` 驗證（若有定義）
- 驗證失敗 → instance 狀態變為 FAILED（錯誤碼 `schema_validation_error`）
- 後續 steps 不會執行
- 若 `output` 未指定，回傳 `null`

### 範例

```yaml
- type: return
  output:
    order_id: ${ steps.load_order.output.id }
    status: "completed"
    payment_id: ${ steps.create_payment.output.payment_id }
```

---

## 提前結束語意

`fail` 和 `return` 都是 **立即終止** 指令：

- 無論出現在哪個巢狀層級（if branch、switch case、foreach iteration、parallel branch），都會終止整個 workflow instance
- 不會自動執行任何 cleanup 或 compensation 邏輯
- 若需要在失敗時進行清理，SHOULD 使用 `on_error` handler（見 [12-error-handling](12-error-handling.md)）

### 隱式完成

若 workflow 的 `steps` 陣列全部執行完畢，且沒有遇到 `return` 或 `fail`：

- Instance 狀態變為 SUCCEEDED
- Output 為 `null`
- 若有定義 `output.schema` 且 schema 有 `required` 欄位 → FAILED（`schema_validation_error`）
