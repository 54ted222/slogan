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
  max_iterations: integer      # MAY, 預設 20 — loop 最大迭代次數
  max_tool_calls: integer      # MAY — 總 tool 呼叫次數上限
  output_format: text | json   # MAY, 預設 json — agent 最終回傳格式
  persist_history: full | summary | none  # MAY, 預設 summary

# 自訂 loop（MAY — 省略時使用系統預設 agentic loop）
loop:
  steps: [...]                 # workflow step 陣列，定義每一輪迭代的邏輯
```

### 設計要點

- **不含 `prompt`**：user prompt 由 workflow step 提供，因為每次呼叫的任務不同
- `system_prompt` 是角色定義（穩定）；`prompt` 是任務指令（隨場景變化）
- `tools` 和 `skills` 是預設值，workflow step 可以合併、覆寫或排除
- **Loop 是可選的**：大多數 agent 使用系統預設 loop 即可；需要精細控制對話流程時才自訂

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
  - type: tool
    name: agent.ask_human

skills:
  - name: risk-analysis

config:
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

  # 5. Tool — 引擎內建的 agent 工具（兩段式命名：namespace.action）
  - type: tool
    name: string                    # MUST — 工具名稱（如 agent.ask_human）
```

### Function Call Schema 轉換規則

| Tool 類型 | Schema 來源 |
|-----------|------------|
| `task` | `task.input_schema` → parameters, `task.description` → description |
| `workflow` | `workflow.input_schema` → parameters, `workflow.description` → description |
| `agent` | `agent.input_schema` → parameters, `agent.description` → description |
| `mcp` | MCP server 回傳的 tool schema（session 初始化時拉取） |
| `tool` | 引擎預定義的 schema |

Tool 的 `name` 和 `description` 欄位可覆寫預設值（來自 definition 或 MCP server）。這允許在不同 context 下為同一個 tool 提供不同的描述，幫助 LLM 更好地理解何時使用。

### 內建 Agent Tools

| Tool 名稱 | 說明 | Input Schema |
|-----------|------|-------------|
| `agent.ask_human` | 暫停 agent，向人類提問並等待回答 | `{ questions: array, context?: string }` |
| `agent.activate_skill` | 載入指定 skill 的完整 SKILL.md 指令到 context（有 skills 時自動注入） | `{ name: string }` |
| `agent.read_skill_file` | 讀取 skill 內的 scripts/references/assets 檔案（有 skills 時自動注入） | `{ skill: string, path: string }` |

> **命名慣例**：所有內建 tool 採用兩段式命名（`namespace.action`），以點號分隔。詳見第十節 10.4。

#### `agent.ask_human` 行為

類似 workflow 層級的 `wait_event`，但在 agent session 內部運作：

1. Agent 呼叫 `agent.ask_human({ questions: [{ id: "confirm", question: "需要確認：此訂單金額異常，是否繼續？", type: "confirm" }] })`
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
  - name: risk-analysis        # 本地 skill 目錄路徑
  - name: compliance-review
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
  - agent.ask_human              # 排除內建 tool（by name）
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
   - 初始化 `session` namespace（見下方）
3. 依 agent definition 是否有 `loop` 決定執行模式：

#### 模式一：系統預設 Agentic Loop（無 `loop`）

當 agent definition **未定義** `loop` 時，使用引擎內建的標準 agentic loop：

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

這是大多數 agent 的推薦模式——簡單、可預測、無需額外配置。

#### 模式二：自訂 Loop（有 `loop`）

當 agent definition **定義了** `loop` 時，引擎不使用內建 loop，改為**重複執行 `loop.steps` 中的 step 序列**，直到遇到 `return` 或 `fail` step 為止。

