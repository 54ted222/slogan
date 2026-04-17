# 06 — Tool Backend Driver

本文件定義 Backend Driver 對 `kind: Tool` 的 `backend.type: exec | http | extension` 的執行細節，以及 callback 的協議實作。對應 [dsl-spec/05-tool.md](../dsl-spec/05-tool.md)。

---

## 通用流程

```
ToolInvocation = (Action, input, context, callback_handler?)

driver.invoke(invocation) -> ToolResult
```

ToolResult：

```
{
  success: bool,
  output:  any,
  error:   { type, message, code? } | null,
  attempts: int,
  duration_ms: int,
}
```

`callback_handler` 由 Engine Loop 提供：是一個函式 `handle(call_id, name, input) -> CallbackResult`，driver 觸發後將結果回灌 tool。

### Schema 驗證職責

所有 backend（exec protocol / exec raw / http / extension）在建立 ToolResult 前 MUST 由 driver 驗證：

1. **input_schema**（tool 定義有此欄位時）：對 `input` 在 spawn 前驗證；不符 → 不啟動 tool，ToolResult `{success: false, error: {type: "schema_violation", details: {direction: "input", path: ...}}}`
2. **output_schema**（tool 定義有此欄位時）：對 `output`（raw 模式下為 mapping 後結果）在成功分支驗證；不符 → ToolResult `{success: false, error: {type: "schema_violation", details: {direction: "output", path: ...}}}`；原始 output 保留於 error.details.raw_output

Engine Loop 不重複驗證；若 driver 未驗證，錯誤將以 tool 回傳結構傳播，但 step 可能已完成（違反 schema 保證）。因此驗證責任在 driver 層。

#### Schema 缺省時的處理

- `input_schema` 缺省（欄位不存在）→ driver **不**做任何 input 驗證、不套 default；CEL 求值後的 map 原樣序列化傳給 backend（exec 為 JSON stdin 或 args；http 為 request body；extension 為 ExtensionRequest.input）
- `output_schema` 缺省 → driver 接收 tool output 後直接寫 checkpoint，不驗證；output 仍受 `engine.max_step_output_bytes` 上限保護
- 無 schema 的 tool **仍可**宣告 `idempotent: true` / `compensate` 等；signature 計算基於 CEL 求值後的 input_snapshot（與有 schema 時一致）
- 建議生產環境 tool **明確宣告 schema**；缺省僅適合 prototyping 與 wrapper tool（如 `builtin.echo`）

#### output_schema 與 mapping 的交互

Raw / HTTP backend 若同時宣告 `stdout.mapping` / `response.mapping` 與 `output_schema`，順序如下：

```
backend 回傳 raw 資料
  │
  ▼
（若有 mapping）CEL 求值 mapping → output
  │ 失敗：step FAILED，error.type == "expression_error.mapping"
  │      error.details.raw 保留原始 raw（便於 debug）
  │      **不**再走 output_schema 驗證
  ▼
（若有 output_schema）對 mapping 後的 output 驗證
  │ 失敗：step FAILED，error.type == "schema_violation"
  │      error.details.direction == "output"
  │      error.details.raw_output 保留 mapping 後的結果
  ▼
step SUCCEEDED，output 進入 checkpoint
```

重點：
- **mapping 失敗優先**於 schema 驗證；兩種失敗的 error.type 不同，使用者可於 catch 分流
- 無 mapping 時 raw 直接為 output（走同一 output_schema 驗證路徑）
- `input_schema` 驗證於 CEL 求值後、backend spawn 前（無 mapping 概念）

---

## exec backend

### Process 啟動

```
1. 解析 backend.command / args / env / working_dir / shell
2. 對 args、env、working_dir 中的 ${ } 做 expression 求值
3. 驗證求值結果（見下「args / env 值型別驗證」）
4. spawn process（依 platform 採 fork/exec）
5. 設定 stdin / stdout / stderr 為非阻塞 pipe
6. 啟動 reader / writer task；start time 記錄為 RUNNING checkpoint 的時間
```

#### args / env 值型別驗證

CEL 求值後每個 args 元素 / env value MUST 為**非 null 字串**；引擎執行以下規則：

| 求值結果 | 處理 |
|---------|------|
| string | 原樣使用 |
| null | step FAILED，`error.type == "invalid_args"`（args）或 `"invalid_env"`（env）；`error.details.field` 為失敗路徑（如 `args[2]`） |
| int / double / bool | step FAILED，同上；使用者 MUST 顯式 `string(x)` 轉型（避免格式意外） |
| list / map | step FAILED，同上（args/env 不支援複合值） |

若使用者確實需要「可省略的 optional 參數」，請於 YAML 上層以 `if` / `switch` 產生不同 args 陣列，或在 CEL 中以 `default(x, "")` 轉為空字串（engine 不自動過濾空字串元素，tool 自行處理）。

### 模式判定

| 條件 | 模式 |
|------|------|
| `mode: protocol` 且 caller 無 `callback:` | **單訊息 protocol** |
| `mode: protocol` 且 caller 有 `callback:` | **NDJSON protocol**（v6-protocol） |
| `mode: raw` 或預設 | **raw** |

