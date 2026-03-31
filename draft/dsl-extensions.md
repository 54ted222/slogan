# DSL 擴充功能草稿

> **狀態**：草稿
> **範圍**：非 agent 專屬的 DSL 層級擴充功能，涵蓋資源宣告、模板系統、條件控制流與命名慣例

---

## 一、`kind: resources` — 共用資源宣告

將 MCP servers、string templates、skills、vars、artifacts 等共用資源統一宣告於獨立的 `kind: Resources` 定義檔，避免散落於各 workflow / agent definition 中。

### 設計動機

- 多個 workflow / agent 可能共用相同的 MCP server、skill、template
- 目前 MCP server 在引擎層級配置，skill 以路徑引用 — 缺乏統一的資源管理層
- 提供單一位置宣告與管理共用資源，提升可維護性

### 初步 Schema 構想

```yaml
apiVersion: resource/v2
kind: Resources

metadata:
  name: shared-resources
  version: 1

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

skills:
  risk-analysis:
    path: ./skills/risk-analysis
    description: "風險分析知識包"

  compliance-review:
    path: ./skills/compliance-review
    description: "合規審查知識包"

string_templates:
  greeting:
    template: "Hello, {name}! Today is {day}."
    input_schema:
      type: object
      properties:
        name: { type: string }
        day: { type: string }
      required: [name, day]

  risk-summary:
    template: |
      風險等級：{level}
      摘要：{summary}
      標記：{flags}
    input_schema:
      type: object
      properties:
        level: { type: string }
        summary: { type: string }
        flags: { type: string }
      required: [level, summary]
```

### 引用方式

其他 definition 以 name 引用資源，不需要知道實際路徑或配置細節：

```yaml
# agent definition 中引用 skill（by name，非 path）
skills:
  - name: risk-analysis
  - name: compliance-review

# agent definition 中引用 MCP server（已有此機制，不變）
tools:
  - type: mcp
    server: github-mcp
```

### 待決定

- `kind: Resources` 是否為獨立檔案，還是可嵌入 workflow/agent definition
- 資源的版本管理與覆寫機制
- 是否支援跨 namespace 引用

---

## 二、`kind: tools` — 工具集合宣告

將多個工具統一定義為具名集合，可被 agent definition 或 workflow step 引用，減少重複宣告。

### 設計動機

- 多個 agent 可能需要相同的工具組合（如「訂單處理工具集」）
- 避免在每個 agent/step 中重複列舉相同 tools
- 提供工具的邏輯分組與重用

### 初步 Schema 構想

```yaml
apiVersion: tool/v2
kind: Tools

metadata:
  name: order-tools
  version: 1
  description: "訂單處理相關工具集"

tools:
  - type: task
    action: order.load
  - type: task
    action: order.update
  - type: task
    action: customer.get_history
  - type: mcp
    server: compliance-mcp
    tools: [check_sanctions_list]
```

### 引用方式

```yaml
# agent definition 中引用工具集合
tools:
  - type: toolset
    name: order-tools           # 引用 kind: Tools 的 name
  - type: tool
    name: agent.ask_human       # 額外加入個別 tool
```

### 待決定

- `type: toolset` 是否為正確的引用方式，或採用其他語法
- 工具集合之間是否可組合（toolset 引用另一個 toolset）
- 與 step 層級 `tools_exclude` 的交互行為

---

## 三、String Template + Variable Substitution

定義可重用的字串模板，支援變數替換，主要用於 prompt 組裝。

### Template 定義

Template 在 `kind: Resources` 中宣告（見第一節），或作為 workflow/agent 內的 inline 定義：

```yaml
# 在 kind: Resources 中定義
string_templates:
  greeting:
    template: "Hello, {name}! Today is {day}."
    input_schema:
      type: object
      properties:
        name: { type: string }
        day: { type: string }
      required: [name, day]
```

### 使用方式

在 prompt 中透過 `use_template` 引用，支援 `override` 與 `append` 兩種模式：

```yaml
prompts:
  override:
    - use_template: "greeting"
      input:
        name: "Bob"
        day: "Tuesday"
    - "abcdef ${vars.name}"
  append:
    - use_template: "greeting"
      input:
        name: "Alice"
        day: "Monday"
    - "xyz ${vars.name}"
```

| 模式 | 行為 |
|------|------|
| `override` | 完全取代原有 prompt（agent definition 的 system_prompt 或預設 prompt） |
| `append` | 追加至原有 prompt 尾部 |

每個項目可以是：
- `use_template`：引用已定義的 string template，傳入 `input` 做變數替換
- 純字串：支援 `${ }` CEL 表達式內插

### 待決定

- template 中的變數定界符是否用 `{name}` 還是 `${ name }` 或其他格式（避免與 CEL 混淆）
- template 是否支援巢狀引用（template 引用另一個 template）
- `override` + `append` 是否可同時存在，語意為何