```
     ┌──────────────────────────────────────┐
     │        loop.steps 開始執行            │ ◄──┐
     │  （使用者定義的 workflow step 序列）    │    │
     └──────────────┬───────────────────────┘    │
                    │                             │
          ┌─────── ▼ ────────┐                   │
          │  遇到 return?    │── YES → 結束，output = return value
          └────────┬─────────┘                   │
                   │ NO                           │
          ┌─────── ▼ ────────┐                   │
          │  遇到 fail?      │── YES → step FAILED
          └────────┬─────────┘                   │
                   │ NO                           │
          ┌─────── ▼ ────────┐                   │
          │ iteration < max? │── YES ─────────────┘
          └────────┬─────────┘
                   │ NO
            agent_max_iterations → FAILED
```

自訂 loop 讓使用者完全掌控每一輪迭代的邏輯：呼叫 LLM、處理 tool calls、編輯對話紀錄、發事件、決定何時退出。

##### `session` Namespace

自訂 loop 的 steps 除了標準 namespace（`input`、`steps`、`vars`）外，額外可存取 `session` namespace：

| 變數 | 型別 | 說明 |
|------|------|------|
| `session.messages` | array | 當前對話紀錄（message 陣列） |
| `session.iteration` | integer | 當前迭代次數（從 0 開始） |
| `session.tool_call_count` | integer | 累計 tool 呼叫次數 |

`session.messages` 中的每個 message 結構：

```yaml
# message 結構
- role: system | user | assistant | tool
  content: string | array       # 文字或 content blocks
  tool_calls: array             # assistant message 的 tool calls（MAY）
  tool_call_id: string          # tool message 的對應 ID（MAY）
```

初始化時，引擎自動注入：
1. `system` message（system_prompt + skills metadata）
2. `user` message（prompt）

##### Agent 內建 Task

自訂 loop 透過以下內建 task 與 LLM 和 tool 系統互動：

| Task Action | 說明 | Input | Output |
|-------------|------|-------|--------|
| `agent.llm_call` | 將 messages 送給 LLM，取得回應 | `{ messages: array }` | `{ message: object, has_tool_calls: bool, tool_calls: array }` |
| `agent.exec_tool_calls` | 執行一組 tool calls | `{ tool_calls: array }` | `{ results: array }` — 每個 result 為 `{ tool_call_id, content }` |
| `agent.set_messages` | 取代整個對話紀錄 | `{ messages: array }` | `{}` |

- `agent.llm_call`：使用 agent definition 的 `model` 設定呼叫 LLM。Input 的 `messages` 會包含完整對話紀錄。Output 的 `message` 為 LLM 回傳的 assistant message。
- `agent.exec_tool_calls`：引擎根據 tool 定義（task / workflow / agent / mcp / builtin）分派執行。Output 的 `results` 為 tool role 的 messages。
- `agent.set_messages`：直接取代 `session.messages`，用於裁剪對話、注入摘要、移除舊訊息等。

##### 自訂 Loop 中可用的 Step 類型

自訂 loop 的 steps 支援所有標準 workflow step 類型：

- `task`（含 `agent.llm_call` 等內建 task）
- `assign`（修改 `vars`）
- `if` / `switch`（條件判斷）
- `foreach` / `parallel`（迴圈與平行）
- `emit`（發送事件）
- `wait_event`（等待外部事件，loop 暫停）
- `return`（結束 loop，產生 output）
- `fail`（中止 loop 並報錯）

**不支援** `sub_workflow` 和 `agent`（巢狀 agent loop 應透過 agent-as-tool 機制）。

##### 自訂 Loop 範例：標準 loop 的等效實作

