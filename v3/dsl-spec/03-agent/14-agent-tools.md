# 14 — Agent Tools

本文件定義 agent 可使用的工具類型、命名慣例、MCP server 配置、執行並行性，以及所有內建 agent tools 的完整規格。

---

## 1. Tool 類型

Agent 可使用的工具分為 5 種類型。所有類型在引擎內部統一抽象為 tool interface，對 LLM 而言都是 function call。

### 1.1 type: task

引用已定義的 task definition 作為工具。

```yaml
- type: task
  action: string                    # MUST — task definition name（兩段式命名）
  version: integer | "latest"       # MAY, 預設 "latest"
  name: string                      # MAY — 覆寫 LLM 看到的 tool 名稱
  description: string               # MAY — 覆寫描述
```

### 1.2 type: workflow

引用已定義的 workflow definition 作為工具。

```yaml
- type: workflow
  workflow: string                  # MUST — workflow definition name
  version: integer | "latest"       # MAY, 預設 "latest"
  name: string                      # MAY — 覆寫 tool 名稱
  description: string               # MAY — 覆寫描述
```

### 1.3 type: agent

引用已定義的 agent definition 作為工具（agent-as-tool）。

```yaml
- type: agent
  agent: string                     # MUST — agent definition name
  version: integer | "latest"       # MAY, 預設 "latest"
  name: string                      # MAY — 覆寫 tool 名稱
  description: string               # MAY — 覆寫描述
```

### 1.4 type: mcp

從外部 MCP server 取得工具。

```yaml
- type: mcp
  server: string                    # MUST — 引擎層級配置的 MCP server 名稱
  tools: array                      # MAY — 限定可用的 tool 名稱列表；省略 = 全部
  resources: array                  # MAY — 限定可用的 resource 名稱列表
```

### 1.5 type: tool

引擎內建的 agent 工具，採用兩段式命名。此類型取代原有的 `type: builtin`。

```yaml
- type: tool
  name: string                      # MUST — 工具名稱（namespace.action 格式）
```

**遷移說明**：`type: builtin` 已移除，所有原本使用 `type: builtin` 的工具改為 `type: tool` 搭配兩段式命名。

```yaml
# 舊寫法（廢棄）
- type: builtin
  name: ask_human

# 新寫法
- type: tool
  name: agent.ask_human
```

---

## 2. 兩段式 Tool 命名

所有工具名稱統一採用 `namespace.action` 格式，以點號（`.`）分隔。提供一致的命名結構，便於分類、搜尋與衝突避免。

### 命名規則

| 類型 | 格式 | 範例 |
|------|------|------|
| 內建 agent tool | `agent.<action>` | `agent.ask_human`、`agent.activate_skill`、`agent.read_skill_file` |
| 內建 agent task | `agent.<action>` | `agent.llm_call`、`agent.exec_tool_calls`、`agent.set_messages` |
| Artifact 操作 | `artifact.<action>` | `artifact.read`、`artifact.write`、`artifact.list` |
| Tool 倉庫 | `registry.<action>` | `registry.search`、`registry.list`、`registry.activate` |
| Task tool | `<namespace>.<action>` | `order.load`、`customer.get_history` |
| Workflow tool | 使用 workflow definition 的 `metadata.name` | `payment_processing` |
| Agent tool | 使用 agent definition 的 `metadata.name` | `order.analyzer` |

### 規則

- 所有內建工具的 namespace 為保留字（`agent`、`artifact`、`registry`），使用者定義的 tool 不得使用這些 namespace
- Task definition 的 `metadata.name` 本身即為兩段式格式，直接作為 tool 名稱
- `name` 與 `description` 欄位可覆寫預設值，允許在不同 context 下為同一工具提供不同描述

---

## 3. Function Call Schema 轉換

引擎將每種 tool 類型轉換為 LLM function call schema，使 LLM 能以統一介面呼叫所有工具。

