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
| `mode: protocol` 且 caller 有 `callback:` | **NDJSON streaming**（v5-stream） |
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

### NDJSON streaming（v5-stream）

driver 與 tool 以行分隔 JSON 雙向通訊。

#### 引擎 → tool（stdin 第一行）

```json
{"protocol": "v5-stream", "input": {...}, "config": {...}, "context": {...}}
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

- 行長上限：建議 1MiB；超過 → 行視為損壞，driver 寫 stderr log 並丟棄。
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
6. 依 exit_code.success 判斷成功
```

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

- `working_dir`：預設為 engine 啟動的 cwd；建議實作為 instance 的 sandbox 子目錄。
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
- mTLS / client cert 由 backend.config 提供路徑；secret 引用解密後的 key 需在記憶體 holding。

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
| Process spawn 失敗 | ToolResult `error.type == "spawn_failed"` |
| stdout 超過 size limit（建議 16MB） | 截斷尾部；保留前段；error 記錄 truncation |
| Process 寫 stderr 速率過高（建議 > 10MB/s） | rate-limit log；其餘丟棄 |
| HTTP body 超過 size limit | 同上 |
| 未終止的 SSE 連線 | timeout 後 driver 主動關閉 |
| Process 留下殘餘 child（forked grandchild） | driver SHOULD 透過 process group 一併 kill |