```yaml
loop:
  steps:
    # 1. 呼叫 LLM
    - id: response
      type: task
      action: agent.llm_call
      input:
        messages: ${ session.messages }

    # 2. 將 assistant message 加入對話紀錄
    - type: assign
      vars:
        updated_messages: ${ session.messages.append(steps.response.output.message) }
    - type: task
      action: agent.set_messages
      input:
        messages: ${ vars.updated_messages }

    # 3. 無 tool call → 結束
    - type: if
      expr: ${ !steps.response.output.has_tool_calls }
      then:
        - type: return
          output: ${ steps.response.output.message.content }

    # 4. 有 tool call → 執行 tools
    - id: tool_results
      type: task
      action: agent.exec_tool_calls
      input:
        tool_calls: ${ steps.response.output.tool_calls }

    # 5. 將 tool results 加入對話紀錄
    - type: task
      action: agent.set_messages
      input:
        messages: ${ session.messages.concat(steps.tool_results.output.results) }

    # 回到 loop 頂部（下一次迭代）
```

##### 自訂 Loop 範例：對話摘要壓縮

```yaml
loop:
  steps:
    - id: response
      type: task
      action: agent.llm_call
      input:
        messages: ${ session.messages }

    - type: task
      action: agent.set_messages
      input:
        messages: ${ session.messages.append(steps.response.output.message) }

    - type: if
      expr: ${ !steps.response.output.has_tool_calls }
      then:
        - type: return
          output: ${ steps.response.output.message.content }

    - id: tool_results
      type: task
      action: agent.exec_tool_calls
      input:
        tool_calls: ${ steps.response.output.tool_calls }

    - type: task
      action: agent.set_messages
      input:
        messages: ${ session.messages.concat(steps.tool_results.output.results) }

    # 每 5 輪壓縮對話紀錄
    - type: if
      expr: ${ session.iteration > 0 && session.iteration % 5 == 0 }
      then:
        - id: summarize
          type: task
          action: agent.llm_call
          input:
            messages:
              - role: user
                content: ${ "請用 200 字摘要以下對話重點：\n" + session.messages.map(m, m.content).join("\n") }
        - type: task
          action: agent.set_messages
          input:
            messages:
              - ${ session.messages[0] }
              - role: user
                content: ${ "（前述對話摘要）" + steps.summarize.output.message.content }
              - ${ session.messages[size(session.messages) - 1] }

##### 自訂 Loop 範例：在 loop 中發事件

```yaml
loop:
  steps:
    - id: response
      type: task
      action: agent.llm_call
      input:
        messages: ${ session.messages }

    - type: task
      action: agent.set_messages
      input:
        messages: ${ session.messages.append(steps.response.output.message) }

    # 每次迭代都發 progress 事件
    - type: emit
      event: agent.iteration_completed
      data:
        iteration: ${ session.iteration }
        has_tool_calls: ${ steps.response.output.has_tool_calls }

    - type: if
      expr: ${ !steps.response.output.has_tool_calls }
      then:
        - type: return
          output: ${ steps.response.output.message.content }

    - id: tool_results
      type: task
      action: agent.exec_tool_calls
      input:
        tool_calls: ${ steps.response.output.tool_calls }

    - type: task
      action: agent.set_messages
      input:
        messages: ${ session.messages.concat(steps.tool_results.output.results) }
