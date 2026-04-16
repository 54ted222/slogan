# v5 → v6 變更摘要

v6 的目標是**精簡至第一個實作版本**：移除 `Agent` 與 `Resources` kind 及其相關功能，聚焦於 Workflow、Tool、Function、Project、Secret 核心。所有 `apiVersion` 從 `*/v5` 升為 `*/v6`。

---

## 移除項目

| # | 項目 | 說明 |
|---|------|------|
| R1 | `kind: Agent` | 移除整個 Agent definition（`06-agent.md`） |
| R2 | `kind: Resources` | 移除整個 Resources definition（`07-resources.md`）：MCP servers、string templates、model aliases、toolsets |
| R3 | `type: agent` step | 移除 agent step 類型 |
| R4 | `session` namespace | 移除 agent session 相關求值上下文 |
| R5 | Agent 相關 builtin tools | `agent.ask_human`、`agent.llm_call`、`agent.exec_tool_calls`、`agent.set_messages` 等 |
| R6 | Toolset 引用語法 | `toolset.<name>` 命名空間字串不再存在 |
| R7 | MCP server tool 引用 | `<server>.*` / `<server>.<tool>` wildcard 引用不再存在 |

---

## 保留項目

以下 v5 功能在 v6 完整保留：

- `kind: Workflow` — 工作流程定義、triggers、config
- `kind: Tool` — 工具定義與 exec / http / extension backend
- `kind: Function` — 可重用邏輯單元
- `kind: Project` — 專案定義
- `kind: Secret` — 加密機密
- 所有 step 類型（task、assign、if、switch、foreach、parallel、emit、wait、fail、return、saga）
- CEL 表達式與所有 namespace（除 `session`）
- Tool callback 協議
- Saga 補償

---

## 未來考量

以下功能在第一個實作版本完成後 MAY 重新引入：

- `kind: Agent` — AI agent 角色模板
- `kind: Resources` — MCP servers、templates、model aliases、toolsets
- `type: agent` step
- `session` namespace
