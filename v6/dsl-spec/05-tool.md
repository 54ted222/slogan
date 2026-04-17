# 05 — Tool Definition（`kind: Tool`）

本文件定義 `kind: Tool` 的 YAML 結構。Tool definition 描述一個可被 workflow step 呼叫的工具及其執行後端。

---

## 設計動機

- **定義與使用分離**：workflow 描述「呼叫什麼」（`type: task` step），tool definition 描述「怎麼執行」
- **多 backend 支援**：同一個 tool 介面可切換不同的執行方式
- **統一命名**：兩段式 `namespace.action` 命名
- **零修改包裝**：現有程式、腳本、CLI 工具可直接作為 tool 使用，無需實作特定協議

---

## 頂層結構

```yaml
apiVersion: tool/v6
kind: Tool

metadata:
  name: order.load # MUST — snake_case + dotted 命名
  version: 3 # MUST
  description: "從資料庫載入訂單" # MAY

input_schema: object # MAY — 輸入 JSON Schema
output_schema: object # MAY — 輸出 JSON Schema

idempotent: bool # MAY, 預設 false — 是否為冪等操作

compensate: # MAY — 預設補償操作
  action: string #   MUST — 補償用的 tool name
  input: map #   MAY — 補償輸入（支援 ${ }，可引用原 step 的 output）

lifecycle: # MAY — 生命週期鉤子
  init: ...
  destroy: ...

backend:
  type: exec | http | extension # MUST
  timeout: 30s                  # MAY — 單次 backend 呼叫的上限（spawn / request 級）；不同於 step-level timeout
  config: { ... }               # MAY — 靜態設定 map（protocol exec / extension 模式可讀；raw 模式無此通道）
  # ... backend 專屬設定（依 type 展開，見「Backend Types」）
```

### backend 共通欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | enum | MUST | `exec` / `http` / `extension` |
| `timeout` | duration | MAY（無預設） | 單次 backend 呼叫的上限。**與 step-level `timeout` 不同**：step timeout 涵蓋整個 step（含 retry），backend.timeout 僅涵蓋當前 attempt 的實際呼叫（spawn process / HTTP request / extension Invoke 返回）。缺省時使用 engine 預設（exec：無；http：60s；extension：無）。超時視為 attempt FAILED，依 `retry` 規則決定是否重試 |
| `config` | map | MAY | 靜態設定鍵值對。**可用模式**：exec protocol 於 stdin JSON 的 `config` 欄位讀取；extension 於 `ExtensionRequest.config` 欄位讀取；**exec raw / http 模式不支援**（實作 MAY 於載入期對這類 backend 宣告 `config` 發出 warning，但不拒絕）。`config` 值為定義時固定，不支援 CEL |

> `builtin` 不是 `backend.type`；系統內建 tool 由引擎直接提供，不需要使用者撰寫 Tool definition。

---

## Compensate 運作規則

`compensate` 宣告於 tool definition 與 step 的映射如下（step 層級優先，見 `03-steps.md`）。**無論在 tool 或 step 層宣告，compensate 的 action MUST 指向 registry 中存在的 tool / function**；否則載入失敗 `registry.action_not_found`。

### compensate.input 求值與驗證

1. `input` 求值上下文與一般 step input 相同（`input` / `steps` / `vars` / `secret` / `artifacts` / `project` 等），**額外** 提供 `output` namespace，指向**原 step 的 output**（該 step MUST 為 SUCCEEDED，否則該 compensate 不會被執行，見步驟 4）
2. CEL 求值後，引擎以 compensate action 的 `input_schema` 驗證（嚴格模式，與一般 task step 呼叫 tool 相同）
3. 驗證失敗 → 該 compensate 視為失敗（計入 `compensation_failures`），`error.type == "schema_violation"`、`error.details.direction == "input"`、`error.details.compensate_origin_step == <origin>`；不中止其他 compensate
4. 僅 SUCCEEDED 的 origin step 會觸發其 compensate；SKIPPED / FAILED 的 step 不補償
5. CEL 不自動型別轉換；若 origin output 型別與 compensate input_schema 不符，請在 CEL 中顯式 `string(output.x)` / `int(output.x)`

### compensate 本身的 retry 與 idempotent