---

## 四、Model Alias

定義 model alias 機制，避免在多處硬編碼 model provider + name。

### 設計動機

- 同一個 model 可能被多個 agent 引用，升級 model 版本時需逐一修改
- 測試環境與生產環境可能使用不同 model
- 提供語意化名稱（如 `fast`、`smart`、`reasoning`），與實際 model 解耦

### 初步 Schema 構想

在引擎層級或 `kind: Resources` 中定義 alias：

```yaml
# 引擎層級配置 或 kind: Resources
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

### Agent 中使用

```yaml
# 方式一：使用 alias
model: fast                    # 引用 model alias

# 方式二：原有完整寫法（仍支援）
model:
  provider: anthropic
  name: claude-sonnet-4-20250514
  config:
    temperature: 0
```

### 待決定

- alias 定義的位置（引擎配置 vs `kind: Resources` vs 兩者皆可）
- 是否允許 agent definition 覆寫 alias 的部分 config（如只改 `max_tokens`）
- alias 名稱的命名規範

---

## 五、Tool 名稱兩段式命名

Tool 名稱統一採用 `namespace.action` 兩段式格式，以點號分隔。

### 設計動機

- 提供一致的命名結構，便於分類、搜尋與衝突避免
- 內建 tool 與自訂 tool 使用相同命名規範
- 符合現有 task action 的命名風格（如 `order.load`）

### 命名規則

| 類型 | 格式 | 範例 |
|------|------|------|
| 內建 tool | `agent.<action>` | `agent.ask_human`、`agent.activate_skill` |
| 內建 task | `agent.<action>` | `agent.llm_call`、`agent.exec_tool_calls`、`agent.set_messages` |
| History 操作 | `history.<action>` | `history.read`、`history.write`、`history.summarize` |
| Tool 倉庫 | `registry.<action>` | `registry.search`、`registry.list` |
| Task tool | `<namespace>.<action>` | `order.load`、`customer.get_history` |

### 對現有規格的影響

- `type: builtin` 移除，統一使用 `type: tool`
- 現有 `name: ask_human` 改為 `name: agent.ask_human`

```yaml
# 舊寫法（廢棄）
- type: builtin
  name: ask_human

# 新寫法
- type: tool
  name: agent.ask_human
```

---

## 六、`expr` 條件關鍵字調整

目前 `if` 和 `switch` step 中使用 `expr` 作為條件欄位名稱，語意不夠直觀。

### 現狀

```yaml
- type: if
  expr: ${ steps.load_order.output.status == "cancelled" }

- type: switch
  expr: ${ steps.load_order.output.status }
```

`expr` 是「expression」的縮寫，但作為條件判斷的欄位名，無法直接推斷「這是一個條件」。

### 替代方案

| 方案 | if 範例 | switch 範例 | 優點 | 缺點 |
|------|---------|------------|------|------|
| `condition` | `condition: ${ ... }` | `condition: ${ ... }` | 語意明確 | 較長 |
| `test` | `test: ${ ... }` | `test: ${ ... }` | 簡短、常見（shell test） | 不夠直觀 |
| `when` | `when: ${ ... }` | `when: ${ ... }` | 自然語言感 | 與獨立 `when` 功能衝突（見第七節） |
| 維持 `expr` | `expr: ${ ... }` | `expr: ${ ... }` | 不需遷移 | 原始問題未解 |

> **待決定**：是否值得為了語意改名而引入 breaking change

---

## 七、分析加入 `when` 的好壞

評估是否引入 `when` 作為通用條件守衛（guard），可附加在任意 step 上。

### 構想

```yaml
# when 作為 step 層級的條件守衛
- id: notify
  type: task
  action: notification.send
  when: ${ steps.analyze.output.risk_level == "high" }
  input:
    message: "高風險訂單"
```

等效於：

```yaml
- type: if
  expr: ${ steps.analyze.output.risk_level == "high" }
  then:
    - id: notify
      type: task
      action: notification.send
      input:
        message: "高風險訂單"
