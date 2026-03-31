# Agent 完整範例

> 本文件以一個「訂單風險分析」場景，展示 agent 從宣告到使用的完整流程。
> 涵蓋概念：agent definition、model alias、skills by name、tool 兩段式命名、
> ask_human 多問題、artifact 讀寫、agent-as-tool、tools_exclude、stream、hook、session ID。

---

## 1. Resources 宣告（`kind: Resources`）

```yaml
apiVersion: resource/v2
kind: Resources

metadata:
  name: order-resources
  version: 1

model_aliases:
  fast:
    provider: anthropic
    name: claude-haiku-4-5-20251001
    config: { temperature: 0, max_tokens: 2048 }
  smart:
    provider: anthropic
    name: claude-sonnet-4-20250514
    config: { temperature: 0, max_tokens: 4096 }

skills:
  risk-analysis:
    path: ./skills/risk-analysis
    description: "風險分析知識包"

mcp_servers:
  compliance-mcp:
    transport: stdio
    command: "python3 ./mcp-servers/compliance/main.py"

string_templates:
  review-prompt:
    template: |
      請審核訂單 {order_id}，客戶等級為 {tier}。
      重點關注金額異常與歷史模式。
    input_schema:
      type: object
      properties:
        order_id: { type: string }
        tier: { type: string }
      required: [order_id]
```

---

## 2. Agent Definition（`kind: Agent`）

```yaml
apiVersion: agent/v2
kind: Agent

metadata:
  name: order.analyzer
  version: 1
  description: "訂單風險分析專家"

model: smart                          # 引用 model alias

system_prompt: |
  你是訂單風險分析專家。根據訂單資料與客戶歷史，
  評估風險等級並產生 JSON 報告。

input_schema:
  type: object
  properties:
    order_id: { type: string }
  required: [order_id]

output_schema:
  type: object
  properties:
    risk_level: { type: string, enum: [low, medium, high] }
    summary: { type: string }
  required: [risk_level, summary]

tools:
  - type: task
    action: order.load
  - type: task
    action: customer.get_history
  - type: tool
    name: agent.ask_human             # 兩段式命名
  - type: tool
    name: artifact.read
  - type: tool
    name: artifact.write

skills:
  - name: risk-analysis               # by name，非 path

config:
  max_iterations: 10
  output_format: json
  stream: iteration                   # iteration-level stream

hooks:
  on_before_tool_call:
    - type: task
      action: audit.log
      input:
        session_id: ${ session.id }   # session ID
        tool: ${ hook.tool_name }
```

---

## 3. 子 Agent Definition（agent-as-tool 用）

```yaml
apiVersion: agent/v2
kind: Agent

metadata:
  name: fraud.detector
  version: 1
  description: "深度詐騙偵測，分析交易模式與異常行為"

model: fast

system_prompt: "你是詐騙偵測專家。分析交易模式，回傳 JSON 結果。"

input_schema:
  type: object
  properties:
    order_id: { type: string }
    amount: { type: number }
  required: [order_id]

output_schema:
  type: object
  properties:
    is_fraud: { type: boolean }
    confidence: { type: number }
    reason: { type: string }
  required: [is_fraud, confidence]

tools:
  - type: mcp
    server: compliance-mcp
    tools: [check_sanctions_list]

config:
  max_iterations: 5
  output_format: json
```

---

## 4. Workflow — 使用 Agent Step

```yaml
apiVersion: workflow/v2
kind: Workflow

metadata:
  name: order_review
  version: 1

input_schema:
  type: object
  properties:
    order_id: { type: string }
  required: [order_id]

artifacts:
  raw_order:
    kind: data
    lifecycle: intermediate
  review_report:
    kind: data
    lifecycle: output

steps:
  # 1. 載入訂單資料存入 artifact
  - id: load
    type: task
    action: order.load
    input: { order_id: ${ input.order_id } }

  - type: task
    action: artifact.write
    input:
      name: raw_order
      content: ${ steps.load.output }

  # 2. Agent 分析 — 使用 prompt template、合併 agent-as-tool、排除 ask_human
  - id: analyze
    type: agent
    agent: order.analyzer
    prompt:
      use_template: review-prompt
      input:
        order_id: ${ input.order_id }
        tier: ${ steps.load.output.customer_tier }
    input:
      order_id: ${ input.order_id }
    tools:                                # 合併額外 tool
      - type: agent
        agent: fraud.detector             # agent-as-tool
    tools_exclude: [agent.ask_human]      # 自動流程不需人工介入
    config:
      persist_history: none
    timeout: 2m

  # 3. 根據風險等級決定後續
  - type: switch
    expr: ${ steps.analyze.output.risk_level }
    cases:
      - value: high
        then:
          # 高風險 → 人工審核 agent（保留 ask_human）
          - id: manual
            type: agent
            agent: order.analyzer
            prompt: |
              深度審核訂單 ${ input.order_id }。
              必須透過 agent.ask_human 向審核人員確認。
            input:
              order_id: ${ input.order_id }
            system_prompt_append: "審核模式：高風險案例，務必謹慎。"
            config:
              persist_history: full
              stream: token               # 即時串流給審核 UI
            timeout: 10m

          - type: task
            action: artifact.write
            input:
              name: review_report
              content: ${ steps.manual.output }

    default:
      - type: task
        action: artifact.write
        input:
          name: review_report
          content: ${ steps.analyze.output }

  # 4. 回傳結果
  - type: return
    output:
      order_id: ${ input.order_id }
      report: ${ artifacts.review_report.content }
```

---

## 概念索引

| 概念 | 出現位置 |
|------|---------|
| Model alias (`fast`, `smart`) | Resources §1, Agent §2 §3 |
| Skills by name | Agent §2 `skills` |
| Tool 兩段式命名 (`agent.ask_human`) | Agent §2 `tools`, Workflow §4 `tools_exclude` |
| `ask_human` 多問題格式 | Agent §2 配置，由 LLM 在 agentic loop 中呼叫 |
| Artifact 讀寫 (`artifact.read/write`) | Agent §2 `tools`, Workflow §4 steps |
| Agent-as-tool | Workflow §4 step `tools` 中引用 `fraud.detector` |
| `tools_exclude` | Workflow §4 自動分析 step |
| `tools` 合併 | Workflow §4 在 definition 預設上加入 `fraud.detector` |
| `system_prompt_append` | Workflow §4 高風險人工審核 step |
| Stream (`iteration`, `token`) | Agent §2 `config`, Workflow §4 高風險 step |
| Hook (`on_before_tool_call`) | Agent §2 `hooks` |
| Session ID (`session.id`) | Agent §2 hook 中使用 |
| String template (`use_template`) | Workflow §4 analyze step `prompt` |
| `persist_history` 差異 | Workflow §4 自動 (`none`) vs 人工 (`full`) |
| `config` 覆寫 | Workflow §4 step 層級覆寫 agent definition 預設 |
