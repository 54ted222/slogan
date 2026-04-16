# 01 — 架構

本文件描述 v6 engine 的核心元件與責任分工。所有元件都是邏輯角色，可由實作合併至同一程序或拆分為微服務。

---

## 元件地圖

```
                    ┌───────────────────────┐
   trigger source ──► Trigger Resolver       │  → 建立 instance
                    └─────────┬─────────────┘
                              │
                    ┌─────────▼─────────────┐
                    │ Instance Store         │  ← 持久化 state / output / log
                    └─────────┬─────────────┘
                              │
   ┌───── event bus ◄────► Engine Loop ────────────────┐
   │                          │                        │
   │              ┌───────────┼─────────────┐          │
   │              │           │             │          │
   │      ┌───────▼──┐   ┌────▼─────┐  ┌────▼─────┐    │
   │      │ Step      │   │ Expr      │ │ Task       │  │
   │      │ Executor  │   │ Evaluator │ │ Registry   │  │
   │      └───────┬──┘   └───────────┘ └────┬─────┘    │
   │              │                         │          │
   │     ┌────────▼──────┐         ┌────────▼──────┐   │
   │     │ Backend Driver│         │ Resource Pool │   │
   │     │ (exec/http/ext)│        │ (MCP/secret/...)│ │
   │     └────────┬──────┘         └───────────────┘   │
   │              │                                    │
   └──────────────┴────────────────────────────────────┘
```

---

## 元件責任

### Trigger Resolver
- 訂閱 event bus 與外部 API；當 trigger 條件成立，建立 workflow instance。
- 依 `input_schema` 驗證輸入；不通過 → 回報 `workflow.input_schema_violation`、不建立 instance。
- 將初始 instance 寫入 Instance Store，狀態 `PENDING`，並向 Engine Loop 發出 `instance.created` 事件。

### Engine Loop
- 主事件迴圈：消費 `instance.created` / `step.completed` / `event.matched` / `wait.timeout` / `tool.callback` 等事件。
- **執行模型**：類 Node.js 的單線程 event loop。單一 engine 進程內只有一個事件迴圈 thread，處理所有 instance 的事件分派與 step transition。所有 I/O（tool process、HTTP、event bus、Instance Store）皆為 non-blocking；主 loop 絕不阻塞在同步 I/O 上。
- **水平擴展**：啟動多個 engine 進程；跨進程透過 lease 分配 instance 擁有權（見 `02-instance-lifecycle.md`）。進程間不共享記憶體，Instance Store 為唯一共享狀態來源。
- **CPU-bound 下放**：大型 JSON 處理、CEL 求值等若有阻塞疑慮，實作 MAY 使用 worker thread pool（Node.js worker / Rust rayon / Go goroutine pool 等）執行後 callback 回主 loop；主 loop 本身 MUST 保持 non-blocking。
- 對每個 instance 維護 **單一 logical execution thread**：同時刻僅一個 step 處於 `RUNNING`（除非該 step 是 parallel/foreach/wait 等多分支類型；此時其子節點各自處於 RUNNING）。由於整個 engine 進程本身就是單線程，同進程內的 instance 之間也不會發生 thread-level 搶佔。
- 不持有 step 的執行邏輯，只負責調度。執行語意委派給 Step Executor。

### Step Executor
- 接收一個 step + 當前 namespace snapshot，回傳 `StepOutcome`（`{status, output, error, side_effects}`）。
- 不直接 I/O；I/O 透過 Backend Driver / Task Registry 取得能力。
- 對 `task` / `wait` / `foreach` 等不同 type 有專用 handler。

### Expression Evaluator
- 純函式 CEL 求值器；輸入 `(expression, namespaces)`，輸出值或拋異常。
- 不知道 instance 持久化、不知道事件；只負責 CEL。
- 失敗時拋 `ExpressionError(message, expr_fragment)`；由呼叫端決定是否轉為 step FAILED。

### Task Registry
- 載入時聚合所有 `kind: Tool` / `kind: Function` definition。
- 提供 `resolve(action_name) → Action`；`Action` 可被執行（內含類別、來源、執行入口）。
- 解析依「第一段命名風格」分流（見 `dsl-spec/01-overview.md` 的「Task registry 解析規則」與本文件 `05-task-registry.md`）。

### Backend Driver
- 為 `kind: Tool` 的 `backend.type` 提供具體執行：
  - **exec**：spawn process、管 stdin/stdout/stderr、處理 NDJSON framing 與 callback。
  - **http**：HTTP / SSE client；處理 `X-Callback-URL` 雙向流。
  - **extension**：呼叫第三方 handler；協議由 handler 自行宣告。
- Driver 是 stateless：所有 lifecycle 狀態（init output、process handle）由 Resource Pool 持有。

