# 13 — Agent Definition（`kind: Agent`）

本文件定義 `kind: Agent` 的完整 YAML 結構、設計原則、對話持久化、錯誤碼、執行策略與 crash recovery。Agent Definition 是 AI agent 的角色模板，由 workflow step 組裝實際能力後執行。

---

## 1. 設計原則

1. **Agent Definition 是角色模板** — 定義「這個 agent 是誰、預設能做什麼」，不包含 workflow 邏輯。
2. **Workflow Step 組裝實際能力** — Workflow 中的 `type: agent` step 引用 agent by name，提供 prompt，可增減 tools / skills。
3. **所有 tool 統一抽象** — task、workflow、agent、MCP、builtin 在引擎內部統一為 tool interface，對 LLM 來說都是 function call。
4. **雙軌制 tool 機制** — task / workflow 自動轉換為 function call schema（引擎代為執行）；MCP 走 MCP 協議（由 MCP server 處理）。兩者對 LLM 透明，皆為 function call。

---

## 2. Top-level Schema

```yaml
apiVersion: agent/v3
kind: Agent

metadata:
  name: string              # MUST — 唯一識別名稱（dotted namespace，如 order.analyzer）
  version: integer           # MUST — 版本號，正整數，單調遞增
  description: string        # MAY — 人類可讀說明；亦作為 agent-as-tool 時的 tool description
  labels: map<string, string> # MAY — 分類鍵值對

# LLM 模型設定（二選一寫法）
model: string | object
  # 寫法一：model alias（如 "fast"、"smart"、"reasoning"）
  #   model: fast
  # 寫法二：完整物件
  #   model:
  #     provider: string       # MUST — LLM provider（如 anthropic, openai）
  #     name: string           # MUST — model 名稱（如 claude-sonnet-4-20250514）
  #     config: map            # MAY — temperature, max_tokens 等

system_prompt: string        # MAY — 系統提示詞，角色定義（支援 ${ } CEL 表達式）

input_schema: object         # MAY — agent 接受的輸入 JSON Schema
output_schema: object        # MAY — agent 回傳的輸出 JSON Schema

tools: array                 # MAY — 預設可用 tool 列表（詳見 14-agent-tools）
skills: array                # MAY — 透過 toolset 引用（詳見 15-agent-skills、18-toolset-definition）

config:
  max_iterations: integer      # MAY, 預設 20 — loop 最大迭代次數
  max_tool_calls: integer      # MAY — 總 tool 呼叫次數上限
  output_format: text | json   # MAY, 預設 json — agent 最終回傳格式
  persist_history: full | summary | none  # MAY, 預設 summary — 對話紀錄保留策略
  stream: token | iteration | none       # MAY, 預設 none — 串流模式

hooks: object                # MAY — 生命週期 hooks（詳見 17-agent-hooks-and-streaming）

loop:                        # MAY — 自訂 loop steps（詳見 16-agent-loop）
  steps: [...]               #   省略時使用系統預設 agentic loop
```

### model 欄位說明

`model` 支援兩種寫法：

| 寫法 | 範例 | 說明 |
|------|------|------|
| string（alias） | `model: fast` | 引用引擎配置或 `kind: Resources` 中定義的 model alias（決策 J1） |
| object（完整） | `model: { provider: anthropic, name: claude-sonnet-4-20250514, config: { ... } }` | 直接指定 provider、name 與 config |

使用 alias 時，Agent Definition MAY 在 `model.config` 中部分覆寫 alias 預設值（決策 J2）：

```yaml
model:
  alias: fast             # 引用 alias
  config:
    max_tokens: 8192      # 僅覆寫 max_tokens，其餘沿用 alias 預設
```

### skills 欄位說明

根據決策 G4，skills MUST 透過 toolset 引用，不可獨立宣告。`skills` 陣列中的每個項目實際上是 toolset 引用：

```yaml
skills:
  - type: toolset
    toolset: risk-analysis-tools    # 引用 kind: Toolset 定義
```