### 單訊息 protocol

```
1. 寫入單一 JSON 至 stdin、close stdin
2. 等待 process 輸出單一 JSON 至 stdout
3. process exit 後解析 stdout
4. 構造 ToolResult：{ success, output, error } 取自 JSON
```

stderr 寫入 execution log，不影響成功判定。

### NDJSON protocol（v6-protocol）

driver 與 tool 以行分隔 JSON 雙向通訊。此 NDJSON framing 為 callback 雙工通道，非 workflow-visible stream。

#### 引擎 → tool（stdin 第一行）

```json
{"protocol": "v6-protocol", "input": {...}, "config": {...}, "context": {...}}
```

stdin 保持開啟；後續行為 `callback_result` 訊息。

#### tool → 引擎（stdout，逐行讀）

| `type` | 處置 |
|--------|------|
| `callback` | 解析 `{call_id, name, input}` → 呼叫 `callback_handler.handle()`（背景 task）→ 完成後寫 `callback_result` 至 stdin |
| `result`   | 解析 `{success, output, error}` → 構造 ToolResult；driver 關閉 stdin 並等 process exit |
| 其他 / 空 | 寫 stderr log；不視為錯誤直接丟棄 |

#### 引擎 → tool（stdin 後續行）

```json
{"type":"callback_result","call_id":"...","output":{...}}
{"type":"callback_result","call_id":"...","error":{"type":"...","message":"..."}}
```

#### 邊界規則

- 行長上限：建議 1MiB；超過 → step FAILED，error.type: `incomplete_protocol`，error.message 包含截斷的前 1KB。
  - driver 發 SIGTERM 至 tool，等 5s 未退出則 SIGKILL；不得繼續等待後續訊息
  - 原因：超長行可能是 `type: result` 訊息，丟棄後 tool 端認為成功但 driver 無法判定終態，造成掛起
- Tool 在收到 `callback_result` 前可繼續發更多 `callback`。
- Driver MUST 將 `call_id` 對應表持久化（key: `(instance_id, step_path, attempt, call_id)`，含 `step_path` 以區分 foreach / parallel 迭代共用同 `step_id` 的情境，含 attempt 以避免 retry 跨 attempt 誤配），避免 process 中途崩潰時 callback 結果遺失。
  - **重複 call_id**：同一 (instance_id, step_path, attempt) 內 tool 發出多筆相同 `call_id` 的 `callback` 訊息 → 以**首筆為準**，第二筆之後以協議錯誤 `incomplete_protocol` 處理（SIGTERM kill tool），避免 handler 被重複呼叫破壞 idempotent 語意
  - **未匹配 call_id**：tool POST `callback_result` 至 `X-Callback-URL`（protocol 模式於 stdin 回寫；HTTP callback 模式於外部 endpoint）但 `call_id` 不在持久化表中 → driver 回 HTTP 404 / 於 stdin 寫 `{"type":"error","reason":"unknown_call_id","call_id":...}`，tool 不得掛起等待；step 本身不因此 FAILED（避免 tool bug 影響業務結果），僅記錄 `tool.protocol_anomaly` observability 事件
  - **進程重啟後的 call_id 表**：driver 重啟時以 `(instance_id, step_path, attempt)` 重建該表；若 attempt 已推進，舊 attempt 的 call_id POST 視同未匹配
- 收到 `result` 後，driver 等待 process exit；若 process 在 `result` 後 5s 內未退出 → SIGTERM；再等 5s → SIGKILL。
- Process exit 但未收到 `result` → ToolResult `{success: false, error: {type: "incomplete_protocol"}}`；exit code 寫入 `error.code`。

### Raw 模式

```
1. 透過 stdin 傳入（依 backend.stdin.format）
2. 程式執行
3. 讀取 stdout / stderr
4. 依 stdout.format 解析 raw
5. 若有 stdout.mapping → 對 raw 套 mapping CEL → output
   - mapping 求值失敗 → step FAILED，error.type: "expression_error.mapping"
     error.details.raw 保留原始解析結果（便於 debug）
6. 依 exit_code.success 判斷成功
   - exit_code 不符 → step FAILED，error.type: "exit_code"（mapping 不再求值）
```

mapping 與 exit_code 的處理順序：先檢查 exit_code，再求 mapping。

stdin format：

| format | 行為 |
|--------|------|
| `none` | 不傳；stdin 立即 close |
| `json` | `json_encode(input)` 寫入後 close |
| `text` | 對 `template` 求值（`${ }`）後寫入 close |

stdout format：

| format | 解析 |
|--------|------|
| `text` | 整段 stdout 去尾換行 → string |
| `json` | `json_decode(stdout)` |
| `lines` | `split(stdout, "\n")`，過濾空行 → list<string> |

raw 模式不支援 callback（caller 即使宣告 `callback:`，driver 在啟動時 raise validation error）。

### Exit code 處理

```
exit_code.success = backend.exit_code.success or [0]
if process.exit_code in success:
    parse output
else:
    ToolResult.success = false
    ToolResult.error = { type: "exit_code", message: stderr.tail(), code: str(exit_code) }
```

### Working directory & shell