- compensate step 走 tool 的 `retry` 規則（若原 compensate tool 宣告 retry）；step 層 retry **不** 自動套用到 compensate
- 建議 compensate tool 宣告 `idempotent: true`；引擎依此在 lease 切換時判斷是否重跑（見 `runtime-spec/03-step-execution.md` 的 saga 補償狀態機）

### compensate.action 的版本解析

compensate 觸發時，其 `action` 字串的版本解析規則如下，目的為確保 saga 補償在跨部署期間行為可重現：

1. **顯式 `@version`**：`compensate.action: payment.refund@3` → 一律使用 version 3；若 registry 無此版本 → compensate 失敗，`error.type == "action_not_found"`（計入 `compensation_failures`）
2. **無 `@version`** 且 **compensate 宣告於 tool definition**（tool 層）：於**原 step 執行 action 時**一併 resolve 並 pin（查詢當下的 instance `action_pins` / `project.defaults.action_versions` / 全域 `default_version_policy`），將 resolved version 寫入 **instance `action_pins` map**（以 `"<origin_canonical>::compensate"` 為 key 區分；見 `runtime-spec/05-task-registry.md` 的「Compensate 的遞迴 pin」）；補償執行時恆使用此 pin，不重查 registry
3. **無 `@version`** 且 **compensate 宣告於 step 層**（step 直接覆寫）：行為同上，pin 於原 step 執行時決定並寫入 instance `action_pins`
4. `default_version_policy: require_explicit` 下，無 `@version` 且無 `action_pin` 可套用 → 原 step 載入階段即失敗（`registry.version_not_specified`），不進入執行期；故 compensate 不會遇到「執行時無版本可用」

由此保證：同一 saga 內，執行與補償看到的 action 實體為同一版本；即使其間 registry 新增 / 刪除版本，補償行為仍可重現。

---

## Backend Types

### exec

執行外部程式。支援兩種模式：`protocol`（引擎 JSON 協議）與 `raw`（任意程式）。

#### 共通欄位

```yaml
backend:
  type: exec
  command: string # MUST — 啟動命令
  args: [string] # MAY — 命令列參數（支援 ${ }）
  shell: string # MAY — 指定 shell（預設引擎自動判斷）
  env: map # MAY — 環境變數（支援 ${ }）
  working_dir: string # MAY — 工作目錄
  mode: protocol | raw # MAY, 預設 raw
```

#### protocol 模式

與引擎交換結構化 JSON。適用於**專為此引擎開發**的 tool。

```yaml
backend:
  type: exec
  mode: protocol
  command: "node ./tools/order/load.js"
  env:
    DB_URL: ${ secret.DATABASE_URL }
  config: # MAY — 傳入 tool 的靜態設定
    db_pool: "primary"
```

引擎 → tool（stdin）：

```json
{
  "input": { ... },
  "config": { ... },
  "context": {
    "idempotency_key": "string",
    "instance_id": "string",
    "step_id": "string",
    "attempt": 1,
    "trace_id": "string"
  }
}
```

`idempotency_key` 恆為 string（engine 一律計算，非 null），與 stdin template 的 `context.idempotency_key` 同源。

tool → 引擎（stdout）：

```json
{
  "success": true,
  "output": { ... },
  "error": null
}
```

Exit code 在 protocol 模式下被忽略，以 JSON response 的 `success` 為準。

#### raw 模式

**直接執行任意程式**，不要求 tool 理解引擎協議。適用於包裝現有腳本、CLI 工具、或其他語言程式。

```yaml
backend:
  type: exec
  mode: raw # 預設值，可省略
  command: "python"
  args:
    - "./scripts/analyze.py"
    - "--order-id"
    - ${ input.order_id }
  env:
    API_KEY: ${ secret.API_KEY }
```

原生 Linux 指令與 Tool 包裝對照：

```bash
python ./scripts/analyze.py --order-id order-123
```

```yaml
backend:
  type: exec
  mode: raw
  command: "python"
  args:
    - "./scripts/analyze.py"
    - "--order-id"
    - ${ input.order_id }
```

```bash
grep -c "ERROR" /var/log/app.log || echo 0
```

```yaml
backend:
  type: exec
  mode: raw
  command: "sh"
  args:
    - "-c"
    - 'grep -c "$PATTERN" "$LOG_FILE" || echo 0'
  env:
    LOG_FILE: ${ input.log_file }
    PATTERN: ${ default(input.pattern, "ERROR") }
```

