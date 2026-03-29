# Agent 規格草稿（Draft）

> **狀態**：討論中，待修正
> **目標**：定義 `kind: Agent` 與 `type: agent` step，讓 AI agent 成為 workflow 的一等公民

---

## 設計原則

1. **Agent Definition 是角色模板** — 定義「這個 agent 是誰、預設能做什麼」，不包含 workflow 邏輯
2. **Workflow Step 組裝實際能力** — 引用 agent by name，提供 prompt，可增減 tools/skills
3. **所有 tool 都是 agent tool** — task、workflow、agent、MCP、builtin 統一抽象為 tool interface
4. **雙軌制 tool 機制** — task/workflow 自動轉 function call schema；MCP 走 MCP 協議

---

## 一、Agent Definition（`kind: Agent`）

獨立 YAML 定義檔，定義 agent 的預設能力與人設。

```yaml
apiVersion: agent/v2
kind: Agent

metadata:
  name: string              # MUST — 唯一識別名稱（dotted namespace）
  version: integer           # MUST — 版本號，正整數
  description: string        # MAY — 人類可讀說明，也作為 agent-as-tool 時的 tool description
  labels: map                # MAY — 分類鍵值對

model:
  provider: string           # MUST — LLM provider（如 anthropic, openai）
  name: string               # MUST — model 名稱（如 claude-sonnet-4-20250514）
  config: map                # MAY — temperature, max_tokens 等

system_prompt: string        # MAY — 系統提示詞（支援 ${ } CEL 表達式）

input_schema: object         # MAY — agent 接受的輸入 JSON Schema
output_schema: object        # MAY — agent 回傳的輸出 JSON Schema

tools: array                 # MAY — 預設可用 tool 列表（見「Tool 定義」）
skills: array                # MAY — 預設 agentskills.io skill 列表

config:
  mode: agentic | single_shot  # MAY, 預設 agentic
  max_iterations: integer      # MAY, 預設 20 — agentic loop 最大迭代次數
  max_tool_calls: integer      # MAY — 總 tool 呼叫次數上限
  output_format: text | json   # MAY, 預設 json — agent 最終回傳格式
  persist_history: full | summary | none  # MAY, 預設 summary
```

### 設計要點

- **不含 `prompt`**：user prompt 由 workflow step 提供，因為每次呼叫的任務不同
- `system_prompt` 是角色定義（穩定）；`prompt` 是任務指令（隨場景變化）
- `tools` 和 `skills` 是預設值，workflow step 可以合併、覆寫或排除

### 生命週期

與 task/workflow definition 相同：

```
DRAFT → VALIDATED → PUBLISHED → DEPRECATED → ARCHIVED
```

### 範例

```yaml
apiVersion: agent/v2
kind: Agent

metadata:
  name: order.analyzer
  version: 1
  description: "訂單風險分析專家，可評估訂單風險等級"

model:
  provider: anthropic
  name: claude-sonnet-4-20250514
  config:
    temperature: 0
    max_tokens: 4096

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

# 預設能力
tools:
  - type: task
    action: order.load
  - type: task
    action: customer.get_history
  - type: builtin
    name: ask_human

skills:
  - path: ./skills/risk-analysis

config:
  mode: agentic
  max_iterations: 10
  output_format: json
```

---

## 二、Tool 定義

Agent 可使用的工具分為 5 種類型。所有類型在引擎內部統一抽象為 tool interface，對 LLM 來說都是 function call。

### Tool 類型

