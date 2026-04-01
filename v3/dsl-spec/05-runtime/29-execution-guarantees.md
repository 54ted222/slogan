# 29 — 執行語意保證

本文件定義 task 與 agent step 執行的 delivery guarantee、冪等 key 在 recovery 中的行為、HTTP retry 與 task retry 的區分、race condition 處理、crash recovery 語意、以及 deterministic replay 原則。

---

## 核心原則

| 原則 | 說明 |
|------|------|
| DB 為唯一 truth | 記憶體狀態僅為快取，crash 後從 DB 恢復 |
| Deterministic replay | 相同的 definition + 相同的事件序列 = 相同的結果 |
| 非確定性函式記錄 | `now()` 和 `uuid()` 的求值結果 MUST 與 step instance 一同持久化；replay 時使用已記錄的值 |
| Optimistic locking | Step instance 更新 SHOULD 使用 version column 防止並行更新 |

---

## Delivery Guarantee

### 整體語意

引擎對 task 及 agent step 執行提供 **at-least-once** 保證：

| 保證 | 說明 |
|------|------|
| At-least-once | 每個 task / agent step 至少被嘗試執行一次。Crash 後 recovery 可能導致重複執行 |
| **非** exactly-once | 引擎無法保證 step 只被執行恰好一次。需要 exactly-once 語意的 step 應透過 `execution_policy` + `idempotency key` 自行實現 |

### 各 Execution Policy 的語意

| Policy | Crash Recovery 行為 | 對 handler 的要求 |
|--------|--------------------|-----------------------|
| `replayable` | 無條件重跑 | Handler MUST 為冪等，或可接受重複執行（如 read-only 操作、有補償邏輯的操作） |
| `idempotent` | 帶 idempotency key 重跑 | Handler SHOULD 依 idempotency key 去重，確保重複呼叫不產生副作用 |
| `non_repeatable` | 直接標記 FAILED | Handler 不需要冪等。引擎寧可失敗也不重複執行（適用於扣款、發送通知等） |

### emit step 的保證

emit step 提供 **at-least-once** 保證：

- 事件 MUST 在 step 標記為 SUCCEEDED 之前被持久化
- Crash recovery 後，若 step 狀態為 RUNNING → 重新 emit（可能重複發送）
- 下游消費者 SHOULD 能處理重複事件（透過 `event.id` 去重）

---

## Idempotency Key 詳細規格

### Key 格式

```
{workflow_instance_id}:{step_id}:{attempt}
```

- `attempt` 從 1 開始
- 每次 retry 遞增 attempt，產生不同的 key
- Crash recovery 重跑**不**遞增 attempt（使用 crash 前的 attempt 值），確保重跑使用相同的 key

### 傳遞方式

| Backend | 傳遞位置 |
|---------|---------|
| stdio | `context.idempotency_key`（request JSON 欄位）+ `SLOGAN_IDEMPOTENCY_KEY` 環境變數 |
| http | `Idempotency-Key` HTTP header |

> `builtin` backend 已在 v3 中移除（詳見 [14-agent-tools](../03-agent/14-agent-tools.md)），改以 `type: tool` 兩段式命名取代。

### Handler 端實作建議

Tool handler 收到 idempotency key 後 SHOULD：

1. 以 key 查詢是否已處理過（如資料庫記錄、external API 的 idempotency 機制）
2. 若已處理 → 回傳之前的結果，不重複執行
3. 若未處理 → 正常執行，並記錄 key 與結果
4. Key 的保留期限由 handler 自行決定（建議至少 24 小時）

### Recovery 場景

```
Step RUNNING → Engine crash
     ↓
Engine restart → 載入 step instance（state = RUNNING, attempt = N）
     ↓
依 execution_policy 決定：
  replayable    → 重跑（attempt 不變 = N）
  idempotent    → 帶 key 重跑（key = instance:step:N，與 crash 前相同）
  non_repeatable → 標記 FAILED
```