| Tool 類型 | function name 來源 | parameters 來源 | description 來源 |
|-----------|-------------------|----------------|-----------------|
| `task` | `action`（或覆寫的 `name`） | task definition 的 `input_schema` | task definition 的 `description`（或覆寫值） |
| `workflow` | `workflow`（或覆寫的 `name`） | workflow definition 的 `input_schema` | workflow definition 的 `description`（或覆寫值） |
| `agent` | `agent`（或覆寫的 `name`） | agent definition 的 `input_schema` | agent definition 的 `description`（或覆寫值） |
| `mcp` | MCP server 回傳的 tool name | MCP server 回傳的 tool schema | MCP server 回傳的 tool description |
| `tool` | `name` 欄位值 | 引擎預定義的 schema | 引擎預定義的 description |

### 覆寫優先順序

Tool 宣告中的 `name` 和 `description` 欄位可覆寫來自 definition 或 MCP server 的預設值。覆寫後的值用於生成 function call schema。

---

## 4. MCP Server 配置

MCP server 在**引擎層級**配置，agent definition 中透過 `tools[type=mcp].server` 引用。

### 配置範例

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

### 欄位定義

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `transport` | string | MUST | 傳輸方式：`stdio` 或 `sse` |
| `command` | string | MUST（stdio） | 啟動 MCP server 的命令 |
| `url` | string | MUST（sse） | SSE endpoint URL |
| `env` | map | MAY | 環境變數（支援 `${ }` CEL 表達式） |
| `headers` | map | MAY | HTTP headers（僅 `sse` 適用） |
| `config` | map | MAY | 傳給 MCP server 的靜態設定 |

### 生命週期

MCP server 採用 **per agent session** 的生命週期（決策 M4）：

- 每個 agent session 啟動時，引擎為該 session 所需的 MCP server 建立獨立連線
- Session 初始化時拉取 MCP server 的 tool schema，轉為 function call schema
- Agent session 結束時，引擎關閉該 session 的所有 MCP server 連線
- 不同 agent session 之間的 MCP server 連線互不影響

此設計確保 MCP server 的狀態隔離，避免跨 session 的副作用。

---

## 5. Tool 執行並行性

當 LLM 在單次回應中返回多個 tool calls 時，引擎的執行策略如下（決策 M5）：

- **預設並行**：引擎同時執行所有 tool calls，提高吞吐量
- **`sequential_only` 標記**：tool 可標記為 `sequential_only: true`，表示此 tool 必須依序執行，不可與其他 tool 同時執行

### sequential_only 標記方式

```yaml
# 在 tool 宣告中標記
tools:
  - type: task
    action: order.update
    sequential_only: true           # 此 tool 不可並行執行

  - type: tool
    name: artifact.write
    sequential_only: true
```

### 執行規則

1. 當所有 tool calls 均為非 `sequential_only`：全部並行執行
2. 當其中包含 `sequential_only` tool：該 tool 依序執行，其餘非 `sequential_only` tool 可並行
3. 多個 `sequential_only` tool 之間依 LLM 回傳順序依序執行

---

## 6. 內建 Agent Tools

以下為引擎預定義的內建工具，使用 `type: tool` 宣告。

### 6.1 agent.ask_human

暫停 agent，向人類提問並等待回答。Agent session 進入 WAITING 狀態，直到外部系統回覆。

#### Input Schema

```yaml
{
  questions: [                        # 問題陣列，支援多個問題
    {
      id: string,                     # MUST — 問題 ID，用於回覆對應
      question: string,               # MUST — 問題內容
      type: "text" | "select" | "multi_select" | "confirm",  # MUST — 問題類型
      options: [                      # MAY — type 為 select / multi_select 時必填
        { value: string, label: string }
      ],
      required: boolean,              # MAY, 預設 true
      default: any                    # MAY — 預設值
    }
  ],
  context: string                     # MAY — 整體上下文說明
}
```

#### 問題類型

| type | 說明 | options | 回覆值型別 |
|------|------|---------|-----------|
| `text` | 自由文字輸入 | 不適用 | `string` |
| `select` | 單選 | MUST | `string`（選中的 value） |
| `multi_select` | 多選 | MUST | `array`（選中的 value 陣列） |
| `confirm` | 是/否確認 | 不適用 | `boolean` |