```

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

### 系統預設 Loop 的 Crash Recovery 限制

與 task step 不同，agent step 的 crash recovery 較困難：

- Agent session 的 LLM conversation context 在記憶體中，crash 後遺失
- `persist_history: full` 時，可從 `agent_messages` 重建 conversation 並繼續
- `persist_history: summary` 或 `none` 時，無法恢復中間狀態，必須重新執行
- 已執行的 tool calls（如 task、workflow）已產生副作用，重跑時需注意 idempotency

### 自訂 Loop 的 Crash Recovery

自訂 loop 的 recovery 比系統預設 loop 更明確：

- 每輪迭代的每個 step 都有持久化的狀態和 output
- `session.messages` 透過 `agent.set_messages` 已持久化至資料庫
- Recovery 時可從最後一個成功的 step 繼續（與一般 workflow step recovery 相同）
- 唯一例外是 `agent.llm_call` 的回應若未被 `agent.set_messages` 持久化就 crash，需重新呼叫 LLM

---

## 十、待設計功能

以下功能尚在構想階段，待後續設計與定案。

> DSL 層級的擴充功能（`kind: resources`、`kind: tools`、string template、`expr` 改名、`when`、`between`、`router`、`exists`、model alias、tool 命名）已移至 [draft/dsl-extensions.md](dsl-extensions.md)。

### 10.1 Session ID 概念

Agent session 需要一個唯一的 session ID，作為整個 agent 執行生命週期的識別符。

#### 產生方式

- 引擎在 agent step 進入 RUNNING 時自動產生（UUID v7，含時間戳排序）
- 格式：`asess_<uuid>`（agent session 前綴，便於辨識）

#### 用途

| 用途 | 說明 |
|------|------|
| 對話持久化 | `agent_sessions.id`、`agent_messages.session_id` 的主鍵 |
| API 操作 | `POST /agents/{session_id}/respond`（ask_human 回覆） |
| Agent-as-Tool | 子 agent session 透過 `parent_session_id` 關聯父 agent |
| 日誌與觀測 | 所有 agent 相關的 log、trace、metric 以 session ID 為維度 |
| Crash Recovery | 透過 session ID 查詢 session 狀態，決定恢復策略 |

#### 在表達式中存取

```yaml
# 在自訂 loop 或 prompt 中可存取 session ID
prompt: "Session ${ session.id } — 請分析訂單 ${ input.order_id }"
```

`session.id` 加入 `session` namespace，與 `session.messages`、`session.iteration` 並列。

#### 與 Workflow Instance 的關係

```
workflow instance (instance_id)
  └── step: agent (step_id)
        └── agent session (session_id)
              └── agent-as-tool 子 agent (child session_id, parent_session_id)
```

一個 workflow instance 中可能有多個 agent step，每個 agent step 產生一個 session。若 agent step 因 retry 重新執行，會產生新的 session ID。

### 10.2 Agent 引用 Artifact

Agent 需要能讀取 workflow artifact 作為上下文，也需要能寫入 artifact 作為產出。

#### 讀取 Artifact

透過內建 tool `artifact.read` 讓 agent 在 agentic loop 中按需讀取 artifact：

```yaml
# Agent 的 tools 中加入 artifact 工具
tools:
  - type: tool
    name: artifact.read
  - type: tool
    name: artifact.write
```

| Tool | Input | Output | 說明 |
|------|-------|--------|------|
| `artifact.read` | `{ name: string }` | `{ kind, content, content_type?, uri? }` | 讀取指定 artifact |
| `artifact.write` | `{ name: string, content: any, content_type?: string }` | `{}` | 寫入或更新 artifact |
| `artifact.list` | `{}` | `{ artifacts: array }` | 列出可用 artifact 名稱與 metadata |

#### Prompt 中直接注入

除了 tool 方式，也可在 prompt 中直接引用 artifact 內容：

```yaml
- type: agent
  agent: document.reviewer
  prompt: |
    請審閱以下文件並給出修改建議：
    ${ artifacts.order_file.content }
```

#### 寫入 Artifact

Agent 透過 `artifact.write` tool 將產出寫入 artifact，後續 workflow step 可透過 `artifacts.<name>` 存取。

```yaml
# agent 在 agentic loop 中呼叫 artifact.write
# → workflow 後續 step 可存取 artifacts.result_report
```

#### 權限控制

Agent 可存取的 artifact 範圍與 workflow step 相同 — 限於同一 workflow instance 的 artifact。Agent-as-tool 的子 agent 不繼承父 agent 的 artifact 存取（與資料隔離原則一致）。

### 10.3 Tool 倉庫搜尋工具

提供一種內建 tool 讓 agent 能在 tool registry 中搜尋與發現可用工具，而非僅限於靜態配置的 tools 列表。

#### 設計動機

- Agent 的可用工具數量可能很大，全部列入 function call schema 會降低 LLM 效能
- Agent 可能需要根據任務動態發現適合的 tool
- 類似 MCP 的 `tools/list` 但跨所有 tool 來源

#### 內建 Tool

```yaml
tools:
  - type: tool
    name: registry.search
