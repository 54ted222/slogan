# 31 — JSON Schema 發布策略

本文件定義 DSL v3 的 JSON Schema 發布方式，涵蓋各 kind 的 schema 結構、step discriminated union 設計、CEL 表達式處理、IDE 整合與 schema 產出流程。

---

## 設計動機

使用者撰寫 workflow、agent、task 等 YAML definition 時，缺乏即時的語法提示與驗證。透過發布 JSON Schema，可讓 VS Code、JetBrains 等 IDE 自動提供：

- **自動完成** — 輸入欄位名稱時顯示可用選項
- **型別檢查** — 即時標示型別錯誤（如 string 填成 integer）
- **文件提示** — hover 時顯示欄位說明
- **結構驗證** — 缺少必填欄位、未知欄位等

---

## Schema 涵蓋範圍

每個 `kind` 各產出一份獨立的 JSON Schema：

| Kind | Schema URL | 說明 |
|------|-----------|------|
| `Workflow` | `https://slogan.dev/schemas/workflow/v3.json` | Workflow definition（steps、triggers、input/output） |
| `Task` | `https://slogan.dev/schemas/task/v3.json` | Task definition（backend、schema） |
| `Agent` | `https://slogan.dev/schemas/agent/v3.json` | Agent definition（model、tools、loop） |
| `Toolset` | `https://slogan.dev/schemas/toolset/v3.json` | 具名 tools + skills 集合 |
| `Resources` | `https://slogan.dev/schemas/resource/v3.json` | 共用資源宣告（MCP servers、templates、model aliases） |
| `Project` | `https://slogan.dev/schemas/project/v3.json` | Definition 檔案 project 組織 |

### URL 格式

```
https://slogan.dev/schemas/<kind-lowercase>/v3.json
```

其中 `<kind-lowercase>` 對應 `apiVersion` 的前半段（如 `workflow`、`task`、`agent`、`toolset`、`resource`、`project`）。

---

## Kind 偵測方式（決策 B1）

以 YAML 文件中的 `kind` 欄位判斷文件類型，不依賴檔案命名慣例。

引擎與 IDE 的偵測流程：

1. 讀取 YAML 文件的 `kind` 欄位
2. 依 `kind` 值選擇對應的 JSON Schema
3. 若無 `kind` 欄位或值不在已知範圍內，則不套用 schema

此策略的優勢：

- 不限制使用者的檔案命名（不需要 `*.workflow.yaml` 之類的命名慣例）
- 同一目錄可混放不同 kind 的 YAML 檔案
- 與引擎實際載入行為一致

---

## Schema 結構設計

