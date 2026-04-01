# 16 — Agentic Loop

本文件定義 `type: agent` step 的完整 schema、能力合併規則、兩種執行模式（系統預設 loop 與自訂 loop），以及 output 處理模型。

---

## Agent Step Schema

Workflow 中的 `type: agent` step 引用 agent definition 並組裝實際能力。

```yaml
- id: string                          # MAY — 需要參照 output 時才必填
  type: agent                         # MUST
  agent: string                       # MUST — agent definition name
  version: integer | "latest"         # MAY, 預設 "latest"
  input: map | CEL expression         # MAY — 傳給 agent 的輸入
  prompt: string                      # MUST — 任務指令（支援 ${ } CEL 表達式）

  # 能力調整
  tools: array                        # MAY — 額外 tools（合併至 definition 預設）
  tools_override: array               # MAY — 完全覆寫 tools（忽略 definition 預設）
  tools_exclude: array                # MAY — 從最終 tools 中排除
  skills: array                       # MAY — 額外 skills（合併至 definition 預設）
  system_prompt_append: string        # MAY — 追加到 system prompt 尾部

  # 執行控制覆寫
  config:
    max_iterations: integer           # MAY
    max_tool_calls: integer           # MAY
    output_format: text | json        # MAY
    persist_history: full | summary | none  # MAY
    stream: token | iteration | none  # MAY

  # 標準 step 屬性
  timeout: duration                   # MAY
  execution_policy: replayable | idempotent | non_repeatable  # MAY, 預設 replayable
  retry:
    max_attempts: integer             # MAY, 預設 1
    delay: duration                   # MAY, 預設 1s
    backoff: fixed | exponential      # MAY, 預設 fixed
  catch: [...]                     # MAY, step 陣列
  on_timeout: [...]                   # MAY, step 陣列
```

### 欄位說明

| 欄位 | 說明 |
|------|------|
| `agent` | Agent definition name（dotted namespace，如 `order.analyzer`） |
| `prompt` | **MUST** — 任務指令。Agent definition 不含 prompt，因為每次呼叫的任務不同 |
| `tools` | 額外 tools，與 agent definition 預設合併 |
| `tools_override` | 完全取代 agent definition 的預設 tools，與 `tools` 互斥 |
| `tools_exclude` | 從合併後的 tools 中移除指定項目 |
| `skills` | 額外 skills，與 agent definition 預設合併 |
| `system_prompt_append` | 追加到 agent definition 的 `system_prompt` 尾部 |
| `config` | Step 層級覆寫 agent definition 的 config 預設值 |

---

## 能力合併規則

Step 層級的設定與 agent definition 的預設值依以下規則合併：

| Step 欄位 | 合併行為 |
|-----------|---------|
| `tools` | 與 agent definition 的 `tools` **合併**（union，去重） |
| `tools_override` | **完全取代** agent definition 的 `tools`（與 `tools` 互斥，同時設定 → validation error） |
| `tools_exclude` | 從合併後的 tools 中**移除**指定項目（by builtin name 或 task action） |
| `skills` | 與 agent definition 的 `skills` **合併**（union，去重） |
| `system_prompt_append` | **追加**到 agent definition 的 `system_prompt` 尾部 |
| `config.*` | Step 層級的值**覆寫** agent definition 的預設值 |

### `tools_exclude` 語法

以 tool 的識別名稱排除：

```yaml
tools_exclude:
  - agent.ask_human              # 排除內建 tool（by name）
  - order.load                   # 排除 task tool（by action）
  - payment_processing           # 排除 workflow tool（by workflow name）
  - fraud.detector               # 排除 agent tool（by agent name）
```

### 合併順序

當引用了 toolset 時，最終能力列表依以下順序產生：

```
1. Agent Definition 的 tools/skills（預設值）
2. Agent Definition 引用的 toolset（展開）
3. Step 層級的 tools/skills（增量合併）
4. Step 層級的 tools_override（若有，取代 1+2+3 的 tools）
5. Step 層級的 tools_exclude（從最終結果移除）
```

---

## 模式一：系統預設 Loop

當 agent definition **未定義** `loop` 時，使用引擎內建的標準 agentic loop。這是大多數 agent 的推薦模式——簡單、可預測、無需額外配置。

### 流程

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

### 限制

| 限制 | 說明 |
|------|------|
| 迭代上限 | `config.max_iterations`（預設 20） |
| Tool 呼叫上限 | `config.max_tool_calls`（可選，無預設） |
| 超過上限 | Step FAILED（error code: `agent_max_iterations` 或 `agent_max_tool_calls`） |

### 執行語意

1. Step 進入 RUNNING
2. 引擎組裝 agent session：
   - 載入 agent definition 預設值
   - 合併 step 層級的 tools / skills / config
   - 求值 `prompt`（CEL 表達式 + input namespace）
   - 組合 system_prompt = definition.system_prompt + step.system_prompt_append
   - 初始化 tool connections（建立 MCP server 連線、解析 task/workflow/agent schema）
   - 載入 skills metadata 注入 system prompt
   - 初始化 `session` namespace
3. 進入標準 agentic loop（上方流程圖）
4. LLM 回傳無 tool call 的 final response → step output

---

## 模式二：自訂 Loop

當 agent definition **定義了** `loop` 時，引擎不使用內建 loop，改為重複執行 `loop.steps` 中的 step 序列，直到遇到 `return` 或 `fail` step 為止。

### 流程

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

