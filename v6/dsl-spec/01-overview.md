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
  version: integer            # MUST — 版本號，正整數
  description: string         # MAY — 人類可讀說明（長度上限 1024 chars，超過 → invalid_metadata）
  labels: map<string, string> # MAY — 分類用鍵值對（value 必為 string，見下）
```

### labels value 型別

- `labels.<key>` value MUST 為 **string**；key 與 value 皆限制：
  - key：`^[a-z][a-z0-9_.-]{0,62}$`（最多 63 chars，kebab / snake / dotted）
  - value：任意 UTF-8 string，長度 0..255；不允許換行（`\n` / `\r`）
- YAML 解析後若 value 為 int / bool / float（如 `version: 3` 或 `enabled: true`）→ **載入失敗**，`registry.invalid_label_value`；使用者需明確加引號（`version: "3"`）避免誤解
- 空字串 `""` 合法（表示「有此 key 但無值」，可用於 selector 匹配存在性）
- Unicode 支援：value 允許 emoji / CJK；key 限 ASCII（便於外部監控系統聚合）
- `project.labels` / `workflow.labels` / `tool.labels` / `function.labels` 共用同一規則

### version 規則

- `version` MUST 為正整數（≥ 1）；0 / 負值 / 浮點 / 字串 → 載入失敗，`registry.invalid_version`
- **允許跳號**（如 v1 → v3）；engine 僅要求 `(name, version)` 全域唯一（見 `05-task-registry.md`）
- v6 不採 semver；無 major/minor/patch 區分。欲表達「破壞性變更」請：(1) bump 整數 version，(2) 於 `metadata.labels.breaking_change` 標記 `"true"` 供 CI 掃描
- **同 `(name, version)` 的 definition 內容不可變更**：一旦載入後再載入同 version 但內容不同的 definition → `registry.duplicate_action` 並拒絕後載入者（即 immutable semantics；需修改請 bump version）
- 已建立的 instance 透過 action_pin 鎖定版本（見 `05-task-registry.md`）；definition 新版上架不影響既有 instance

### labels 傳遞規則

| 來源 | 目的地 | 行為 |
|------|--------|------|
| Workflow definition labels | Workflow instance labels | 建立 instance 時複製（同 key 由 instance-time 覆寫） |
| Project defaults.labels | Workflow / Tool / Function definition labels | 載入期合併（見 `06-project-and-secret.md` 的合併規則） |
| Tool definition labels | step 執行紀錄 | **不**自動寫入 step instance；僅供 registry 觀測與 metric label 用（`tool_invocations_total{action_name}`；不含 labels dimension 以避免 cardinality 爆炸） |
| Function definition labels | Function instance labels | 建立 function instance 時複製 |

如 workflow step 需在執行紀錄中保留 Tool labels，應於 `emit` 或 `assign` 中顯式攜帶（引擎不隱式注入）。

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

### YAML anchors / aliases

v6 DSL 允許**單檔內**使用標準 YAML anchors (`&ref`) 與 aliases (`*ref`)、以及 merge key (`<<: *ref`)：

- 範圍僅限單一 YAML 文件；**不跨檔案**
- 解析器在載入期展開所有 anchor/alias，定義在 registry 中的結構不保留 anchor 資訊
- 循環 anchor（`&a ... *a`）→ 載入失敗（由 YAML 解析器報錯，engine 不額外處理）
- 跨檔案共享結構請用 Function（DSL 層）或 secret/env（資料層）；v6 不支援 `!include` 等擴充 tag

### Duration 格式

```
duration := segment+
segment  := <int> ("h" | "m" | "s")
```

- 至少一個 segment；各 segment 可以任意順序出現，但同一單位 MUST NOT 重複。
- 解析時將各 segment 換算為秒後加總。
- 範例：`10s`、`5m`、`1h`、`2h30m`、`1h30m15s`。
- 反例（非法）：`2h2h`、`30`（缺單位）、`1.5h`（不支援小數）。
