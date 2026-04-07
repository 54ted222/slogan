# 01 — 總覽

本規格定義一套以 YAML 撰寫的 workflow DSL，用於描述可持久化、事件驅動的工作流程。v4 在 v3 基礎上大幅簡化，聚焦於 DSL 語法呈現。

---

## 設計原則

| 原則 | 說明 |
|------|------|
| Declarative | 描述「做什麼」，不描述「怎麼做」 |
| YAML-native | 所有定義以 YAML 表達，不內嵌通用程式語言 |
| CEL expressions | 動態運算使用 CEL (cel.dev)，以 `${ }` 定界 |
| Agent as first-class | AI agent 是 workflow 的一等公民 |
| Tool unification | 所有工具統一抽象為 tool interface |

---

## Kind 一覽

| Kind | apiVersion | 說明 |
|------|------------|------|
| `Workflow` | `workflow/v4` | 工作流程定義 |
| `Tool` | `tool/v4` | 可被呼叫的工具及其執行後端（v3 的 `Task`） |
| `Function` | `function/v4` | 以 steps 定義的可重用邏輯單元，有獨立變數空間 |
| `Agent` | `agent/v4` | AI agent 角色模板 |
| `Resources` | `resource/v4` | 共用資源（MCP servers、templates、model aliases、toolsets） |
| `Project` | `project/v4` | 定義檔案的 project 層級組織 |
| `Secret` | `secret/v4` | 加密的機密鍵值對 |

### v3 → v4 變更摘要

| 項目 | v3 | v4 |
|------|----|----|
| Task | `kind: Task` | **改名 `kind: Tool`** |
| Toolset | 獨立 `kind: Toolset` | **併入 Resources 的 `toolsets` 屬性** |
| sub_workflow | `type: sub_workflow` step | **獨立為 `kind: Function`，透過 `type: task` 呼叫** |

---

## 共通 metadata 結構

所有 kind 的 YAML 文件 MUST 包含 `apiVersion`、`kind`、`metadata` 三個頂層欄位：

```yaml
apiVersion: <kind>/v4
kind: <Kind>
metadata:
  name: string               # MUST — 唯一識別名稱
  version: integer            # MUST — 版本號，正整數，單調遞增
  description: string         # MAY — 人類可讀說明
  labels: map<string, string> # MAY — 分類用鍵值對
```

### name 命名規則

| Kind | 格式 | 範例 |
|------|------|------|
| Workflow | `snake_case` | `order_fulfillment` |
| Tool | `snake_case` + dotted | `order.load`、`memory.write_line` |
| Function | `snake_case` + dotted | `payment.process`、`order.validate` |
| Agent | `snake_case` + dotted | `order.analyzer`、`fraud.detector` |
| Resources | `kebab-case` | `shared-resources` |
| Project | `kebab-case` | `order` |
| Secret | `snake_case` | `payment_secrets` |

---

## 術語表

| 術語 | 定義 |
|------|------|
| **Workflow Definition** | `kind: Workflow` YAML 文件，描述流程結構 |
| **Workflow Instance** | 一次 workflow 的執行實例 |
| **Step** | workflow 中的最小執行單元 |
| **Tool Definition** | `kind: Tool` YAML 文件，定義可被呼叫的工具（v3 稱 Task） |
| **Function Definition** | `kind: Function` YAML 文件，以 steps 定義的可重用邏輯單元 |
| **Agent Definition** | `kind: Agent` YAML 文件，定義 AI agent 的角色模板 |
| **Agent Session** | 一次 agent step 的執行實例 |
| **Resources Definition** | `kind: Resources` YAML 文件，集中宣告共用資源 |
| **Project Definition** | `kind: Project` YAML 文件，定義 project 層級組織 |
| **Trigger** | 啟動 workflow instance 的機制 |
| **Event** | 系統內的具名訊息 |
| **CEL** | Common Expression Language (cel.dev) |

---

## 文件索引

| 文件 | 說明 |
|------|------|
| [02-workflow](02-workflow.md) | Workflow kind — 頂層結構、triggers、config |
| [03-steps](03-steps.md) | 所有 step 類型 |
| [04-expressions](04-expressions.md) | CEL 表達式語法與求值上下文 |
| [05-tool](05-tool.md) | Tool kind（v3 的 Task） |
| [05b-function](05b-function.md) | Function kind（v3 的 sub_workflow） |
| [06-agent](06-agent.md) | Agent kind — 定義、step、agentic loop |
| [07-resources](07-resources.md) | Resources kind — MCP servers、templates、model aliases、toolsets |
| [08-project-and-secret](08-project-and-secret.md) | Project kind + Secret kind |
| [99-examples](99-examples.md) | 完整範例集 |

---

## 慣例說明

- 使用 **MUST** / **SHOULD** / **MAY** 表示要求強度（RFC 2119）
- 所有識別字 MUST 使用 `snake_case`（step id、variable name、event type）
- Duration 格式：`10s`、`5m`、`1h`、`2h30m`