- `working_dir`：預設為 `artifacts._workspace_path`（即 `<engine.workspace_root>/<instance_id>`）；CEL 支援 `${ }`（如引用 `artifacts.<name>.path`）
  - Engine MUST 在 tool spawn **前**確保 `working_dir` 存在（若引用 artifact path，則確保對應 artifact 目錄已建立）
  - `working_dir` 求值結果 MUST 在 workspace 下；engine 於 CEL 求值後、spawn 前以下列程序驗證（**canonical 路徑定義**）：
    1. 將 `working_dir` 字串與 `<engine.workspace_root>` 同時做**平台原生的 realpath 解析**（遞迴展開所有 symlink，直到非 symlink 為止；POSIX 用 `realpath()` / Windows 用 `GetFinalPathNameByHandle`）
    2. 標準化 `.` / `..` segments，於 case-insensitive 檔案系統（HFS+ / NTFS default）執行 case folding
    3. 以**bytewise prefix match** 比對 canonicalized `working_dir` 是否以 `canonicalized <workspace_root>/<instance_id>` 為前綴（允許完全相等或 `/` 分隔的子路徑；拒絕 `<root>-sibling` 類似字首誤判）
    4. Symlink 解析失敗（如 broken link、permission denied、循環 symlink）→ 視為 escape 意圖拒絕
    5. 路徑不存在：allowed — engine 於 step 1 解析 symlink 成功但目標不存在時，以現有祖先目錄的 realpath + 剩餘 literal segments 重建；若剩餘 segments 無 `..` 跳脫 workspace 即通過
  - 驗證於 **每次 spawn 前** 重做（即便同一 step 的 retry 也重驗；避免 TOCTOU：symlink 在 retry 間被替換）
  - escape 外部 → ToolResult `{success: false, error: {type: "spawn_failed.working_dir_escape", details: {canonical_path, workspace_path, reason: "prefix_mismatch" | "symlink_resolve_failed" | "broken_symlink"}}}`（`spawn_failed.working_dir` 舊錯誤碼繼續適用於目錄不存在或無權限的情況）
- `shell`：明確指定時用該 shell 執行 `command`；未指定時直接 `exec()` `command + args`，不經 shell（避免 quoting 問題）。
  - 接受值：**絕對路徑**（如 `/bin/bash`、`/usr/bin/env zsh`）或**檔名**（`sh`、`bash`、`zsh`、`dash`、`ash`、`busybox`）
  - 檔名會透過 `PATH` 查找；找不到 → ToolResult `{success: false, error: {type: "spawn_failed.not_found"}}`
  - 以 shell 執行時：engine 實際 exec 為 `<shell> -c <command>`；`args` 附加於 command 字串後（engine 自動 shell-escape 每個 arg）
  - 不支援非 POSIX shell（如 `fish` / `nushell` / `powershell`）作為自動 `-c` 模式；若需要請以明確路徑 + 自訂 args 包裝
  - 未指定 `shell` 但 `command` 含 shell 語法（`|` / `>` / `&&`）→ engine 不解析，視為字面參數；如需管線請顯式 `shell: sh` + `command: "..."`

### Env 變數

最終 env = OS 環境 ⨁ project env ⨁ backend.env（後者覆寫前者，**shallow key-level merge**）。

- **Merge 語意**：同 key 由後者覆寫、不同 key 皆保留；即若 backend.env 未宣告 `PATH` / `HOME`，process 仍繼承 OS 的值（不會被清空）
- **系統必要變數**：若 backend.env 明確設 `PATH: "/custom/bin"` → process 只看到 `/custom/bin`（覆寫 OS PATH）；使用者若想「追加」請顯式拼接：`PATH: "${ env.PATH }:/custom/bin"`（需先在 engine 的 `env` namespace 中看到 `env.PATH`）
- **敏感變數攔截**：engine **不**預先過濾 OS env（不建議在系統環境傳機密）；若 operator 啟動 engine 時該環境變數已存在，tool process 會自然繼承；建議部署時以最小 env 啟動 engine
- **預留 key（SLOGAN_* 前綴）**：engine 保留 `SLOGAN_*` 前綴作為 metadata 注入通道；規則如下：
  - backend.env 使用 `SLOGAN_*` key → **載入失敗**，`registry.invalid_env_key`、`details.reason: "reserved_prefix"`；由使用者設值可能遮蔽 engine 注入值並造成 tool 混淆
  - Engine 於每次 spawn tool process 時自動注入以下 SLOGAN_* 變數（exec / raw 模式皆注入；extension 模式以 context map 傳遞同樣欄位）：
    - `SLOGAN_INSTANCE_ID` = 當前 instance uuid
    - `SLOGAN_STEP_ID` = 當前 task step 的 step_id
    - `SLOGAN_STEP_PATH` = 含巢狀路徑（見 `08-persistence.md` steps.step_path）
    - `SLOGAN_ATTEMPT` = 當前 attempt 序號（retry 計數，從 1 起算）
    - `SLOGAN_IDEMPOTENCY_KEY` = 見 `08-persistence.md` 的 idempotency_key 定義
    - `SLOGAN_TRACE_ID` = instance 的 trace_id
    - `SLOGAN_ACTION_NAME` = resolved canonical action name（含 project 前綴與 version）
    - `SLOGAN_ENGINE_ID` = 當前持有 lease 的 engine_id
  - 未來若新增 SLOGAN_* 變數，tool 讀取未知 key 視為 optional；不存在則 fallback 至 tool 預設行為
