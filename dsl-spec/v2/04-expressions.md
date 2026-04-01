# 04 — Expressions

本文件定義 CEL 表達式在 workflow DSL 中的語法、求值上下文與可用函式。

---

## 表達式語法

本 DSL 的表達式語言為 **CEL** (Common Expression Language, [cel.dev](https://cel.dev))。

### 定界符

所有 CEL 表達式 MUST 使用 `${ }` 定界符包裹：

```yaml
# ✅ 正確：CEL 表達式
order_id: ${ input.order_id }
is_pay: ${ input.action == "pay" }
full_name: ${ input.first_name + " " + input.last_name }

# ✅ 正確：純字串（無 ${ }）
label: "hello world"
status: "completed"
```

### 規則

- YAML 值中出現 `${ ... }` 時，引擎 MUST 將其內容解析為 CEL 表達式並求值
- 不含 `${ }` 的 YAML 值視為純字串或 YAML 原生型別（number、boolean 等）
- `${ }` 內的空白不影響語意：`${ input.x }` 與 `${input.x}` 等價
- 一個 YAML 值中 MAY 包含多個 `${ }`，引擎會分別求值後串接為字串：

```yaml
message: "Order ${ input.order_id } is ${ steps.check.output.status }"
```

- 若整個 YAML 值是單一 `${ }` 表達式，求值結果保留原始型別（不強制轉為字串）：

```yaml
# 結果為 integer（非字串）
amount: ${ steps.load.output.amount }

# 結果為 boolean
should_notify: ${ input.notify_customer == true }
```

### 特定欄位的表達式

某些欄位（如 `expr`、`match`、`when`）本身即預期為 CEL 表達式，在這些欄位中也 MUST 使用 `${ }` 定界符：

```yaml
- id: check
  type: if
  expr: ${ steps.load_order.output.status == "cancelled" }

- id: wait
  type: wait_event
  event: payment.confirmed
  match: ${ event.data.order_id == input.order_id }
```

---

## 求值上下文（Namespaces）

CEL 表達式在求值時可存取以下 namespaces：

### input

Workflow instance 的輸入資料。

```
input.order_id
input.items[0].sku
input.notify_customer
```

可用時機：所有 steps。

### steps

已完成 step 的輸出。格式為 `steps.<step_id>.output.<field>`。

```
steps.load_order.output.id
steps.load_order.output.amount
steps.create_payment.output.payment_id
```

可用時機：僅限該 step 之後的 steps。引擎 MUST 確保被參照的 step 在當前 step 之前已執行完成。

### vars

由 `assign` step 設定的變數。

```
vars.payment_input
vars.payment_input.amount
vars.is_pay_action
```

可用時機：`assign` step 之後的 steps。後續 `assign` 可覆寫同名變數。

### loop

`foreach` 迴圈內的迭代資料。

| 變數 | 型別 | 說明 |
|------|------|------|
| `loop.<as>` | any | 當前迭代的元素（`<as>` 為 `foreach` 的 `as` 欄位值，預設為 `item`） |
| `loop.index` | int | 當前迭代的索引（從 0 開始） |

```
# as: item（預設）
loop.item.sku
loop.item.qty

# as: order
loop.order.sku
loop.order.qty

# index 不受 as 影響
loop.index
```

可用時機：僅限 `foreach` 的 `do` 區塊內。

### event

事件資料。

```
event.type
event.data.order_id
event.data.source
event.id
event.timestamp
```

可用時機：
- `trigger` 的 `when` 欄位：事件觸發時
- `wait_event` 的 `match` 欄位：事件比對時
- `wait_event` 完成後的後續 steps：透過 `steps.<wait_step_id>.output`

### artifacts

Artifact 的中繼資料。

```
artifacts.order_file.uri
artifacts.order_file.size
artifacts.order_file.content_type
```

可用時機：所有 steps。

### env

OS 環境變數 / `.env` 檔中的值。

```
env.DATABASE_HOST
env.API_BASE_URL
env.LOG_LEVEL
```

可用時機：所有 steps 與 task definition。值一律為 `string`，不存在時回傳 `null`。

### secret

加密機密檔案（`kind: Secret`）中的值。

```
secret.PAYMENT_API_KEY
secret.STRIPE_SECRET
```

可用時機：所有 steps 與 task definition。值一律為 `string`，不存在時回傳 `null`。

詳見 [18-secrets-and-env](18-secrets-and-env.md)。

### error（限 on_error handler 內）

錯誤資訊，僅在 `on_error` handler 的 steps 中可用。

| 變數 | 型別 | 說明 |
|------|------|------|
| `error.message` | string | 錯誤訊息 |
| `error.code` | string | 錯誤碼 |
| `error.step_id` | string | 發生錯誤的 step id |

### timeout（限 on_timeout handler 內）

Timeout 資訊，僅在 `on_timeout` handler 的 steps 中可用。

| 變數 | 型別 | 說明 |
|------|------|------|
| `timeout.step_id` | string | 超時的 step id |
| `timeout.duration` | duration | 設定的 timeout 值 |

---

## 型別系統

CEL 原生型別：

| 型別 | 範例 |
|------|------|
| `int` | `42` |
| `uint` | `42u` |
| `double` | `3.14` |
| `bool` | `true`, `false` |
| `string` | `"hello"` |
| `bytes` | `b"data"` |
| `list` | `[1, 2, 3]` |
| `map` | `{"key": "value"}` |
| `null_type` | `null` |
| `timestamp` | `timestamp("2024-01-01T00:00:00Z")` |
| `duration` | `duration("30m")` |

---

## 標準函式與巨集

以下為 CEL 標準函式，引擎 MUST 支援：

### 字串函式

| 函式 | 回傳 | 說明 |
|------|------|------|
| `s.contains(sub)` | bool | 是否包含子字串 |
| `s.startsWith(prefix)` | bool | 是否以 prefix 開頭 |
| `s.endsWith(suffix)` | bool | 是否以 suffix 結尾 |
| `s.matches(regex)` | bool | 正則比對 |
| `s.size()` | int | 字串長度 |

### 集合巨集

| 巨集 | 回傳 | 說明 |
|------|------|------|
| `list.exists(x, expr)` | bool | 是否存在元素滿足條件 |
| `list.all(x, expr)` | bool | 是否所有元素滿足條件 |
| `list.exists_one(x, expr)` | bool | 是否恰好一個元素滿足條件 |
| `list.filter(x, expr)` | list | 過濾元素 |
| `list.map(x, expr)` | list | 映射元素 |
| `list.size()` | int | 陣列長度 |

### 其他標準函式

| 函式 | 回傳 | 說明 |
|------|------|------|
| `has(field)` | bool | 欄位是否存在 |
| `type(value)` | type | 取得值的型別 |
| `int(value)` | int | 轉型為 int |
| `double(value)` | double | 轉型為 double |
| `string(value)` | string | 轉型為 string |
| `timestamp(string)` | timestamp | 解析 ISO 8601 時間字串 |
| `duration(string)` | duration | 解析 duration 字串 |

---

## 擴充函式

以下為本 DSL 定義的擴充函式，引擎 MUST 支援：

| 函式 | 回傳 | 說明 |
|------|------|------|
| `now()` | timestamp | 當前 UTC 時間（非確定性，見下方 Replay 語意） |
| `uuid()` | string | 產生 UUID v4（非確定性，見下方 Replay 語意） |
| `json_encode(value)` | string | 將值編碼為 JSON 字串 |
| `json_decode(string)` | any | 將 JSON 字串解碼為值 |
| `coalesce(a, b, ...)` | any | 回傳第一個非 null 的值 |
| `default(value, fallback)` | any | 若 value 為 null 則回傳 fallback |

### 非確定性函式的 Replay 語意

`now()` 與 `uuid()` 為非確定性函式，會破壞 deterministic replay 保證。為確保「相同輸入 + 相同事件序列 = 相同結果」：

- Runtime MUST 在首次求值時記錄這些函式的回傳值至持久化儲存（見 [15-runtime-persistence](15-runtime-persistence.md) 的 `expression_cache`）
- Replay（crash recovery）時，runtime MUST 使用記錄的值，而非重新求值
- 記錄的 key 為 `workflow_instance_id + step_id + expression_position`（expression_position 為該表達式在 step 定義中的位置識別）

---

## 求值語意

1. **純函式**：CEL 表達式求值 MUST 為純函式，不產生副作用（`now()` 和 `uuid()` 除外，見「非確定性函式的 Replay 語意」）
2. **求值失敗**：表達式求值失敗（型別錯誤、null reference 等）→ 所在 step 狀態變為 FAILED
3. **Null 傳播**：存取不存在的欄位回傳 `null`；對 null 呼叫方法將導致求值失敗。使用 `has()` 或 `default()` 做防禦性存取
4. **求值時機**：表達式在 step 進入 RUNNING 狀態時求值，不提前求值