#### 回覆格式

外部系統透過 API 回傳答案：

```
POST /agents/{session_id}/respond
```

```yaml
{
  answers: {
    "<question_id>": any              # key 為問題 ID，value 為回答
  }
}
```

#### 範例

```yaml
# Agent 呼叫 ask_human — 多問題
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

#### 向後相容簡寫

保留單一問題的簡寫形式，等效於 `questions` 陣列中一個 `type: text` 的問題：

```yaml
{ question: "需要確認：此訂單金額異常，是否繼續？", context: "..." }
```

#### 行為流程

1. Agent 呼叫 `agent.ask_human`
2. Agent session 進入 WAITING 狀態
3. 引擎發出外部通知（webhook / event）
4. 外部系統透過 `POST /agents/{session_id}/respond` 回傳答案
5. Agent 收到答案後恢復 RUNNING，繼續 agentic loop

### 6.2 agent.activate_skill

載入指定 skill 的完整 SKILL.md 指令到 agent context。當 agent definition 配置了 `skills` 時，此工具自動注入可用工具清單。

| 欄位 | 說明 |
|------|------|
| **Input** | `{ name: string }` — skill 名稱 |
| **Output** | `{ content: string }` — SKILL.md 的完整內容 |
| **行為** | 將 SKILL.md body 載入 context，LLM 後續可依據指令行動 |

### 6.3 agent.read_skill_file

讀取 skill 目錄中的 scripts / references / assets 檔案。當 agent definition 配置了 `skills` 時，此工具自動注入。

| 欄位 | 說明 |
|------|------|
| **Input** | `{ skill: string, path: string }` — skill 名稱與檔案路徑 |
| **Output** | `{ content: string }` — 檔案內容 |
| **行為** | 按需讀取 skill 內的檔案（progressive disclosure） |

### 6.4 artifact.read / artifact.write / artifact.list

管理 workflow artifact，讓 agent 能讀取上下文資料與寫入產出。

| Tool | Input | Output | 說明 |
|------|-------|--------|------|
| `artifact.read` | `{ name: string }` | `{ kind, content, content_type?, uri? }` | 讀取指定 artifact |
| `artifact.write` | `{ name: string, content: any, content_type?: string }` | `{}` | 寫入或更新 artifact |
| `artifact.list` | `{}` | `{ artifacts: array }` | 列出可用 artifact 名稱與 metadata |

#### 存取範圍

- Agent 可存取的 artifact 範圍與 workflow step 相同 — 限於同一 workflow instance 的 artifact
- **Agent-as-tool 的子 agent 不繼承父 agent 的 artifact 存取**，與資料隔離原則一致

#### 在 prompt 中直接注入

除 tool 方式外，也可在 prompt 中以 CEL 表達式直接引用 artifact 內容：

```yaml
- type: agent
  agent: document.reviewer
  prompt: |
    請審閱以下文件並給出修改建議：
    ${ artifacts.order_file.content }