- **空字串清空**：設為空字串 `""` → process env 仍含該 key 但值為空；若需**完全刪除**該 key（如禁止子 process 繼承 `AWS_ACCESS_KEY_ID`），目前 v6 **不支援**；替代方案為不使用 OS 環境傳遞機密，改用 `secret.*`；若環境變數刪除為剛性需求，建議以 wrapper script 於 tool 啟動前清理

`secret.*` 注入後在 process 環境中以明文存在；driver 啟動後**立即**從本身記憶體中清除其副本（best-effort），避免 dump core 時暴露。

---

## http backend

### 一般請求

```
1. 解析 url / method / headers / request.body
2. CEL 求值 ${ } 欄位（含 secret 注入）
3. 決定 Content-Type 與 body 序列化（見下「Request Body 規則」）
4. 發送 HTTP；遵守 timeout（預設 30s，可由 backend 覆寫）
5. 依 HTTP status code 分類決策（見下表）
6. 解析 response body → raw → （可選）response.mapping → output
```

#### Request Body 規則

| `request.body` 型別 | 自動 Content-Type | 序列化 |
|--------------------|-------------------|---------|
| 缺省 / `null` | 不設（無 body） | 空 |
| CEL 求值結果為 map / list | `application/json`（除非 `headers.Content-Type` 明確覆寫） | `json_encode()` |
| CEL 求值結果為 string | `text/plain`（除非 `headers.Content-Type` 明確覆寫） | 原字串（UTF-8） |
| CEL 求值結果為 number / boolean | `text/plain` | `string(value)` |

- 使用者於 `headers.Content-Type` 顯式指定者**優先**；engine 不強制 body 與 Content-Type 相符（由 tool 作者保證）
- **Request body 大小上限**：`engine.http_request_body_limit`（預設 16 MB）；序列化後超過 → step FAILED，`error.type == "http_request_body_too_large"`；不實際發出請求
- 此限制不涵蓋 chunked upload（v6 不支援 HTTP chunked transfer body；若需上傳大檔請改用 exec backend 呼叫 curl 等工具）
- `input_schema` / CEL 求值後 body 與上限檢查於 transport 前一次性完成；確保不會發出 HTTP 才被對端 413 拒絕

#### 狀態碼分類決策

| 條件（依序判定，首個匹配生效） | 決策 |
|-------------------------------|------|
| status 同時列於 `error_on_status` 與 `retry_on_status` | **`error_on_status` 優先**（直接 FAILED，不重試） |
| status ∈ `error_on_status` | step FAILED，`error.type == "http_error"`、`error.code == "<status>"`（未進 retry 流程） |
| status ∈ `retry_on_status` 且 `step.retry` 已設且尚有 attempts | 由 step.retry 重排一次 attempt；每次 attempt 的 backoff 依 step.retry 計算（見下） |
| status ∈ `retry_on_status` 但 step 無 `retry` 或 attempts 耗盡 | step FAILED，`error.type == "http_error"`、`error.details.exhausted == true` |

**backoff 疊加規則**：

- HTTP backend 的 `retry_on_status` 僅決定「本次 HTTP 呼叫的失敗**是否具有重試資格**」；不增加額外的 backoff
- 實際重試由 **step-level retry** 接手：整個 tool 呼叫視為 step 的一次 attempt；HTTP 收到 503 後 → step 的本次 attempt 標記 FAILED → step.retry 按 `delay` / `backoff` 排程下次 attempt
- HTTP 內建**不**有自己的指數退避；若 API 需要「重試前先等」，請在 step.retry 設 `delay` 與 `backoff: exponential`
- Attempt 計數統一由 step-level 遞增；HTTP 層失敗與 exec 層失敗共用同一 attempt 號碼（避免「HTTP 重試 3 次 + step 重試 3 次 = 9 次實際呼叫」的疊加混亂）
- `Retry-After` header：引擎 MAY 於 HTTP 503/429 時讀取 `Retry-After`（秒數或日期），若 > step.retry 的下次 delay → **取較大者**；`Retry-After` 值會被裁切至 `retry.max_delay` 上限
| status 2xx（200-299） | 成功，進入 response parse |
| status 3xx（300-399） | **不 follow redirect**：引擎 MUST 於 HTTP client 初始化時顯式關閉自動 redirect（Go `CheckRedirect: ErrUseLastResponse` / Python `allow_redirects=False` / Node `redirect: 'manual'` 等）；若底層 library 不支援關閉，engine MUST 以 wrapper 攔截 3xx 並改走以下失敗路徑。除非 status 被 `error_on_status` / `retry_on_status` 明示處理，3xx 預設 → step FAILED，`error.type == "http_redirect_blocked"`、`error.details.location = Location header`、`error.code = "<status>"`；此設計避免注入於 URL / Authorization header 的 secret 洩漏至非預期 endpoint |
| status 其他（4xx / 5xx） | 依 response 解析；若不在 `error_on_status` / `retry_on_status` 列表則仍視為成功 body（由 response.mapping 解讀） |

