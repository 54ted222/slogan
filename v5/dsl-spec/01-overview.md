# 01 — 總覽

本規格定義一套以 YAML 撰寫的 workflow DSL，用於描述可持久化、事件驅動的工作流程。v5 在 v4 基礎上收斂矛盾、補齊缺漏，**不新增大功能**。語法大致向 v4 看齊，但對分歧命名做單一選擇、對「待補」協議做完整定義，詳見 [CHANGES-FROM-V4.md](CHANGES-FROM-V4.md)。

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
| `Workflow` | `workflow/v5` | 工作流程定義 |
| `Tool` | `tool/v5` | 可被呼叫的工具及其執行後端（v3 的 `Task`） |
| `Function` | `function/v5` | 以 steps 定義的可重用邏輯單元，有獨立變數空間 |
| `Agent` | `agent/v5` | AI agent 角色模板 |
| `Resources` | `resource/v5` | 共用資源（MCP servers、templates、model aliases、toolsets） |
| `Project` | `project/v5` | 定義檔案的 project 層級組織 |
| `Secret` | `secret/v5` | 加密的機密鍵值對 |

### v4 → v5 變更摘要

| 類別 | 項目 | 說明 |
|------|------|------|
| Breaking | Tool backend type `stdio` 移除 | 統一為 `exec` |
| Breaking | Agent `system_prompt` 欄位移除 | 統一為 `system` |
| Breaking | Wait step `step_id` 欄位改名 | 統一為 `step`（值可為 string 或 [string]） |
| Breaking | Agent / toolset `tools` 以 `type:` 分流的寫法移除 | 統一為字串簡寫或 `{action, name?, description?, input?}` |
| Clarification | `prev` 遇 SKIPPED、`when` 求值失敗、saga 並行補償、lifecycle init 求值上下文、model 合併優先級 | 明文化，見 `04-expressions.md` / `05-tool.md` / `06-agent.md` |
| Additions | `foreach.count` 參數 | `items: range(n)` 的快捷 |
| Additions | Tool callback 協議 | `05-tool.md` 新增章節，對接 `05b-function.md` 的 caller `callback:` |

完整差異見 [CHANGES-FROM-V4.md](CHANGES-FROM-V4.md)。

---

## 共通 metadata 結構

所有 kind 的 YAML 文件 MUST 包含 `apiVersion`、`kind`、`metadata` 三個頂層欄位：

```yaml
apiVersion: <kind>/v5
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

### Duration 格式

```
duration := segment+
segment  := <int> ("h" | "m" | "s")
```

- 至少一個 segment；各 segment 可以任意順序出現，但同一單位 MUST NOT 重複。
- 解析時將各 segment 換算為秒後加總。
- 範例：`10s`、`5m`、`1h`、`2h30m`、`1h30m15s`。
- 反例（非法）：`2h2h`、`30`（缺單位）、`1.5h`（不支援小數）。
