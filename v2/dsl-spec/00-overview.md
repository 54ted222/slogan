# 00 — 總覽

本規格定義一套以 YAML 撰寫的 workflow DSL，用於描述可持久化、事件驅動的工作流程。引擎依據此 DSL 建立 workflow instance 並依序（或平行）執行其中的 steps。

---

## 設計原則

| 原則 | 說明 |
|------|------|
| Declarative | workflow 定義描述「做什麼」，不描述「怎麼做」 |
| YAML-native | 所有定義以 YAML 表達，不內嵌任何通用程式語言 |
| CEL expressions | 所有動態運算使用 CEL (cel.dev)，以 `${ }` 定界 |
| Durable state machine | 每個 instance 都是可持久化的狀態機，crash 後可恢復 |
| DB as single source of truth | 資料庫是唯一權威狀態來源，記憶體僅為快取 |
| Deterministic replay | 相同輸入 + 相同事件序列 = 相同結果 |
| DSL / runtime 解耦 | 規格僅定義語意，不綁定特定 runtime 實作 |

---

## 術語表

| 術語 | 定義 |
|------|------|
| **Workflow Definition** | 一份 YAML 文件，描述 workflow 的結構、輸入輸出、steps |
| **Workflow Instance** | 一次 workflow 的執行實例，擁有獨立狀態與資料 |
| **Step** | workflow 中的最小執行單元，具有 `id` 與 `type` |
| **Task Definition** | 獨立的 YAML 文件（`kind: Task`），定義一個可被呼叫的工作單元及其執行後端 |
| **Task Step** | workflow 中呼叫 task definition 的 step 類型 |
| **Trigger** | 啟動 workflow instance 的機制（manual 或 event） |
| **Event** | 系統內的具名訊息，包含 `type` 與 `data` |
| **Artifact** | workflow 使用或產生的外部資源（檔案、資料） |
| **CEL** | Common Expression Language (cel.dev)，本 DSL 的表達式語言 |
| **Execution Policy** | 定義 task 在 crash recovery 時的重跑策略 |
| **Namespace** | CEL 表達式中的資料存取路徑前綴（如 `input`、`steps`、`vars`、`env`、`secret`） |
| **Secret Definition** | 獨立的 YAML 文件（`kind: Secret`），儲存加密的機密鍵值對 |

---

## 文件索引

| 文件 | 說明 |
|------|------|
| [01-document-structure](01-document-structure.md) | Workflow 頂層 YAML 結構 |
| [02-triggers](02-triggers.md) | Trigger 模型（manual / event） |
| [03-input-output-schema](03-input-output-schema.md) | Input / Output schema 定義 |
| [04-expressions](04-expressions.md) | CEL 表達式語法與求值上下文 |
| [05-steps-overview](05-steps-overview.md) | Step 共通屬性與生命週期 |
| [06-step-task](06-step-task.md) | task 步驟（使用端） |
| [07-step-assign](07-step-assign.md) | assign 步驟 |
| [08-step-control-flow](08-step-control-flow.md) | if / switch / foreach / parallel |
| [09-step-events](09-step-events.md) | emit / wait_event |
| [10-step-terminal](10-step-terminal.md) | fail / return |
| [11-step-sub-workflow](11-step-sub-workflow.md) | sub_workflow 步驟 |
| [12-error-handling](12-error-handling.md) | 錯誤處理模型 |
| [13-artifacts](13-artifacts.md) | Artifact 系統 |
| [14-lifecycle](14-lifecycle.md) | 生命週期狀態機 |
| [15-runtime-persistence](15-runtime-persistence.md) | Runtime 架構與持久化 |
| [16-validation-rules](16-validation-rules.md) | 靜態驗證規則 |
| [17-task-definition](17-task-definition.md) | Task Definition（定義端：stdio / http / builtin） |
| [18-secrets-and-env](18-secrets-and-env.md) | Secrets 與環境變數（`env` / `secret` namespace、CLI） |
| [19-versioning](19-versioning.md) | 版本管理與遷移策略 |
| [20-execution-guarantees](20-execution-guarantees.md) | 執行語意保證（delivery guarantee、race condition） |
| [99-examples](99-examples.md) | 完整範例集 |

---

## 版本歷史

| 版本 | 變更摘要 |
|------|----------|
| v1 | 初版範例（非正式規格） |
| v2 | 正式規格：CEL + `${ }` 定界符、新增 `sub_workflow`、統一錯誤處理三層模型、`config` 區塊、task definition 獨立化（`kind: Task`）、event trigger `input_mapping`、`kind: Secret` 加密機密管理 |

---

## 慣例說明

- 本規格使用 **MUST** / **SHOULD** / **MAY** 表示要求強度，語意同 RFC 2119
- YAML 鍵值與程式碼識別字使用英文，說明內容使用繁體中文
- 所有識別字（step id、variable name、event type）MUST 使用 `snake_case`
- YAML 範例使用 `yaml` 語法的 fenced code block
- Duration 格式：`10s`、`5m`、`1h`、`2h30m`