- 兩列表不可互相包含相同 status 以外的情境；載入期 SHOULD 警告重疊（非錯誤）
- `retry_on_status` 僅決定「是否具備可重試資格」，實際是否重試仍看 step.retry 配額

#### Response body 解析

```
Content-Type 以 application/json 為主（含 +json suffix 如 application/ld+json）：
  → 嘗試 json_decode → 成功則 raw = decoded value
  → 失敗（body 為 JSON 但語法錯）→ step FAILED，error.type == "http_body_malformed_json"、error.details.preview 保留前 1 KB
其他 Content-Type：
  → raw = string body（依下述「charset 解析順序」決定編碼）
  → 解碼失敗 → error.type == "http_body_decode_failed"、error.details.encoding_tried 保留嘗試的編碼名
```

**charset 解析順序**（依序嘗試，首個適用即採用）：

1. 若 `Content-Type` header 含 `charset=<name>` 參數 → 以該 charset 解碼（IANA 名稱大小寫不敏感；未知 charset 名 → `http_body_decode_failed`、`error.details.reason: "unknown_charset"`）
2. 否則檢查 body 前 4 bytes 之 BOM：`EF BB BF` → UTF-8、`FE FF` → UTF-16-BE、`FF FE` → UTF-16-LE、`00 00 FE FF` → UTF-32-BE、`FF FE 00 00` → UTF-32-LE；BOM 本身於解碼後移除
3. 無 charset 且無 BOM → 預設 **UTF-8**（符合 RFC 3629 對 JSON 的規範；對非 JSON body 亦採同預設以求一致）
4. 若 content-type 為 `application/json` 無 charset 且解碼結果含 non-UTF-8 序列 → `http_body_decode_failed`，不做自動偵測（engine 不猜測 Latin-1 / Windows-1252；需要請設 `response.mapping` 自訂處理或改 tool backend 為 exec + stdout raw）

`response.mapping` 若存在則對 raw 套 CEL 得最終 output；失敗 → `expression_error.mapping`。

### Callback 模式（caller 宣告 callback:）

```
1. 設定 Accept: text/event-stream
2. 發送請求
3. 檢查 response header：MUST 含 X-Callback-URL
4. 解析 SSE 事件流：
   - event: callback → 解析 data → 呼叫 callback_handler → POST result 至 X-Callback-URL
   - event: result   → 構造 ToolResult，關閉 SSE 連線
5. 若連線中斷且未收到 result → ToolResult { success: false, error.type: "incomplete_protocol" }
```

X-Callback-URL endpoint 由 driver 啟動：

```
POST <callback_url>
Content-Type: application/json
{ "call_id": "...", "output": {...} } | { "call_id": "...", "error": {...} }
```

URL 應含一次性 token（query string 或 path），驗證來源；engine 在 SSE 連線結束後撤銷該 URL。

#### SSE 連線中斷與重連

SSE 連線具狀態（持有 `X-Callback-URL` token 與 in-flight call_id 表）；本次呼叫的**一次連線生命週期對應一次 tool attempt**：

- **Partial SSE message**：engine 以 line-based 累積（`data:` 行至空行為止才組成一則 event）；若連線在 event 結束前中斷（無終止空行），該 partial event 以**丟棄**處理（不構成 `incomplete_protocol` 立即失敗；engine 繼續判定下一步）
- **連線中斷**：engine **不**自動重連同一 attempt 的 SSE；本次 attempt 終結為 `incomplete_protocol`（見步驟 5）；重試由 step-level `retry` 決定，下一 attempt 以新 HTTP request 重新建立 SSE（含新的 `X-Callback-URL` token）
- **X-Callback-URL token 的 TTL**：連線存續期間有效；連線結束（正常 `result` 或中斷）後 engine 立即撤銷 token；過期 token 的 POST → HTTP 410 Gone、`error.reason: "callback_url_expired"`
- **跨 attempt 的 call_id 隔離**：call_id dedup 表 key 已含 `attempt`（見 NDJSON 模式同規則）；前一 attempt 的 call_id POST 抵達新 attempt → HTTP 404 / 老 URL 已撤銷；不會誤配
- **同一 attempt 內 tool 對 `X-Callback-URL` 的重複 POST**（例：tool 以 at-least-once 方式投遞 callback result）：driver 以 `call_id` 為 key 冪等處理；第二次 POST 若 engine 已處理第一次 → 直接回 200 OK + 原始 response body；tool 可安全重試
- **engine POST 至 tool 的 `X-Callback-URL` 以外的失敗處理**：若 tool 以 callback mode 但未按規範暴露 `X-Callback-URL` header → step FAILED，`error.type: "incomplete_protocol"`、`details.reason: "missing_callback_url"`

### 認證 / TLS

