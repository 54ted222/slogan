# 06 — Tool Backend Driver

本文件定義 Backend Driver 對 `kind: Tool` 的 `backend.type: exec | http | extension` 的執行細節，以及 callback / streaming 的協議實作。對應 [dsl-spec/05-tool.md](../dsl-spec/05-tool.md)。

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
  stream_events: int,            # 計數，非保留訊息本體
}
```

`callback_handler` 由 Engine Loop 提供：是一個函式 `handle(call_id, name, input) -> CallbackResult`，driver 觸發後將結果回灌 tool。

### Schema 驗證職責

所有 backend（exec protocol / exec raw / http / extension）在建立 ToolResult 前 MUST 由 driver 驗證：

1. **input_schema**（tool 定義有此欄位時）：對 `input` 在 spawn 前驗證；不符 → 不啟動 tool，ToolResult `{success: false, error: {type: "schema_violation", details: {direction: "input", path: ...}}}`
2. **output_schema**（tool 定義有此欄位時）：對 `output`（raw 模式下為 mapping 後結果）在成功分支驗證；不符 → ToolResult `{success: false, error: {type: "schema_violation", details: {direction: "output", path: ...}}}`；原始 output 保留於 error.details.raw_output

Engine Loop 不重複驗證；若 driver 未驗證，錯誤將以 tool 回傳結構傳播，但 step 可能已完成（違反 schema 保證）。因此驗證責任在 driver 層。

---

## exec backend

### Process 啟動

```
1. 解析 backend.command / args / env / working_dir / shell
2. 對 args、env、working_dir 中的 ${ } 做 expression 求值
3. spawn process（依 platform 採 fork/exec）
4. 設定 stdin / stdout / stderr 為非阻塞 pipe
5. 啟動 reader / writer task；start time 記錄為 RUNNING checkpoint 的時間
```

### 模式判定

| 條件 | 模式 |
|------|------|
| `mode: protocol` 且 caller 無 `callback:` | **單訊息 protocol** |
| `mode: protocol` 且 caller 有 `callback:` | **NDJSON streaming**（v6-stream） |
| `stream.enabled: true` | **NDJSON streaming**（強制） |
| `mode: raw` 或預設 | **raw** |

### 單訊息 protocol

```
1. 寫入單一 JSON 至 stdin、close stdin
2. 等待 process 輸出單一 JSON 至 stdout
3. process exit 後解析 stdout
4. 構造 ToolResult：{ success, output, error } 取自 JSON
```

stderr 寫入 execution log，不影響成功判定。

### NDJSON streaming（v6-stream）

driver 與 tool 以行分隔 JSON 雙向通訊。

#### 引擎 → tool（stdin 第一行）

```json
{"protocol": "v6-stream", "input": {...}, "config": {...}, "context": {...}}
```

stdin 保持開啟；後續行為 `callback_result` 訊息。

#### tool → 引擎（stdout，逐行讀）

| `type` | 處置 |
|--------|------|
| `callback` | 解析 `{call_id, name, input}` → 呼叫 `callback_handler.handle()`（背景 task）→ 完成後寫 `callback_result` 至 stdin |
| `stream`   | 解析 `{sequence, data, final}` → 發 `tool.stream` 事件至 Event Bus |
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
- Tool 在收到 `callback_result` 前可繼續輸出 `stream` / 發更多 `callback`。
- Driver MUST 將 `call_id` 對應表持久化（key: `(instance_id, step_id, call_id)`），避免 process 中途崩潰時 callback 結果遺失。
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

raw 模式不支援 callback 與 stream（caller 即使宣告 `callback:`，driver 在啟動時 raise validation error）。

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
  - `working_dir` 求值結果 MUST 在 workspace 下（canonical 化後以 workspace_path 為前綴）；escape 外部 → ToolResult `{success: false, error: {type: "spawn_failed.working_dir"}}`
- `shell`：明確指定時用該 shell 執行 `command`；未指定時直接 `exec()` `command + args`，不經 shell（避免 quoting 問題）。

### Env 變數

最終 env = OS 環境 ⨁ project env ⨁ backend.env（後者覆寫前者）。

`secret.*` 注入後在 process 環境中以明文存在；driver 啟動後**立即**從本身記憶體中清除其副本（best-effort），避免 dump core 時暴露。

---

## http backend

### 一般請求

```
1. 解析 url / method / headers / request.body
2. CEL 求值 ${ } 欄位（含 secret 注入）
3. 發送 HTTP；遵守 timeout（預設 30s，可由 backend 覆寫）
4. retry_on_status：列表內的 HTTP code 觸發 retry（搭配 step.retry）
5. error_on_status：列表內的 HTTP code 直接 FAILED
6. 解析 response：若 response.mapping 存在 → 對 raw response body 套 CEL；否則 raw 即 output
```

raw 為 JSON-decoded body（content-type 為 JSON 時）或 string body（其他）。

### Callback 模式（caller 宣告 callback:）

```
1. 設定 Accept: text/event-stream
2. 發送請求
3. 檢查 response header：MUST 含 X-Callback-URL
4. 解析 SSE 事件流：
   - event: callback → 解析 data → 呼叫 callback_handler → POST result 至 X-Callback-URL
   - event: stream   → 發 tool.stream 事件
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
2. 構造 ExtensionRequest：{ tool_name, input, context, callback_endpoint? }
3. 呼叫 handler.invoke(request)
4. handler 回傳 ExtensionResponse 或 stream
```

協議由 handler 自行宣告；engine 僅保證：

- 提供與 exec/http 相同的 callback_handler 介面
- 提供 stream 事件投遞通道
- 將最終結果包裝為 ToolResult

extension handler 必須是 in-process（Go plugin、Python entry point、WASM module 等）；out-of-process 由其自行包裝為 exec / http backend。

---

## Lifecycle

### init

- Resource Pool 維護 cache：`(tool_name, version, instance_id) → init_output`
- 第一次解析該 tool 時：driver 透過同樣機制執行 `lifecycle.init.backend`，取得 output，寫入 cache
- 後續解析直接讀 cache；`context.lifecycle.init` 注入該 output
- init 失敗：cache 寫入 sentinel `__failed__`；後續解析直接 raise `lifecycle_init_failed`

### destroy

- Instance 終結時，Resource Pool 列出所有 init 過的 tool；逐一執行 destroy backend
- destroy 失敗僅記錄；不影響 instance 終態
- destroy 順序：與 init 反序

---

## 觀測性 / 限制

| 監測項 | 行為 |
|--------|------|
| Process spawn 失敗 | ToolResult `error.type` 依原因細分（見 `09-error-model.md`）：`spawn_failed.not_found` / `permission_denied` / `resource_exhausted` / `oom` / `working_dir`；error.code 保留 errno 數值 |
| stdout 超過 size limit（建議 16MB） | 截斷尾部；保留前段；error 記錄 truncation |
| Process 寫 stderr 速率過高（建議 > 10MB/s） | rate-limit log；其餘丟棄 |
| HTTP body 超過 size limit | 同上 |
| 未終止的 SSE 連線 | timeout 後 driver 主動關閉 |
| Process 留下殘餘 child（forked grandchild） | driver SHOULD 透過 process group 一併 kill |
