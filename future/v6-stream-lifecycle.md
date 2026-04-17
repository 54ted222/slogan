# Stream Tool（含生命週期與優雅關閉）— 延後至未來版本

> **狀態**：v6 原本規劃支援 tool 的即時串流輸出（content-level stream）；於 2026-04-17 的審閱中，因發現其生命週期與優雅關閉機制有明確缺口，決定自 v6 移除、延後至未來版本再引入。
>
> 本檔完整歸檔原設計與缺口分析，未來重新規劃時可作為起點。

---

## 背景：為何從 v6 移除

### 原設計範圍（v6 第四十一輪前）

v6 原本在 DSL 與 runtime 中定義了 content-level stream，允許 tool 在執行過程中逐段輸出資料供 workflow 訂閱：

1. DSL 層 `backend.stream.enabled: true`（exec backend）
2. Exec NDJSON 訊息型別 `{"type":"stream", "sequence", "data", "final"}`
3. HTTP SSE `event: stream` 事件
4. Event Bus 內部事件 `tool.stream`（workflow 可用 `wait` 訂閱）
5. Tool crash 時 partial stream 的處置、sequence 單調遞增驗證等

### 四大缺口（質疑後核對確認成立）

使用者回饋：workflow 創建應用後開啟 stream tool，當工作流程結束時 stream 無法得知、無法主動正常關閉。核對後確認：

1. **HTTP/SSE stream 無語義層 graceful close 訊號**
   - v6 只做「close connection」（`runtime-spec/10-concurrency.md` cancel propagation）
   - Tool 端無法區分「workflow 正常結束」與「網路故障」
2. **無通用 finalizer / on_close hook**
   - Cancel 路徑有 SIGTERM（exec）/ close connection（HTTP）/ `handler.cancel()`（extension）
   - 但 **succeeded 路徑**未定義通知機制 — tool 若需要在 workflow 正常結束時清理資源（flush buffer、關閉下游連線），沒有標準化管道
3. **長駐 application 非一等公民**
   - v6 有 `Function`（子 workflow）、`Tool`（有限生命週期的 call），但無「長駐 application / session」概念
   - 將長駐服務包為 stream tool，其生命週期綁在單次 task step — 易產生「step 等 stream 結束 vs stream 等 step 結束」的雞蛋問題
4. **Stream 沒有獨立 `max_duration`**
   - 只能靠外層 step / workflow timeout 兜底
   - 若 workflow 為長流程（如 7 天人工審批）、stream tool 原設計只想跑 1 小時，無獨立上限欄位

---

## 原設計（完整歸檔）

以下原文自 v6 規格（於移除前的最終版本）完整搬移至此。

### A. DSL：串流輸出（原 `v6/dsl-spec/05-tool.md` 的「串流輸出（Streaming）」章）

exec backend 支援持續性輸出，適用於長時間執行且需要中間結果回報的 tool。

```yaml
backend:
  type: exec
  command: "python ./tools/batch_process.py"
  stdin:
    format: json
  stream:
    enabled: true # MUST
    format: jsonl | delimited | grpc # MAY, 預設 jsonl；grpc 目前暫不支援
    # jsonl: 每行一個 JSON object
    # delimited: 使用 begin/end 標記分隔
    # grpc: 保留格式，目前暫不支援
    begin: "---BEGIN---" # MAY — 僅 delimited 格式
    end: "---END---" # MAY — 僅 delimited 格式
```

#### 串流行為

- 引擎對每段解析出的資料發出 `tool.stream` 事件，可被 workflow 的 event handler 接收
- tool 結束後（exit），最後一段輸出作為 step 的 `output`
- `grpc` 為保留值，現階段不可使用
- 串流中的每段資料結構：

```json
{
  "sequence": 1,
  "data": { ... },
  "final": false
}
```

#### jsonl 範例

```python
import sys, json

items = json.loads(sys.stdin.read())["items"]
for item in items:
    result = process(item)
    print(json.dumps({"item_id": item["id"], "status": "done"}))
    sys.stdout.flush()

# 最後一行作為最終 output
print(json.dumps({"total": len(items), "status": "completed"}))
```