差異在於：Linux 指令直接寫死參數；Tool 定義把可變部分提升為 `input` / `env` / `stdin` / `stdout` 設定，讓 workflow 可重複呼叫與解析結果。

##### stdin — 輸入控制

控制透過 stdin 傳送給程式的內容。

```yaml
backend:
  type: exec
  command: "jq"
  args: [".items[] | .sku"]
  stdin:
    format: json | text | none # MAY, 預設 none
    # format: json 時 — 將 input 序列化為 JSON 字串傳入
    # format: text 時 — 使用 template 產生純文字
    # format: none 時 — 不傳送 stdin
    template: string # MAY — 僅 format: text 時使用，支援 ${ }
```

format 說明：

| format | stdin 內容                 | 適用場景                        |
| ------ | -------------------------- | ------------------------------- |
| `none` | 不傳送                     | 程式僅靠 args / env 取得輸入    |
| `json` | `input` 序列化為 JSON 字串 | 程式自行解析 JSON               |
| `text` | `template` 求值結果        | 需要自訂格式（CSV、指令字串等） |

`text` template 範例：

原生 Linux 指令：

```bash
printf '%s\n' "https://api.example.com/orders/order-123" | sh -c 'read line && curl -s "$line"'
```

對應 Tool 包裝：

```yaml
backend:
  type: exec
  command: "sh"
  args: ["-c", "read line && curl -s $line"]
  stdin:
    format: text
    template: "https://api.example.com/orders/${ input.order_id }"
```

template 的求值上下文：

