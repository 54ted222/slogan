# 03 — Expressions

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

某些欄位（如 `when`、`match`、`when`）本身即預期為 CEL 表達式，在這些欄位中也 MUST 使用 `${ }` 定界符：

```yaml
- id: check
  type: if
  when: ${ steps.load_order.output.status == "cancelled" }

- id: wait
  type: wait
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

已完成 step 的輸出與狀態。

**Output 存取**：`steps.<step_id>.output.<field>`

```
steps.load_order.output.id
steps.load_order.output.amount
steps.create_payment.output.payment_id
```

**Status 存取**：`steps.<step_id>.status`

| 值 | 說明 |
|----|------|
| `"SUCCEEDED"` | step 成功完成 |
| `"FAILED"` | step 失敗（已被 catch 處理，workflow 繼續） |
| `"SKIPPED"` | step 被跳過（when 為 false 或分支未選中） |
| `null` | step 尚未執行 |

```yaml
# 使用 status 檢查 step 是否已執行
- type: if
  when: ${ steps.load_order.status == "SUCCEEDED" }
  then: [...]
```

可用時機：僅限該 step 之後的 steps。引擎 MUST 確保被參照的 step 在當前 step 之前已執行完成。

**非 SUCCEEDED step 的 output**：

- **SKIPPED**（`when` 為 false 或所在分支未被選中）：`steps.<id>.output` 回傳 `null`
- **FAILED**（錯誤已被 `catch` handler 處理，workflow 繼續執行）：`steps.<id>.output` 回傳 `null`

使用 `default()` 做防禦性存取：

```
default(steps.process_payment.output.payment_id, "")
```

### prev

前一個 step 的輸出與狀態。使用 `prev` 可免去為 step 定義 `id` 僅為了參照其輸出。

**Output 存取**：`prev.output.<field>`

```
prev.output.id
prev.output.amount
prev.output.status
```

**Status 存取**：`prev.status`

```yaml
steps:
  - type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  # 不需要知道前一個 step 的 id
  - type: task
    action: order.process
    input:
      amount: ${ prev.output.amount }

  - type: if
    when: ${ prev.output.success }
    then:
      - type: emit
        event: order.completed
```

`prev` 為**靜態綁定**——永遠指向 YAML 語法上的前一個 step，不因執行狀態（如 SKIPPED）而改變指向。

| 情境 | `prev` 指向 |
|------|------------|
| steps 陣列中的第 N 個 step（N > 0） | 第 N-1 個 step（語法位置，固定不變） |
| steps 陣列中的第 0 個 step | `null`（不可用） |
| `then` / `else` / `do` 等巢狀陣列中的第 0 個 step | `null`（不可用） |
| 前一個 step 為 SKIPPED | `prev.output` 回傳 `null`，`prev.status` 回傳 `"SKIPPED"` |
| 前一個 step 為 FAILED（已被 handler 處理） | `prev.output` 回傳 `null`，`prev.status` 回傳 `"FAILED"` |

> **注意**：`prev` 僅指向同一個 step 陣列中語法上的前一個 step，不會跨越巢狀邊界（如不會從 `then` 陣列指向 `if` step 本身），也不會因為前一個 step 被跳過而自動指向更前面的 step。

---

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
| `loop.item` | any | 當前迭代的元素 |
| `loop.index` | int | 當前迭代的索引（從 0 開始） |

可用時機：僅限 `foreach` 的 `do` 區塊內。

### session

Agent 自訂 loop 的 session 資料。

| 變數 | 型別 | 說明 |
|------|------|------|
| `session.id` | string | Agent session 唯一識別符（`asess_<uuid>`） |
| `session.messages` | array | 當前對話紀錄（message 陣列） |
| `session.iteration` | integer | 當前迭代次數（從 0 開始） |
| `session.tool_call_count` | integer | 累計 tool 呼叫次數 |

可用時機：僅限 agent definition 的 `loop.steps` 內、以及 agent 的 `prompt` 與 `system_prompt` 中。詳見 [16-agent-loop](../03-agent/16-agent-loop.md)。

### hook

Agent hook 的觸發上下文。

可用時機：僅限 agent definition 的 `hooks` 區塊 steps 內。每個 hook 觸發點提供不同的 `hook.*` 變數。詳見 [17-agent-hooks-and-streaming](../03-agent/17-agent-hooks-and-streaming.md)。

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
- `wait`（event 模式）的 `match` 欄位：事件比對時
- `wait` 完成後的後續 steps：透過 `steps.<wait_step_id>.output`

### artifacts

Artifact 的 workspace 路徑與中繼資料。

```
artifacts._workspace_path
artifacts.order_file.path
artifacts.order_file.size
artifacts.order_file.content_type
artifacts.order_file.content
artifacts.order_file.access
artifacts.order_file.exists
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

可用時機：所有 steps、task definition、agent definition 的 `system_prompt`、Resources definition 的 MCP server `env`。值一律為 `string`，不存在時回傳 `null`。

詳見 [25-secrets-and-env](../04-resources/25-secrets-and-env.md)。

### error（限 catch handler 內）

錯誤資訊，僅在 `catch` handler 的 steps 中可用。

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

## 存在性檢查