重點：`idempotent` policy 在 crash recovery 時使用與 crash 前相同的 key，讓 handler 能辨識為重複請求。

---

## HTTP Retry 與 Task Retry 的關係

Task step 的 `retry` 機制與 HTTP backend 的 `retry_on_status` 是兩個**獨立的層級**：

### 層級區分

```
Task Step retry（workflow 層）
  ↓ 每次 retry attempt 呼叫一次 Task Executor
  ↓
Task Executor（HTTP backend）
  ↓ 依 retry_on_status 判斷是否為暫時性失敗
  ↓
HTTP 請求結果
```

### 交互行為

| HTTP 結果 | `retry_on_status` 匹配 | Task Executor 回報 | Task Step retry |
|-----------|----------------------|-------------------|----------------|
| 2xx | — | 成功 | 不觸發 |
| 429 | 在列表中 | 可重試失敗 | 觸發（若 attempts 未用盡） |
| 500 | 不在列表中 | 永久性失敗 | 觸發（若 attempts 未用盡） |
| 連線失敗 | — | 可重試失敗 | 觸發（若 attempts 未用盡） |

### 重要區別

- **HTTP backend 不自行重試**：`retry_on_status` 僅影響 Task Executor 回報的失敗類型，不會在 HTTP 層做重試
- **Task step retry 控制所有重試**：無論是暫時性還是永久性失敗，都由 task step 的 `retry` 設定統一控制
- **每次 retry attempt = 一次完整的 HTTP 請求**

### 可重試失敗 vs 永久性失敗

| 失敗類型 | 說明 | retry 行為 |
|---------|------|-----------|
| 可重試（transient） | 暫時性問題，重試可能成功 | 正常 retry（respect delay + backoff） |
| 永久性（permanent） | 不可能透過重試解決 | 仍然 retry（引擎不區分），直到 attempts 用盡 |

引擎**不區分**可重試與永久性失敗的 retry 行為。無論失敗類型，只要 `max_attempts` 未用盡就重試。區分的用途在於讓使用者在 `catch` handler 中透過 `error.code` 判斷失敗原因。

---

## Recovery 機制

### Instance Recovery 規則

引擎重啟時，載入所有非終態（CREATED / RUNNING / WAITING）的 instance，從 DB 重建狀態：

| Instance 狀態 | Recovery 行為 |
|--------------|---------------|
| CREATED | 轉為 RUNNING，開始執行 |
| RUNNING | 依據 step 狀態恢復（見下方 Step Recovery） |
| WAITING | 重建 wait_subscription，繼續等待 |

### Step Recovery 規則

`execution_policy` 適用於 `task`、`sub_workflow` 與 `agent` step。其他 step 類型（assign、emit、if、switch、foreach、parallel、saga）在 RUNNING 狀態下的 recovery 行為等同 `replayable`（安全重跑）。

| Step 狀態 | Execution Policy | Recovery 行為 |
|-----------|-----------------|---------------|
| RUNNING | `replayable`（或無 policy） | 重跑 step |
| RUNNING | `idempotent` | 帶 idempotency key 重跑 |
| RUNNING | `non_repeatable` | 標記 FAILED |
| WAITING | — | 重建 wait_subscription，繼續等待 |
| READY | — | 正常執行 |
| PENDING | — | 不動作，等待前置 step |

### 關鍵保證

- Timeout schedule MUST 在 crash 後被重建（根據 DB 中的 `fires_at`）
- Wait subscription MUST 在 crash 後被重建
- 過期的 timeout / wait 在重啟時立即處理
- 非確定性函式（`now()`、`uuid()`）的求值結果 MUST 在首次執行時持久化，replay 時使用已記錄的值

---

## Agent Step Crash Recovery 語意

Agent step 的 crash recovery 比 task step 更複雜，因為 agent session 包含 LLM conversation context。

### 各 Policy 的 Recovery 行為

