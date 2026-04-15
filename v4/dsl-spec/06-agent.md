# 06 — Agent Definition（`kind: Agent`）

本文件定義 `kind: Agent` 的 YAML 結構、agent step 的使用方式，以及 agent 執行行為。

---

## 設計原則

- **統一呼叫模型**：agent 能使用的「工具」就是 task action。任何可被 `type: task` 呼叫的對象（Tool / Function / Agent / MCP / builtin）皆可作為 agent tool，不需另設類型。
- **角色與任務分離**：Agent Definition 是角色模板（「這個 agent 是誰、預設能做什麼」），實際任務由 workflow step 提供。
- **只有 `steps`，沒有特別的 loop 概念**：agent 的執行行為就是一段 `steps`。迴圈由一般 step 類型（`foreach` / `if` / `return`）組合而成。省略 `steps` 時引擎使用內建預設行為。

---

## 頂層結構

```yaml
apiVersion: agent/v4
kind: Agent

metadata:
  name: order.analyzer # MUST — snake_case + dotted
  version: 1
  description: "訂單風險分析 agent"

model: fast # MUST — alias 或完整物件

system: | # MAY — 角色定義（system prompt）
  你是訂單風險分析專家。

prompt: | # MAY — 預設任務提示
  分析訂單資料，評估風險等級（low/medium/high）並提供建議。

input_schema: object # MAY
output_schema: object # MAY

tools: # MAY — 可用的 task action 清單
  - order.load
  - customer.get_history
  - agent.ask_human

config:
  max_iterations: 20 # MAY, 預設 20
  max_tool_calls: 50 # MAY
  output_format: json # MAY — text | json, 預設 json
  persist_history: summary # MAY — full | summary | none, 預設 summary
  stream: none # MAY — token | iteration | none, 預設 none

hooks: { ... } # MAY — 生命週期 hooks

steps: [...] # MAY — 自訂執行行為（省略時用引擎預設 agentic loop）
```

### model 欄位

| 寫法             | 範例                                                             | 說明                          |
| ---------------- | ---------------------------------------------------------------- | ----------------------------- |
| string（alias）  | `model: fast`                                                    | 引用 Resources 的 model alias |
| object（完整）   | `model: { provider: anthropic, name: claude-sonnet-4-20250514 }` | 直接指定                      |
| alias + 部分覆寫 | `model: { alias: fast, config: { max_tokens: 8192 } }`           | 覆寫部分設定                  |

---

## tools — 可用工具清單

`tools` 是一個 task action 清單。每一項可以是字串（簡寫）或物件（帶覆寫）。

### 簡寫：字串

直接寫 action name。引擎依照 task registry 解析，對 LLM 以 function call 呈現，function name、description、parameters 取自該 action 的 metadata 與 `input_schema`。

```yaml
tools:
  - order.load # Tool
  - payment.process # Function
  - fraud.detector # 其他 Agent（agent-as-tool）
  - agent.ask_human # builtin
```

> Agent Definition 本身也註冊在 task registry，故任一 agent 都可作為其他 agent 的 tool。

### 物件形式：可覆寫

當需要重新命名、改寫說明，或預綁部分輸入時，使用物件形式：

```yaml
tools:
  - action: customer.get_history # MUST — 引用的 action name
    name: get_history # MAY — 覆寫 LLM 看到的 function 名稱
    description: "取得客戶歷史訂單" # MAY — 覆寫 description
    input: # MAY — 預綁輸入（LLM 看不到，執行時合併）
      include_refunds: true
```

`input` 預綁的欄位會從 LLM 看到的 JSON schema 中隱藏，執行時與 LLM 提供的引數合併（LLM 的引數優先）。

### Wildcard：批次引入

以 `namespace.*` 形式引入同一命名空間下的所有 action。常見用途是引入 MCP server 或 registry 下的 tool 集：

```yaml
tools:
  - "github.*" # 引入 github 命名空間下所有 action
  - "registry.search_ops.*" # 引入某個 registry 分類
```

