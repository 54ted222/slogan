# 00 — 總覽

本規格定義一套以 YAML 撰寫的 workflow DSL，用於描述可持久化、事件驅動的工作流程。引擎依據此 DSL 建立 workflow instance 並依序（或平行）執行其中的 steps。v3 新增 AI agent 作為一等公民、saga 補償模式、資源與工具集合管理等功能。

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
| Agent as first-class citizen | AI agent 是 workflow 的一等公民，與 task、sub_workflow 並列 |
| Tool unification | 所有工具（task、workflow、agent、MCP、builtin）統一抽象為 tool interface |

---

## 術語表

| 術語 | 定義 |
|------|------|
| **Workflow Definition** | 一份 YAML 文件（`kind: Workflow`），描述 workflow 的結構、輸入輸出、steps |
| **Workflow Instance** | 一次 workflow 的執行實例，擁有獨立狀態與資料 |
| **Step** | workflow 中的最小執行單元，具有 `id` 與 `type` |
| **Task Definition** | 獨立的 YAML 文件（`kind: Task`），定義一個可被呼叫的工作單元及其執行後端 |
| **Agent Definition** | 獨立的 YAML 文件（`kind: Agent`），定義 AI agent 的角色模板（model、system prompt、預設 tools/skills） |
| **Agent Session** | 一次 agent step 的執行實例，擁有獨立的對話紀錄與狀態 |
| **Toolset Definition** | 獨立的 YAML 文件（`kind: Toolset`），定義具名的 tools + skills 集合 |
| **Resources Definition** | 獨立的 YAML 文件（`kind: Resources`），集中宣告 MCP servers、string templates、model aliases 等共用資源 |
| **Project Definition** | 獨立的 YAML 文件（`kind: Project`），定義所有 definition 檔案的 project 層級組織 |
| **Trigger** | 啟動 workflow instance 的機制（manual 或 event） |
| **Event** | 系統內的具名訊息，包含 `type` 與 `data` |
| **Artifact** | workflow 使用或產生的外部資源（檔案、資料） |
| **CEL** | Common Expression Language (cel.dev)，本 DSL 的表達式語言 |
| **Execution Policy** | 定義 task 在 crash recovery 時的重跑策略 |
| **Namespace** | CEL 表達式中的資料存取路徑前綴（如 `input`、`steps`、`vars`、`env`、`secret`、`session`） |
| **Secret Definition** | 獨立的 YAML 文件（`kind: Secret`），儲存加密的機密鍵值對 |
| **Saga** | 補償模式，用於長交易的失敗回復（step-level compensate 或 saga 區塊） |
| **Instance Label** | 附加在 workflow instance 上的自訂 key-value 標籤，用於分類與查詢 |
| **Model Alias** | 語意化的 LLM model 別名（如 `fast`、`smart`），與實際 provider/model 解耦 |
| **String Template** | 可重用的字串模板，支援 `${ }` 變數替換，主要用於 prompt 組裝 |
| **Agent Skill** | 遵循 agentskills.io 規格的知識包，提供 agent 專業知識與行為指引 |

---

## 文件索引