| Policy | Recovery 行為 |
|--------|---------------|
| `replayable`（預設） | 查詢 agent session 狀態：RUNNING → 無法恢復（LLM context 已遺失），重新建立 session 重跑；WAITING → 恢復等待；已完成 → 取結果 |
| `idempotent` | 同 replayable，但已執行的 tool calls 透過 idempotency key 避免副作用重複 |
| `non_repeatable` | 若 session 狀態不明確 → step 標記 FAILED |

### 系統預設 Loop 的 Crash Recovery 限制

系統預設 agentic loop 的 crash recovery 受限於對話歷史的保存程度：

- Agent session 的 LLM conversation context 在記憶體中，crash 後遺失
- `persist_history: full` 時，可從 `agent_messages` 重建 conversation 並繼續
- `persist_history: summary` 或 `none` 時，無法恢復中間狀態，必須重新執行
- 已執行的 tool calls（如 task、workflow）已產生副作用，重跑時需注意 idempotency

詳見 [13-agent-definition](../03-agent/13-agent-definition.md) 中的 `config.persist_history`。

### 自訂 Loop 的 Crash Recovery 優勢

自訂 loop 的 recovery 比系統預設 loop 更明確，因為每個操作皆為獨立的 workflow step：

- 每輪迭代的每個 step 都有持久化的狀態和 output
- `session.messages` 透過 `agent.set_messages` 已持久化至資料庫
- Recovery 時可從最後一個成功的 step 繼續（與一般 workflow step recovery 相同）
- 唯一例外：`agent.llm_call` 的回應若未被 `agent.set_messages` 持久化就 crash，需重新呼叫 LLM

與系統預設 loop 相比，自訂 loop 的中間狀態更明確（因為每個 step 都有持久化的 output），recovery 更可靠。

詳見 [16-agent-loop](../03-agent/16-agent-loop.md)。

---

## MCP Server 生命週期

MCP server（stdio transport）的 process 生命週期為 per-session（決策 M4）：

- 每個 agent session 獨立啟動其引用的 MCP server processes
- Agent session 結束時（無論成功、失敗或取消），關閉對應的 MCP server processes
- Crash recovery 時：MCP server process 已隨 engine crash 終止，recovery 時重新啟動

### 生命週期與 Session 的關係

```
Agent session 開始
     ↓
初始化 MCP server connections（啟動 stdio process / 建立 SSE 連線）
     ↓
Agent loop 執行（tool calls 透過已建立的 connections）
     ↓
Agent session 結束
     ↓
關閉 MCP server connections（SIGTERM → SIGKILL stdio process）
```

此設計確保 MCP server 的狀態與 agent session 一對一綁定，避免跨 session 的狀態污染。

詳見 [19-resources-definition](../04-resources/19-resources-definition.md) 中的 MCP server 配置。

---

## Tool Execution 並行性

LLM 可能在一次回傳中包含多個 tool calls。引擎預設並行執行這些 tool calls，但支援 `sequential_only` 標記（決策 M5）：

### 執行規則

| 情境 | 行為 |
|------|------|
| 所有 tool calls 均為非 `sequential_only` | 全部並行執行 |
| 其中包含 `sequential_only` tool | 該 tool 依序執行，其餘非 `sequential_only` tool 可並行 |
| 多個 `sequential_only` tool | 依 LLM 回傳順序依序執行 |

### `sequential_only` 標記方式

在 tool 定義中標記：

```yaml
tools:
  - type: task
    action: payment.charge
    sequential_only: true           # 此 tool 不可並行執行
  - type: mcp
    server: database
    sequential_only: true
```

### 適用場景

- `sequential_only: true` 適用於有副作用且順序敏感的操作（如扣款、資料庫寫入）
- 大多數 read-only 操作無需設定 `sequential_only`，預設並行可提升效能

詳見 [14-agent-tools](../03-agent/14-agent-tools.md)。

---

## Race Condition 處理

### Timeout 與 Event 同時到達

當 `wait` step（event 模式）的 timeout 和匹配事件幾乎同時到達時：

