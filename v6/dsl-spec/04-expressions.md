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

### 值語意（value semantics）

所有 namespace 中的資料（`input` / `steps` / `vars` / `loop.item` / `event.data` / ...）在 CEL 求值時遵循**值語意（deep copy by value）**，非引用：

- CEL 讀取 `input.items[0].price` → 取的是當下 snapshot 的拷貝；後續 `input` 若因實作細節變動（不應發生，但並行場景中可能），已取得的值不受影響
- `assign vars.items: ${ input.product_list }` → `vars.items` 為 `input.product_list` 的 deep copy；後續對 `vars.items[0].price = 99` 不影響 `input.product_list`（事實上 assign 只能整體覆寫 `vars.items`，沒有部分更新語法；此處僅示意值獨立）
- foreach 並行迭代中，`loop.item` 為各迭代**獨立**的值拷貝；對 `loop.item` 的讀取不會競爭
- 深層結構寫入 `vars.foo.bar.baz` 的語法不存在（`assign` 只支援頂層 key 替換）；若需更新深層請先 `assign vars.foo: ${ merge(vars.foo, {bar: {baz: new}}) }`（目前 v6 無內建 `merge`，需透過 CEL 手動組 map）
- 實作上 engine MAY 以 copy-on-read 或 structural sharing 優化，但對使用者呈現的語意恆為 deep copy

### vars

Instance 級變數空間；僅 `type: assign` 可寫入。

- 建立 instance 時初始為 **空 map `{}`**（非 null）；`has(vars) == true`、`vars.size() == 0`、`has(vars.anything) == false`
- 未寫入的 key 讀取：`vars.foo` → `expression_error.identifier_not_found`；使用 `has(vars.foo)` 或 `default(vars.foo, <fallback>)` 防禦
- 子 function instance **不繼承**父 vars；函式呼叫邊界即為 vars 邊界
- 寫入語意見 `dsl-spec/03-steps.md` 的 `type: assign`；並行寫入規則見 `runtime-spec/10-concurrency.md`
- v6 不支援 `vars_schema` 驗證（使用者自行保證型別）；相關功能延後至未來版本

範例：

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

| 路徑 | 型別 | 說明 |
|------|------|------|
| `event.id` | string | 事件 UUID v4（訂閱端去重用） |
| `event.type` | string | 事件類型（dotted snake_case） |
| `event.data` | any | 業務 payload；結構由 emit 端決定 |
| `event.scope` | string | `workflow` / `project` / `global` |
| `event.timestamp` | timestamp | emit 時刻（RFC 3339 Nano） |
| `event.source.instance_id` | string | 發出 emit 的 instance id |
| `event.source.step_id` | string \| null | 發出 emit 的 step id（見 `runtime-spec/07-event-bus.md` 「source 於特殊上下文的歸屬」） |
| `event.source.project` | string | 發出 instance 的 project name（根目錄為 `""`） |
| `event.source.sequence` | int | 同源全序序號；同 instance 內的 emit 嚴格遞增 |
| `event.trace_id` | string | 跨 instance 追蹤鏈 id |

```
event.type                              # "order.created"
event.data.order_id                     # 業務欄位
event.source.step_id                    # "emit_step_id"
event.source.instance_id == context.instance_id   # 是否來自同一 instance
```

**可變性**：`event.*` 在 match 求值期間為不可變 snapshot；CEL 不可寫。

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

### callback

僅於 caller 的 `type: task` 之 `callback:` handler 內可用（以 `steps.<handler_id>.<...>` 等呼叫端 namespace 共存）。提供當前 callback 的輸入與識別資訊：

| 路徑 | 型別 | 說明 |
|------|------|------|
| `callback.input` | any | function 端 `type: callback` step 傳入的 input |
| `callback.name` | string | 觸發的 callback 名稱（同一 task 多種 callback 共用 handler 時可分流） |
| `callback.call_id` | string | 本次 callback 的唯一 id（跨 handler attempt 一致） |

於 function 端（`type: callback` step 外部的 function body）**不可見**；其 namespace 與使用端同 workflow 的其他 steps 共享（見 `05b-function.md` 的 callback 小節）。

### context