- TLS 由 HTTP client 預設處理；引擎 SHOULD 拒絕 plain http 至非 loopback 且非 secret-allowlisted 主機。
- mTLS 設定欄位（於 backend.tls.*）：

  ```yaml
  backend:
    type: http
    url: "https://mtls.example.com/api"
    tls:
      client_cert: ${ secret.CLIENT_CERT_PEM }   # PEM 字串
      client_key:  ${ secret.CLIENT_KEY_PEM }    # PEM 字串
      ca_bundle:   ${ secret.CA_BUNDLE_PEM }     # MAY — 自訂 CA 驗證
      insecure_skip_verify: false                 # MAY, 預設 false
  ```

  - 解密後的 key PEM 於 HTTP client 建立時載入記憶體，client 釋放時清除
  - TLS handshake 失敗 → step FAILED，error.type: `connection_error`，error.details.tls_reason 含驗證失敗原因（cert_expired / hostname_mismatch / unknown_ca 等）
- **Bearer token expiry**：v6 不自動處理 OAuth2 refresh；有效期到期由 tool lifecycle.init 負責取得新 token，或以 HTTP 401 觸發 retry 並讓下次 init 重新取 token（搭配 `retry_on_status: [401]`）。完整 OAuth2 flow 延後至未來版本。

---

## extension backend

```
1. 載入 handler（extension registry）
2. 構造 ExtensionRequest：{ tool_name, input, config, context, callback_endpoint? }（`config` 對應 tool definition 的 `backend.config`；無則為空 map，與 exec protocol stdin JSON 的 `config` 欄位語意同源）
3. 呼叫 handler.invoke(request)
4. handler 回傳 ExtensionResponse
```

#### ExtensionRequest.context 欄位

Extension 的 `context` map 結構與 exec protocol stdin JSON 的 `context` 欄位（見 `dsl-spec/05-tool.md` 的 Tool stdin JSON schema）**完全一致**，含以下欄位：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `instance_id` | string | 當前 workflow / function instance id |
| `step_id` | string | 呼叫此 tool 的 step id |
| `step_path` | string | 含巢狀路徑（`saga.0.if.then.0` 等） |
| `attempt` | int | 本次 attempt 計數（首次 = 1） |
| `idempotency_key` | string | SHA256 hex（公式見 `08-persistence.md` Idempotency 章節） |
| `trace_id` | string | W3C 相容 trace id |
| `action_name` | string | resolved canonical action name（含 project 前綴與 version） |
| `engine_id` | string | 當前持有 lease 的 engine_id |
| `lifecycle.init` | map \| null | 若 tool 宣告 `lifecycle.init` 且已執行成功，為其 output；未宣告或失敗為 `null` |

其他欄位（例如未來版本新增）MAY 存在，handler 應以「未知欄位忽略」的寬鬆策略解析（避免 v6 升級破壞相容性）。

協議由 handler 自行宣告；engine 僅保證：

- 提供與 exec/http 相同的 callback_handler 介面
- 將最終結果包裝為 ToolResult

extension handler 必須是 in-process（Go plugin、Python entry point、WASM module 等）；out-of-process 由其自行包裝為 exec / http backend。

#### Extension callback_handler 介面契約

當 tool invoke 時 caller 宣告 `callback:`，engine 於 `ExtensionRequest` 中注入 `callback_endpoint`（非 nil）。Handler 可於 invoke 過程中以該 endpoint 觸發 callback 並同步等待 caller handler 結果：

```
CallbackEndpoint {
  invoke(call_id: string, name: string, input: map) -> CallbackResult
}

CallbackResult {
  output: map | null,
  error:  { type: string, message: string } | null,   # 成功時 error == null；失敗時 output == null
}
```

契約要點：