```

| Tool | Input | Output |
|------|-------|--------|
| `registry.search` | `{ query: string, type?: string, limit?: integer }` | `{ tools: [{ name, type, description, input_schema }] }` |
| `registry.list` | `{ type?: string, namespace?: string }` | `{ tools: [{ name, type, description }] }` |

#### 搜尋範圍

`registry.search` 搜尋引擎已註冊的所有 tool definition：

- 所有已 PUBLISHED 的 task definition（轉為 tool schema）
- 所有已 PUBLISHED 的 workflow definition
- 所有已 PUBLISHED 的 agent definition（agent-as-tool）
- 所有已配置的 MCP server 提供的 tools

#### 動態載入

搜尋到的 tool 並非自動可用。Agent 需要透過另一個內建 tool 將其加入當前 session 的可用工具清單：

```yaml
tools:
  - type: tool
    name: registry.search
  - type: tool
    name: registry.activate       # 將搜尋到的 tool 加入可用列表
```

| Tool | Input | Output | 說明 |
|------|-------|--------|------|
| `registry.activate` | `{ name: string }` | `{ tool: object }` | 將 tool 加入當前 session，後續迭代可使用 |

#### 安全考量

- `registry.activate` 只能啟用引擎中已註冊的 tool，不能任意呼叫外部服務
- 可透過 agent definition 的 config 限制可啟用的 tool namespace

### 10.4 `ask_human` 格式擴充

擴充 `agent.ask_human` 工具，支援提供選項（選擇題）與一次問多個問題。

#### 現有格式

```yaml
# 現有：純文字提問
{ question: string, context?: string }
```

#### 擴充後格式

```yaml
# 擴充後的 input schema
{
  questions: [                    # 問題陣列，支援多個問題
    {
      id: string,                 # 問題 ID，用於回覆對應
      question: string,           # 問題內容
      type: "text" | "select" | "multi_select" | "confirm",
      options?: [                 # type 為 select / multi_select 時
        { value: string, label: string }
      ],
      required?: boolean,         # 預設 true
      default?: any               # 預設值
    }
  ],
  context?: string                # 整體上下文說明
}
```

#### 回覆格式

```yaml
# 回覆 API：POST /agents/{session_id}/respond
{
  answers: {
    "<question_id>": any          # key 為問題 ID，value 為回答
  }
}
```

#### 範例

```yaml
# Agent 呼叫 ask_human
{
  "name": "agent.ask_human",
  "input": {
    "questions": [
      {
        "id": "approve",
        "question": "此訂單金額異常（$50,000），是否核准？",
        "type": "confirm"
      },
      {
        "id": "reason",
        "question": "請說明核准/拒絕原因",
        "type": "text",
        "required": false
      },
      {
        "id": "priority",
        "question": "處理優先級",
        "type": "select",
        "options": [
          { "value": "low", "label": "低" },
          { "value": "medium", "label": "中" },
          { "value": "high", "label": "高" }
        ],
        "default": "medium"
      }
    ],
    "context": "訂單 ORD-001，客戶 CUST-789，歷史訂單平均金額 $5,000"
  }
}
```

#### 向後相容

保留單一問題的簡寫形式：

```yaml
# 簡寫（等效於 questions 陣列中一個 type: text 的問題）
{ question: "需要確認：此訂單金額異常，是否繼續？", context: "..." }
```

### 10.5 Prompt Override / Append 語意

評估 agent 使用時 exclude、append 或 override prompt/tools 的功能。應優先使用 input 傳遞參數，避免過度覆寫。

> **原則**：input 優先於 prompt override，降低耦合
>
> 詳細的 string template 與 prompt 組裝語法見 [draft/dsl-extensions.md](dsl-extensions.md) 第三節。

### 10.6 Agent 標準 Hook

為 agent 提供標準 hook 機制，允許在 agent 生命週期的關鍵節點執行自訂邏輯。

#### Hook 觸發點

| Hook | 觸發時機 | 可用資料 | 典型用途 |
|------|---------|---------|---------|
| `on_session_start` | Agent session 初始化完成後 | `session.id`、`input` | 初始化外部資源、記錄啟動 |
| `on_iteration_start` | 每輪迭代開始前 | `session.iteration`、`session.messages` | 注入動態 context、檢查前置條件 |
| `on_before_llm_call` | 送 messages 給 LLM 前 | `messages` | 修改 messages、注入系統訊息 |
| `on_after_llm_call` | 收到 LLM 回應後 | `response`、`has_tool_calls` | 記錄回應、攔截特定內容 |
| `on_before_tool_call` | 執行 tool 前 | `tool_name`、`tool_input` | 權限檢查、輸入驗證、記錄 |
| `on_after_tool_call` | Tool 執行完成後 | `tool_name`、`tool_output`、`success` | 結果轉換、錯誤處理 |
| `on_session_end` | Agent session 結束時 | `final_output`、`status` | 清理資源、記錄結果 |

#### 配置方式

在 agent definition 中以 `hooks` 欄位定義，每個 hook 指向 workflow step 陣列：

```yaml
hooks:
  on_before_tool_call:
    - type: if
      expr: ${ hook.tool_name == "order.delete" }
      then:
        - type: task
          action: audit.log
          input:
            action: "agent_tool_call"
            tool: ${ hook.tool_name }
            session_id: ${ session.id }

  on_session_end:
    - type: task
      action: metrics.record
      input:
        session_id: ${ session.id }
        iterations: ${ session.iteration }
        tool_calls: ${ session.tool_call_count }