僅於 Tool stdin / http template 求值時可用（不出現在 workflow step 的一般 CEL）。提供本次 tool 呼叫的執行上下文：

| 路徑 | 型別 | 說明 |
|------|------|------|
| `context.instance_id` | string | 當前 workflow / function instance id |
| `context.step_id` | string | 呼叫 tool 的 step id |
| `context.attempt` | int | 本次 attempt 計數（首次 = 1） |
| `context.idempotency_key` | string | 依 `(instance_id + step_id + attempt + input_snapshot)` 計算的 hash；與 `05-tool.md` protocol mode stdin JSON 的 `context.idempotency_key` 同源 |
| `context.trace_id` | string | W3C 相容 trace id（見 `runtime-spec/07-event-bus.md` 的 trace 傳播） |

與 protocol mode stdin JSON 的 `context` 結構欄位一致；tool 開發者可以 CEL 取同一資料集、也可於 stdin JSON 中直接讀。

### project

當前 workflow / tool / function 所屬 project 的 metadata（read-only）。根目錄下無 project 的 definition 亦可存取，此時 `project` 為預留的 null-object（所有欄位為空字串 / 空 map）。

| 路徑 | 型別 | 說明 |
|------|------|------|
| `project.name` | string | project `metadata.name`（kebab-case）；根目錄 definition 為 `""` |
| `project.path` | string | 從根目錄起的 project 路徑；巢狀如 `"order/domestic"`；根目錄 definition 為 `""` |
| `project.labels` | map<string, string> | 合併後的 `defaults.labels`（含父 project 繼承與自身覆寫，見 `06-project-and-secret.md`） |
| `project.owner` | string | `metadata.owner`（若 project 宣告）；否則 `""` |

CEL 範例：

```
project.name                           # "order"
project.labels.domain                  # "order"
has(project.labels.team)               # true
```

適用：所有 CEL 求值位置（workflow step、tool backend、callback handler、lifecycle init / destroy）；屬「載入期常量」語意，instance 內不變動。

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
| Workflow `config.catch` / step `catch` 執行中 | **寫鎖尚未加上**；catch handler 內的 task step 仍可讀寫 artifact（catch 為「仍在 RUNNING 的終結前處理」） |
| Instance 真正進入終態（catch 消化或上拋完成後） | Engine 對 `<instance_id>/` workspace 加**寫鎖**；此後 lifecycle destroy hook / 背景清理以唯讀（`ro` mount）存取 artifact；destroy 若嘗試寫入 → 該 destroy 被視為失敗（記錄 `lifecycle.destroy_failed`，不影響終態） |
| Instance 終結 | 依 `retention_until` 時刻清理整個 `<instance_id>/` 目錄（跟隨 instance 保存期；見 `08-persistence.md`） |
| Instance CANCELLED 或 FAILED | 保留至保存期（預設 30 天 FAILED / 7 天 CANCELLED），便於 debug |

- Artifact 不跨 instance 共享；每 instance 獨立 workspace
- 複雜 artifact（宣告式 artifact definition、retain policy、遠端 source）為未來版本功能（見 `FUTURE.md`），v6 僅支援 per-instance 隱式建立

**Function instance 的 workspace**：

- Function instance（由 `type: task` 呼叫 function 建立）擁有**獨立** workspace（以其自身 `instance_id` 為路徑前綴）；**不繼承**父 workflow / 父 function instance 的 artifact 目錄
- 父子 instance 間若需共享檔案：顯式於 function `input` 傳遞**父 workspace 下某檔案的路徑字串**（或檔案內容本身），由 function 內部 tool 讀取該絕對路徑。此讀取路徑位於父 workspace 內，**不受** function 自身 workspace 的 `working_dir` canonical 檢查攔截（`working_dir` 檢查僅限制 tool 的 CWD，不限制 tool 讀寫的檔案路徑）
- 或改以 `type: emit` + `type: wait` 以事件傳遞資料（適合小量 metadata；大量二進位資料仍建議以共享檔案系統路徑傳遞）
- 父 instance 終結時，engine 依 retention 清理其自身 workspace；**子 function instance 的 workspace 獨立 retention**（跟子 instance 的終態走，不受父 instance 退場影響），但子 instance 本身「跟父走」的 retention 規則（見 `runtime-spec/08-persistence.md`）在父清理時一併清理子 instance 與其 workspace

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
| `s.size()` | int | 字串 Unicode codepoint 數（**非** byte 數） |