詳見 [15-agent-skills](15-agent-skills.md) 與 [18-toolset-definition](../04-resources/18-toolset-definition.md)。

---

## 3. 設計要點

- **不含 `prompt` 欄位** — user prompt 由 workflow 中的 `type: agent` step 提供，因為每次呼叫的任務不同。
- **`system_prompt` = 角色（穩定）；`prompt` = 任務指令（隨場景變化）** — system_prompt 定義在 Agent Definition 中，prompt 在每次 step 呼叫時由 workflow 提供。
- **`tools` 和 `skills` 是預設值** — workflow step 可以合併（`tools`）、覆寫（`tools_override`）或排除（`tools_exclude`）Agent Definition 中的預設 tools。
- **`loop` 是可選的** — 大多數 agent 使用系統預設 agentic loop 即可，僅需精細控制對話流程時才自訂。詳見 [16-agent-loop](16-agent-loop.md)。

---

## 4. 生命週期

與 Workflow / Task Definition 相同（詳見 [01-kind-definitions](../01-core/01-kind-definitions.md)）：

```
DRAFT → VALIDATED → PUBLISHED → DEPRECATED → ARCHIVED
```

| 狀態 | 說明 |
|------|------|
| `DRAFT` | 草稿，不可被引用 |
| `VALIDATED` | 通過結構驗證 |
| `PUBLISHED` | 正式發布，可被 workflow step 引用 |
| `DEPRECATED` | 已棄用，既有引用仍可執行，不可新增引用 |
| `ARCHIVED` | 已封存，不可執行 |

---

## 5. 對話歷史持久化

Agent session 的 conversation history 根據 `config.persist_history` 決定保留策略。

### 三種模式

| 模式 | 保留內容 | 適用場景 |
|------|---------|---------|
| `full` | 完整 LLM 來回、所有 tool calls + results | audit、debug、合規要求 |
| `summary`（預設） | final output + tool call 摘要（名稱、成功/失敗、耗時） | 一般使用，平衡可除錯性與 storage 成本 |
| `none` | 僅 final output | 低成本、隱私敏感場景 |

### Storage Schema

```sql
-- agent session 主表
CREATE TABLE agent_sessions (
  id TEXT PRIMARY KEY,                -- asess_<uuid>（UUID v7）
  instance_id TEXT NOT NULL,          -- 所屬 workflow instance
  step_id TEXT NOT NULL,              -- 所屬 step
  parent_session_id TEXT,             -- 若為 agent-as-tool，指向 parent agent session
  agent_name TEXT NOT NULL,
  agent_version INTEGER NOT NULL,
  status TEXT NOT NULL,               -- RUNNING | WAITING | SUCCEEDED | FAILED
  persist_history TEXT NOT NULL,      -- full | summary | none
  iteration_count INTEGER DEFAULT 0,
  tool_call_count INTEGER DEFAULT 0,
  final_output JSONB,
  created_at TIMESTAMPTZ NOT NULL,
  completed_at TIMESTAMPTZ
);

-- 完整對話紀錄（persist_history = full 時寫入）
CREATE TABLE agent_messages (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL REFERENCES agent_sessions(id),
  sequence INTEGER NOT NULL,
  role TEXT NOT NULL,                 -- system | user | assistant | tool
  content JSONB NOT NULL,
  tool_call_id TEXT,
  created_at TIMESTAMPTZ NOT NULL
);

-- tool call 摘要（persist_history = summary 或 full 時寫入）
CREATE TABLE agent_tool_calls (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL REFERENCES agent_sessions(id),
  sequence INTEGER NOT NULL,
  tool_name TEXT NOT NULL,
  tool_type TEXT NOT NULL,            -- task | workflow | agent | mcp | builtin
  input_summary JSONB,
  output_summary JSONB,
  success BOOLEAN,
  error_code TEXT,
  error_message TEXT,
  duration_ms INTEGER,
  created_at TIMESTAMPTZ NOT NULL
);
```