```

#### Hook Namespace

Hook steps 中額外可存取 `hook` namespace，包含該 hook 觸發時的上下文資料。

### 10.7 Agent Loop 中修改 Prompt / System Prompt

定義在 agent loop 迭代過程中修改 prompt 與 system prompt 的方式。

#### 方式一：透過 `agent.set_messages`（已有）

直接操作 `session.messages` 陣列，替換或修改 system / user message：

```yaml
# 在自訂 loop 中替換 system prompt
- type: task
  action: agent.set_messages
  input:
    messages:
      - role: system
        content: ${ "原始 system prompt\n\n追加 context：" + vars.new_context }
      - ${ session.messages.slice(1) }   # 保留其餘 messages
```

#### 方式二：新增 `agent.update_system_prompt`（待評估）

提供專用 task，不需手動操作整個 messages 陣列：

```yaml
- type: task
  action: agent.update_system_prompt
  input:
    append: ${ "目前進度：已完成 " + string(session.iteration) + " 輪分析" }
```

| Task | Input | 說明 |
|------|-------|------|
| `agent.update_system_prompt` | `{ content?: string, append?: string }` | `content` 完全取代；`append` 追加至尾部 |
| `agent.update_prompt` | `{ content?: string, append?: string }` | 修改最新的 user message |

> **待決定**：是否需要方式二，或 `agent.set_messages` 已足夠

### 10.8 Agent 支援 `sub_workflow` 與 `agent` Step

移除自訂 loop 中「不支援 `sub_workflow` 和 `agent`」的限制。

#### 設計變更

原規格第五節限制自訂 loop 中不支援 `sub_workflow` 和 `agent` step，理由是「巢狀 agent loop 應透過 agent-as-tool 機制」。但實際場景中，自訂 loop 內直接使用這兩種 step 更直觀：

```yaml
loop:
  steps:
    - id: response
      type: task
      action: agent.llm_call
      input:
        messages: ${ session.messages }

    # 根據 LLM 判斷，啟動子 workflow 做複雜處理
    - type: if
      expr: ${ steps.response.output.needs_deep_analysis }
      then:
        - id: deep
          type: sub_workflow
          workflow: deep_analysis
          input:
            data: ${ steps.response.output.analysis_target }

    # 或呼叫另一個 agent 做專項分析
    - type: if
      expr: ${ steps.response.output.needs_expert }
      then:
        - id: expert
          type: agent
          agent: domain.expert
          prompt: ${ steps.response.output.expert_question }

    - type: return
      output: ${ steps.response.output.final_answer }