- **同步阻塞語意**：`invoke(...)` 為阻塞呼叫；engine 於 caller handler 執行完成（或 caller step timeout / cancel）後返回；handler 實作 MUST 以此為前提（不需自行輪詢）
- **call_id 由 handler 產生**：格式同 exec / http（tool 產生唯一字串）；engine 以 `(instance_id, step_path, attempt, call_id)` 建立 dedup 表，規則與 exec protocol 一致（重複 / 未匹配處理見 [NDJSON protocol](#邊界規則)）
- **並發 callback**：handler MAY 於同一 invoke 內從多個 goroutine / task 並發呼叫 `CallbackEndpoint.invoke`；engine 以 `call_id` 路由，並發安全；caller handler 執行序由 engine 依 caller 能力決定
- **Callback timeout / cancel 傳播**：若 caller step 觸發 timeout，engine 返回 `CallbackResult{error: {type: "timeout", message: ...}}`（與 exec `callback_result.error.type == "timeout"` 對應）；handler SHOULD 儘速結束 invoke
- **Panic 隔離**：caller handler 內未捕捉例外經 engine 攔截後，以 `CallbackResult{error: {type: "extension_handler_panic"}}` 回傳；不影響 tool invoke 主流程（見「Extension Handler 異常安全」）
- **Endpoint 生命週期**：`callback_endpoint` 於 `handler.invoke` 返回後立即失效；後續呼叫 → `CallbackResult{error: {type: "callback_endpoint_expired"}}`；同 TTL 語意對應 http callback 的 `X-Callback-URL` 撤銷
- 若 caller 未宣告 `callback:` → `ExtensionRequest.callback_endpoint` 為 nil；handler 若仍呼叫則為實作錯誤，engine MAY raise `extension_handler_panic`

實作綁定範例（Go）：

```go
type ExtensionHandler interface {
    Invoke(ctx context.Context, req *ExtensionRequest) (*ExtensionResponse, error)
}
type ExtensionRequest struct {
    ToolName         string
    Input            map[string]any
    Config           map[string]any
    Context          map[string]any
    CallbackEndpoint CallbackEndpoint // nil if caller has no callback:
}
type CallbackEndpoint interface {
    Invoke(callID, name string, input map[string]any) (*CallbackResult, error)
}
```

其他語言綁定（Python entry point、WASM component model）MUST 維持相同語意（同步阻塞、`call_id` 去重、panic 隔離）；簽名細節可依語言慣例調整（如 Python 以 dict / Python WASM component 以 interface type）。

#### Cancel 傳播契約

當 engine 需取消正在執行中的 extension tool invoke（例如父 instance cancel、step timeout），以下列順序觸發 handler 終止：

1. **Context cancellation**：`ctx` 被 cancel（`context.Context.Done()` closed / `Canceled` error）；handler SHOULD 於感知到 `ctx.Err() != nil` 時儘速返回（return `ExtensionResponse{Error: {type: "cancelled"}}` 或 `ctx.Err()` 直接包裝）
2. **Grace 期滿強制中止**：若 handler 於 `cancel_grace_period` 內未返回，engine 以 `go handler.invoke` 的 goroutine 視為「失聯」；不強制 kill（Go 語言無安全 kill goroutine 手段），但：
   - ToolResult 以 `{success: false, error: {type: "cancelled", message: "extension handler did not honor cancel"}}` 回傳父 step；原 goroutine 可能持續執行（視為 resource leak，engine 記錄 `lifecycle.destroy_abandoned` observability 事件）
   - 同一 engine 進程後續呼叫同一 handler MAY 繼續；不因「上次未終止」拒絕新呼叫（handler 實作者自行處理 thread / goroutine safety）
3. **WASM 與受限 runtime 的強制終止**：對 WASM component 等支援 host-side termination 的 runtime，engine MAY 於 grace 期滿後觸發 instance trap（見 WASM component model 的 `interrupt` 機制）；Go plugin / Python 等無此能力的實作則僅能依賴 context cancellation

實作要點：

- Handler MUST 在**所有**阻塞路徑（I/O、sleep、channel receive）處理 `ctx` cancellation；於 callback_endpoint.Invoke 阻塞中 engine 會自動傳播 cancel 至 endpoint（endpoint 回 `CallbackResult{error: {type: "cancelled"}}`），不需 handler 額外處理
- Engine 不暴露獨立的 `handler.Cancel()` 方法；所有 cancel 語意以 `ctx` 為單一通道（Go idiom）；其他語言綁定以該語言的等價 cancellation 機制（Python `asyncio.CancelledError` / WASM component 的 interrupt trap）
- 若 handler 為 CPU-bound long loop 且無 ctx 檢查點，engine 無法主動中止；屬 handler 實作品質問題，engine 層僅能以 grace period 超時後視為 cancelled 並記錄 observability 事件

### Extension Handler 異常安全

- Extension handler 執行期間若發生 **未捕捉例外 / panic / WASM trap**，engine MUST：
  1. 攔截並記錄完整 stack trace 至 execution_log（redact secret 後）
  2. 構造 ToolResult `{success: false, error: {type: "extension_handler_panic", message: <摘要>, details: {handler_name, stack_digest}}}`
  3. Engine 進程本身**不**因 handler panic 而崩潰（以 recover / try-catch 包裹 invocation）
- Panic 後 handler 實例的狀態被視為**不可信**；同 engine 進程若後續再呼叫同 handler，MAY 重新初始化（對 Go plugin / Python 為重新 import；WASM 為重建 instance）
- Handler 若宣稱 `idempotent: true` 但發生 panic → 本次 attempt 仍計入 retry 計數；retry 前清空 signature cache（同一 attempt 的 panic 不視為「已完成」）
- 若 panic 發生於 callback_handler 執行中（而非 tool invoke 主流程）→ 以 `callback_result.error = {type: "extension_handler_panic"}` 回覆 tool；不中斷 tool 主流程

### Extension Registry

Extension handler 透過 **engine 啟動期註冊**的方式登記；v6 不支援執行期動態載入：

- 實作 MUST 在 engine 啟動前（或首個 instance 建立前）完成所有 handler 註冊
- 註冊方式由實作決定（常見：
  - Go 以 `engine.RegisterExtension("<handler_name>", handlerFn)` 編譯期綁定
  - Python 以 entry point (`slogan.extensions` group) 於 engine 啟動時掃描
  - 靜態設定檔列出 `handler_name → plugin_path` 由 engine 載入）
- `handler_name` 格式：`^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`（dotted snake_case）；保留前綴 `slogan.*` 歸 engine 核心使用
- 同一 `handler_name` 多次註冊 → engine 啟動失敗（fail-fast；不允許覆寫）
- 載入 tool definition 時，若 `backend.type: extension` 的 `backend.handler` 在 registry 中找不到 → 載入期拒絕，`registry.extension_handler_not_found`
- Extension handler 無版本管理（v6 簡化）；同 engine build 下所有 instance 共用同一 handler 實例

---

## Lifecycle

### init

- Resource Pool 維護 cache：`(tool_name, version, instance_id) → init_output`
- 第一次解析該 tool 時：driver 透過同樣機制執行 `lifecycle.init.backend`，取得 output，寫入 cache
- 後續解析直接讀 cache；`context.lifecycle.init` 注入該 output
- init 失敗：cache 寫入 sentinel `__failed__`；後續解析直接 raise `lifecycle_init_failed`

#### 重啟後的 init cache 有效性

Engine crash → 新 engine 進程接管 instance 後對 cache 的處理：

| init output 類型 | 預設行為 |
|------------------|----------|
| 預設（未標記） | **忽略舊 cache**，重啟後重新執行 init（避免 session token 已過期） |
| Tool 標記 `lifecycle.init.reusable_across_restart: true` | 條件性復用既存 `init_output`（見下） |

- resource_pool 表新增欄位 `reusable_across_restart: bool`（預設 false），由 init 階段依 tool definition 寫入
- 舊 cache 若被忽略，MUST 在新 init 成功寫入前以 `init_output=null, init_error=null` 重設該 row；避免「舊值殘留又無新值」的不一致視圖

**`reusable_across_restart: true` 的復用規則**（避免跨 engine 復用不可用資源）：

接管者取得 lease 後檢查 `resource_pool.init_engine_id`：

| 條件 | 行為 |
|------|------|
| `init_engine_id == 當前 engine_id` | 視為「同進程重啟」；復用 `init_output`（假設是原進程 crash 後同 PID 重啟，或 in-process graceful restart） |
| `init_engine_id != 當前 engine_id` | 視為「跨 engine 接管」；**忽略舊 cache**，重新執行 init（與 `reusable_across_restart: false` 相同路徑） |

理由：init_output 可能持有 process-local 的參照（連線池 handle、本地 cache key 等），跨 engine 進程不可移植；`reusable_across_restart` 的「可復用」僅針對同 engine 進程的生命週期（重啟後 engine_id 相同的實作較罕見，但若實作以 persistent engine_id 設計則可支援）。

- 跨 engine 接管時 MUST 先 UPDATE `init_output=null, init_error=null, init_engine_id=<new_engine_id>` 再執行 init（同 row 更新，避免 race）
- 實作若要支援真正的「跨 engine 可復用」（如 session token 這類純資料型 output），建議 tool 作者於 init 內**不依賴 process-local 狀態**，並於 tool invoke 時檢查 cache 值有效性（如 token 過期 → 觸發 HTTP 401 retry 重新 init）

### destroy

- Instance 終結時，Resource Pool 列出所有 init 過的 tool；逐一執行 destroy backend
- destroy 失敗僅記錄；不影響 instance 終態
- destroy 順序：與 init 反序

---

## 觀測性 / 限制

| 監測項 | 行為 |
|--------|------|
| Process spawn 失敗 | ToolResult `error.type` 依原因細分（見 `09-error-model.md`）：`spawn_failed.not_found` / `permission_denied` / `resource_exhausted` / `oom` / `working_dir`；error.code 保留 errno 數值 |
| stdout 累積超過 `engine.tool_stdout_raw_limit`（預設 64 MB） | 立即 SIGTERM（5s grace → SIGKILL）；ToolResult `{success: false, error.type: "stdout_too_large"}`；保留前 4 KB 為 error.details.preview。**不**嘗試解析殘留內容 |
| 解析後的 output JSON 超過 `engine.max_step_output_bytes`（預設 16 MB） | step FAILED，`error.type: "output_too_large"`（與 persistence 規則一致） |
| Process 寫 stderr 速率過高（預設 > 10 MB/s 且持續 ≥ 5s） | rate-limit log；其餘丟棄；不影響成功判定 |
| HTTP body 超過 `engine.http_body_limit`（預設 16 MB） | 同 stdout：不解析、`error.type: "http_body_too_large"` |
| 未終止的 SSE 連線 | `engine.sse_idle_timeout`（預設 60s）無訊息後 driver 主動關閉，視為 `incomplete_protocol` |
| Process 留下殘餘 child（forked grandchild） | driver SHOULD 透過 process group 一併 kill |

**stdout_raw vs output 的關係**：

- `tool_stdout_raw_limit` 是「process 寫至 stdout 的原始 bytes 總量」上限（含 NDJSON framing、log 行等），用於防止 tool 失控輸出撐爆 driver buffer
- `max_step_output_bytes` 是「parsed JSON output 序列化後寫入 Instance Store 的 bytes」上限
- 先觸發者先生效；raw limit 觸發 → tool 被 kill、step FAILED；output limit 觸發 → tool 正常退出但 step FAILED
- 兩者皆可由 engine config 獨立覆寫；建議 `raw_limit ≥ 4 × output_bytes_limit`（保留 protocol overhead 空間）