### 頂層 Schema（以 Workflow 為例）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://slogan.dev/schemas/workflow/v3.json",
  "title": "Slogan Workflow Definition",
  "description": "DSL v3 workflow definition schema",
  "type": "object",
  "required": ["apiVersion", "kind", "metadata", "steps"],
  "properties": {
    "apiVersion": {
      "type": "string",
      "const": "workflow/v3"
    },
    "kind": {
      "type": "string",
      "const": "Workflow"
    },
    "metadata": { "$ref": "#/$defs/metadata" },
    "input_schema": { "$ref": "#/$defs/jsonSchema" },
    "output_schema": { "$ref": "#/$defs/jsonSchema" },
    "triggers": { "$ref": "#/$defs/triggers" },
    "config": { "$ref": "#/$defs/workflowConfig" },
    "artifacts": { "$ref": "#/$defs/artifacts" },
    "steps": {
      "type": "array",
      "items": { "$ref": "#/$defs/step" }
    }
  }
}
```

### Step Discriminated Union

每種 step type 以 `type` 欄位作為 discriminator，使用 `oneOf` 區分：

```json
{
  "$defs": {
    "step": {
      "discriminator": { "propertyName": "type" },
      "oneOf": [
        { "$ref": "#/$defs/stepTask" },
        { "$ref": "#/$defs/stepAssign" },
        { "$ref": "#/$defs/stepIf" },
        { "$ref": "#/$defs/stepSwitch" },
        { "$ref": "#/$defs/stepForeach" },
        { "$ref": "#/$defs/stepParallel" },
        { "$ref": "#/$defs/stepEmit" },
        { "$ref": "#/$defs/stepWaitEvent" },
        { "$ref": "#/$defs/stepReturn" },
        { "$ref": "#/$defs/stepFail" },
        { "$ref": "#/$defs/stepSubWorkflow" },
        { "$ref": "#/$defs/stepAgent" },
        { "$ref": "#/$defs/stepSaga" }
      ]
    }
  }
}
```

每個 step 子 schema 以 `const` 限定 `type` 值：

```json
{
  "$defs": {
    "stepTask": {
      "type": "object",
      "required": ["type", "action"],
      "properties": {
        "type": { "const": "task" },
        "id": { "type": "string" },
        "action": {
          "type": "string",
          "description": "Task definition name（兩段式命名，如 order.load）"
        },
        "input": { "type": "object" },
        "when": {
          "type": "string",
          "description": "CEL 表達式，支援 ${ } 語法。為 false 時 step 被 SKIPPED"
        },
        "timeout": { "type": "string" },
        "retry": { "$ref": "#/$defs/retryConfig" },
        "catch": {
          "type": "array",
          "items": { "$ref": "#/$defs/step" }
        },
        "compensate": {
          "type": "array",
          "items": { "$ref": "#/$defs/step" }
        }
      }
    },
    "stepIf": {
      "type": "object",
      "required": ["type", "when"],
      "properties": {
        "type": { "const": "if" },
        "when": {
          "type": "string",
          "description": "CEL 表達式，支援 ${ } 語法。求值為 true 時執行 then 分支"
        },
        "then": {
          "type": "array",
          "items": { "$ref": "#/$defs/step" }
        },
        "else": {
          "type": "array",
          "items": { "$ref": "#/$defs/step" }
        }
      }
    },
    "stepAgent": {
      "type": "object",
      "required": ["type", "agent"],
      "properties": {
        "type": { "const": "agent" },
        "id": { "type": "string" },
        "agent": {
          "type": "string",
          "description": "Agent definition name（dotted namespace，如 order.analyzer）"
        },
        "prompt": {
          "description": "User prompt，支援 string 或 template 引用",
          "oneOf": [
            { "type": "string" },
            { "$ref": "#/$defs/promptTemplate" }
          ]
        },
        "input": { "type": "object" },
        "tools": {
          "type": "array",
          "items": { "$ref": "#/$defs/toolRef" }
        },
        "tools_exclude": {
          "type": "array",
          "items": { "type": "string" }
        },
        "config": { "$ref": "#/$defs/agentStepConfig" },
        "timeout": { "type": "string" }
      }
    }
  }
}
```

---

## CEL 表達式欄位處理

`${ }` CEL 表達式無法透過 JSON Schema 驗證語法正確性。處理策略：

| 處理方式 | 說明 |
|----------|------|
| Schema 型別 | 含 CEL 的欄位型別設為 `string` |
| description 標註 | 在 `description` 中明確標註「支援 `${ }` CEL 表達式」 |
| 深度驗證 | CEL 語法驗證交由 `slogan validate` CLI 命令或 VS Code extension |

範例：

```json
{
  "when": {
    "type": "string",
    "description": "CEL 表達式，支援 ${ } 語法。求值結果 MUST 為 boolean"
  }
}
```

對於接受靜態值或 CEL 表達式的欄位（如 `input` 中的值），schema 使用寬鬆型別（`string | number | boolean | object | array`），不限制 CEL 語法。

---

## IDE 整合（決策 B2：SchemaStore）

透過 [SchemaStore](https://www.schemastore.org/) 登錄，IDE 自動根據 `kind` 欄位套用 schema。

### SchemaStore Catalog Entry

```json
{
  "name": "Slogan Workflow",
  "description": "Slogan DSL v3 Workflow definition",
  "url": "https://slogan.dev/schemas/workflow/v3.json",
  "versions": {
    "v3": "https://slogan.dev/schemas/workflow/v3.json"
  }
}
```

由於 SchemaStore 的 `fileMatch` 機制以檔名 pattern 匹配，而本 DSL 採用 `kind` 欄位偵測（決策 B1），因此需搭配 YAML Language Server 的 `$schema` 指示或 IDE 設定：

### 方式一：YAML 內嵌 Schema 指示

使用者在 YAML 開頭加入 modeline，IDE 即自動套用：

```yaml
# yaml-language-server: $schema=https://slogan.dev/schemas/workflow/v3.json
apiVersion: workflow/v3
kind: Workflow
```

### 方式二：VS Code settings.json

在專案 `.vscode/settings.json` 中配置：

```json
{
  "yaml.schemas": {
    "https://slogan.dev/schemas/workflow/v3.json": "*.yaml"
  }
}
```

### 方式三：SchemaStore fileMatch（通用場景）

若團隊採用命名慣例，可在 SchemaStore 中同時登錄 fileMatch：

```json
{
  "name": "Slogan Workflow",
  "description": "Slogan DSL v3 Workflow definition",
  "fileMatch": ["*.workflow.yaml", "*.workflow.yml"],
  "url": "https://slogan.dev/schemas/workflow/v3.json"
},
{
  "name": "Slogan Task",
  "description": "Slogan DSL v3 Task definition",
  "fileMatch": ["*.task.yaml", "*.task.yml"],
  "url": "https://slogan.dev/schemas/task/v3.json"
},
{
  "name": "Slogan Agent",
  "description": "Slogan DSL v3 Agent definition",
  "fileMatch": ["*.agent.yaml", "*.agent.yml"],
  "url": "https://slogan.dev/schemas/agent/v3.json"
},
{
  "name": "Slogan Toolset",
  "description": "Slogan DSL v3 Toolset definition",
  "fileMatch": ["*.toolset.yaml", "*.toolset.yml"],
  "url": "https://slogan.dev/schemas/toolset/v3.json"
},
{
  "name": "Slogan Resources",
  "description": "Slogan DSL v3 Resources definition",
  "fileMatch": ["*.resources.yaml", "*.resources.yml"],
  "url": "https://slogan.dev/schemas/resource/v3.json"
}
```

> **建議**：對使用者文件推薦「方式一：YAML 內嵌 Schema 指示」作為主要方式，適用於任何檔案命名。

---

## Schema 產出方式（決策 B3：從引擎程式碼生成）

JSON Schema 從引擎程式碼的型別定義自動生成，確保 schema 與引擎實作同步：

| 項目 | 說明 |
|------|------|
| 來源 | 引擎程式碼中的 struct / type definition |
| 生成工具 | 依引擎實作語言選用（如 Go 的 `jsonschema`、TypeScript 的 `zod-to-json-schema`） |
| 發布流程 | CI pipeline 在引擎版本發布時自動生成並上傳至 `https://slogan.dev/schemas/` |
| 版本對應 | Schema URL 中的 `v3` 對應 DSL major version，與引擎 patch version 無關 |

### 生成與發布流程

```
引擎程式碼 → Schema 生成工具 → JSON Schema 檔案 → 上傳至 CDN → SchemaStore 更新
```

---

## 相關文件

- [01-kind-definitions](../01-core/01-kind-definitions.md) — Kind 一覽與 metadata 結構
- [05-steps-overview](../02-steps/05-steps-overview.md) — Step 類型與共通屬性
- [08-step-control-flow](../02-steps/08-step-control-flow.md) — if / switch 的 `when` 欄位
- [13-agent-definition](../03-agent/13-agent-definition.md) — Agent Definition schema