| 文件 | 說明 |
|------|------|
| [01-kind-definitions](01-kind-definitions.md) | Kind 定義總覽（6 種 kind） |
| [02-document-structure](02-document-structure.md) | Workflow 頂層 YAML 結構 |
| [03-expressions](03-expressions.md) | CEL 表達式語法與求值上下文 |
| [04-input-output-schema](04-input-output-schema.md) | Input / Output schema 定義 |
| [05-steps-overview](../02-steps/05-steps-overview.md) | Step 共通屬性與生命週期 |
| [06-step-task](../02-steps/06-step-task.md) | task 步驟 + Task Definition |
| [07-step-assign](../02-steps/07-step-assign.md) | assign 步驟 |
| [08-step-control-flow](../02-steps/08-step-control-flow.md) | if / switch / foreach / parallel |
| [09-step-events](../02-steps/09-step-events.md) | emit / wait_event |
| [10-step-terminal](../02-steps/10-step-terminal.md) | fail / return / continue-as-new（renew） |
| [11-step-sub-workflow](../02-steps/11-step-sub-workflow.md) | sub_workflow 步驟 |
| [12-step-saga](../02-steps/12-step-saga.md) | saga 步驟與 step-level compensate |
| [13-agent-definition](../03-agent/13-agent-definition.md) | Agent Definition（`kind: Agent`） |
| [14-agent-tools](../03-agent/14-agent-tools.md) | Agent 的 tool 類型與內建 tools |
| [15-agent-skills](../03-agent/15-agent-skills.md) | Agent Skills（agentskills.io） |
| [16-agent-loop](../03-agent/16-agent-loop.md) | Agentic Loop（預設 / 自訂） |
| [17-agent-hooks-and-streaming](../03-agent/17-agent-hooks-and-streaming.md) | Agent Hooks 與 Streaming |
| [18-toolset-definition](../04-resources/18-toolset-definition.md) | Toolset Definition（`kind: Toolset`） |
| [19-resources-definition](../04-resources/19-resources-definition.md) | Resources Definition（`kind: Resources`） |
| [20-project](../04-resources/20-project.md) | Project Definition（`kind: Project`） |
| [21-error-handling](../05-runtime/21-error-handling.md) | 錯誤處理模型 |
| [22-artifacts](../05-runtime/22-artifacts.md) | Artifact 系統 |
| [23-lifecycle](../05-runtime/23-lifecycle.md) | 生命週期狀態機 |
| [24-instance-labels](../05-runtime/24-instance-labels.md) | Instance Labels |
| [25-secrets-and-env](../04-resources/25-secrets-and-env.md) | Secrets 與環境變數 |
| [26-triggers](../05-runtime/26-triggers.md) | Trigger 模型（manual / event） |
| [27-validation-rules](../06-publish/27-validation-rules.md) | 靜態驗證規則 |
| [28-versioning](../06-publish/28-versioning.md) | 版本管理與遷移策略 |
| [29-execution-guarantees](../05-runtime/29-execution-guarantees.md) | 執行語意保證 |
| [30-testing](../06-publish/30-testing.md) | 本地測試模式（`slogan test`） |
| [31-json-schema-publication](../06-publish/31-json-schema-publication.md) | JSON Schema 發布策略 |
| [99-examples](../99-examples.md) | 完整範例集 |

---

## 版本歷史

| 版本 | 變更摘要 |
|------|----------|
| v1 | 初版範例（非正式規格） |
| v2 | 正式規格：CEL + `${ }` 定界符、新增 `sub_workflow`、統一錯誤處理三層模型、`config` 區塊、task definition 獨立化（`kind: Task`）、event trigger `input_mapping`、`kind: Secret` 加密機密管理 |
| v3 | Agent 一等公民（`kind: Agent`、`type: agent` step）、Saga 補償模式（`type: saga` step + step-level `compensate`）、continue-as-new（`return` + `renew`）、`kind: Toolset`（統一 tools + skills 集合）、`kind: Resources`（MCP servers、string templates、model aliases）、`kind: Project`（definition 檔案 project 組織）、instance labels、`expr` 改名為 `when`、兩段式 tool 命名（`namespace.action`）、本地測試模式、JSON Schema 發布 |

---

## 慣例說明

- 本規格使用 **MUST** / **SHOULD** / **MAY** 表示要求強度，語意同 RFC 2119
- YAML 鍵值與程式碼識別字使用英文，說明內容使用繁體中文
- 所有識別字（step id、variable name、event type）MUST 使用 `snake_case`
- YAML 範例使用 `yaml` 語法的 fenced code block
- Duration 格式：`10s`、`5m`、`1h`、`2h30m`
