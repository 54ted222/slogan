# 19 — Resources Definition（`kind: Resources`）

本文件定義 `kind: Resources` 的完整 YAML 結構，涵蓋三種共用資源類型：MCP server 配置、字串模板、model alias。Resources 將散落於各處的共用資源統一宣告於獨立檔案，提升可維護性。

---

## 1. 設計原則

1. **獨立檔案**（決策 H1）— `kind: Resources` MUST 定義於獨立 YAML 檔案，不可嵌入 workflow 或 agent definition。
2. **不支援跨 namespace 引用**（決策 H2）— Resources 僅在同一 namespace 內可見。
3. **統一資源管理** — MCP servers、string templates、model aliases 集中宣告，其他 definition 以 name 引用。

---

## 2. Top-level Schema

```yaml
apiVersion: resource/v3
kind: Resources

metadata:
  name: string               # MUST — 唯一識別名稱（kebab-case）
  version: integer            # MUST — 版本號，正整數，單調遞增
  description: string         # MAY — 人類可讀說明
  labels: map<string, string> # MAY — 分類鍵值對

mcp_servers: map              # MAY — MCP server 配置
string_templates: map         # MAY — 可重用字串模板
model_aliases: map            # MAY — model 語意化別名
skills: map                   # MAY — skill 引用（供 toolset 消費）
```

---

## 3. mcp_servers — MCP Server 配置

宣告外部 MCP server 的連線資訊，供 agent tools 中 `type: mcp` 引用。

### Schema

```yaml
mcp_servers:
  <server-name>:               # string — server 唯一名稱
    transport: stdio | sse     # MUST — 傳輸方式
    command: string            # COND — stdio 時 MUST，啟動命令
    url: string                # COND — sse 時 MUST，SSE endpoint URL
    env: map<string, string>   # MAY — 環境變數（支援 ${ } CEL 表達式）
    headers: map<string, string> # MAY — HTTP headers（sse 時使用）
    config: map                # MAY — 額外配置參數
```

### 範例

```yaml
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
    command: "python -m compliance_mcp_server"
    env:
      DB_URL: ${ secret.COMPLIANCE_DB_URL }
    config:
      timeout_seconds: 30
```

### 引用方式

Agent tools 中以 server name 引用：

```yaml
tools:
  - type: mcp
    server: github-mcp          # 引用 mcp_servers 中的名稱
    tools: [create_issue, list_repos]
```

---

## 4. string_templates — 可重用字串模板

定義可重用的字串模板，主要用於 prompt 組裝。

### Schema

```yaml
string_templates:
  <template-name>:             # string — template 唯一名稱
    template: string           # MUST — 模板內容，支援 ${ name } 變數替換
    input_schema: object       # MAY — 輸入參數的 JSON Schema
```

### 變數定界符

根據決策 I1，template 中的變數使用 `${ name }` 定界符，與 CEL 表達式統一：

```yaml
string_templates:
  greeting:
    template: "Hello, ${ name }! Today is ${ day }."
    input_schema:
      type: object
      properties:
        name: { type: string }
        day: { type: string }
      required: [name, day]

  risk-summary:
    template: |
      風險等級：${ level }
      摘要：${ summary }
      標記：${ flags }
    input_schema:
      type: object
      properties:
        level: { type: string }
        summary: { type: string }
        flags: { type: string }
      required: [level, summary]
```

### 巢狀限制

根據決策 I2，template **不支援**巢狀引用（template 引用另一個 template）。所有 template MUST 為獨立單元。

### 在 Prompts 中使用

Template 透過 `use_template` 在 prompts 中引用，支援 `override` 與 `append` 兩種模式：

```yaml
prompts:
  override:
    - use_template: greeting
      input:
        name: "Bob"
        day: "Tuesday"
    - "額外指令：${ vars.instruction }"
  append:
    - use_template: risk-summary
      input:
        level: ${ steps.analyze.output.level }
        summary: ${ steps.analyze.output.summary }
        flags: ${ steps.analyze.output.flags }
    - "請根據以上摘要做出判斷。"
```

| 模式 | 行為 |
|------|------|
| `override` | 完全取代原有 prompt（agent definition 的 system_prompt 或預設 prompt） |
| `append` | 追加至原有 prompt 尾部 |

根據決策 I3，`override` 與 `append` 可同時存在。語意為：先以 `override` 取代原有 prompt，再以 `append` 追加。每個模式內的項目可以是 `use_template`（引用 template 並傳入 input）或純字串（支援 `${ }` CEL 表達式內插）。

---

