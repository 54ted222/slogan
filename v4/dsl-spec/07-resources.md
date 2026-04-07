# 07 — Resources Definition（`kind: Resources`）

本文件定義 `kind: Resources` 的 YAML 結構。Resources 集中宣告共用資源：MCP servers、string templates、model aliases、toolsets。

v4 將 v3 的獨立 `kind: Toolset` 併入 Resources 的 `toolsets` 屬性。

---

## 頂層結構

```yaml
apiVersion: resource/v4
kind: Resources

metadata:
  name: shared-resources        # MUST — kebab-case
  version: 1
  description: "共用資源定義"

mcp_servers: { ... }            # MAY — MCP server 配置
string_templates: { ... }       # MAY — 可重用字串模板
model_aliases: { ... }          # MAY — model 語意化別名
toolsets: { ... }               # MAY — 具名的 tools + skills 集合（v3 的 kind: Toolset）
```

---

## mcp_servers

宣告外部 MCP server 的連線資訊，供 agent tools 中 `type: mcp` 引用。

```yaml
mcp_servers:
  github-mcp:
    transport: stdio                          # MUST — stdio | sse
    command: "npx -y @modelcontextprotocol/server-github"  # MUST (stdio)
    env:
      GITHUB_TOKEN: ${ secret.GITHUB_TOKEN }

  slack-mcp:
    transport: sse
    url: "https://mcp.slack.example.com/sse"  # MUST (sse)
    headers:
      Authorization: "Bearer ${ secret.SLACK_TOKEN }"

  compliance-mcp:
    transport: stdio
    command: "python -m compliance_mcp_server"
    config:
      timeout_seconds: 30
```

| 欄位 | 型別 | 說明 |
|------|------|------|
| `transport` | string | MUST — `stdio` 或 `sse` |
| `command` | string | stdio 時 MUST — 啟動命令 |
| `url` | string | sse 時 MUST — endpoint URL |
| `env` | map | MAY — 環境變數（支援 `${ }`） |
| `headers` | map | MAY — HTTP headers（sse） |
| `config` | map | MAY — 額外設定 |

---

## string_templates

可重用的字串模板，支援 `${ }` 變數替換，主要用於 prompt 組裝。

```yaml
string_templates:
  order_analysis_prompt:
    template: |
      你是訂單分析專家。
      當前訂單：${ input.order_id }
      客戶等級：${ vars.customer_tier }
    description: "訂單分析 prompt 模板"

  risk_warning:
    template: "訂單金額 ${ input.amount } 超過閾值"
```

在 agent 的 `system_prompt` 或 `prompt` 中引用：

```yaml
system_prompt: ${ templates.order_analysis_prompt }
```

---

## model_aliases

語意化的 LLM model 別名，與實際 provider/model 解耦。

```yaml
model_aliases:
  fast:
    provider: anthropic
    name: claude-haiku-4-5-20251001
    config:
      max_tokens: 4096
      temperature: 0.3

  smart:
    provider: anthropic
    name: claude-sonnet-4-20250514
    config:
      max_tokens: 8192

  reasoning:
    provider: openai
    name: o1
```

Agent definition 中以 alias 引用：

```yaml
model: fast
```

---

## toolsets

具名的 tools + skills 集合（v3 的 `kind: Toolset` 併入此處）。Agent definition 透過 toolset name 引用整組能力。

```yaml
toolsets:
  order-processing:
    description: "訂單處理工具集"
    tools:
      - type: task
        action: order.load
      - type: task
        action: order.update
      - type: mcp
        server: compliance-mcp
        tools: [check_sanctions_list]
    skills:
      - name: risk-analysis
      - name: compliance-review

  customer-management:
    description: "客戶管理工具集"
    tools:
      - type: task
        action: customer.get_history
      - type: task
        action: customer.update_tier

  full-order-suite:
    description: "完整訂單套件"
    includes:                    # 組合其他 toolset
      - order-processing
      - customer-management
    tools:                       # 額外 tools
      - type: tool
        name: agent.ask_human
```

### toolset 欄位

| 欄位 | 說明 |
|------|------|
| `description` | MAY — 說明 |
| `tools` | MAY — tool 列表（同 agent tools 語法） |
| `skills` | MAY — skill 引用列表 |
| `includes` | MAY — 引用其他 toolset（遞迴展開，不可循環） |

### 在 Agent Definition 中引用

```yaml
# Agent Definition
tools:
  - type: toolset
    name: order-processing
```

---

## 完整範例

```yaml
apiVersion: resource/v4
kind: Resources

metadata:
  name: order-resources
  version: 1
  description: "訂單領域共用資源"

mcp_servers:
  compliance-mcp:
    transport: stdio
    command: "python -m compliance_server"
    env:
      DB_URL: ${ secret.COMPLIANCE_DB_URL }

model_aliases:
  fast:
    provider: anthropic
    name: claude-haiku-4-5-20251001
    config: { max_tokens: 4096 }
  smart:
    provider: anthropic
    name: claude-sonnet-4-20250514

string_templates:
  order_system_prompt:
    template: |
      你是訂單處理助手。
      可用工具：order.load、order.update

toolsets:
  order-tools:
    tools:
      - type: task
        action: order.load
      - type: task
        action: order.update
      - type: mcp
        server: compliance-mcp
    skills:
      - name: risk-analysis
```