### Resource Pool
- 持有 workflow instance 等級的共享資源：
  - Tool lifecycle init 的快取 output（key: `(tool_name, version, instance_id)`；per-instance，見 `06-tool-backend.md` Lifecycle 節）
  - HTTP keep-alive client、connection pool
  - Secret 解密後的明文（in-memory only，不寫盤）
- Instance 結束時觸發 destroy hook 並釋放資源。

### Event Bus
- 訊息匯流排：trigger 訂閱、`emit` step 發送、`wait` step 訂閱、`tool.stream` / `tool.callback` 內部事件。
- 支援 `scope`：`workflow`（僅本 instance）/ `project` / `global`。
- 不保證跨 scope 順序；同 scope 內依產生時間 FIFO。
- 重試與 dead-letter 由實作決定，但 SHOULD 提供至少一次（at-least-once）投遞。

### Instance Store
- 持久化所有 instance state、step output、execution log、wait subscription、saga 狀態。
- 提供原子 checkpoint：`(instance_id, step_id) → outcome` 的寫入是 atomic。
- 詳見 `08-persistence.md`。

---

## 跨元件 invariant

- **Step Executor 永不直接寫 Instance Store**；它回傳 `StepOutcome`，由 Engine Loop 寫入。
- **同一 instance 在同一時間僅由一個 engine 進程擁有**（lease 機制）；跨進程串行化、同進程內單線程 event loop 天然串行化。
- **Event Bus 的訊息是 effect-only**：訊息內不夾帶引擎控制流；engine 從 Instance Store 拉取真正的執行狀態。
- **Expression 求值結果不直接回寫 namespace**；step output 才是 namespace 的唯一寫入源。
- **Lifecycle init / destroy 不算 step**：不出現在 `steps` namespace、不寫入 step output；其結果存入 Resource Pool。

---

## 失效模型

| 失效類型 | 預期行為 |
|----------|----------|
| Engine Loop crash | Lease 過期，其他 engine 進程接管；從 Instance Store 的最新 checkpoint resume |
| Backend tool process crash | Step FAILED，`error.type == "backend_crashed"`；可被 `retry` / `catch` 處理 |
| Event Bus 投遞失敗 | 由 bus 實作重試；engine 視為事件未到達，繼續等待至 `timeout` |
| Instance Store 不可寫 | Engine Loop 該 instance 的 transition 延後（non-blocking 重試，不阻塞主 loop）；超過 SLO 則整個 engine 進程拒絕新 instance |
| Tool callback 路徑 handler FAILED | callback_result.error 回 tool；tool 自行決定處置 |

---

## 與 dsl-spec 的對應

| dsl-spec 概念 | runtime 元件 |
|---------------|--------------|
| `kind: Workflow` 載入 | Trigger Resolver 訂閱 |
| `triggers` | Trigger Resolver |
| `steps[]` 推進 | Engine Loop + Step Executor |
| `${ }` 求值 | Expression Evaluator |
| `type: task` action 解析 | Task Registry |
| `backend:` 執行 | Backend Driver + Resource Pool |
| `emit` / `wait` | Event Bus |
| `idempotent` / `retry` / `catch` / `saga` | Step Executor + Engine Loop |
| `kind: Secret` 解密 | Resource Pool |

---

## Engine Config

Engine 的各項 `engine.*_limit` / `engine.*_policy` 設定（散見於其他章節，如 `engine.max_function_call_depth`、`engine.tool_stdout_raw_limit`、`engine.event_bus_max_data_bytes`）由下列來源決定，**由高至低**優先級覆蓋：

| 來源 | 優先級 | 範例 |
|------|--------|------|
| CLI flag | 1（最高） | `slogan engine --max-function-call-depth=256` |
| 環境變數 | 2 | `SLOGAN_MAX_FUNCTION_CALL_DEPTH=256`（命名規則：`SLOGAN_` 前綴 + 欄位名 UPPER_SNAKE） |
| Engine config file | 3 | `/etc/slogan/engine.yaml` 的 `engine.max_function_call_depth: 256` |
| 內建預設值 | 4（最低） | 各章節標註的 `預設 N`（如 128） |

- 同一欄位於多來源存在 → 高優先級完全覆寫（不合併）
- 數值解析錯誤（如環境變數為非數字字串）→ engine 啟動失敗，log error；不回退至低優先級
- Config file 路徑預設 `/etc/slogan/engine.yaml`，可由 `--config` 或 `SLOGAN_CONFIG` 覆寫
- 啟動時 engine MUST 於 log 輸出有效配置表（含每欄位最終值與來源），便於 debug
- Runtime 不支援動態重載；修改後需重啟 engine