MCP server 與外部 registry 如何把自身 tool 註冊進 task registry，見 [07-resources](07-resources.md)。

---

## Agent Step

Workflow 透過 `type: agent` step 呼叫 agent definition。

```yaml
- id: analyze
  type: agent
  agent: order.analyzer # MUST — agent definition name
  system: "特別注意高額訂單。" # MAY — 附加到 definition 的 system 尾部
  prompt: | # MAY — 附加的任務指令（user message）
    分析訂單 ${ steps.load_order.output.id } 的風險等級。
  input: # MAY — 傳入 agent 的輸入（對應 input_schema）
    order: ${ steps.load_order.output }

  # 工具調整
  tools: # MAY — 附加的 tools（與 definition 合併）
    - customer.get_history
  tools_override: [...] # MAY — 完全取代 definition 的 tools（與 tools 互斥）
  tools_exclude: # MAY — 從最終結果移除
    - agent.ask_human

  # 執行控制覆寫
  config:
    max_iterations: 10

  # 標準 step 屬性
  timeout: 2m
  retry: { ... }
  catch: [...]
```

### 欄位覆寫規則

| 欄位       | 合併行為                                            |
| ---------- | --------------------------------------------------- |
| `system`   | append 到 definition 的 `system` 尾部               |
| `prompt`   | append 到 definition 的 `prompt` 尾部（user message） |
| `input`    | 作為 `input` namespace 傳入                         |
| `config.*` | 逐欄覆寫 definition 的 `config`                     |

### tools 合併順序

```
1. Agent Definition 的 tools
2. Step 層級的 tools（append）
3. Step 層級的 tools_override（若有，取代 1+2）
4. Step 層級的 tools_exclude（從最終結果移除）
```

> `tools` 與 `tools_override` 互斥，同時出現時引擎 MUST 回報 validation error。

### 呼叫 agent 的另一種方式：task

因為 agent 註冊於 task registry，也可以用 `type: task` 呼叫：

```yaml
- id: analyze
  type: task
  action: order.analyzer
  input:
    order: ${ steps.load_order.output }
```

兩種寫法差異：

| 需求                                     | 建議                                    |
| ---------------------------------------- | --------------------------------------- |
| 只需要傳 input、取 output                | `type: task`（最單純）                  |
| 需要覆寫 system / prompt / tools / config | `type: agent`（支援覆寫欄位）           |

---

## 執行行為（steps）

### 預設行為

省略 `steps` 時，引擎執行內建的 agentic loop：

```
1. 組裝 messages（system + user prompt + 歷史）
2. 呼叫 LLM
3. 若回傳 tool calls → 依序執行對應的 task → 將結果加入 messages → 回到 2
4. 若無 tool calls → 結束，output = LLM 最終回應
5. 達到 max_iterations 或 max_tool_calls → 結束
```

### 自訂 steps

`steps` 是一般的 step 序列，所有 step 類型都可用，迴圈透過 `foreach` / `if` / `return` 組合達成。內部所有指令皆以 `type: task` 呼叫 `agent.*` 命名空間下的 builtin。

執行到 `type: return` 時結束 agent 並以其 `output` 為結果；自然 fall through 則以最後一個 step 的 output 為結果。

可用的 namespace：

| Namespace | 說明                                             |
| --------- | ------------------------------------------------ |
| `input`   | agent 的輸入                                     |
| `steps`   | 同一次 agent 執行中已完成 step 的 output         |
| `session` | 當前 agent 會話狀態（`messages` 等）             |
| `vars`    | `type: assign` 設定的變數                        |
| `config`  | agent config（`max_iterations` 等）              |

### 範例：以 foreach 實作 agentic loop

```yaml
steps:
  - type: foreach
    count: ${ config.max_iterations }
    do:
      - id: response
        type: task
        action: agent.llm_call
        input:
          messages: ${ session.messages }

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
          messages: ${ session.messages
            .append(steps.response.output.message)
            .concat(steps.tool_results.output.results) }

  - type: fail
    message: "max_iterations exceeded"
```

