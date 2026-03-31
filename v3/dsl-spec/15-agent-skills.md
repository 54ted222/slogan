# 15 — Agent Skills（agentskills.io）

Agent skills 遵循 [agentskills.io](https://agentskills.io/) 規格，是提供專業知識、行為指引與可執行腳本的知識包。Skills 與 tools 互補：skill 指引 agent 如何思考與行動，tool 提供實際執行能力。

---

## 概觀

Skills 不是可呼叫的 function，而是注入 agent context 的知識單元。每個 skill 包含：

- **指引（Instructions）**：告訴 agent 在特定領域應如何判斷與行動
- **參考資料（References）**：領域知識文件、範例、規範
- **可執行腳本（Scripts）**：agent 可按指引呼叫的腳本
- **資產（Assets）**：模板、範本等靜態資源

---

## 配置

### Name-based 引用

```yaml
skills:
  - name: risk-analysis        # 指定單一 skill
  - name: compliance-review
  - name: "compliance-*"       # 通配符：載入所有符合 pattern 的 skills
  - name: "*"                  # 載入所有可用的 skills
```

### 通配符規則

| 規則 | 說明 |
|------|------|
| `*` | 匹配零或多個字元（不含 `/`） |
| 展開時機 | 引擎啟動時展開為符合 pattern 的所有已註冊 skill name |
| 合併 | 展開結果與明確指定的 skill 合併（去重） |
| 未匹配 | 通配符未匹配到任何 skill 時，不產生錯誤（靜默忽略） |

---

## Skills 必須透過 Toolset 引用

> **決策 G4**：skill 不可在 agent definition 中直接引用，必須透過 `kind: Toolset` 組裝。

Skills 作為 agent 能力的一部分，應與 tools 一同透過 toolset 管理。直接在 agent definition 的 `skills` 欄位引用 skill 的方式已 **deprecated**。

### 正確做法：透過 `type: toolset` 引用

```yaml
# kind: Toolset — 將 tools + skills 組合為具名集合
apiVersion: toolset/v3
kind: Toolset

metadata:
  name: order-analysis-suite
  version: 1
  description: "訂單分析能力集合"

tools:
  - type: task
    action: order.load
  - type: task
    action: customer.get_history
  - type: tool
    name: agent.ask_human

skills:
  - name: risk-analysis
  - name: compliance-review
```

```yaml
# Agent definition 透過 toolset 引用
tools:
  - type: toolset
    name: order-analysis-suite    # 引用 kind: Toolset，包含 tools + skills
  - type: tool
    name: artifact.read           # 可額外加入個別 tool
```

### 已 deprecated 的直接引用

```yaml
# ❌ deprecated — 不應直接在 agent definition 中引用 skills
skills:
  - name: risk-analysis
  - name: compliance-review
```

> 引擎仍接受直接引用（向後相容），但會發出 deprecation warning。

---

## Skill 目錄結構

每個 skill 遵循 agentskills.io 標準目錄結構：

```
risk-analysis/
  SKILL.md          # MUST — metadata + instructions（frontmatter + body）
  scripts/          # MAY — 可執行腳本
  references/       # MAY — 參考文件（PDF、Markdown 等）
  assets/           # MAY — 模板、範本、靜態資源
```

### SKILL.md 結構

```markdown
---
name: risk-analysis
description: "訂單風險分析指引，包含評估框架與判斷標準"
---

# Risk Analysis

## 評估框架
...（完整的 skill instructions）
```

- **Frontmatter**：`name`（MUST）、`description`（MUST）— 用於 discovery 注入
- **Body**：完整的指引內容 — 透過 `agent.activate_skill` 按需載入

### Skills 專案資料夾

> **決策 F1–F4**：skill 可透過 project 資料夾組織。

當 skill 數量增多時，使用 `kind: SkillProject` 的 `project.yaml` 組織：

```
skills/
  order/                         # project: order
    project.yaml                 # MUST — 無此檔案則不視為 project
    risk-analysis/
      SKILL.md
    fraud-detection/
      SKILL.md
  customer/                      # project: customer
    project.yaml
    sentiment-analysis/
      SKILL.md
```

- **Project prefix 自動加入**：`risk-analysis` 位於 `order/` 下，name 自動成為 `order/risk-analysis`（決策 F4）
- **巢狀 project**：支援多層巢狀（如 `order/domestic/risk-analysis`）（決策 F2）
- **`project.yaml` 必須存在**：無此檔案的資料夾不視為 project（決策 F3）

---

## 引擎行為

Agent session 啟動時，引擎依序執行以下步驟處理 skills：

### 1. 啟動時載入 metadata

讀取每個 skill 的 `SKILL.md` frontmatter（`name`、`description`），建立 skill registry。

### 2. Discovery 注入

將 agent 可用的所有 skills 的 name + description 注入 system prompt，讓 LLM 知道有哪些 skill 可用：

```
你可以使用以下 skills：
- risk-analysis：訂單風險分析指引，包含評估框架與判斷標準
- compliance-review：合規審查流程與法規要求
使用 activate_skill(name) 載入完整指引。
```

### 3. 按需啟用（On-demand Activation）

LLM 透過內建 tool `agent.activate_skill(name)` 載入完整的 SKILL.md body 到 context。啟用後 skill 的完整指引成為對話的一部分。

### 4. Progressive Disclosure

Skill 內的 `scripts/`、`references/`、`assets/` 檔案透過 `agent.read_skill_file` 按需讀取，避免一次性載入過多內容。

```yaml
# 內建 tool 呼叫示例
agent.activate_skill({ name: "risk-analysis" })
agent.read_skill_file({ skill: "risk-analysis", path: "references/risk-matrix.md" })
```

---

## 與 MCP Tools 的差異

| 維度 | Agent Skills | MCP Tools |
|------|-------------|-----------|
| 本質 | 知識 + 指令 + 腳本 | 可程式化呼叫的工具 |
| 載入方式 | 注入 system prompt / context | 轉為 function call schema |
| 執行方式 | Agent 按指引自行決定行動 | Agent 呼叫 → 引擎代為執行 |
| 互動模式 | Discovery → 按需啟用 → progressive disclosure | Session 初始化時拉取 tool schema |
| 互補關係 | Skill 指引 agent 如何使用 tools | Tool 提供 skill 指引的執行能力 |

---

## 版本管理

> **決策 M6**：skills 版本管理 v1 不做，延後處理。

v1 階段 skills 僅支援本地路徑引用，不支援版本號或遠端 registry。Skill 的更新透過直接修改本地檔案完成。

未來版本規劃：

- Skill registry 與版本號
- 遠端 skill 引用
- Skill 版本鎖定與相容性檢查

---

## 相關文件

- [01-kind-definitions](01-kind-definitions.md) — `kind: Toolset` 定義
- [05-steps-overview](05-steps-overview.md) — step 類型總覽
- [16-agent-loop](16-agent-loop.md) — Agent step schema 與能力合併規則
