# 01 — 總覽

本規格定義一套以 YAML 撰寫的 workflow DSL，用於描述可持久化、事件驅動的工作流程。v6 基於 v5 精簡，移除 `Agent` 與 `Resources` kind，聚焦於第一個實作版本所需的核心功能。

---

## 設計原則

| 原則 | 說明 |
|------|------|
| Declarative | 描述「做什麼」，不描述「怎麼做」 |
| YAML-native | 所有定義以 YAML 表達，不內嵌通用程式語言 |
| CEL expressions | 動態運算使用 CEL (cel.dev)，以 `${ }` 定界 |
| Tool unification | 所有工具統一抽象為 tool interface |

---

## Kind 一覽

| Kind | apiVersion | 說明 |
|------|------------|------|
| `Workflow` | `workflow/v6` | 工作流程定義 |
| `Tool` | `tool/v6` | 可被呼叫的工具及其執行後端（v3 的 `Task`） |
| `Function` | `function/v6` | 以 steps 定義的可重用邏輯單元，有獨立變數空間 |
| `Project` | `project/v6` | 定義檔案的 project 層級組織 |
| `Secret` | `secret/v6` | 加密的機密鍵值對 |

### v5 → v6 變更摘要

| 類別 | 項目 | 說明 |
|------|------|------|
| 移除 | `kind: Agent` | 第一個實作版本不做 agent |
| 移除 | `kind: Resources` | 第一個實作版本不做 resources（MCP servers、templates、model aliases、toolsets） |
| 移除 | `type: agent` step | 移除 agent step 類型 |
| 移除 | `session` namespace | 移除 agent session 相關求值上下文 |

---

## 共通 metadata 結構

所有 kind 的 YAML 文件 MUST 包含 `apiVersion`、`kind`、`metadata` 三個頂層欄位：

```yaml
apiVersion: <kind>/v6
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
| Project | `kebab-case` | `order` |
| Secret | `snake_case` | `payment_secrets` |

### Task registry 解析規則

`type: task` 的 `action` 透過 task registry 解析。v6 只接受 `snake_case` + dotted 格式作為 action 名稱主體：

| action 形式 | 判定為 | 範例 |
|-------------|--------|------|
| `<namespace>.<action>` | Tool / Function / builtin 動作 | `order.load`、`payment.process`、`builtin.echo` |
| `<project>/<namespace>.<action>` | 跨 project 引用 | `order/order.load` |

- 主體每一段 MUST 符合 `^[a-z][a-z0-9_]*$`（snake_case）；含 `-` / 大寫 / 其他字元 → 載入錯誤 `registry.invalid_action_name`。
- Project 前綴以 `/` 分隔（project name 為 kebab-case，見 [06-project-and-secret](06-project-and-secret.md)）；巢狀 project 以 `/` 層層展開。

---

## 術語表

| 術語 | 定義 |
|------|------|
| **Workflow Definition** | `kind: Workflow` YAML 文件，描述流程結構 |
| **Workflow Instance** | 一次 workflow 的執行實例 |
| **Step** | workflow 中的最小執行單元 |
| **Tool Definition** | `kind: Tool` YAML 文件，定義可被呼叫的工具（v3 稱 Task） |
| **Function Definition** | `kind: Function` YAML 文件，以 steps 定義的可重用邏輯單元 |
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
| [06-project-and-secret](06-project-and-secret.md) | Project kind + Secret kind |
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