```

#### 與 Agent-as-Tool 的差異

| 維度 | `agent` step（自訂 loop 中） | Agent-as-Tool |
|------|------------------------------|---------------|
| 觸發者 | Workflow step（程式碼控制） | LLM 決定（function call） |
| Prompt 來源 | Step 的 `prompt` 欄位 | LLM 的 tool call input |
| 流程控制 | 可搭配 if/switch 做條件判斷 | LLM 自主判斷 |
| 適用場景 | 確定性流程中嵌入 agent | Agent 自主決定是否委託子 agent |

#### 深度限制

`sub_workflow` 和 `agent` step 在自訂 loop 中的深度限制與一般 workflow 相同（建議上限 5 層）。

### 10.9 Agent Stream 支援

定義 agent 如何支援 streaming 輸出，讓外部系統即時接收 agent 的部分結果。

#### 設計動機

- Agent 執行可能耗時數分鐘，外部系統需要即時回饋
- 使用者互動場景（如 chat）需要逐 token 串流
- 不同場景對 stream 的需求不同（token-level vs iteration-level）

#### Stream 層級

| 層級 | 說明 | 適用場景 |
|------|------|---------|
| `token` | LLM 回應逐 token 串流 | Chat UI、即時互動 |
| `iteration` | 每輪迭代結果完整送出 | 進度追蹤、Dashboard |
| `none`（預設） | 僅最終結果 | 批次處理、後台任務 |

#### 配置

```yaml
# Agent definition 或 step 層級
config:
  stream: token | iteration | none    # MAY, 預設 none
```

#### Token-level Stream

引擎在 `agent.llm_call` 時啟用 LLM 的 streaming API，透過引擎的 event 系統即時推送：

```yaml
# 引擎發出的 stream event
event: agent.stream.token
data:
  session_id: "asess_xxx"
  iteration: 3
  delta: "根據分析結果，"            # 增量文字
```

外部系統透過 SSE / WebSocket 訂閱 agent session 的 stream event。

#### Iteration-level Stream

每輪迭代完成後發出完整的迭代結果：

```yaml
# 引擎發出的 stream event
event: agent.stream.iteration
data:
  session_id: "asess_xxx"
  iteration: 3
  message: { role: assistant, content: "..." }
  tool_calls: [...]
  tool_results: [...]
```

#### API 端點

```
GET /agents/{session_id}/stream    # SSE 端點，訂閱 stream events
```

#### 自訂 Loop 中的 Stream

自訂 loop 可透過 `emit` step 手動發送 stream event：

```yaml
loop:
  steps:
    - id: response
      type: task
      action: agent.llm_call
      input:
        messages: ${ session.messages }
        stream: true                   # 啟用 token-level stream

    # 手動發送自訂 stream event
    - type: emit
      event: agent.stream.custom
      data:
        progress: ${ session.iteration }
        partial_result: ${ steps.response.output.message.content }