### B. DSL：exec（stream 模式）NDJSON 訊息路由

`stream.format: jsonl` 下沿用同一 NDJSON framing。引擎依 `type` 欄位分流：

| `type`            | 處置                                     |
|-------------------|------------------------------------------|
| `callback`        | 路由至 caller handler |
| `stream`          | 發出 `tool.stream` 事件 |
| `result`          | 最終結果，接收後關閉 |
| 缺少 `type` 或其他 | 協議錯誤 → step FAILED |

Tool 可混用 `stream` 與 `callback` 訊息，最後以單一 `result` 訊息收尾。

### C. DSL：HTTP SSE stream 事件

HTTP backend 在 caller 宣告 `callback:` 時啟用 SSE（`Content-Type: text/event-stream`）雙向串流。Tool 可輸出 `callback` / `stream` / `result` 三種事件：

```
event: callback
data: {"call_id": "cb-1", "name": "<callback_name>", "input": {...}}

event: stream
data: {"sequence": 1, "data": {...}, "final": false}

event: result
data: {"success": true, "output": {...}, "error": null}
```

### D. Runtime：Stream 訊息驗證規則（原 `06-tool-backend.md`）

- `sequence` MUST 為非負整數；驗證值的**單調遞增**（每筆 > 前筆）；違反 → 協議錯誤 `incomplete_protocol`，SIGTERM kill tool
- `data` 為任意 JSON 型別；單筆 `data` 序列化後 size 受 `engine.event_bus_max_data_bytes`（預設 1 MB）限制；超過 → 協議錯誤
- `final` MUST 明確存在且為 boolean；缺失或非 boolean → 協議錯誤
- **終結規則**：收到 `final: true` 之後，若再收到任何 `stream` 訊息 → 協議錯誤
- `final: true` 不等同 `result`：tool 仍可在 `final: true` 後發 `callback`；但 MUST 最終以單一 `result` 訊息收尾
- Stream 訊息的 `sequence` 與 Event Bus 的 `source.sequence` 為**獨立計數**；tool.stream 事件的 `source.sequence` 由 engine 依 event bus 規則分配，不等同 stream message 的 sequence 值（後者保留於 `tool.stream.data.stream_sequence` 方便使用者串接）

### E. Runtime：Tool crash 時 partial stream 的處置

若 tool 在發出 N 筆 stream 訊息後（尚未送出 `final: true` 或 `result`）**異常終止**（SIGKILL / OOM / segfault / exit 非 0），引擎處置：

- 已寫入 event bus 的 N 筆 `tool.stream` 事件**不撤回**（event bus 本身無 rollback；訂閱者已接收的事件照常存在）
- Step 終態為 FAILED，`error.type == "backend_crashed"`（若能取得 exit code）或 `incomplete_protocol`（exit 但無 result）
- `tool.stream` 事件的最後一筆**不會** 帶 `final: true`（tool 未送達）；訂閱 `tool.stream` 的 wait step 若以 `match: ${ event.data.final }` 等待結束 → 不會自動喚醒，依 `wait.timeout` 兜底
- 建議訂閱者於 `wait.catch` 同時處理 `tool.stream` 匹配與 tool 呼叫本身的 FAILED（兩者為獨立訊號）
- `step.completed`（呼叫 tool 的 task step）於 step 終態後 emit，訂閱者可據此判定 stream 是否為完整序列（step SUCCEEDED + last stream 的 `final: true`）或不完整（step FAILED）

### F. Runtime：`tool.stream` Event Bus 規格

- `tool.stream`：由 Tool Backend Driver 在 `mode: stream` 下發出；scope: `internal`；data: stream payload
- workflow 可訂閱 `tool.stream` 事件透過 `wait` 取得資料（將 event_type 設為 `tool.stream` 並 match）
- delivery：`broadcast`（內部）；target: null；供 workflow 的 wait 訂閱
- 持久化：中間訊息**不持久化**（即時投遞至訂閱者後丟棄）
- 內部事件目錄中定位：由 Backend Driver 發送，Engine Loop / 訂閱 wait 接收，用途為串流投遞
- 保留前綴：`tool.*`（tool backend 事件，如 `tool.stream` / `tool.callback`），`emit` 不得宣告