```yaml
tools:
  # 1. Task — 引用已定義的 task definition
  - type: task
    action: order.load              # MUST — task definition name
    version: integer | "latest"     # MAY, 預設 "latest"
    name: string                    # MAY — 覆寫 tool 名稱（LLM 看到的名稱）
    description: string             # MAY — 覆寫描述

  # 2. Workflow — 引用已定義的 workflow definition
  - type: workflow
    workflow: payment_processing    # MUST — workflow definition name
    version: integer | "latest"     # MAY
    name: string                    # MAY
    description: string             # MAY

  # 3. Agent — 引用已定義的 agent definition（agent-as-tool）
  - type: agent
    agent: research_agent           # MUST — agent definition name
    version: integer | "latest"     # MAY
    name: string                    # MAY
    description: string             # MAY

  # 4. MCP — 從外部 MCP server 取得
  - type: mcp
    server: string                  # MUST — 引擎層級配置的 MCP server 名稱
    tools: array                    # MAY — 限定可用的 tool 名稱列表，空 = 全部
    resources: array                # MAY — 限定可用的 resource 名稱列表

  # 5. Builtin — 引擎內建的 agent 工具
  - type: builtin
    name: string                    # MUST — 內建 tool 名稱
```

### Function Call Schema 轉換規則

| Tool 類型 | Schema 來源 |
|-----------|------------|
| `task` | `task.input_schema` → parameters, `task.description` → description |
| `workflow` | `workflow.input_schema` → parameters, `workflow.description` → description |
| `agent` | `agent.input_schema` → parameters, `agent.description` → description |
| `mcp` | MCP server 回傳的 tool schema（session 初始化時拉取） |
| `builtin` | 引擎預定義的 schema |

Tool 的 `name` 和 `description` 欄位可覆寫預設值（來自 definition 或 MCP server）。這允許在不同 context 下為同一個 tool 提供不同的描述，幫助 LLM 更好地理解何時使用。

### 內建 Agent Tools

| Tool 名稱 | 說明 | Input Schema |
|-----------|------|-------------|
| `ask_human` | 暫停 agent，向人類提問並等待回答 | `{ question: string, context?: string }` |
| `activate_skill` | 載入指定 skill 的完整 SKILL.md 指令到 context（有 skills 時自動注入） | `{ name: string }` |
| `read_skill_file` | 讀取 skill 內的 scripts/references/assets 檔案（有 skills 時自動注入） | `{ skill: string, path: string }` |

#### `ask_human` 行為

類似 workflow 層級的 `wait_event`，但在 agent session 內部運作：

1. Agent 呼叫 `ask_human({ question: "需要確認：此訂單金額異常，是否繼續？" })`
2. Agent session 進入 WAITING 狀態
3. 引擎發出外部通知（webhook / event）
4. 外部系統透過 API 回傳答案：`POST /agents/{session_id}/respond`
5. Agent 收到答案後恢復 RUNNING，繼續 agentic loop

---

## 三、Agent Skills（agentskills.io）

