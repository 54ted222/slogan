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

### YAML 與 CEL 的 escape 交互

- YAML 先一層 unescape（依 scalar 類型：plain / single-quoted / double-quoted / block），再由引擎掃 `${ }` 解析內部 CEL
- CEL 字串字面值使用 `"..."` 或 `'...'`，escape 規則為 CEL 標準（`\"`、`\\`、`\n` 等）
- 當 CEL 字串含 YAML 特殊字元（`:`、`#`、`{}` 等）時 SHOULD 以 block scalar (`|` / `>`) 避免雙層 escape 地獄：

  ```yaml
  # 推薦：block scalar 包裹含引號的 CEL
  query: |
    ${ "SELECT * FROM orders WHERE status = \"pending\"" }

  # 避免：plain scalar 中混用 YAML 與 CEL escape
  # query: ${ "SELECT * FROM orders WHERE status = \"pending\"" }  # YAML 解析歧義
  ```

- 多行 CEL 表達式：YAML 的 `|` block scalar 保留換行，但 CEL 解析前會先將換行視為 whitespace（CEL 語法允許跨行）。建議將長表達式拆分為 `assign` 中間變數以提升可讀性
- 單一 `${ }` 佔整值時型別保留；若 `${ }` 外有任何其他字元（含空白），整值強制轉 string

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

**Workspace 目錄結構**（由 engine 管理，每個 instance 獨立）：

```
<engine.workspace_root>/
  <instance_id>/
    artifacts/
      <artifact_name>/         # 每個 artifact 一個子目錄
        <files>
    _shared/                   # 跨 step 共享暫存；tool 可用 artifacts._workspace_path 寫入
```

- `artifacts._workspace_path` = `<engine.workspace_root>/<instance_id>`（絕對路徑）
- `artifacts.<name>.path` = `<workspace_path>/artifacts/<name>`（絕對路徑）
- 名稱合法字元：`^[a-z][a-z0-9_-]*$`；含 `/` / `..` / 其他特殊字元 → 載入驗證失敗
- Engine MUST 在 tool spawn 前檢查路徑 canonical 化後仍在 `<workspace_path>` 內（防 `../` escape）
- Instance 終結時 engine MAY 根據 artifact 的 `retain` 設定刪除 workspace（見 artifact DSL，v6 保留為 implementation detail）

**Artifact lifecycle（v6 最小實作）**：

| 階段 | 行為 |
|------|------|
| Instance PENDING → RUNNING | Engine 建立 `<workspace_root>/<instance_id>/` 目錄（mode 0700） |
| First tool 引用 `artifacts.<name>.path` | Engine 建立 `<workspace_path>/artifacts/<name>/` 子目錄（lazy） |
| Step 執行中 | Tool 可讀寫 `artifacts.<name>.path` 下檔案；engine 不介入內容 |
| `artifacts.<name>.exists` 求值 | 檢查該 artifact 目錄存在且至少含一個檔案（非空目錄視為 exists）— **snapshot 語意**：僅保證求值瞬間結果；不阻止後續刪除。若欲安全讀檔請直接在 tool 內打開檔案並處理 `ENOENT` |
| Instance 進入終態（SUCCEEDED / FAILED / CANCELLED） | Engine 即刻對 `<instance_id>/` workspace 加**寫鎖**；此後所有 tool spawn 以唯讀 mount 形式掛入（`ro`），`catch` handler 仍可讀取 artifact 但**不得寫入** |
| Instance 終結 | 依 `retention_until` 時刻清理整個 `<instance_id>/` 目錄（跟隨 instance 保存期；見 `08-persistence.md`） |
| Instance CANCELLED 或 FAILED | 保留至保存期（預設 30 天 FAILED / 7 天 CANCELLED），便於 debug |

- Artifact 不跨 instance 共享；每 instance 獨立 workspace
- 複雜 artifact（宣告式 artifact definition、retain policy、遠端 source）為未來版本功能（見 `FUTURE.md`），v6 僅支援 per-instance 隱式建立

**並發與生命週期規則**：