### 範例：固定程序（非 loop）

也可以寫成單次固定程序，不做迭代：

```yaml
steps:
  - id: classify
    type: task
    action: agent.llm_call
    input:
      messages:
        - { role: user, content: ${ input.text } }

  - type: return
    output:
      category: ${ steps.classify.output.message.content }
```

---

## Builtin Agent Actions

以下為引擎在 task registry 中預先註冊、供 agent 與 loop 使用的 builtin action。均以 `type: task` 呼叫。

| Action                    | 用途                                   |
| ------------------------- | -------------------------------------- |
| `agent.ask_human`         | 暫停 agent，向人類提問                 |
| `agent.llm_call`          | 呼叫 LLM（自訂 loop 使用）             |
| `agent.exec_tool_calls`   | 執行一組 tool calls                    |
| `agent.set_messages`      | 取代 `session.messages`                |
| `agent.activate_skill`    | 載入 skill 的 SKILL.md 指令            |
| `agent.read_skill_file`   | 讀取 skill 目錄中的檔案                |
| `artifact.list`           | 列出 workflow artifact 中繼資料與路徑  |
| `registry.search`         | 搜尋 action registry                   |
| `registry.list`           | 列出 registry 下的 action              |
| `registry.activate`       | 動態啟用 action（加入目前 agent 可用範圍） |

### agent.llm_call

呼叫 LLM 並回傳結果。output 結構固定，CEL 可直接存取 tool call 的名稱與參數。

```yaml
# Input
{
  messages: [ { role, content, ... } ],
  tools: [...]          # MAY — 覆寫本次可用的 tools
}

# Output
{
  message: { role: "assistant", content: "..." },  # LLM 回覆訊息
  tool_calls: [                                    # 本次要求呼叫的工具清單（空陣列表示無）
    {
      id: "call_abc123",      # 該次 tool call 的唯一 ID（回傳結果時用來對應）
      name: "order.load",     # 工具名稱（= task action name）
      arguments: {            # LLM 產生的引數（已解析為 map）
        order_id: "ord-001"
      }
    }
  ],
  has_tool_calls: true,       # 便利旗標，等同 size(tool_calls) > 0
  finish_reason: "tool_use",  # stop | tool_use | length | ...
  usage: { input_tokens, output_tokens }
}
```

CEL 存取範例：

```cel
steps.response.output.tool_calls[0].name           # "order.load"
steps.response.output.tool_calls[0].arguments      # { order_id: "ord-001" }
steps.response.output.tool_calls[0].arguments.order_id
steps.response.output.tool_calls.map(c, c.name)    # ["order.load", ...]
steps.response.output.tool_calls.exists(c, c.name == "agent.ask_human")
```

### agent.exec_tool_calls

執行一組 tool calls。input 接收 `agent.llm_call` 的 `tool_calls`，output 為對應的 tool result 陣列（順序與長度對齊 input）。

```yaml
# Input
{
  tool_calls: [ { id, name, arguments }, ... ]
}

# Output
{
  results: [
    {
      id: "call_abc123",        # 對應 input tool_call 的 id
      name: "order.load",
      success: true,
      output: { ... },          # 該 tool 的 output（success 時）
      error: null               # 或 { type, message } （failure 時）
    }
  ]
}
```

若某個 tool call FAILED，`results[i].success` 為 `false`，其他 tool 不受影響；是否繼續由自訂 steps 判斷。

### agent.ask_human

```yaml
# Input
{
  questions: [
    { id: "approve", question: "是否核准？", type: "confirm" }
  ]
}
# type: text | select | multi_select | confirm
```

---

## TODO

- Agent Definition 的生命週期 hooks 詳細規格
- 動態 system prompt 與 prompt 的使用範例
- 考慮 agent CEL 函數化（在 expression 層直接呼叫 agent）
