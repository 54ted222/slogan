# 18 — Toolset Definition（`kind: Toolset`）

本文件定義 `kind: Toolset` 的完整 YAML 結構、組合機制、合併規則與生命週期。Toolset 將 tools 與 skills 統一為具名的能力集合，是 agent 能力組裝的核心抽象。

---

## 1. 設計原則

1. **統一能力抽象** — Tools 和 skills 本質都是「agent 的能力」，Toolset 將兩者統一管理，使用者只需學習一套機制。
2. **取代 `kind: Tools`**（決策 G1）— `kind: Toolset` 完全取代 `kind: Tools`，避免兩個近似 kind 並存。
3. **僅在 Agent Definition 引用**（決策 G3）— Toolset 透過 `type: toolset` 在 agent definition 的 `tools` 陣列中引用，step 層級不可直接引用 toolset。
4. **Skills 必須透過 Toolset**（決策 G4）— Skills 不可在 agent definition 中獨立引用，必須包含在某個 toolset 內。

---

## 2. Top-level Schema

```yaml
apiVersion: toolset/v3
kind: Toolset

metadata:
  name: string               # MUST — 唯一識別名稱（kebab-case）
  version: integer            # MUST — 版本號，正整數，單調遞增
  description: string         # MAY — 人類可讀說明
  labels: map<string, string> # MAY — 分類鍵值對

tools: array                  # MAY — tool 列表
skills: array                 # MAY — skill 列表
includes: array               # MAY — 引用其他 toolset（組合）
```

### 2.1 tools 陣列

與 [14-agent-tools](14-agent-tools.md) 中定義的 tool 類型相同，支援 `task`、`workflow`、`agent`、`mcp`、`tool` 五種類型：

```yaml
tools:
  - type: task
    action: order.load
  - type: task
    action: order.update
  - type: mcp
    server: compliance-mcp
    tools: [check_sanctions_list]
  - type: tool
    name: agent.ask_human
```

### 2.2 skills 陣列

引用已註冊的 agentskills.io 知識包：

```yaml
skills:
  - name: risk-analysis
  - name: compliance-review
  - name: "compliance-*"       # 支援通配符
```

通配符規則與 [15-agent-skills](15-agent-skills.md) 相同。

### 2.3 includes 陣列

引用其他 toolset，實現能力組合：

```yaml
includes:
  - order-processing             # 引用另一個 kind: Toolset 的 name
  - customer-management
```

展開規則：

| 規則 | 說明 |
|------|------|
| 遞迴展開 | includes 中的 toolset 若含 includes，繼續遞迴展開 |
| 循環引用 | Validation error — 引擎 MUST 在載入時偵測並拒絕 |
| 版本解析 | 引用 name 時解析為最新 PUBLISHED 版本 |

---

## 3. Agent Definition 中的引用方式

根據決策 G2，toolset 透過 `type: toolset` 作為 tool 的一種引用方式，放置於 agent definition 的 `tools` 陣列中：

```yaml
# Agent Definition
apiVersion: agent/v3
kind: Agent

metadata:
  name: order.analyzer
  version: 1

tools:
  - type: toolset
    name: order-processing          # 引用 kind: Toolset
  - type: tool
    name: artifact.read             # 額外加入個別 tool
```

`type: toolset` 欄位：

```yaml
- type: toolset
  name: string                     # MUST — Toolset name（kebab-case）
  version: integer | "latest"      # MAY, 預設 "latest"
```

---

## 4. 合併規則

引入 toolset 後，agent 的最終能力列表由以下順序合併：

```
1. Agent Definition defaults — tools/skills 預設值
2. Toolset expansion         — 展開 type: toolset 引用（含遞迴 includes）
3. Step overrides            — step 層級的 tools/skills（增量合併）
4. tools_override            — 若指定，取代 1 + 2 + 3 的結果
5. tools_exclude             — 從最終結果移除指定 tool/skill
```

### 4.1 去重規則

| 類型 | 去重 key | 說明 |
|------|----------|------|
| task | `type` + `action` | 同一 action 出現多次，保留最後一個（後者覆寫前者的 name/description） |
| workflow | `type` + `workflow` | 同上 |
| agent | `type` + `agent` | 同上 |
| mcp | `type` + `server` + `tools` | server 相同且 tools 陣列相同時去重 |
| tool | `type` + `name` | 同一 name 出現多次，保留最後一個 |
| skill | `name` | 同一 name 出現多次去重 |

### 4.2 合併範例

```yaml
# kind: Toolset — order-processing
tools:
  - type: task
    action: order.load
  - type: task
    action: order.update
skills:
  - name: risk-analysis

# Agent Definition
tools:
  - type: toolset
    name: order-processing           # 展開為上方 toolset 的 tools + skills
  - type: tool
    name: artifact.read              # 額外加入

# Workflow step
- type: agent
  agent: order.analyzer
  tools_exclude:
    - type: task
      action: order.update           # 移除 order.update
```

最終能力列表：`order.load`（task）、`artifact.read`（tool）、`risk-analysis`（skill）。

---

## 5. 生命週期

與其他 definition 相同：

```
DRAFT → VALIDATED → PUBLISHED → DEPRECATED → ARCHIVED
```

| 狀態 | 說明 |
|------|------|
| DRAFT | 編輯中，不可被引用 |
| VALIDATED | 通過 schema 驗證 |
| PUBLISHED | 可被 agent definition 引用 |
| DEPRECATED | 仍可運作但不建議使用 |
| ARCHIVED | 不可被引用 |

---

## 6. 驗證規則

| 規則 | 說明 |
|------|------|
| 循環引用 | includes 形成循環 → validation error |
| 不存在的引用 | includes 引用不存在或非 PUBLISHED 的 toolset → validation error |
| 空 toolset | tools 與 skills 與 includes 皆為空 → validation warning |
| name 格式 | MUST 為 kebab-case |

---

## 7. 完整範例

### 7.1 基礎 Toolset

```yaml
apiVersion: toolset/v3
kind: Toolset

metadata:
  name: order-processing
  version: 1
  description: "訂單處理能力集合：包含訂單讀寫 task、合規 MCP 工具及風險分析 skill"
  labels:
    domain: order

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
  - type: tool
    name: agent.ask_human

skills:
  - name: order/risk-analysis
  - name: order/compliance-review
```

### 7.2 組合 Toolset（使用 includes）

```yaml
apiVersion: toolset/v3
kind: Toolset

metadata:
  name: full-order-suite
  version: 1
  description: "完整訂單能力套件：組合訂單處理與客戶管理，並加入欺詐偵測 agent"

includes:
  - order-processing                 # 引用 7.1 的 toolset
  - customer-management              # 引用另一個 toolset

tools:
  - type: agent
    agent: fraud.detector            # 額外加入 agent-as-tool

skills:
  - name: shared/escalation-procedures  # 額外加入 skill
```

### 7.3 Agent Definition 引用 Toolset

```yaml
apiVersion: agent/v3
kind: Agent

metadata:
  name: order.handler
  version: 1
  description: "訂單處理 agent，具備完整訂單能力套件"

model: smart

system_prompt: |
  你是訂單處理專員。根據客戶需求處理訂單相關事務，
  必要時諮詢人類主管。

tools:
  - type: toolset
    name: full-order-suite           # 引用組合 toolset
  - type: tool
    name: artifact.read              # 額外加入個別 tool

config:
  max_iterations: 30
  output_format: json
```
