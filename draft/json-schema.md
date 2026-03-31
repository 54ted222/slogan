# JSON Schema 發布

> **狀態**：草稿
> **範圍**：為 YAML DSL 產出 JSON Schema，提供 IDE autocompletion / linting

---

## 設計動機

使用者撰寫 workflow、agent、task 等 YAML definition 時，缺乏即時的語法提示與驗證。透過發布 JSON Schema，可讓 VS Code、JetBrains 等 IDE 自動提供：

- **自動完成**：輸入欄位名稱時顯示可用選項
- **型別檢查**：即時標示型別錯誤（如 string 填成 integer）
- **文件提示**：hover 時顯示欄位說明
- **結構驗證**：缺少必填欄位、未知欄位等

---

## Schema 涵蓋範圍

需為以下 `kind` 各產出一份 JSON Schema：

| Kind | 檔案 | 說明 |
|------|------|------|
| `Workflow` | `workflow.schema.json` | Workflow definition（steps、triggers、input/output） |
| `Task` | `task.schema.json` | Task definition |
| `Agent` | `agent.schema.json` | Agent definition |
| `Resources` | `resources.schema.json` | 共用資源宣告（待 dsl-extensions 定案） |
| `Tools` | `tools.schema.json` | 工具集合宣告（待 dsl-extensions 定案） |

---

## Schema 結構設計

### 頂層 Schema（以 Workflow 為例）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://slogan.dev/schemas/workflow/v2.json",
  "title": "Slogan Workflow Definition",
  "description": "DSL v2 workflow definition schema",
  "type": "object",
  "required": ["apiVersion", "kind", "metadata", "steps"],
  "properties": {
    "apiVersion": {
      "type": "string",
      "const": "workflow/v2"
    },
    "kind": {
      "type": "string",
      "const": "Workflow"
    },
    "metadata": { "$ref": "#/$defs/metadata" },
    "input_schema": { "$ref": "#/$defs/jsonSchema" },
    "output_schema": { "$ref": "#/$defs/jsonSchema" },
    "triggers": { "$ref": "#/$defs/triggers" },
    "steps": {
      "type": "array",
      "items": { "$ref": "#/$defs/step" }
    }
  }
}
```

### Step 的 Discriminated Union

每種 step type 以 `type` 欄位區分：

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
        { "$ref": "#/$defs/stepAgent" }
      ]
    }
  }
}
```

---

## 發布方式

### 方案 A：Schema Store 登錄

將 schema 上傳至 [SchemaStore](https://www.schemastore.org/)，IDE 自動根據檔案 pattern 套用：

```json
{
  "fileMatch": ["*.workflow.yaml", "*.workflow.yml"],
  "url": "https://slogan.dev/schemas/workflow/v2.json"
}
```

### 方案 B：YAML 內嵌 Schema 指示

使用者在 YAML 開頭加入 modeline：

```yaml
# yaml-language-server: $schema=https://slogan.dev/schemas/workflow/v2.json
apiVersion: workflow/v2
kind: Workflow
```

### 方案 C：VS Code Extension

開發專用 VS Code extension，內含 schema 並提供額外功能（CEL expression 驗證、definition 跳轉等）。

---

## Schema 產出方式

| 方案 | 說明 | 優點 | 缺點 |
|------|------|------|------|
| 手動撰寫 | 直接寫 JSON Schema | 精確控制 | 與規格同步成本高 |
| 從規格生成 | 解析 spec markdown 產出 schema | 單一事實來源 | 需要建置工具 |
| 從程式碼生成 | 引擎的型別定義生成 schema | 與實作一致 | 依賴引擎實作語言 |

---

## CEL 表達式的 Schema 處理

`${ }` CEL 表達式無法透過 JSON Schema 驗證。處理方式：

- Schema 中將含 CEL 的欄位型別設為 `string`
- 在 `description` 中標註「支援 `${ }` CEL 表達式」
- CEL 的深度驗證交由 `slogan validate` CLI 命令或 VS Code extension

---

## 待決定

- 檔案命名慣例（`*.workflow.yaml` vs 以 `kind` 欄位判斷）
- Schema 版本管理策略（URL 是否含版本號）
- 是否需要 VS Code extension 或僅靠 Schema Store
- Schema 產出方式（手動 vs 自動生成）
- `${ }` 表達式的驗證深度