Agent skills 遵循 [agentskills.io](https://agentskills.io/) 規格，是給 agent 提供專業知識、行為指引和可執行腳本的知識包。

### 配置

```yaml
skills:
  - path: ./skills/risk-analysis        # 本地 skill 目錄路徑
  - path: ./skills/compliance-review
```

每個 skill 目錄遵循 agentskills.io 結構：

```
risk-analysis/
  SKILL.md          # 必需：metadata + instructions
  scripts/          # 可選：可執行腳本
  references/       # 可選：參考文件
  assets/           # 可選：模板、資源
```

### 引擎行為

1. **啟動時載入 metadata**：讀取每個 skill 的 `SKILL.md` frontmatter（name, description）
2. **Discovery**：將所有 skills 的 name + description 注入 system prompt，讓 LLM 知道有哪些 skill 可用
3. **按需載入**：LLM 透過 `activate_skill(name)` 載入完整的 SKILL.md body 到 context
4. **Progressive disclosure**：skill 內的檔案透過 `read_skill_file` 按需讀取

### 與 MCP Tools 的差異

| 維度 | Agent Skills | MCP Tools |
|------|-------------|-----------|
| 本質 | 知識 + 指令 + 腳本 | 可程式化呼叫的工具 |
| 載入方式 | 注入 system prompt / context | 轉為 function call schema |
| 執行方式 | Agent 按指令自行決定行動 | Agent 呼叫 → 引擎代為執行 |
| 互補 | Skill 指引 agent 如何使用 tools | Tool 提供 skill 指引的執行能力 |

---

## 四、MCP Server 配置

MCP server 在**引擎層級**配置，agent definition 中透過 `tools[type=mcp].server` 引用。

```yaml
# 引擎層級配置（runtime config）
mcp_servers:
  github-mcp:
    transport: stdio
    command: "npx -y @modelcontextprotocol/server-github"
    env:
      GITHUB_TOKEN: ${ secret.GITHUB_TOKEN }

  slack-mcp:
    transport: sse
    url: "https://mcp.slack.example.com/sse"
    headers:
      Authorization: "Bearer ${ secret.SLACK_TOKEN }"

  compliance-mcp:
    transport: stdio
    command: "python3 ./mcp-servers/compliance/main.py"
    config:
      database: compliance_db
```

### MCP Server 欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `transport` | string | MUST | `stdio` 或 `sse` |
| `command` | string | MUST（stdio） | 啟動 MCP server 的命令 |
| `url` | string | MUST（sse） | SSE endpoint URL |
| `env` | map | MAY | 環境變數（支援 ${ } CEL） |
| `headers` | map | MAY | HTTP headers（僅 sse） |
| `config` | map | MAY | 傳給 MCP server 的靜態設定 |

---

## 五、Step: agent（Workflow 中組裝實際能力）

Workflow step 引用 agent definition 並**組裝實際要用的能力**。Step 層級的設定會與 agent definition 的預設值合併或覆寫。

### Schema

```yaml
- id: string                          # MAY — 需要參照 output 時才必填
  type: agent                         # MUST
  agent: string                       # MUST — agent definition name
  version: integer | "latest"         # MAY, 預設 "latest"
  input: map | CEL expression         # MAY — 傳給 agent 的輸入
  prompt: string                      # MUST — 任務指令（支援 ${ } CEL）

  # 能力調整
  tools: array                        # MAY — 額外 tools（合併至 definition 預設）
  tools_override: array               # MAY — 完全覆寫 tools（忽略 definition 預設）
  tools_exclude: array                # MAY — 從最終 tools 中排除
  skills: array                       # MAY — 額外 skills（合併至 definition 預設）
  system_prompt_append: string        # MAY — 追加到 system prompt 尾部

  # 執行控制覆寫
  config:
    mode: agentic | single_shot       # MAY
    max_iterations: integer           # MAY
    output_format: text | json        # MAY
    persist_history: full | summary | none  # MAY

  # 標準 step 屬性
  timeout: duration                   # MAY
  execution:
    policy: replayable | idempotent | non_repeatable  # MAY, 預設 replayable
  retry:
    max_attempts: integer             # MAY, 預設 1
    delay: duration                   # MAY, 預設 1s
    backoff: fixed | exponential      # MAY, 預設 fixed
  on_error: [...]                     # MAY, step 陣列
  on_timeout: [...]                   # MAY, step 陣列
```

### 能力合併規則

| Step 欄位 | 行為 |
|-----------|------|
| `tools` | 與 agent definition 的 `tools` **合併**（union） |
| `tools_override` | **完全取代** agent definition 的 `tools`（與 `tools` 互斥） |
| `tools_exclude` | 從合併後的 tools 中**移除**指定項目（by builtin name 或 task action） |
| `skills` | 與 agent definition 的 `skills` **合併** |
| `system_prompt_append` | 追加到 agent definition 的 `system_prompt` 尾部 |
| `config.*` | Step 層級的值**覆寫** agent definition 的預設 |

### `tools` 與 `tools_override` 互斥

- 設定 `tools`：合併至 definition 預設（增量）
- 設定 `tools_override`：完全取代 definition 預設
- 兩者同時設定 → validation error

### `tools_exclude` 語法

```yaml
tools_exclude:
  - ask_human                    # 排除 builtin tool（by name）
  - order.load                   # 排除 task tool（by action）
  - payment_processing           # 排除 workflow tool（by workflow name）
  - fraud.detector               # 排除 agent tool（by agent name）
```

### 執行語意

1. Step 進入 RUNNING
2. 引擎組裝 agent session：
   - 載入 agent definition 預設值
   - 合併 step 層級的 tools / skills / config
   - 求值 `prompt`（CEL 表達式 + input namespace）
   - 組合 system_prompt = definition.system_prompt + step.system_prompt_append
   - 初始化 tool connections（建立 MCP server 連線、解析 task/workflow/agent schema）
   - 載入 skills metadata 注入 system prompt
3. 依 `config.mode` 執行：

#### Agentic Loop（預設）

```
                    ┌──────────────────────┐
                    │   送 prompt 給 LLM    │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
              ┌─ NO │  LLM 回傳 tool call?  │
              │     └──────────┬───────────┘
              │                │ YES
              │     ┌──────────▼───────────┐
              │     │   引擎執行 tool(s)    │
              │     └──────────┬───────────┘
              │                │
              │     ┌──────────▼───────────┐
              │     │  結果送回 LLM         │──→（回到頂部）
              │     └──────────────────────┘
              │
              │     ┌──────────────────────┐
              └──→  │  Final response      │
                    │  → step output       │
                    └──────────────────────┘
```

- 迭代上限：`config.max_iterations`（預設 20）
- Tool 呼叫上限：`config.max_tool_calls`（可選）
- 超過上限 → step FAILED（error code: `agent_max_iterations`）

#### Single-Shot

1. 送 prompt + tool schemas 給 LLM
2. LLM 回傳 tool calls → 引擎執行所有 tool calls → 結果送回 LLM 取得 final answer
3. 或 LLM 直接回傳 answer（無 tool call）
4. 最多一輪 tool call

### Output 模型

Agent 的最終回應依 `output_format` 處理後作為 step output：

| `output_format` | 處理方式 |
|-----------------|---------|
| `json`（預設） | 從 LLM 回應中解析 JSON → `steps.<id>.output` 為 map |
| `text` | LLM 回應直接作為字串 → `steps.<id>.output.text` |

若有 `output_schema`（來自 agent definition），引擎在解析後驗證 output。驗證失敗 → step FAILED（error code: `agent_output_invalid`）。

```yaml
- id: analyze
  type: agent
  agent: order.analyzer
  prompt: "分析訂單 ${ input.order_id } 的風險等級"
  input:
    order_id: ${ input.order_id }

# output_format: json → steps.analyze.output 為 map
- type: if
  expr: ${ steps.analyze.output.risk_level == "high" }
  then:
    - type: task
      action: order.flag_for_review
      input:
        order_id: ${ input.order_id }
        reason: ${ steps.analyze.output.summary }
```

---

## 六、Agent-as-Tool

Agent A 的 tools 列表中引用 Agent B，Agent B 成為 Agent A 的 tool。

### 執行語意

1. Agent A 的 LLM 決定呼叫 Agent B
2. 引擎為 Agent B 建立獨立 session
3. Agent A 的 tool call input → Agent B 的 input
4. Agent A 的 tool call input 中 MAY 包含一個 `prompt` 欄位，作為 Agent B 的 prompt
5. Agent B 執行完成 → output 回傳給 Agent A 作為 tool result
6. **深度限制**：建議上限 5 層（同 sub_workflow 的 `max_depth_exceeded`）

### Agent-as-Tool 的 prompt

當 Agent B 作為 tool 被呼叫時，prompt 來源：

| 情境 | Prompt 來源 |
|------|------------|
| Tool call input 包含 `prompt` 欄位 | 使用該 prompt |
| Tool call input 不含 `prompt` | 引擎從 input 自動組合描述性 prompt |

### 資料隔離

與 sub_workflow 相同原則 — Agent B 無法存取 Agent A 的 context（conversation history、tool results、variables 等）。

### 範例

```yaml
# Agent A (order.analyzer) 的 tools 中包含 Agent B
tools:
  - type: agent
    agent: fraud.detector
    description: "深度詐騙偵測，用於高風險案例的進階分析"

# Agent A 在 agentic loop 中可能產生的 tool call：
# {
#   "name": "fraud.detector",
#   "input": {
#     "prompt": "請對訂單 ORD-001 進行詐騙偵測分析",
#     "order_id": "ORD-001",
#     "transaction_amount": 50000,
#     "customer_id": "CUST-789"
#   }
# }
```

---

## 七、對話持久化

Agent session 的 conversation history 根據 `config.persist_history` 決定保留策略：

| 模式 | 保留內容 | 適用場景 |
|------|---------|---------|
| `full` | 完整 LLM 來回、所有 tool calls + results | audit、debug、合規要求 |
| `summary`（預設） | final output + tool call 摘要（名稱、成功/失敗、耗時） | 一般使用，平衡 debug 與 storage |
| `none` | 僅 final output | 低成本、隱私敏感場景 |

### Storage Schema

```sql
-- agent session
CREATE TABLE agent_sessions (
  id TEXT PRIMARY KEY,
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

-- conversation history（persist_history = full）
CREATE TABLE agent_messages (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL REFERENCES agent_sessions(id),
  sequence INTEGER NOT NULL,
  role TEXT NOT NULL,                 -- system | user | assistant | tool
  content JSONB NOT NULL,
  tool_call_id TEXT,
  created_at TIMESTAMPTZ NOT NULL
);

-- tool call 摘要（persist_history = summary 或 full）
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

## 八、錯誤碼

新增 agent 相關的標準錯誤碼：

| 錯誤碼 | 觸發條件 |
|--------|---------|
| `agent_max_iterations` | 達到 `max_iterations` 上限但未完成 |
| `agent_max_tool_calls` | 達到 `max_tool_calls` 上限 |
| `agent_output_invalid` | Agent 回傳的 output 不符合 `output_schema` |
| `agent_llm_error` | LLM API 呼叫失敗（網路錯誤、rate limit、provider 錯誤） |
| `agent_tool_failed` | Agent 呼叫的 tool 執行失敗（傳遞給 LLM 作為 tool result，不直接終止 agent） |
| `agent_not_found` | 指定的 agent definition 不存在或非 PUBLISHED |

注意：`agent_tool_failed` 不會直接導致 agent step 失敗。Tool 失敗的結果會送回 LLM，由 LLM 決定下一步（重試、使用其他 tool、或放棄）。只有當 LLM 明確表示無法完成任務，或達到迭代上限時，agent step 才會 FAILED。

---

## 九、Execution Policy 與 Crash Recovery

Agent step 的 `execution.policy` 決定 crash recovery 行為：

| Policy | Recovery 行為 |
|--------|---------------|
| `replayable`（預設） | 查詢 agent session 狀態：RUNNING → 無法恢復（LLM context 已遺失），重新建立 session 重跑；WAITING → 恢復等待；已完成 → 取結果 |
| `idempotent` | 同 replayable，但避免副作用 tool 重複執行 |
| `non_repeatable` | 若 session 狀態不明確 → step 標記 FAILED |

### Agentic Loop 的 Crash Recovery 限制

與 task step 不同，agent step 的 crash recovery 較困難：

- Agent session 的 LLM conversation context 在記憶體中，crash 後遺失
- `persist_history: full` 時，可從 `agent_messages` 重建 conversation 並繼續
- `persist_history: summary` 或 `none` 時，無法恢復中間狀態，必須重新執行
- 已執行的 tool calls（如 task、workflow）已產生副作用，重跑時需注意 idempotency

---

## 討論議題

> 以下列出待討論和修正的設計問題，歡迎直接在此標記意見。

### 議題 1：Agent-as-Tool 的 prompt 來源

目前設計：tool call input 可包含 `prompt` 欄位，或由引擎自動組合。

**疑問**：Agent B 作為 tool 時，description 已定義「何時使用」，但具體指令（prompt）的來源需要更明確的規範。是否應該讓 Agent A 總是能透過 tool call 傳入 prompt？

### 議題 2：Agentic Loop 的 Crash Recovery

`persist_history: full` 允許從 agent_messages 重建 conversation，但重建後繼續的行為需要更精確的定義（例如：重建 context 時是否包含 system prompt 的 skill 載入狀態？）。

### 議題 3：MCP Server 生命週期

MCP server（stdio transport）的 process 生命週期應該如何管理？

- 選項 A：每個 agent session 獨立啟動/關閉
- 選項 B：引擎層級的 connection pool，多個 session 共用
- 選項 C：可配置（per-session vs shared）

### 議題 4：Tool Execution 的並行性

LLM 可能在一次回傳中包含多個 tool calls。引擎是否應該並行執行這些 tool calls？

- 選項 A：總是循序執行（簡單、可預測）
- 選項 B：預設並行，除非 tool 標記為 `sequential_only`
- 選項 C：可配置

### 議題 5：Skills 的版本管理

目前 skills 以本地路徑引用，沒有版本概念。是否需要支援遠端 skill（如 git repo URL）或版本鎖定？

### 議題 6：Agent Step 在 Expression 中的 Namespace

Agent step 執行過程中，`prompt` 可存取哪些 namespace？

目前假設與 task step 相同：`input`、`steps`、`vars`、`env`、`secret`、`artifacts`。

---

## 完整範例

### Agent Definition — 角色模板

```yaml
apiVersion: agent/v2
kind: Agent

metadata:
  name: order.analyzer
  version: 1
  description: "訂單風險分析專家，可評估訂單風險等級"

model:
  provider: anthropic
  name: claude-sonnet-4-20250514
  config:
    temperature: 0
    max_tokens: 4096

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
  - type: task
    action: order.load
  - type: task
    action: customer.get_history
  - type: builtin
    name: ask_human

skills:
  - path: ./skills/risk-analysis

config:
  mode: agentic
  max_iterations: 10
  output_format: json
```

### Workflow A — 自動化批次分析

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: order_batch_review
  version: 1

steps:
  - id: analyze
    type: agent
    agent: order.analyzer
    prompt: "分析訂單 ${ input.order_id } 的風險等級，直接給出判斷"
    input:
      order_id: ${ input.order_id }
    tools_exclude: [ask_human]
    config:
      persist_history: none
    timeout: 1m

  - type: switch
    expr: ${ steps.analyze.output.risk_level }
    cases:
      - value: high
        steps:
          - type: task
            action: order.flag_for_review
            input:
              order_id: ${ input.order_id }
              reason: ${ steps.analyze.output.summary }
    default:
      - type: return
        output:
          status: approved
```

### Workflow B — 人工審核流程

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: order_manual_review
  version: 1

steps:
  - id: deep_analyze
    type: agent
    agent: order.analyzer
    prompt: |
      深度審核訂單 ${ input.order_id }。
      請使用制裁名單檢查和詐騙偵測工具。
      高風險案例必須透過 ask_human 確認。
    input:
      order_id: ${ input.order_id }
    tools:
      - type: mcp
        server: compliance-mcp
        tools: [check_sanctions_list]
      - type: agent
        agent: fraud.detector
    skills:
      - path: ./skills/compliance-review
    system_prompt_append: |
      審核模式：這是人工審核流程，你必須謹慎行事。
    config:
      persist_history: full
    timeout: 10m

  - type: return
    output:
      result: ${ steps.deep_analyze.output }
```