## 5. model_aliases — Model 語意化別名

定義語意化的 model 別名，避免在多處硬編碼 provider + name。

### Schema

```yaml
model_aliases:
  <alias-name>:                # string — alias 名稱（如 fast、smart、reasoning）
    provider: string           # MUST — LLM provider（如 anthropic、openai）
    name: string               # MUST — model 名稱
    config: map                # MAY — temperature、max_tokens 等預設配置
```

### 定義位置

根據決策 J1，model aliases 可在兩處定義：

| 位置 | 說明 | 優先權 |
|------|------|--------|
| 引擎配置 | 全域預設，適用所有 namespace | 低 |
| `kind: Resources` | namespace 層級定義 | 高（覆寫引擎配置） |

同一 alias name 在兩處都有定義時，`kind: Resources` 中的定義優先。

### Agent 部分覆寫

根據決策 J2，Agent Definition MAY 在 `model.config` 中部分覆寫 alias 的預設配置：

```yaml
# Agent Definition
model:
  alias: fast               # 引用 alias
  config:
    max_tokens: 8192         # 僅覆寫 max_tokens，其餘沿用 alias 預設
```

### 範例

```yaml
model_aliases:
  fast:
    provider: anthropic
    name: claude-haiku-4-5-20251001
    config:
      temperature: 0
      max_tokens: 2048

  smart:
    provider: anthropic
    name: claude-sonnet-4-20250514
    config:
      temperature: 0
      max_tokens: 4096

  reasoning:
    provider: anthropic
    name: claude-opus-4-6
    config:
      temperature: 0
      max_tokens: 8192
```

---

## 6. skills — Skill 引用

Resources 可宣告 skill 引用資訊，供 toolset 消費：

```yaml
skills:
  risk-analysis:
    path: ./skills/risk-analysis
    description: "風險分析知識包"

  compliance-review:
    path: ./skills/compliance-review
    description: "合規審查知識包"
```

Toolset 以 skill name 引用（詳見 [18-toolset-definition](18-toolset-definition.md)），不需要知道實際路徑。

---

## 7. 驗證規則

| 規則 | 說明 |
|------|------|
| 獨立檔案 | MUST 為獨立 YAML 檔案，不可嵌入其他 definition |
| mcp_servers transport | stdio 時 MUST 有 `command`；sse 時 MUST 有 `url` |
| string_templates 巢狀 | template 內容 MUST NOT 包含 `use_template` 引用 |
| model_aliases name | 建議使用語意化名稱（fast、smart、reasoning 等） |
| 跨 namespace | MUST NOT 引用其他 namespace 的 Resources |

---

## 8. 完整範例

```yaml
apiVersion: resource/v3
kind: Resources

metadata:
  name: shared-resources
  version: 1
  description: "共用資源定義：MCP servers、字串模板與 model aliases"
  labels:
    env: production

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
    command: "python -m compliance_mcp_server"
    env:
      DB_URL: ${ secret.COMPLIANCE_DB_URL }
    config:
      timeout_seconds: 30

string_templates:
  greeting:
    template: "Hello, ${ name }! Today is ${ day }."
    input_schema:
      type: object
      properties:
        name: { type: string }
        day: { type: string }
      required: [name, day]

  risk-summary:
    template: |
      風險等級：${ level }
      摘要：${ summary }
      建議行動：${ action }
    input_schema:
      type: object
      properties:
        level: { type: string }
        summary: { type: string }
        action: { type: string }
      required: [level, summary]

  order-context:
    template: |
      訂單編號：${ order_id }
      客戶：${ customer_name }
      金額：${ amount }
      狀態：${ status }
    input_schema:
      type: object
      properties:
        order_id: { type: string }
        customer_name: { type: string }
        amount: { type: number }
        status: { type: string }
      required: [order_id, customer_name]

model_aliases:
  fast:
    provider: anthropic
    name: claude-haiku-4-5-20251001
    config:
      temperature: 0
      max_tokens: 2048

  smart:
    provider: anthropic
    name: claude-sonnet-4-20250514
    config:
      temperature: 0
      max_tokens: 4096

  reasoning:
    provider: anthropic
    name: claude-opus-4-6
    config:
      temperature: 0
      max_tokens: 8192

skills:
  risk-analysis:
    path: ./skills/order/risk-analysis
    description: "風險分析知識包"

  compliance-review:
    path: ./skills/order/compliance-review
    description: "合規審查知識包"

  sentiment-analysis:
    path: ./skills/customer/sentiment-analysis
    description: "客戶情緒分析知識包"
```