```

---

## 十一、待定義項目

以下項目需要獨立的規格設計，目前僅列出範圍與方向。

### 11.1 Agent Built-in 工具清單

列出 agent 內建工具（如 `get_tools`、`call_llms` 等），定義各工具的介面與行為。

> **現有**：`agent.ask_human`、`agent.activate_skill`、`agent.read_skill_file`（見第二節）
> **待擴充**：是否需要更多 builtin tools

### 11.2 History 定義與處理

定義 history 的資料結構、操作方式（讀取、截斷、摘要）及持久化方案。

> **現有**：`persist_history: full | summary | none`、`agent.set_messages`（見第四節、第七節）
> **待補充**：history 的讀取 API、截斷策略、摘要生成機制

### 11.3 Sub-agent / Fork Agent

如何從 agent 中啟動子 agent 或 fork 新 agent，定義父子關係與通訊機制。

> **現有**：agent-as-tool 機制（見第六節）
> **待補充**：fork 語意、父子 context 共享範圍、生命週期關聯

### 11.4 RAG / Memory 機制

評估是否透過 tools 方式實現 RAG 與長期記憶，而非內建於 agent 核心。

> **方向**：傾向以 MCP tools 或 builtin tools 實現，保持 agent 核心簡潔

### 11.5 Agent Team

多 agent 協作機制，包含共用 team tools、message box、任務分派與結果匯集。

> **待設計**：team 定義格式、通訊模型（shared message box vs direct call）、任務分派策略

---

## 十二、討論議題

> 以下列出待討論和修正的設計問題，歡迎直接在此標記意見。

### 議題 1：Agent-as-Tool 的 prompt 來源

目前設計：tool call input 可包含 `prompt` 欄位，或由引擎自動組合。

**疑問**：Agent B 作為 tool 時，description 已定義「何時使用」，但具體指令（prompt）的來源需要更明確的規範。是否應該讓 Agent A 總是能透過 tool call 傳入 prompt？

### 議題 2：Agentic Loop 的 Crash Recovery

`persist_history: full` 允許從 agent_messages 重建 conversation，但重建後繼續的行為需要更精確的定義（例如：重建 context 時是否包含 system prompt 的 skill 載入狀態？）。

### 議題 3：自訂 Loop 的 Crash Recovery

自訂 loop 的每一輪迭代包含多個 workflow steps，crash recovery 需要追蹤：
- 當前迭代次數
- 迭代內哪個 step 正在執行
- `session.messages` 的狀態（透過 `agent.set_messages` 已持久化）

與系統預設 loop 相比，自訂 loop 的中間狀態更明確（因為每個 step 都有持久化的 output），recovery 可能更容易。

### 議題 4：自訂 Loop 中 `agent.llm_call` 的 messages 格式

`agent.llm_call` 的 input `messages` 是否應該強制使用 `session.messages`？還是允許使用者傳入任意 messages（例如做 summarization 時用不同的 messages）？目前設計為任意 messages，但需考慮 tool schema injection 的時機。

### 議題 5：MCP Server 生命週期

MCP server（stdio transport）的 process 生命週期應該如何管理？

- 選項 A：每個 agent session 獨立啟動/關閉
- 選項 B：引擎層級的 connection pool，多個 session 共用
- 選項 C：可配置（per-session vs shared）

### 議題 6：Tool Execution 的並行性

LLM 可能在一次回傳中包含多個 tool calls。引擎是否應該並行執行這些 tool calls？

- 選項 A：總是循序執行（簡單、可預測）
- 選項 B：預設並行，除非 tool 標記為 `sequential_only`
- 選項 C：可配置

### 議題 7：Skills 的版本管理

目前 skills 以本地路徑引用，沒有版本概念。是否需要支援遠端 skill（如 git repo URL）或版本鎖定？

### 議題 8：Agent Step 在 Expression 中的 Namespace

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
  - type: tool
    name: agent.ask_human

skills:
  - name: risk-analysis

config:
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
    tools_exclude: [agent.ask_human]
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
      高風險案例必須透過 agent.ask_human 確認。
    input:
      order_id: ${ input.order_id }
    tools:
      - type: mcp
        server: compliance-mcp
        tools: [check_sanctions_list]
      - type: agent
        agent: fraud.detector
    skills:
      - name: compliance-review
    system_prompt_append: |
      審核模式：這是人工審核流程，你必須謹慎行事。
    config:
      persist_history: full
    timeout: 10m

  - type: return
    output:
      result: ${ steps.deep_analyze.output }
```