### 集合

| 函式 | 回傳 | 說明 |
|------|------|------|
| `list.exists(x, expr)` | bool | 存在元素滿足條件 |
| `list.all(x, expr)` | bool | 所有元素滿足條件 |
| `list.filter(x, expr)` | list | 過濾 |
| `list.map(x, expr)` | list | 映射 |
| `list.size()` | int | 長度 |

#### Lambda 變數作用域

`exists` / `all` / `filter` / `map` 的第一個參數為 lambda 局部變數（如 `x`）：

- lambda 變數**遮蔽**外層同名 namespace；例：`list.map(steps, ...)` 內 `steps` 指 lambda 變數（元素），不再指向 `steps.*` namespace
- lambda 內仍可透過**不同名稱**存取外層 namespace；遮蔽僅發生在同名情況
- **載入期警告**：若 lambda 變數名與 CEL 保留 namespace（`input` / `steps` / `prev` / `vars` / `loop` / `event` / `env` / `secret` / `error` / `callback` / `context` / `project` / `artifacts`）相同 → 產生 lint warning 但不拒絕；建議使用 `it` / `elem` / `x` 等短名避免誤解
- lambda 不能跨呼叫共享狀態（純函式）；連續 `list.filter(x, ...).map(y, ...)` 彼此獨立

#### size() 的通用型別語意

`size()` 統一規則（與 CEL 標準一致）：

| 對象型別 | 回傳 |
|---------|------|
| string | Unicode codepoint 數 |
| list | 元素數 |
| map / object | top-level key 數 |
| bytes（若出現） | byte 數 |
| null | `expression_error.type_error`（不回 0，避免與 `[]` / `""` / `{}` 混淆） |
| int / double / bool / timestamp / duration | `expression_error.type_error` |

**防禦模式**：對可能為 null 的 collection 先以 `has()` 或 `default()` 保護：

```
has(input.items) ? input.items.size() : 0
default(input.items, []).size()
```

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
| `s.extractMatches(regex)` | list<string> | 正則匹配；見下 |

#### extractMatches 語意

`s.extractMatches(regex)`：以 `regex`（RE2 語法）對字串 `s` 做 **首次** 匹配（非 global）：

- 無匹配 → `[]`
- 有匹配 → list，index 0 為完整匹配字串；index 1..N 依序為 capture group（未匹配的 optional group 為 `""`）
- regex 本身語法錯誤 → `expression_error.type_error`
- 需要 global / 多次匹配請連續切分或使用 builtin tool 處理

範例：

```
"Order #42 total: $99".extractMatches("Order #(\\d+)")       # ["Order #42", "42"]
"Order #42 total: $99".extractMatches("total: \\$(\\d+)")   # ["total: $99", "99"]
"".extractMatches("foo")                                         # []
```

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

**對 SKIPPED step 的語意**：

- SKIPPED step 仍在 `steps.*` namespace；`has(steps.foo) == true`（StepRef 存在）
- SKIPPED 的 output 為 `null`；`has(steps.foo.output) == false`（null 視為不存在）
- `has(steps.foo.output.field) == false`（短路，不拋 `expression_error.identifier_not_found`）
- `has(steps.foo.status) == true`，`steps.foo.status == "SKIPPED"` 可用於分流

**對 `secret` namespace 的特例**：

- `has(secret.X)` 僅檢查 key 是否存在於當前 project scope 的 SecretAccessor，**不觸發解密**；不產生 log / trace 副作用
- 跨 project 且目標 secret 未標記 `shared: true` → `has(secret["other/X"])` **靜默回 false**（不拋 `expression_error`；維持防禦式查詢的習慣用法）；訪問受權限制的 secret 其存在性本身即屬敏感資訊，故以「不存在」呈現
- `has(secret)` 整體回 `true`（SecretAccessor 本身恆存在，代表 namespace 可用）

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
