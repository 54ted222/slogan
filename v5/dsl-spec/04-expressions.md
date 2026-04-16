# 04 — Expressions

本文件定義 CEL 表達式在 workflow DSL 中的語法、求值上下文與可用函式。

---

## 定界符

所有 CEL 表達式 MUST 使用 `${ }` 定界符：

```yaml
# CEL 表達式
order_id: ${ input.order_id }
full_name: ${ input.first_name + " " + input.last_name }

# 字串插值（一個值中多個 ${ }）
message: "Order ${ input.order_id } is ${ steps.check.output.status }"

# 單一 ${ } 保留原始型別（不強制轉字串）
amount: ${ steps.load.output.amount }    # integer
```

---

## 求值上下文（Namespaces）

### input

Workflow instance 的輸入資料。所有 steps 可用。

```
input.order_id
input.items[0].sku
```

### steps

已完成 step 的輸出與狀態。僅限該 step 之後的 steps。

```
steps.load_order.output.id          # output 存取
steps.load_order.status             # "SUCCEEDED" | "FAILED" | "SKIPPED" | null
```

### prev

語法上前一個 step 的輸出（靜態綁定，不因 SKIPPED 改變指向）。

```yaml
- type: task
  action: order.load
  input: { order_id: ${ input.order_id } }

- type: task
  action: order.process
  input: { amount: ${ prev.output.amount } }   # 不需要前一步的 id
```

當前一步被 SKIPPED：

| 存取 | 結果 |
|------|------|
| `prev.status` | `"SKIPPED"` |
| `prev.output` | `null` |
| `prev.output.<field>` | 求值異常 → 若欄位為 CEL 則該 step FAILED |

需要「最近一個 SUCCEEDED 步驟」的輸出時，以 `steps.<id>.output` 具名引用。防禦性寫法：

```yaml
- type: task
  action: order.process
  when: ${ prev.status == "SUCCEEDED" }     # 上一步未跳過才執行
  input: { amount: ${ prev.output.amount } }

# 或：上一步 SKIPPED 時帶預設值
- type: task
  action: order.process
  input: { amount: ${ default(prev.output, {amount: 0}).amount } }
```

### vars

由 `assign` step 設定的變數。

```
vars.payment_input
vars.is_pay_action
```

### loop

`foreach` 迴圈內可用。

| 變數 | 說明 |
|------|------|
| `loop.item` | 當前迭代元素 |
| `loop.index` | 索引（從 0） |

### session

Agent 自訂 loop 內可用。

| 變數 | 說明 |
|------|------|
| `session.id` | session 唯一識別符 |
| `session.messages` | 當前對話紀錄 |
| `session.iteration` | 迭代次數 |
| `session.tool_call_count` | 累計 tool 呼叫次數 |

### event

事件資料。在 trigger `when`、wait `match`、wait 後續 steps 中可用。

```
event.type
event.data.order_id
event.id
event.timestamp
```

### env

OS 環境變數 / `.env` 檔中的值。所有 steps 可用，值為 `string`，不存在時 `null`。

```
env.DATABASE_HOST
env.API_BASE_URL
```

### secret

加密機密檔案中的值。所有 steps 可用，值為 `string`，不存在時 `null`。

```
secret.PAYMENT_API_KEY
secret.STRIPE_SECRET
```

### error

在 `catch` handler 中可用。透過 `error.type` 區分錯誤類型（如 `"timeout"`、`"step_error"`）。見 [03-steps](03-steps.md) 錯誤處理模型。

### artifacts

Artifact 的 workspace 路徑與中繼資料。所有 steps 可用。

```
artifacts._workspace_path
artifacts.order_file.path
artifacts.order_file.exists
```

---

## 標準函式

### 字串

| 函式 | 回傳 | 說明 |
|------|------|------|
| `s.contains(sub)` | bool | 包含子字串 |
| `s.startsWith(prefix)` | bool | 以 prefix 開頭 |
| `s.endsWith(suffix)` | bool | 以 suffix 結尾 |
| `s.matches(regex)` | bool | 正則比對 |
| `s.size()` | int | 字串長度 |

### 集合

| 函式 | 回傳 | 說明 |
|------|------|------|
| `list.exists(x, expr)` | bool | 存在元素滿足條件 |
| `list.all(x, expr)` | bool | 所有元素滿足條件 |
| `list.filter(x, expr)` | list | 過濾 |
| `list.map(x, expr)` | list | 映射 |
| `list.size()` | int | 長度 |

### 擴充函式

| 函式 | 回傳 | 說明 |
|------|------|------|
| `now()` | timestamp | 當前 UTC 時間 |
| `uuid()` | string | UUID v4 |
| `json_encode(value)` | string | 編碼為 JSON |
| `json_decode(string)` | any | 解碼 JSON |
| `default(value, fallback)` | any | null 時回傳 fallback |
| `coalesce(a, b, ...)` | any | 第一個非 null 值 |
| `has(field)` | bool | 欄位是否存在 |

### List 操作

| 函式 | 回傳 | 說明 |
|------|------|------|
| `list.append(elem)` | list | 尾部加入元素 |
| `list.concat(other)` | list | 串接兩個 list |
| `list.slice(start, end)` | list | 子 list |
| `list.join(sep)` | string | 串接為字串 |

所有 list 操作回傳新 list（純函式）。

---

## 求值語意

- **純函式**：表達式求值不產生副作用（`now()`、`uuid()` 除外）
- **求值失敗**：型別錯誤、null reference、引用不存在的識別字（`steps.<unknown_id>`）→ 所在 step FAILED，`error.type == "expression_error"`，`error.message` 包含失敗的表達式片段
- **Null 安全**：用 `has()`、`default()`、`coalesce()` 做防禦性存取
- **求值時機**：step 進入 RUNNING 狀態時求值
- **`when` 優先**：`when` 為 false → step SKIPPED，不求值其他欄位；`when` 求值異常 → step FAILED

---

## Lifecycle 鉤子的求值上下文

Tool definition 的 `lifecycle.init` / `lifecycle.destroy` 在 tool 首次使用前 / tool 釋放時執行，不隸屬於任何 step，求值上下文受限：

| Namespace | 可用 | 說明 |
|-----------|------|------|
| `secret` | ✅ | 加密機密 |
| `env` | ✅ | OS / .env |
| `project` | ✅ | Project-level 設定 |
| `artifacts._workspace_path` | ✅ | workspace 根目錄 |
| `input` / `steps` / `prev` / `vars` / `loop` / `event` / `error` / `session` | ❌ | 非 step context |

init 於 tool 首次被 step 呼叫前執行、每個 workflow instance 執行一次；destroy 於 instance 結束時執行。init 失敗 → 依賴此 tool 的 step 進入 FAILED，`error.type == "lifecycle_init_failed"`。