- 同一 step 執行中對 artifact 目錄的並發寫入（tool 自身產生的 I/O）engine 不仲裁；tool 自行負責檔案鎖。
- `artifacts.<name>.exists` 求值採 snapshot：若在同一 CEL 表達式中多次引用同一 artifact，求值器 MUST 在該表達式生命期內快取首次結果（避免單一表達式內部不一致）。
- Tool lifecycle `destroy` hook（見 tool spec）若需存取 artifact：MUST 在 instance 進入終態前執行並完成；`destroy` 執行期間 engine 暫緩 workspace 寫鎖切換。超過 lifecycle 宣告的 `destroy.timeout` → 強制加鎖並視 `destroy` 為失敗（記錄 observability 事件，不影響 instance 終態）。
- 清理（retention 到期刪除 workspace）MUST 以 atomic rename 至 `<workspace_root>/_pending_delete/<instance_id>/` 後再刪除；任何殘存的讀取（例如 ops 工具直接開啟檔案）不會看到半刪除狀態。

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
| `now()` | timestamp | 當前 UTC 時間（RFC 3339 Nano 精度） |
| `uuid()` | string | UUID v4 |
| `json_encode(value)` | string | 編碼為 JSON |
| `json_decode(string)` | any | 解碼 JSON |
| `default(value, fallback)` | any | null 時回傳 fallback |
| `coalesce(a, b, ...)` | any | 第一個非 null 值 |
| `has(path)` | bool | 欄位是否存在（見下方語意） |
| `timestamp(string)` | timestamp | 解析 RFC 3339 字串為 timestamp |
| `duration(string)` | duration | 解析 `5m` 等 duration string |

### Timestamp / Duration 型別

- `now()` 與 `timestamp(...)` 皆回傳 CEL **timestamp** 型別，內部 UTC；兩 timestamp 相減得 duration
- CEL 的 `.getFullYear()` / `.getMonth()` 等 accessor 以 UTC 為基準；**不提供時區參數**
- 本地時間格式化 / 時區轉換 **不在 CEL 中支援**；需要時由 tool backend 處理（如 `date` CLI、Python tool）
- Duration 比較與加減合法：`now() + duration("5m") > steps.x.ended_at`
- Timestamp 與字串的 `==` 永不匹配（型別不符）；需先 `string(now())` 才能與字面字串比較

### List 操作

| 函式 | 回傳 | 說明 |
|------|------|------|
| `list.append(elem)` | list | 尾部加入元素 |
| `list.concat(other)` | list | 串接兩個 list |
| `list.slice(start, end)` | list | 子 list |
| `list.join(sep)` | string | 串接為字串 |

所有 list 操作回傳新 list（純函式）。

### has() 語意

`has(path)` 對 map field / namespace path 檢查「欄位存在且非 null」；路徑途中任一層為 null → 回 `false`，**不拋異常**（與 CEL 標準 `has()` 對齊，但對 null 寬容處理）：

```
# steps.x 已 SUCCEEDED，output: { amount: 100 }
has(steps.x.output)         # true
has(steps.x.output.amount)  # true
has(steps.x.output.missing) # false（欄位不存在）

# steps.x 已 SKIPPED，output: null
has(steps.x)                # true（StepRef 存在）
has(steps.x.output)         # false（null 視為不存在）
has(steps.x.output.amount)  # false（short-circuit，不拋異常）

# steps.unknown_id（從未定義）
has(steps.unknown_id)       # false
has(steps.unknown_id.output)# false
```

配合 `default()` / `coalesce()` 做防禦性存取：

```yaml
amount: ${ has(steps.x.output.amount) ? steps.x.output.amount : 0 }
```

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
| `input` / `steps` / `prev` / `vars` / `loop` / `event` / `error` | ❌ | 非 step context |

init 於 tool 首次被 step 呼叫前執行、每個 workflow instance 執行一次；destroy 於 instance 結束時執行。init 失敗 → 依賴此 tool 的 step 進入 FAILED，`error.type == "lifecycle_init_failed"`。