```

### 優點

- 語意直觀，貼近自然語言（「當...時執行此步驟」）
- 減少巢狀，單一 step 的條件判斷不需包一層 `if`
- 許多 workflow 引擎都支援類似語法（GitHub Actions `if:`、Ansible `when:`）

### 缺點

- 與 `if` step 的 `expr` 語意重疊，使用者可能困惑何時用 `when` 何時用 `if`
- 增加驗證複雜度（每個 step 都可能有 `when`）
- `when` 為 false 時的 step 狀態（SKIPPED）需與其他機制一致

### 建議區分

| 場景 | 推薦 |
|------|------|
| 單一 step 的條件執行 | `when`（簡潔） |
| if-then-else 分支 | `if` step（有 else） |
| 多值分支 | `switch` step |

> **待決定**：是否引入、以及與 `expr` 改名的交互影響

---

## 八、`switch` 加入 `between`

在 switch step 中支援 `between` 區間比對，用於數值範圍判斷。

### 構想

```yaml
- type: switch
  expr: ${ steps.analyze.output.score }
  cases:
    - between: [0, 30]          # 0 ≤ score ≤ 30
      then:
        - type: assign
          vars: { level: "low" }
    - between: [31, 70]         # 31 ≤ score ≤ 70
      then:
        - type: assign
          vars: { level: "medium" }
    - between: [71, 100]        # 71 ≤ score ≤ 100
      then:
        - type: assign
          vars: { level: "high" }
  default:
    - type: fail
      message: "score out of range"
```

### 語意

- `between: [min, max]` — 閉區間，`min ≤ expr ≤ max`
- `value` 與 `between` 在同一個 case 中互斥
- 同一個 switch 中可混用 `value` 和 `between`
- 區間重疊時匹配第一個符合的 case（與 `value` 行為一致）

### 待決定

- 是否支援開區間（`(0, 30)`、`[0, 30)`）
- 是否支援無上界/無下界（`>= 71`）
- 是否直接用 CEL 表達式更靈活（`expr: ${ score >= 0 && score <= 30 }`）

---

## 九、考慮加入 `router`

簡化版 switch，以 key → steps 映射簡化路由邏輯。

### 設計動機

許多場景只需根據一個 key 值分派到對應的處理邏輯，`switch` 的 `cases` + `value` + `then` 結構過於冗長。

### 構想

```yaml
# router — 簡潔的 key → steps 映射
- type: router
  key: ${ input.action }       # CEL 表達式，求值結果作為路由 key
  routes:
    create:                     # key == "create" 時
      - type: task
        action: order.create
        input: ${ input.data }
    update:                     # key == "update" 時
      - type: task
        action: order.update
        input: ${ input.data }
    delete:                     # key == "delete" 時
      - type: task
        action: order.delete
        input: { id: ${ input.id } }
  default:                      # 無匹配時
    - type: fail
      message: "unknown action: ${ input.action }"
```

### 與 switch 的等效對照

```yaml
# 等效 switch 寫法（較冗長）
- type: switch
  expr: ${ input.action }
  cases:
    - value: "create"
      then:
        - type: task
          action: order.create
          input: ${ input.data }
    - value: "update"
      then:
        - type: task
          action: order.update
          input: ${ input.data }
    - value: "delete"
      then:
        - type: task
          action: order.delete
          input: { id: ${ input.id } }
  default:
    - type: fail
      message: "unknown action: ${ input.action }"
```

### 差異定位

| 維度 | `router` | `switch` |
|------|----------|----------|
| 比對方式 | key 值直接映射（相等比對） | `value` 相等比對 + `between` 區間 |
| 語法 | `routes` map（key → steps） | `cases` array（value + then） |
| 適用場景 | 簡單的值分派 | 複雜條件分支 |
| 功能 | `switch` 的子集 | 完整功能 |

### 待決定

- 是否真的需要 `router`，還是改善 `switch` 語法即可
- `router` 是否支援 `between`（若支援則與 `switch` 重疊更多）
- `key` vs `expr` 命名

---

## 十、`exists` 檢核機制

提供判斷變數、物件、ID 是否存在的能力。

### 方案比較

#### 方案 A：CEL 函式

在 CEL 中新增 `exists()` 函式，最輕量的實作方式：

```yaml
- type: if
  expr: ${ exists(steps.load_order.output) }
  then: [...]

- type: if
  expr: ${ exists(vars.customer_id) }
  then: [...]
```

#### 方案 B：`has()` 巨集（CEL 原生）

CEL 原生支援 `has()` 巨集，用於檢查 map 中的 key 是否存在：

```yaml
- type: if
  expr: ${ has(steps.load_order.output.address) }
  then: [...]
```

`has()` 的限制：只能檢查 map field 是否存在，不能檢查「一個 step 是否已執行」或「一個變數是否已被 assign」。

#### 方案 C：擴充 namespace 預設值

讓未執行的 step output 和未 assign 的 vars 回傳 `null`，透過 `!= null` 判斷：

```yaml
- type: if
  expr: ${ steps.load_order.output != null }
  then: [...]
```

### 建議

- 優先採用 CEL 原生 `has()`，不額外發明語法
- 對於「step 是否已執行」，提供 `steps.<id>.status` 欄位（值為 `SUCCEEDED`、`FAILED`、`SKIPPED`、`null`）

> **待決定**：最終選用方案，以及是否需要額外的 `exists` step 類型