---

## 6. 錯誤碼

Agent 相關的標準錯誤碼：

| 錯誤碼 | 觸發條件 | 是否終止 agent |
|--------|---------|---------------|
| `agent_max_iterations` | 達到 `config.max_iterations` 上限但未完成 | 是 — step FAILED |
| `agent_max_tool_calls` | 達到 `config.max_tool_calls` 上限 | 是 — step FAILED |
| `agent_output_invalid` | Agent 回傳的 output 不符合 `output_schema` | 是 — step FAILED |
| `agent_llm_error` | LLM API 呼叫失敗（網路錯誤、rate limit、provider 錯誤） | 是 — step FAILED |
| `agent_tool_failed` | Agent 呼叫的 tool 執行失敗 | **否** — 結果送回 LLM |
| `agent_not_found` | 指定的 agent definition 不存在或非 PUBLISHED | 是 — step FAILED |

> **重要**：`agent_tool_failed` 不會直接導致 agent step 失敗。Tool 失敗的結果會作為 tool result 送回 LLM，由 LLM 決定下一步（重試、使用其他 tool、或放棄）。只有當 LLM 明確表示無法完成任務，或達到迭代 / tool call 上限時，agent step 才會 FAILED。

---

## 7. Execution Policy 與 Crash Recovery

Agent step 的 `execution.policy` 決定 crash recovery 行為。

### Policy 對照表

| Policy | Recovery 行為 |
|--------|---------------|
| `replayable`（預設） | 查詢 agent session 狀態：RUNNING → 無法恢復（LLM context 已遺失），重新建立 session 重跑；WAITING → 恢復等待；已完成 → 取結果 |
| `idempotent` | 同 replayable，但避免副作用 tool 重複執行 |
| `non_repeatable` | 若 session 狀態不明確 → step 標記 FAILED |

### 系統預設 Loop 的限制

系統預設 agentic loop 的 crash recovery 較為困難：

- Agent session 的 LLM conversation context 在記憶體中，crash 後遺失。
- `persist_history: full` 時，可從 `agent_messages` 重建 conversation 並繼續。
- `persist_history: summary` 或 `none` 時，無法恢復中間狀態，必須重新執行。
- 已執行的 tool calls（如 task、workflow）已產生副作用，重跑時需注意 idempotency。

### 自訂 Loop 的優勢

自訂 loop 的 crash recovery 比系統預設 loop 更明確（詳見 [16-agent-loop](16-agent-loop.md)）：

- 每輪迭代的每個 step 都有持久化的狀態和 output。
- `session.messages` 透過 `agent.set_messages` 已持久化至資料庫。
- Recovery 時可從最後一個成功的 step 繼續（與一般 workflow step recovery 相同）。
- 唯一例外：`agent.llm_call` 的回應若未被 `agent.set_messages` 持久化就 crash，需重新呼叫 LLM。

---

## 8. Agent-as-Tool

Agent B 可出現在 Agent A 的 `tools` 列表中，成為 Agent A 的 tool。

### 執行語意

1. Agent A 的 LLM 決定呼叫 Agent B。
2. 引擎為 Agent B 建立**獨立 session**（independent session）。
3. Agent A 的 tool call input → Agent B 的 input。
4. Agent A MAY 在 tool call input 中包含 `prompt` 欄位，作為 Agent B 的 prompt（決策 M1）。若未提供 `prompt`，引擎從 input 自動組合描述性 prompt。
5. Agent B 執行完成 → output 回傳給 Agent A 作為 tool result。
6. **深度限制**：上限 5 層（與 sub_workflow 的 `max_depth_exceeded` 相同機制）。

### 資料隔離

與 sub_workflow 相同原則 — Agent B 無法存取 Agent A 的 context（conversation history、tool results、variables 等）。Agent B 的 session 透過 `parent_session_id` 關聯 Agent A 的 session，但僅作為觀測與溯源用途。