```
Timeline:
  t=0    wait 進入 WAITING
  t=30m  timeout fires_at

  Case A: event 在 t=29m59s 到達，timeout 在 t=30m 觸發
  Case B: timeout 在 t=30m 觸發，event 在 t=30m01s 到達
```

#### 解決規則

引擎 MUST 使用以下規則確保唯一贏家：

1. **Atomic check-and-update**：無論是 event 匹配還是 timeout 觸發，更新 step 狀態前 MUST 檢查 step 是否仍為 WAITING
2. **先到先贏**：第一個成功將 step 狀態從 WAITING 更新為其他狀態的操作為贏家
3. **Optimistic locking**：使用 step_instance 的 version column 防止並行更新

| 贏家 | 輸家行為 |
|------|---------|
| Event 先到 | Timeout 觸發時發現 step 不再是 WAITING → 忽略 |
| Timeout 先到 | Event 匹配時發現 step 不再是 WAITING → 忽略（事件不會被消費，其他 subscription 可能匹配） |

#### 對 Wait Subscription 的清理

贏家操作 MUST 在同一 transaction 中：
- 刪除 wait_subscription
- 刪除 timeout_schedule（若存在）

### Cancellation 與 Step 執行的 Race

當 instance 被取消（API cancel 或 parent timeout）時，可能有 step 正在執行：

| Step 狀態 | Cancellation 行為 |
|-----------|------------------|
| RUNNING（task） | 引擎嘗試終止 task 執行（stdio: SIGTERM → SIGKILL；http: 不等待 response）。Step → CANCELLED |
| RUNNING（agent） | 引擎終止 agent session（關閉 MCP connections、停止 LLM call）。Step → CANCELLED |
| RUNNING（sub_workflow） | Child instance → CANCELLED |
| WAITING | 刪除 wait_subscription，step → CANCELLED |
| READY | Step → CANCELLED（不執行） |
| PENDING | Step → CANCELLED |

### Parallel Branch 失敗與 fail_fast 的 Race

當 `parallel` 的 `failure_policy` 為 `fail_fast` 時，一個 branch 失敗要取消其他 branches：

1. Branch A 失敗
2. 引擎設定 parallel step 為 FAILED
3. 引擎對其他仍在執行的 branches 發送 cancellation
4. 其他 branches 的 step MAY 在收到 cancellation 之前完成 → 完成結果被忽略

引擎 MUST 確保：
- Parallel step 的最終狀態只被設定一次
- 使用 optimistic locking 防止多個 branch 同時嘗試設定 parallel step 狀態

---

## 並行控制

- Step instance 更新 SHOULD 使用 optimistic locking（version column）
- 多 worker 架構下，使用 lease-based ownership 確保同一 instance 不被多個 worker 同時處理
- Idempotency key = `workflow_instance_id + step_id + attempt`

---

## 交叉參照

| 主題 | 文件 |
|------|------|
| Step 類型與共通屬性 | [05-steps-overview](../02-steps/05-steps-overview.md) |
| Task step | [06-step-task](../02-steps/06-step-task.md) |
| 事件 step（wait） | [09-step-events](../02-steps/09-step-events.md) |
| Agent Definition 與 persist_history | [13-agent-definition](../03-agent/13-agent-definition.md) |
| Agent Tools 與 sequential_only | [14-agent-tools](../03-agent/14-agent-tools.md) |
| Agent Loop（自訂 loop） | [16-agent-loop](../03-agent/16-agent-loop.md) |
| Resources（MCP servers） | [19-resources-definition](../04-resources/19-resources-definition.md) |
| 錯誤處理 | [21-error-handling](21-error-handling.md) |
| Lifecycle 狀態 | [23-lifecycle](23-lifecycle.md) |
| 驗證規則 | [27-validation-rules](../06-publish/27-validation-rules.md) |
| 版本管理 | [28-versioning](../06-publish/28-versioning.md) |