### `session` Namespace

自訂 loop 的 steps 除了標準 namespace（`input`、`steps`、`vars`）外，額外可存取 `session` namespace：

| 變數 | 型別 | 說明 |
|------|------|------|
| `session.id` | string | Agent session 唯一識別符（`asess_<uuid>`） |
| `session.messages` | array | 當前對話紀錄（message 陣列） |
| `session.iteration` | integer | 當前迭代次數（從 0 開始） |
| `session.tool_call_count` | integer | 累計 tool 呼叫次數 |

`session.messages` 中的每個 message 結構：

```yaml
- role: system | user | assistant | tool
  content: string | array       # 文字或 content blocks
  tool_calls: array             # assistant message 的 tool calls（MAY）
  tool_call_id: string          # tool message 的對應 ID（MAY）
```

初始化時，引擎自動注入：

1. `system` message（system_prompt + skills metadata）
2. `user` message（prompt）

### Agent 內建 Task

自訂 loop 透過以下內建 task 與 LLM 和 tool 系統互動：

| Task Action | 說明 | Input | Output |
|-------------|------|-------|--------|
| `agent.llm_call` | 將 messages 送給 LLM | `{ messages: array }` | `{ message: object, has_tool_calls: bool, tool_calls: array }` |
| `agent.exec_tool_calls` | 執行一組 tool calls | `{ tool_calls: array }` | `{ results: array }` — 每個 result 為 `{ tool_call_id, content }` |
| `agent.set_messages` | 取代整個對話紀錄 | `{ messages: array }` | `{}` |

- **`agent.llm_call`**：使用 agent definition 的 `model` 設定呼叫 LLM。Input 的 `messages` 包含完整對話紀錄。Output 的 `message` 為 LLM 回傳的 assistant message。允許傳入任意 messages（如做 summarization），不限於 `session.messages`（決策 M2）。
- **`agent.exec_tool_calls`**：引擎根據 tool 定義（task / workflow / agent / mcp / builtin）分派執行。預設並行執行，tool 可標記 `sequential_only`（決策 M5）。Output 的 `results` 為 tool role 的 messages。
- **`agent.set_messages`**：直接取代 `session.messages`，用於裁剪對話、注入摘要、移除舊訊息等。

### 自訂 Loop 中可用的 Step 類型

> **決策 M7**：自訂 loop 支援所有標準 workflow step 類型，不再限制 `sub_workflow` 和 `agent`。

自訂 loop 的 steps 支援**所有**標準 workflow step 類型：

- `task`（含 `agent.llm_call` 等內建 task）
- `assign`（修改 `vars`）
- `if` / `switch`（條件判斷）
- `foreach` / `parallel`（迴圈與平行）
- `emit`（發送事件）
- `wait`（等待外部事件或時間，loop 暫停）
- `return`（結束 loop，產生 output）
- `fail`（中止 loop 並報錯）
- `sub_workflow`（呼叫子 workflow）
- `agent`（呼叫另一個 agent）

#### 深度限制

`sub_workflow` 和 `agent` step 在自訂 loop 中的深度限制為 **5 層**，與一般 workflow 相同。超過深度限制 → step FAILED（error code: `max_depth_exceeded`）。

---

## Output 模型

Agent 的最終回應依 `output_format` 處理後作為 step output：

| `output_format` | 處理方式 | 存取方式 |
|-----------------|---------|---------|
| `json`（預設） | 從 LLM 回應中解析 JSON | `steps.<id>.output` 為 map |
| `text` | LLM 回應直接作為字串 | `steps.<id>.output.text` |

### output_schema 驗證

若 agent definition 定義了 `output_schema`，引擎在解析 output 後進行 JSON Schema 驗證：

- 驗證通過 → output 正常寫入 step output
- 驗證失敗 → step FAILED（error code: `agent_output_invalid`）

```yaml
- id: analyze
  type: agent
  agent: order.analyzer
  prompt: "分析訂單 ${ input.order_id } 的風險等級"
  input:
    order_id: ${ input.order_id }

# output_format: json → steps.analyze.output 為 map
- type: if
  when: ${ steps.analyze.output.risk_level == "high" }
  then:
    - type: task
      action: order.flag_for_review
      input:
        order_id: ${ input.order_id }
        reason: ${ steps.analyze.output.summary }
```

---

## 自訂 Loop 範例

### 範例一：標準 loop 的等效實作

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
      when: ${ !steps.response.output.has_tool_calls }
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

### 範例二：對話摘要壓縮

每 5 輪迭代自動壓縮對話紀錄，避免 context window 溢出：

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
      when: ${ !steps.response.output.has_tool_calls }
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
      when: ${ session.iteration > 0 && session.iteration % 5 == 0 }
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
```

### 範例三：在 loop 中發事件

每次迭代完成後發送 progress 事件，供外部系統追蹤進度：

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
      when: ${ !steps.response.output.has_tool_calls }
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

---

## 相關文件

- [05-steps-overview](../02-steps/05-steps-overview.md) — step 類型總覽
- [06-step-task](../02-steps/06-step-task.md) — task step 與 task definition
- [11-step-sub-workflow](../02-steps/11-step-sub-workflow.md) — sub_workflow step（自訂 loop 中可使用）
- [15-agent-skills](15-agent-skills.md) — Agent skills 與 agentskills.io 規格
- [17-agent-hooks-and-streaming](17-agent-hooks-and-streaming.md) — Agent hooks 與 streaming