使用 CEL 原生 `has()` 巨集檢查 map field 是否存在：

```yaml
# 檢查 output 中是否有 address 欄位
- type: if
  when: ${ has(steps.load_order.output.address) }
  then: [...]
```

使用 `steps.<id>.status` 檢查 step 是否已執行：

```yaml
# 檢查 step 是否已執行且成功
- type: if
  when: ${ steps.load_order.status == "SUCCEEDED" }
  then: [...]
```

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

### CEL ↔ JSON/YAML 型別映射

| CEL 型別 | JSON 型別 | YAML 型別 |
|----------|----------|----------|
| `int` / `uint` | number | integer |
| `double` | number | float |
| `bool` | boolean | boolean |
| `string` | string | string |
| `list` | array | array |
| `map` | object | map |
| `null_type` | null | null |
| `timestamp` | string (ISO 8601) | string |
| `duration` | string (e.g. "30m") | string |

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

### 通用函式

| 函式 | 回傳 | 說明 |
|------|------|------|
| `now()` | timestamp | 當前 UTC 時間 |
| `uuid()` | string | 產生 UUID v4 |
| `json_encode(value)` | string | 將值編碼為 JSON 字串 |
| `json_decode(string)` | any | 將 JSON 字串解碼為值 |
| `coalesce(a, b, ...)` | any | 回傳第一個非 null 的值 |
| `default(value, fallback)` | any | 若 value 為 null 則回傳 fallback |

### List 操作函式

| 函式 | 回傳 | 說明 |
|------|------|------|
| `list.append(element)` | list | 回傳新 list，尾部加入一個元素 |
| `list.concat(other_list)` | list | 回傳新 list，串接兩個 list |
| `list.slice(start, end)` | list | 回傳子 list（`start` 含、`end` 不含），索引從 0 開始 |
| `list.join(separator)` | string | 將 `list<string>` 以 separator 串接為字串 |

```yaml
# append — 加入單一元素
messages: ${ session.messages.append(steps.response.output.message) }

# concat — 串接兩個 list
messages: ${ session.messages.concat(steps.tool_results.output.results) }

# slice — 取子 list（保留前 10 筆）
trimmed: ${ session.messages.slice(0, 10) }

# 組合使用
messages: ${ session.messages.slice(0, 1).concat(session.messages.slice(-3, session.messages.size())) }
```

所有 list 操作函式皆回傳**新 list**，不修改原 list（純函式語意）。

---

## 求值語意

1. **純函式**：CEL 表達式求值 MUST 為純函式，不產生副作用（`now()` 和 `uuid()` 除外）
2. **求值失敗**：表達式求值失敗（型別錯誤、null reference 等）→ 所在 step 狀態變為 FAILED
3. **Null 傳播**：存取不存在的欄位回傳 `null`；對 null 呼叫方法將導致求值失敗。使用 `has()` 或 `default()` 做防禦性存取
4. **求值時機**：表達式在 step 進入 RUNNING 狀態時求值，不提前求值

---

## 邊界行為

### Null 上呼叫函式

| 表達式 | 結果 |
|--------|------|
| `null.size()` | 求值失敗 → step FAILED |
| `size(null)` | 求值失敗 → step FAILED |
| `null + "text"` | 求值失敗 → step FAILED |
| `null == null` | `true` |
| `null != "hello"` | `true` |
| `has(obj.missing_field)` | `false`（安全） |
| `default(null, "fallback")` | `"fallback"`（安全） |
| `coalesce(null, null, "x")` | `"x"`（安全） |

建議：對可能為 null 的值使用 `has()`、`default()` 或 `coalesce()` 防禦。

### 數值精度

| 型別 | 精度 | 說明 |
|------|------|------|
| `int` | 64-bit signed integer | 範圍：-2^63 ~ 2^63-1 |
| `uint` | 64-bit unsigned integer | 範圍：0 ~ 2^64-1 |
| `double` | IEEE 754 64-bit | 與 JSON number 相容 |

- 整數溢位 → 求值失敗
- 浮點運算遵循 IEEE 754 語意（含 `NaN`、`Infinity`）
- JSON 反序列化的數字：無小數點 → `int`，有小數點 → `double`

### 字串編碼

- 所有字串 MUST 為 UTF-8 編碼
- `s.size()` 回傳 Unicode code point 數量，非 byte 數量
- 字串比較為 Unicode code point 逐一比較

### List / Map 迭代順序

| 容器 | 順序保證 |
|------|---------|
| `list` | 保持插入順序（索引順序） |
| `map`（來自 YAML） | 保持 YAML 定義順序 |
| `map`（CEL 字面值 `{"a": 1, "b": 2}`） | 保持定義順序 |

List 的 `filter()`、`map()` 等巨集 MUST 保持原始順序。

### 求值順序

當 step 有多個欄位含 CEL 表達式時（如 `input` map 的多個 key）：

- 引擎 MUST 在同一時間點求值所有表達式（語意上為原子）
- 同一 map 內各表達式的求值順序為**未定義**（implementation-defined）
- 表達式之間 MUST NOT 互相依賴（同一 step 內的表達式不可參照彼此的結果）
- `when` 表達式 MUST 在其他欄位的表達式之前求值（when 為 false 時不求值其他欄位）