> 完整的 tool 類型定義與 function call schema 轉換規則，詳見 [14-agent-tools](14-agent-tools.md)。

---

## 9. Session ID

每個 agent 執行實例擁有唯一的 session ID，作為整個 agent session 生命週期的識別符。

### 格式

- **前綴**：`asess_`（agent session 前綴，便於辨識）
- **ID 本體**：UUID v7（含時間戳排序）
- **完整格式**：`asess_<uuid>`，例如 `asess_019513a2-7b4c-7f00-8000-abcdef123456`

### 產生時機

引擎在 agent step 進入 RUNNING 時自動產生。若 agent step 因 retry 重新執行，會產生新的 session ID。

### Session 與 Workflow Instance 的關係

```
workflow instance (instance_id)
  └── step: agent (step_id)
        └── agent session (asess_<uuid>)
              └── agent-as-tool 子 agent (child asess_<uuid>, parent_session_id)
```

一個 workflow instance 中可能有多個 agent step，每個 agent step 產生一個 session。

### 在表達式中存取

`session.id` 屬於 `session` namespace，與 `session.messages`、`session.iteration` 並列：

```yaml
prompt: "Session ${ session.id } — 請分析訂單 ${ input.order_id }"
```

### 用途

| 用途 | 說明 |
|------|------|
| 對話持久化 | `agent_sessions.id`、`agent_messages.session_id` 的主鍵 |
| API 操作 | `POST /agents/{session_id}/respond`（ask_human 回覆） |
| Agent-as-Tool | 子 agent session 透過 `parent_session_id` 關聯父 agent |
| 日誌與觀測 | 所有 agent 相關的 log、trace、metric 以 session ID 為維度 |
| Crash Recovery | 透過 session ID 查詢 session 狀態，決定恢復策略 |

---

## 10. 完整範例

```yaml
apiVersion: agent/v3
kind: Agent

metadata:
  name: order.analyzer
  version: 1
  description: "訂單風險分析專家，可評估訂單風險等級並產生結構化報告"
  labels:
    domain: order
    capability: risk-analysis

model: smart                    # 引用 model alias（決策 J1）

system_prompt: |
  你是訂單風險分析專家。根據訂單資料與客戶歷史，
  評估風險等級並產生結構化分析報告。
  輸出格式必須為 JSON，包含 risk_level、summary、flags。

input_schema:
  type: object
  properties:
    order_id:
      type: string
    include_history:
      type: boolean
      default: true
  required: [order_id]

output_schema:
  type: object
  properties:
    risk_level:
      type: string
      enum: [low, medium, high]
    summary:
      type: string
    flags:
      type: array
      items:
        type: string
  required: [risk_level, summary]

tools:
  # Task tools — 自動轉為 function call schema
  - type: task
    action: order.load
  - type: task
    action: customer.get_history

  # Agent-as-Tool — 高風險案例交由專屬 agent 深度分析
  - type: agent
    agent: fraud.detector
    description: "深度詐騙偵測，用於高風險案例的進階分析"

  # Builtin tool — human-in-the-loop
  - type: tool
    name: agent.ask_human

skills:
  # 透過 toolset 引用（決策 G4）
  - type: toolset
    toolset: risk-analysis-tools

config:
  max_iterations: 10
  max_tool_calls: 50
  output_format: json
  persist_history: summary
  stream: none
```

### 在 Workflow 中使用此 Agent

```yaml
# workflow step 引用 agent 並提供 prompt
- id: analyze
  type: agent
  agent: order.analyzer
  prompt: "分析訂單 ${ input.order_id } 的風險等級，特別注意金額異常與地區風險"
  input:
    order_id: ${ input.order_id }
    include_history: true

# 根據 agent output 決定後續流程
- type: if
  when: ${ steps.analyze.output.risk_level == "high" }
  then:
    - type: task
      action: order.flag_for_review
      input:
        order_id: ${ input.order_id }
        reason: ${ steps.analyze.output.summary }
```
