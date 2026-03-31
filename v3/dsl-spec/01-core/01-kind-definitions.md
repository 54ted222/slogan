# 01 — Kind 定義總覽

本文件列出 DSL v3 支援的所有 kind，定義共通的 metadata 結構與 kind 偵測規則。每個 kind 的完整規格見對應的專屬文件。

---

## Kind 一覽

| Kind | apiVersion | 說明 | 詳見 |
|------|------------|------|------|
| `Workflow` | `workflow/v3` | 工作流程定義 | [02-document-structure](02-document-structure.md) |
| `Task` | `task/v3` | 可被呼叫的工作單元及其執行後端 | [06-step-task](06-step-task.md) |
| `Agent` | `agent/v3` | AI agent 角色模板 | [13-agent-definition](13-agent-definition.md) |
| `Toolset` | `toolset/v3` | 具名的 tools + skills 集合 | [18-toolset-definition](18-toolset-definition.md) |
| `Resources` | `resource/v3` | 共用資源宣告（MCP servers、templates、model aliases） | [19-resources-definition](19-resources-definition.md) |
| `Project` | `project/v3` | 所有 definition 檔案的 project 層級組織 | [20-project](20-project.md) |
| `Secret` | `secret/v3` | 加密的機密鍵值對 | [25-secrets-and-env](25-secrets-and-env.md) |

---

## 共通 metadata 結構

所有 kind 的 YAML 文件 MUST 包含 `apiVersion`、`kind`、`metadata` 三個頂層欄位：

```yaml
apiVersion: <kind>/v3       # MUST
kind: <Kind>                 # MUST
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
| Task | `namespace.action`（dotted） | `order.load`、`customer.get_history` |
| Agent | `namespace.role`（dotted） | `order.analyzer`、`fraud.detector` |
| Toolset | `kebab-case` | `order-processing`、`full-order-suite` |
| Resources | `kebab-case` | `shared-resources` |
| Project | `kebab-case` | `order`、`customer` |
| Secret | `snake_case` | `payment_secrets` |

### version 規則

- MUST 為正整數（1, 2, 3, ...）
- MUST 單調遞增
- 同一 name 可有多個版本同時處於 PUBLISHED 狀態
- `"latest"` 在引用時解析為最大版本號的 PUBLISHED 版本

---

## Kind 偵測規則

引擎與 JSON Schema 透過 YAML 文件中的 `kind` 欄位判斷文件類型，不依賴檔案命名慣例：

1. 讀取 YAML 文件的 `kind` 欄位
2. 根據 `kind` 值決定適用的 JSON Schema 與驗證規則
3. 驗證 `apiVersion` 與 `kind` 的一致性（如 `kind: Workflow` 必須搭配 `apiVersion: workflow/v3`）

```yaml
# 引擎透過 kind 欄位辨識
apiVersion: workflow/v3
kind: Workflow        # ← 偵測依據
```

---

## 生命週期

各 kind 的生命週期狀態機詳見 [23-lifecycle](23-lifecycle.md)。概要：

| Kind | 生命週期 |
|------|---------|
| Workflow, Task, Agent | DRAFT → VALIDATED → PUBLISHED → DEPRECATED → ARCHIVED |
| Toolset, Resources, Project, Secret | DRAFT → PUBLISHED → ARCHIVED |