### G. Runtime：extension backend 的 stream 投遞

extension handler 自行定義傳輸方式；engine 保證提供 `stream` 事件投遞通道，將中間事件注入 event bus。

### H. Runtime：raw 限制考量

`tool_stdout_raw_limit` 是「process 寫至 stdout 的原始 bytes 總量」上限，含 NDJSON framing、stream 訊息、log 行等；建議 `raw_limit ≥ 4 × output_bytes_limit`（保留 protocol overhead 與 stream 空間）。

---

## 未來重新引入時需解決的設計議題

### 議題 1：Graceful close 訊號（補足缺口 1 與 2）

提案：擴充 callback / NDJSON 協議，讓引擎在 workflow 結束時主動通知 tool：

```json
{"type":"workflow_ended","reason":"succeeded|cancelled|timeout","grace_period_ms": 5000}
```

- **succeeded**：workflow 正常結束；tool SHOULD 儘速 flush 並發 `result`
- **cancelled / timeout**：對應現行 cancel 路徑，但走統一訊息型別
- HTTP SSE：新增 `event: workflow_ended`
- extension：handler 新增 `on_workflow_ended(reason)` callback

### 議題 2：DSL 層 finalizer / on_close hook

```yaml
backend:
  type: exec
  command: "..."
  stream:
    enabled: true
    on_close:
      timeout: 10s          # grace period
      expect_final: true    # tool 是否必須在此期間發 final: true
```

區分 succeeded 與 cancelled 路徑的清理動作；tool 作者可在 hook 內 flush buffer、回收資源。

### 議題 3：Stream 獨立 `max_duration`

```yaml
stream:
  enabled: true
  max_duration: 1h    # 與 step.timeout / workflow.timeout 獨立
```

觸發時視為「自然結束」走 succeeded close 路徑（而非 cancel）。

### 議題 4：Application / Session 一等公民

若要支援「workflow 創建長駐應用」場景，需在 DSL / runtime 引入新概念：

```yaml
apiVersion: application/v7  # 示意
kind: Application
metadata:
  name: user_session
spec:
  lease:
    heartbeat_interval: 30s
    max_idle: 10m
  lifecycle:
    on_start: ...
    on_stop: ...
    on_workflow_ended: ...
```

- 與 tool 的差別：tool 生命週期綁在 step，application 生命週期獨立、被 workflow explicit start/stop
- 需要新 runtime-spec 章節處理 lease、heartbeat、session end 事件
- 對 stream tool 的關係：application 可 host 多個 stream 並共享生命週期

### 議題 5：再次引入 content-level stream 時與 `v6-protocol`（NDJSON framing）關係

v6 移除 content-level stream 時已將 NDJSON framing 改名為 `v6-protocol`（原 `v6-stream`），專用於 callback 雙工通道。未來若要重引 content-level stream：

- 建議新增獨立 `type: stream` 訊息型別（不改 framing 本身）
- 若改 framing 協議版本 → 採 `v7-protocol` 或 protocol 內 capability flag 表達

---

## 涉及的現行 v6 章節參考

移除時影響的 v6 章節（以本決策時的行號為準）：

- `v6/dsl-spec/05-tool.md` L695-745（串流輸出整章）、L811-822（exec stream 模式）、L829 SSE stream 事件
- `v6/dsl-spec/03-steps.md` L344（保留前綴範例 `tool.stream`）
- `v6/dsl-spec/05b-function.md` L242（http stream 提及）
- `v6/runtime-spec/06-tool-backend.md` L24、L105-178（NDJSON stream 段）、L212、L253、L293、L347、L401-407、L481-484
- `v6/runtime-spec/07-event-bus.md` L12、L50、L95、L296-299、L319
- `v6/runtime-spec/01-architecture.md` L82
- `v6/runtime-spec/02-instance-lifecycle.md` L135、L262

---

## 決策時間線

- 2026-04-17：使用者回饋 stream 關閉機制缺口 → 核對確認四大缺口成立 → 決定自 v6 移除並歸檔至本檔
