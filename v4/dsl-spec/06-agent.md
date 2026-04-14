# 06 — Agent Definition（`kind: Agent`）

本文件定義 `kind: Agent` 的 YAML 結構、agent step 的使用方式、tool 類型與 agentic loop。

---

## TODO

- Agent Definition 的生命週期 hooks
- Agent-as-tool 的使用範例
- Agentic Loop 的使用範例
- tools_override 與 tools_exclude 的使用範例
- agent 使用固定程式
- 動態 system prompt 與 prompt 的使用範例
- before_loop 與 after_loop 的使用範例

## Agent Definition

Agent Definition 是 AI agent 的**角色模板**——定義「這個 agent 是誰、預設能做什麼」。實際任務（prompt）由 workflow step 提供。

### 頂層結構

```yaml
apiVersion: agent/v4
kind: Agent

metadata:
  name: order.analyzer # MUST — dotted namespace
  version: 1
  description: "訂單風險分析 agent"

model: fast # MUST — alias 或完整物件

system: | # MAY — 角色定義
  你是訂單風險分析專家。

prompt: | # MAY — 任務定義
  分析訂單資料，評估風險等級（low/medium/high），並提供建議。

input_schema: object # MAY
output_schema: object # MAY

tools: # MAY — 預設可用 tool 列表
  - type: task
    action: order.load
  - type: mcp
    server: compliance-mcp
  - type: tool
    name: agent.ask_human

config:
  max_iterations: 20 # MAY, 預設 20
  max_tool_calls: 50 # MAY
  output_format: json # MAY — text | json, 預設 json
  persist_history: summary # MAY — full | summary | none, 預設 summary
  stream: none # MAY — token | iteration | none, 預設 none

hooks: { ... } # MAY — 生命週期 hooks
loop: # MAY — 自訂 loop steps
  steps: [...]
```

### model 欄位

| 寫法             | 範例                                                             | 說明                                |
| ---------------- | ---------------------------------------------------------------- | ----------------------------------- |
| string（alias）  | `model: fast`                                                    | 引用 Resources 中定義的 model alias |
| object（完整）   | `model: { provider: anthropic, name: claude-sonnet-4-20250514 }` | 直接指定                            |
| alias + 部分覆寫 | `model: { alias: fast, config: { max_tokens: 8192 } }`           | 覆寫部分設定                        |

---

## Agent Step

Workflow 中透過 `type: agent` step 呼叫 agent definition。

```yaml
- id: analyze
  type: agent
  agent: order.analyzer # MUST — agent definition name
  system: "你是訂單風險等級專家。" # MAY — 給 agent 附加的系統提示
  prompt: | # MAY — 給 agent 附加的任務提示
    分析訂單 ${ steps.load_order.output.id } 的風險等級。
  input: # MAY
    order: ${ steps.load_order.output }

  # 能力調整
  tools: # MAY — 額外附加的 tools（合併）
    - type: task
      action: customer.get_history
  tools_override: [...] # MAY — 完全覆寫 tools（與 tools 互斥）
  tools_exclude: # MAY — 排除特定 tools
    - agent.ask_human

  # 執行控制覆寫
  config:
    max_iterations: 10

  # 標準 step 屬性
  timeout: 2m
  retry: { ... }
  catch: [...]
```

### 欄位說明

| 欄位             | 說明                                                      |
| ---------------- | --------------------------------------------------------- |
| `system`         | 附加到 agent definition 的 `system_prompt` 尾部           |
| `prompt`         | 附加的任務指令，作為 user message 傳入                    |
| `tools`          | 額外 tools，與 agent definition 預設合併                  |
| `tools_override` | 完全取代 agent definition 的預設 tools（與 `tools` 互斥） |
| `tools_exclude`  | 從合併後的 tools 中移除指定項目                           |

### 能力合併規則

```
1. Agent Definition 的 tools（預設）
2. Agent Definition 引用的 toolsets（展開）
3. Step 層級的 tools（增量合併）
4. Step 層級的 tools_override（若有，取代上面全部）
5. Step 層級的 tools_exclude（從最終結果移除）
```

---

## Tool 類型

Agent 可使用 5 種 tool 類型，對 LLM 統一為 function call。

### type: task

引用 tool definition 作為工具。

```yaml
- type: task
  action: order.load # tool definition name
  name: "載入訂單" # MAY — 覆寫 LLM 看到的名稱
  description: "..." # MAY — 覆寫描述
```

### type: workflow

引用 workflow definition 作為工具。

```yaml
- type: workflow
  workflow: payment_processing
```

### type: agent

引用另一個 agent definition 作為工具（agent-as-tool）。

```yaml
- type: agent
  agent: fraud.detector
```

### type: mcp

從 MCP server 取得工具。

```yaml
- type: mcp
  server: github-mcp # Resources 中定義的 server name
  tools: [search_repos] # MAY — 限定工具，省略 = 全部
```

### type: tool

引擎內建工具，兩段式命名。

```yaml
- type: tool
  name: agent.ask_human
```

---

## 內建 Agent Tools

### agent.ask_human

暫停 agent，向人類提問。

```yaml
# Input
{ questions: [{ id: "approve", question: "是否核准？", type: "confirm" }] }
# type: text | select | multi_select | confirm
```

### agent.activate_skill / agent.read_skill_file

載入 skill 的 SKILL.md 指令 / 讀取 skill 目錄中的檔案。

### agent.llm_call

自訂 loop 中呼叫 LLM。Input 為 `{ messages: array }`。

### agent.exec_tool_calls

執行一組 tool calls。Input 為 `{ tool_calls: array }`。

### agent.set_messages

取代 `session.messages` 對話紀錄。

### artifact.list

列出 workflow artifact 的中繼資料與路徑。

### registry.search / registry.list / registry.activate

動態工具發現與啟用。

---

## Agentic Loop

### 系統預設 Loop

省略 `loop` 時，引擎執行預設的 agentic loop：

```
1. 組裝 messages（system prompt + user prompt）
2. 呼叫 LLM
3. 若 LLM 回傳 tool calls → 執行 tools → 將結果加入 messages → 回到 2
4. 若無 tool calls → 結束，agent output = LLM 最終回應
5. 達到 max_iterations 或 max_tool_calls → 結束
```

### 自訂 Loop

透過 `loop.steps` 完全控制 loop 邏輯：

```yaml
loop:
  steps:
    - id: response
      type: task
      action: agent.llm_call
      input:
        messages: ${ session.messages }

    - type: if
      when: ${ steps.response.output.has_tool_calls }
      then:
        - id: tool_results
          type: task
          action: agent.exec_tool_calls
          input:
            tool_calls: ${ steps.response.output.tool_calls }

        - type: task
          action: agent.set_messages
          input:
            messages: ${ session.messages
              .append(steps.response.output.message)
              .concat(steps.tool_results.output.results) }
      else:
        - type: return
          output: ${ steps.response.output.message.content }
```

自訂 loop 內可使用 `session` namespace（見 [04-expressions](04-expressions.md)）。
