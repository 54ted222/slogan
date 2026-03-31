# Toolset 概念

> **狀態**：草稿
> **範圍**：將 tools 與 skills 統一為 toolset 抽象，skill 視同 tool 的一種，簡化能力組裝與合併規則

---

## 設計動機

目前 agent 的能力由兩個獨立機制組裝：

- `tools`：task、workflow、agent、mcp、builtin 等可呼叫工具
- `skills`：agentskills.io 知識包（注入 context）

兩者在 agent definition 和 step 中分開宣告、分開合併，造成：

- 配置冗長：需要分別管理 `tools`、`tools_override`、`tools_exclude`、`skills`
- 概念分裂：tools 和 skills 本質都是「agent 的能力」，使用者需學習兩套機制
- 組裝困難：常見場景是「給 agent 一整套能力」，需要同時配 tools + skills

---

## Toolset 設計

### `kind: Toolset` — 具名能力集合

將 tools 和 skills 統一定義為一個具名集合：

```yaml
apiVersion: toolset/v2
kind: Toolset

metadata:
  name: order-processing
  version: 1
  description: "訂單處理能力集合"

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
  - name: risk-analysis
  - name: compliance-review
```

### Agent Definition 引用 Toolset

```yaml
# 方案 A：toolset 作為獨立欄位
tools:
  - type: toolset
    name: order-processing          # 引用 kind: Toolset
  - type: tool
    name: artifact.read             # 額外加入個別 tool

skills:
  - name: general-knowledge         # 額外加入個別 skill

# 方案 B：統一為 capabilities
capabilities:
  - toolset: order-processing       # 引用整個 toolset
  - tool:
      type: tool
      name: artifact.read           # 個別 tool
  - skill:
      name: general-knowledge       # 個別 skill
```

### Toolset 組合

Toolset 可引用其他 toolset，實現能力組合：

```yaml
apiVersion: toolset/v2
kind: Toolset

metadata:
  name: full-order-suite
  version: 1

includes:
  - order-processing                # 引用另一個 toolset
  - customer-management

tools:
  - type: agent
    agent: fraud.detector           # 額外加入

skills:
  - name: escalation-procedures     # 額外加入
```

---

## 合併規則

引入 toolset 後，能力的最終列表由以下順序合併：

```
1. Agent Definition 的 tools/skills（預設值）
2. Agent Definition 引用的 toolset（展開）
3. Step 層級的 tools/skills（增量）
4. Step 層級的 tools_override（若有，取代 1+2+3）
5. Step 層級的 tools_exclude（從最終結果移除）
```

合併策略：

| 操作 | 行為 |
|------|------|
| 相同 tool 出現多次 | 去重（by type + action/name） |
| 相同 skill 出現多次 | 去重（by name） |
| toolset 循環引用 | Validation error |

---

## 與 `kind: Tools`（dsl-extensions）的關係

`dsl-extensions.md` 第二節已定義 `kind: Tools`（工具集合宣告）。Toolset 是其擴展：

| 維度 | `kind: Tools` | `kind: Toolset` |
|------|---------------|-----------------|
| 包含 | 僅 tools | tools + skills |
| 組合 | 未定義 | `includes` 支援引用其他 toolset |
| 定位 | 工具分組 | 完整能力集合 |

**建議**：若採用 Toolset 概念，將 `kind: Tools` 合併進 `kind: Toolset`，避免兩個近似的 kind。

---

## 待決定

- 是否將 `kind: Tools` 升級為 `kind: Toolset`（統一） 或保持兩者共存
- 方案 A（toolset 作為 tool type）vs 方案 B（capabilities 統一欄位）
- Toolset 組合的深度限制
- Toolset 是否有獨立的版本管理
- Step 層級是否可直接引用 toolset（目前僅 agent definition 可以）
- Skill 是否需要在 toolset 外仍可獨立引用（向後相容）