| Namespace | 說明                                              |
| --------- | ------------------------------------------------- |
| `input`   | tool 的輸入資料                                   |
| `env`     | 環境變數                                          |
| `secret`  | 機密值                                            |
| `context` | 執行上下文（`instance_id` / `step_id` / `attempt` / `idempotency_key` / `trace_id`）— 完整結構見 [04-expressions.md#context](04-expressions.md#context) |

##### stdout — 輸出解析

定義如何解析程式的 stdout 為結構化輸出。

```yaml
backend:
  type: exec
  command: "python ./scripts/analyze.py"
  stdout:
    format: json | text | lines # MAY, 預設 text
    mapping: map # MAY — 將解析結果映射至 output_schema
```

format 說明：

| format  | `raw` 變數型別 | 說明                                  |
| ------- | -------------- | ------------------------------------- |
| `text`  | `string`       | 整段 stdout 作為單一字串（自動 trim） |
| `json`  | `any`          | 解析 stdout 為 JSON                   |
| `lines` | `list<string>` | 按換行拆分為字串陣列（空行忽略）      |

**無 mapping 時**：`raw` 直接作為 output。

- `format: text` → output 為字串
- `format: json` → output 為 JSON 解析結果
- `format: lines` → output 為字串陣列

**有 mapping 時**：透過 CEL 表達式將 `raw` 映射至結構化 output。

原生 Linux 指令與輸出：

```bash
./bin/order-export --format json
./bin/order-export --format lines
./bin/order-export --format text
```

```yaml
# stdout 為 JSON，映射特定欄位
stdout:
  format: json
  mapping:
    order_id: ${ raw.data.id }
    total: ${ raw.data.amount }
    items: ${ raw.data.line_items.map(i, i.sku) }
```

```yaml
# stdout 為 lines，解析固定格式輸出
# 程式輸出：
#   OK
#   order-123
#   1500
stdout:
  format: lines
  mapping:
    status: ${ raw[0] }
    order_id: ${ raw[1] }
    amount: ${ int(raw[2]) }
```

```yaml
# stdout 為 text，使用 extractMatches 正則解析
# 程式輸出：Order #12345 total: $150.00
stdout:
  format: text
  mapping:
    order_id: ${ raw.extractMatches("Order #(\\d+)")[1] }
    total: ${ double(raw.extractMatches("total: \\$(\\d+\\.\\d+)")[1]) }
```

mapping 的求值上下文：

| Namespace | 說明                                          |
| --------- | --------------------------------------------- |
| `raw`     | stdout 解析後的原始資料（型別依 format 而定） |
| `input`   | tool 的輸入資料                               |

##### exit code — 成功判斷

```yaml
backend:
  type: exec
  command: "grep"
  args: ["-q", ${ input.pattern }, ${ input.file }]
  exit_code:
    success: [0]                # MAY, 預設 [0] — 視為成功的 exit code
    # 不在 success 列表中的 exit code → step FAILED
```

原生 Linux 指令：

```bash
grep -q "ERROR" /var/log/app.log
echo $?
```

exit code 不在 `success` 列表中時，step 進入 FAILED 狀態，`error` 包含：

| 欄位            | 說明                       |
| --------------- | -------------------------- |
| `error.type`    | `"exit_code"`              |
| `error.message` | stderr 內容（若有）        |
| `error.code`    | exit code 數值（字串型別） |

##### stderr 處理

- raw 模式下 stderr 不影響成功判斷（僅看 exit code）
- step FAILED 時 stderr 會被放入 `error.message`
- 引擎 MAY 將 stderr 寫入執行日誌

#### raw 模式完整範例

包裝 Python 腳本：

原生 Linux 指令：

```bash
printf '%s' "This product is great" | python ./tools/sentiment.py
```

對應 Tool 包裝：

```yaml
apiVersion: tool/v6
kind: Tool

metadata:
  name: sentiment.analyze
  version: 1
  description: "分析文本情感傾向"

input_schema:
  type: object
  properties:
    text: { type: string }
  required: [text]

output_schema:
  type: object
  properties:
    score: { type: number }
    label: { type: string }

backend:
  type: exec
  command: "python"
  args: ["./tools/sentiment.py"]
  stdin:
    format: text
    template: ${ input.text }
  stdout:
    format: json
```

對應的 `sentiment.py`（無需任何引擎相關邏輯）：

```python
import sys, json
from textblob import TextBlob

text = sys.stdin.read()
blob = TextBlob(text)
score = blob.sentiment.polarity

print(json.dumps({
    "score": score,
    "label": "positive" if score > 0 else "negative" if score < 0 else "neutral"
}))
```

包裝 shell 管線：

原生 Linux 指令：

```bash
grep -c "ERROR" /var/log/app.log || echo 0
```

對應 Tool 包裝：

```yaml
apiVersion: tool/v6
kind: Tool

metadata:
  name: log.count_errors
  version: 1
  description: "計算日誌檔中的錯誤數量"

input_schema:
  type: object
  properties:
    log_file: { type: string }
    pattern: { type: string, default: "ERROR" }
  required: [log_file]

output_schema:
  type: object
  properties:
    count: { type: integer }

backend:
  type: exec
  command: "sh"
  args:
    - "-c"
    - 'grep -c "$PATTERN" "$LOG_FILE" || echo 0'
  env:
    LOG_FILE: ${ input.log_file }
    PATTERN: ${ default(input.pattern, "ERROR") }
  stdout:
    format: text
    mapping:
      count: ${ int(raw) }
  exit_code:
    success: [0, 1] # grep 無匹配時 exit 1，仍視為成功
```

包裝現有 CLI 工具：

原生 Linux 指令：

```bash
convert input.png -resize 300x200 /tmp/resized.png
```

對應 Tool 包裝：

```yaml
apiVersion: tool/v6
kind: Tool

metadata:
  name: image.resize
  version: 1
  description: "使用 ImageMagick 縮放圖片"

input_schema:
  type: object
  properties:
    source: { type: string }
    width: { type: integer }
    height: { type: integer }
  required: [source, width, height]

output_schema:
  type: object
  properties:
    path: { type: string }

backend:
  type: exec
  command: "convert"
  args:
    - ${ input.source }
    - "-resize"
    - "${ string(input.width) }x${ string(input.height) }"
    - "/tmp/resized_${ context.step_id }.png"
  stdout:
    format: text
    mapping:
      path: "/tmp/resized_${ context.step_id }.png"
  exit_code:
    success: [0]
```

---

### http

呼叫 HTTP endpoint。預設採用 OpenAPI 協議：request body = `input_schema`，response body = `output_schema`。

```yaml
backend:
  type: http
  url: "https://api.payment.example.com/v1/charges" # MUST
  method: POST # MAY, 預設 POST
  headers: # MAY
    Authorization: "Bearer ${ secret.PAYMENT_API_KEY }"
  timeout: 10s # MAY
  retry_on_status: [429, 502, 503] # MAY
```

支援的 `method`：`GET` / `POST` / `PUT` / `DELETE` / `PATCH` / `HEAD` / `OPTIONS`（不區分大小寫，engine 內部正規化為大寫）。其他值 → 載入失敗，`registry.invalid_http_method`。

- `GET` / `HEAD` / `OPTIONS`：若 `request.body` 存在且非空 → 載入警告但仍發出（某些 API 會讀 GET body，非標準但常見）；`HEAD` response 無 body，`output` 為 `null`，`response.mapping` 仍可針對 headers 取值（透過 `raw.headers.<name>`）
- `POST` / `PUT` / `PATCH` / `DELETE`：可含 request body；無 body 時 Content-Length 為 0
- CORS `OPTIONS` 通常由 HTTP client 自動處理，明確宣告為 `OPTIONS` 會被視為顯式 preflight 呼叫

#### 進階 HTTP 設定

當 API 不符合預設 OpenAPI 協議時，可自訂 request / response 映射：

```yaml
backend:
  type: http
  url: "https://legacy.example.com/api"
  method: POST
  headers:
    Content-Type: "application/x-www-form-urlencoded"
  request:
    # MAY — 自訂 request body，預設為 input 直接序列化
    body: "order_id=${ input.order_id }&amount=${ string(input.amount) }"
  response:
    # MAY — 映射 response body 至 output_schema
    mapping:
      payment_id: ${ raw.result.txn_id }
      status: ${ raw.result.state == "OK" ? "confirmed" : "pending" }
  error_on_status: [400, 401, 403, 404, 500]  # MAY — 這些 status code 視為 FAILED
```

JSON request 範例：

```yaml
backend:
  type: http
  url: "https://risk.example.com/v1/evaluate"
  method: POST
  headers:
    Authorization: "Bearer ${ secret.RISK_API_KEY }"
    Content-Type: "application/json"
  request:
    body:
      order_id: ${ input.order_id }
      amount: ${ input.amount }
      customer:
        id: ${ input.customer_id }
        tier: ${ default(input.customer_tier, "standard") }
  response:
    mapping:
      level: ${ raw.risk.level }
      score: ${ raw.risk.score }
```

---

### extension

引擎擴充點，允許第三方開發者定義自訂 backend 類型。引擎會將請求轉發給對應 handler，由 handler 負責執行並回傳結果。

```yaml
backend:
  type: extension
  handler: string # MUST — handler 名稱
  config: {} # MAY
```

---

## 生命週期（Lifecycle）

定義 tool 的初始化與銷毀行為。適用於需要 OAuth 登入、連線池建立等前置作業的場景。

```yaml
lifecycle:
  init: # MAY — 首次使用前執行
    reusable_across_restart: false # MAY, 預設 false — engine 重啟 / lease 接管後是否復用既存的 init output
    backend:
      type: exec
      command: "node ./tools/auth/login.js"
      env:
        CLIENT_ID: ${ secret.OAUTH_CLIENT_ID }
        CLIENT_SECRET: ${ secret.OAUTH_CLIENT_SECRET }
    # init 的 output 會注入至主 backend 的 context.lifecycle.init
  destroy: # MAY — workflow instance 結束時執行
    timeout: 30s # MAY, 預設 30s — destroy hook 執行上限
    backend:
      type: exec
      command: "node ./tools/auth/logout.js"
```

### lifecycle.destroy 欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `timeout` | duration | MAY（預設 30s） | destroy hook 的執行上限；超過視為失敗（走「destroy 失敗」路徑，排入重試佇列，見 `runtime-spec/08-persistence.md` 的「lifecycle.destroy 重試政策」）。若 destroy 正在存取 artifact 而 timeout → engine 強制加 workspace 寫鎖並視 destroy 失敗（見 `dsl-spec/04-expressions.md` 的 artifact 寫鎖規則） |
| `backend` | object | MUST | exec / http backend spec；不支援 extension（同 `lifecycle.init.backend`） |

### lifecycle.init 欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `reusable_across_restart` | bool | MAY（預設 false） | 標記 init output 是否跨 engine 重啟 / lease 接管時仍有效。`false`（預設）：engine crash / 接管後一律重新執行 init（安全預設，避免過期 session token / 失效連線池引用）。`true`：條件性復用（僅同 engine_id 進程可復用；跨 engine 仍重新 init）。詳見 `runtime-spec/06-tool-backend.md` 的「reusable_across_restart 的復用規則」 |
| `backend` | object | MUST | exec / http backend spec；不支援 extension（見下） |

### 執行時機

| 階段      | 觸發條件                                                |
| --------- | ------------------------------------------------------- |
| `init`    | 該 tool 在 workflow instance 中**首次被呼叫前**（lazy） |
| `destroy` | workflow instance 結束時（不論成功或失敗）              |

### 求值上下文

`init` / `destroy` 不隸屬於任何 step，`${ }` 僅允許引用 `secret` / `env` / `project` / `artifacts._workspace_path`，不可引用 `input` / `steps` / `prev` / `vars` / `loop` / `event`。完整表格見 `04-expressions.md`。

`init` 失敗 → 依賴此 tool 的所有後續 step 進入 FAILED，`error.type == "lifecycle_init_failed"`。

### init 循環依賴

`lifecycle.init.backend` 的 `type: exec` / `http` **不** 經 task registry（直接 spawn process 或發 HTTP），因此 **不能** 呼叫其他 tool 或 function —— 不存在 init 互相依賴的情況。

若 init backend 間接觸發其他 tool（例如 init 內呼叫 CLI 而該 CLI 又觸發本 engine 的 API 建立新 instance），視為外部系統行為、不在 engine 保護範圍內。建議做法：

- Init backend SHOULD 為自足腳本（OAuth client credentials flow、讀本地 keychain 等）
- 需要共用資料的多個 tool 應透過 `init output` 的共享 cache 傳遞，而非互相呼叫
- Engine 在載入期 MUST 驗證 `lifecycle.init.backend.type` 與 `lifecycle.destroy.backend.type` **同屬** `{exec, http}`（不支援 `extension`，避免引入未知調用路徑且讓清理路徑與啟動路徑對稱）；違反 → `registry.invalid_lifecycle_backend`，`error.details.hook` 為 `"init"` 或 `"destroy"`

若未來擴充允許 init 呼叫其他 tool，屆時將引入 init-depth 限制與 SCC 檢查；v6 不開放此能力以根絕死鎖風險。

### init output 存取

init 的 output 會被快取，後續同 instance 內所有該 tool 的呼叫都能透過 `context.lifecycle.init` 存取。**三種 backend 模式皆可引用**：

| Backend 模式 | context.lifecycle.init 可用位置 |
|-------------|--------------------------------|
| protocol（exec / http） | stdin JSON 的 `context.lifecycle.init` 欄位；CEL 求值（`args` / `env` / `headers` / `url` / `request.body`）可用 `${ context.lifecycle.init.* }` |
| raw（exec） | CEL 求值位置皆可（`args` / `env` / `stdin.template` / `stdout.mapping`）；**stdin 的 JSON / text 內容本身不自動注入**，需透過 CEL 明示引用 |
| extension | extension handler 收到的 `ExtensionRequest` 含 `context` map，路徑同 protocol |

Lazy 觸發規則（所有模式）：

- 首次任一 step 呼叫此 tool 時 → engine 先執行 lifecycle.init → cache `init.output` → 再呼叫本 step 的 backend
- 即使該 step 呼叫的是 raw 模式且未引用 `context.lifecycle.init`，若 tool definition 有 `lifecycle.init`，init 仍**被觸發**（lifecycle 依 tool 而非 backend mode 而定；避免同 tool 不同引用方式造成狀態不一致）
- init 失敗 → 該 step FAILED，`error.type == "lifecycle_init_failed"`；後續同 tool 的其他 step 亦 FAILED（cache sentinel）

```yaml
# init 取得 OAuth token
lifecycle:
  init:
    backend:
      type: exec
      mode: raw
      command: "curl"
      args:
        [
          "-s",
          "-X",
          "POST",
          "https://auth.example.com/token",
          "-d",
          "grant_type=client_credentials",
          "-d",
          "client_id=${ secret.CLIENT_ID }",
          "-d",
          "client_secret=${ secret.CLIENT_SECRET }",
        ]
      stdout:
        format: json

# 主 backend 使用 init 取得的 token
backend:
  type: http
  url: "https://api.example.com/orders"
  headers:
    Authorization: "Bearer ${ context.lifecycle.init.access_token }"
```

---

## Callback 協議

Tool 在執行中可能需要 caller（workflow）回覆資料後再繼續 —— 常見於人工審批、需要外部系統注入驗證碼等場景。caller 端以 `type: task` step 的 `callback:` 區塊提供 handler（見 `05b-function.md` 的「Caller 端：callback 區塊」）。Tool 端以 backend 協議發出 callback 請求、等待引擎回覆。

### exec（protocol 模式）

宣告 `callback:` 的 caller 會觸發 tool 的「多訊息協議」模式：**stdin 與 stdout 皆以 `\n` 分隔的 JSON line（JSON Lines / NDJSON）通訊**，取代原本的「一次 stdin / 一次 stdout」單訊息格式。Tool 可由檢測 stdin 首個 line 的 `"protocol": "v6-protocol"` 欄位判斷。

#### Framing 規則

- 每則訊息 MUST 為單行 JSON（內部不得含裸換行），以 `\n` 結尾。
- stdout 中 `type` 欄位區分訊息角色；沒有 `type` 的訊息視為協議錯誤。
- 每則 callback 請求 MUST 附 `call_id`（tool 產生、唯一字串），引擎回覆以相同 `call_id` 對應。
- Tool 可在未收到先前 callback 回覆前發出更多 callback（pipeline）；引擎 SHOULD 依 caller handler 處理能力決定回覆順序，tool MUST 以 `call_id` 對應而非依賴順序。
- Tool 發出 `{"type":"result", ...}` 後 MUST 結束 stdout 與程序；此訊息即傳統 `{success, output, error}` 的承載。

#### 訊息類型

引擎 → tool（首訊息，stdin 第一行）：

```json
{"protocol": "v6-protocol", "input": {...}, "config": {...}, "context": {...}}
```

tool → 引擎（發出 callback 請求）：

```json
{"type": "callback", "call_id": "cb-1", "name": "<callback_name>", "input": {...}}
```

引擎 → tool（stdin 後續行；成功）：

```json
{"type": "callback_result", "call_id": "cb-1", "output": {...}}
```

引擎 → tool（失敗）：

```json
{"type": "callback_result", "call_id": "cb-1", "error": {"type": "string", "message": "string"}}
```

引擎 → tool（callback 被 caller step timeout 中斷；特例 error type）：

```json
{"type": "callback_result", "call_id": "cb-1", "error": {"type": "timeout", "message": "caller timeout"}}
```

收到 `"type":"timeout"` 的 callback_result 表示 caller 已放棄本 step；tool SHOULD 儘速輸出 `result` 結束（引擎也會關閉 stdin）。超時見 [runtime-spec/02-instance-lifecycle.md 的「Callback timeout 規則」](../runtime-spec/02-instance-lifecycle.md#callback-timeout-規則)。

tool → 引擎（最終結果，之後 close stdout）：

```json
{"type": "result", "success": true, "output": {...}, "error": null}
```

若 caller 未宣告 `callback:`，tool 仍可使用單訊息協議；引擎不會發送 `v6-protocol` 首訊息，tool 端可沿用 v4 行為。

### exec（raw 模式）

raw 模式**不支援** callback。若需要，請改以 `mode: protocol` 並手動處理協議訊息。

### http

HTTP backend 在 caller 宣告 `callback:` 時啟用 SSE（`Content-Type: text/event-stream`）作為雙工通道：

- Tool 的 response header MUST 含 `X-Callback-URL: <url>`，指向一個引擎代管的 POST endpoint，用於接收 callback result。
- Tool 以 SSE 事件輸出 `callback` / `result`：

```
event: callback
data: {"call_id": "cb-1", "name": "<callback_name>", "input": {...}}

event: result
data: {"success": true, "output": {...}, "error": null}
```

- 引擎收到 `callback` 事件後，執行 caller handler，然後對 `X-Callback-URL` POST：

```json
{"call_id": "cb-1", "output": {...}}
```

或：

```json
{"call_id": "cb-1", "error": {"type": "string", "message": "string"}}
```

- Tool MUST 以 `call_id` 對應回覆，而非依賴事件順序。`result` 事件收到後引擎關閉 SSE 連線。

### extension

extension handler 自行定義 callback 傳輸方式；引擎只保證以同一份 `{name, input}` / `{name, output|error}` 結構與 caller 的 `callback:` 區塊對接。

### 與 function callback 的對稱性

`type: task` 呼叫 tool 與呼叫 function 共用同一 `callback:` 語法（見 `05b-function.md`）。Caller handler 以 `type: return` 結束並以 `output` 回傳的資料，即為 tool 端 `callback_result.output` 的內容。Handler FAILED → 以 `callback_result.error` 回傳，tool 可自行決定是否中止。

---

## 在 Workflow 中使用

Tool definition 透過 `type: task` step 呼叫（`action` 對應 `metadata.name`）：

```yaml
# Tool Definition
apiVersion: tool/v6
kind: Tool
metadata:
  name: order.load
  version: 1
input_schema:
  type: object
  properties:
    order_id: { type: string }
  required: [order_id]
backend:
  type: exec
  command: "node ./tools/order/load.js"
  stdin:
    format: json
  stdout:
    format: json

---
# Workflow Step（使用端）
- id: load_order
  type: task
  action: order.load # 參照 tool definition name
  input:
    order_id: ${ input.order_id }
```

---

## 完整範例

### HTTP Tool with Compensation

```yaml
apiVersion: tool/v6
kind: Tool

metadata:
  name: payment.create
  version: 2
  description: "建立付款請求"

input_schema:
  type: object
  properties:
    order_id:
      type: string
    amount:
      type: number
    currency:
      type: string
      default: "TWD"
  required: [order_id, amount]

output_schema:
  type: object
  properties:
    payment_id:
      type: string
    status:
      type: string

compensate:
  action: payment.refund
  input:
    payment_id: ${ output.payment_id }

lifecycle:
  init:
    backend:
      type: exec
      mode: raw
      command: "node"
      args: ["./tools/payment/get_token.js"]
      env:
        CLIENT_ID: ${ secret.PAYMENT_CLIENT_ID }
        CLIENT_SECRET: ${ secret.PAYMENT_CLIENT_SECRET }
      stdout:
        format: json

backend:
  type: http
  url: "https://api.payment.example.com/v1/charges"
  method: POST
  headers:
    Authorization: "Bearer ${ context.lifecycle.init.access_token }"
  timeout: 10s
  retry_on_status: [429, 502, 503]
```

### Raw Exec Tool — 包裝 Go Binary

```yaml
apiVersion: tool/v6
kind: Tool

metadata:
  name: risk.evaluate
  version: 1
  description: "評估訂單風險等級"

input_schema:
  type: object
  properties:
    order_id: { type: string }
    amount: { type: number }
    customer_id: { type: string }
  required: [order_id, amount]

output_schema:
  type: object
  properties:
    level: { type: string, enum: [low, medium, high] }
    score: { type: number }
    reasons: { type: array, items: { type: string } }

backend:
  type: exec
  command: "./bin/risk-evaluator"
  args:
    - "--order-id=${ input.order_id }"
    - "--amount=${ string(input.amount) }"
    - "--customer=${ default(input.customer_id, 'anonymous') }"
    - "--format=json"
  stdout:
    format: json
```

### Raw Exec Tool — 包裝 Shell 管線

```yaml
apiVersion: tool/v6
kind: Tool

metadata:
  name: data.aggregate
  version: 1
  description: "彙總 CSV 資料"

input_schema:
  type: object
  properties:
    file: { type: string }
    column: { type: integer }
  required: [file, column]

output_schema:
  type: object
  properties:
    sum: { type: number }
    count: { type: integer }
    avg: { type: number }

backend:
  type: exec
  command: "sh"
  args:
    - "-c"
    - |
      awk -F',' -v col=$COL '
        NR>1 { sum+=$col; count++ }
        END { printf "{\"sum\":%.2f,\"count\":%d,\"avg\":%.2f}", sum, count, (count>0 ? sum/count : 0) }
      ' "$FILE"
  env:
    FILE: ${ input.file }
    COL: ${ string(input.column) }
  stdout:
    format: json
  exit_code:
    success: [0]
```