```

### 6.5 registry.search / registry.list / registry.activate

動態工具發現與啟用，讓 agent 在執行時期搜尋並載入所需工具。

| Tool | Input | Output | 說明 |
|------|-------|--------|------|
| `registry.search` | `{ query: string, type?: string, limit?: integer }` | `{ tools: [{ name, type, description, input_schema }] }` | 依關鍵字搜尋 tool registry |
| `registry.list` | `{ type?: string, namespace?: string }` | `{ tools: [{ name, type, description }] }` | 列出已註冊的工具 |
| `registry.activate` | `{ name: string }` | `{ tool: object }` | 將 tool 加入當前 session 的可用工具清單 |

#### 搜尋範圍

`registry.search` 搜尋引擎已註冊的所有 tool definition：

- 所有已 PUBLISHED 的 task definition（轉為 tool schema）
- 所有已 PUBLISHED 的 workflow definition
- 所有已 PUBLISHED 的 agent definition（agent-as-tool）
- 所有已配置的 MCP server 提供的 tools

#### 動態載入流程

搜尋到的 tool 並非自動可用，必須透過 `registry.activate` 明確啟用：

1. Agent 呼叫 `registry.search` 搜尋符合需求的工具
2. Agent 從結果中選擇適合的 tool
3. Agent 呼叫 `registry.activate` 將該 tool 加入當前 session
4. 後續迭代中，該 tool 出現在 LLM 的 function call schema 中

#### 安全限制

- `registry.activate` 只能啟用引擎中已註冊的 tool，不能任意呼叫外部服務
- 可透過 agent definition 的 config 限制可啟用的 tool namespace

---

## 7. agent.llm_call

內建 task，用於自訂 loop 中呼叫 LLM。允許傳入任意 messages（決策 M2），不強制使用 `session.messages`。

| 欄位 | 說明 |
|------|------|
| **Action** | `agent.llm_call` |
| **Input** | `{ messages: array }` — 完整的 messages 陣列 |
| **Output** | `{ message: object, has_tool_calls: bool, tool_calls: array }` |

### 行為

- 使用 agent definition 的 `model` 設定呼叫 LLM
- Input 的 `messages` 為完整對話紀錄（含 system、user、assistant、tool messages）
- Output 的 `message` 為 LLM 回傳的 assistant message
- Output 的 `tool_calls` 為 LLM 請求的 tool call 陣列（若有）

### 任意 messages 支援

`messages` 參數不限於 `session.messages`，可傳入任意組合的 messages。典型用途：

- **對話摘要**：傳入不同的 messages 請 LLM 做 summarization
- **獨立推理**：使用臨時 messages 進行輔助推理，不影響主對話

```yaml
# 使用 session.messages（一般用法）
- type: task
  action: agent.llm_call
  input:
    messages: ${ session.messages }

# 使用任意 messages（summarization）
- id: summarize
  type: task
  action: agent.llm_call
  input:
    messages:
      - role: user
        content: ${ "請用 200 字摘要以下對話重點：\n" + session.messages.map(m, m.content).join("\n") }
```

---

## 8. agent.exec_tool_calls

內建 task，執行一組 tool calls 並回傳結果。

| 欄位 | 說明 |
|------|------|
| **Action** | `agent.exec_tool_calls` |
| **Input** | `{ tool_calls: array }` — LLM 回傳的 tool call 陣列 |
| **Output** | `{ results: array }` — 每個 result 為 `{ tool_call_id, content }` |

### 行為

- 引擎根據 tool 定義（task / workflow / agent / mcp / tool）分派執行
- Output 的 `results` 為 tool role 的 messages，可直接加入 `session.messages`
- 並行性遵循第 5 節的規則（預設並行，`sequential_only` tool 除外）

```yaml
- id: tool_results
  type: task
  action: agent.exec_tool_calls
  input:
    tool_calls: ${ steps.response.output.tool_calls }

# 將 tool results 加入對話紀錄
- type: task
  action: agent.set_messages
  input:
    messages: ${ session.messages.concat(steps.tool_results.output.results) }
```

---

## 9. agent.set_messages

內建 task，取代整個 `session.messages` 對話紀錄。

| 欄位 | 說明 |
|------|------|
| **Action** | `agent.set_messages` |
| **Input** | `{ messages: array }` — 新的完整 messages 陣列 |
| **Output** | `{}` |

### 行為

- 直接取代 `session.messages`，用於裁剪對話、注入摘要、移除舊訊息等
- 取代後的 messages 立即持久化至資料庫

### 不需要 agent.update_system_prompt

`agent.set_messages` 已足夠處理 system prompt 的修改（決策 M3）。透過替換 messages 陣列中的 system message 即可達成：

```yaml
# 修改 system prompt
- type: task
  action: agent.set_messages
  input:
    messages:
      - role: system
        content: ${ "原始 system prompt\n\n追加 context：" + vars.new_context }
      - ${ session.messages.slice(1) }
```
