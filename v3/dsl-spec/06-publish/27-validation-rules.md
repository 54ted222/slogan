# 27 — 靜態驗證規則

本文件定義 definition 在發布前 MUST 通過的靜態驗證規則，涵蓋所有 kind（Workflow、Task、Agent、Toolset、Resources、Project、Secret）。

---

## 驗證時機

驗證在 definition lifecycle 從 DRAFT → VALIDATED 轉換時執行。所有規則 MUST 通過，否則轉換失敗。

---

## 結構驗證

### 頂層結構（Workflow）

- `apiVersion` MUST 為 `workflow/v3`
- `kind` MUST 為 `Workflow`
- `metadata.name` MUST 存在且為有效的 `snake_case` 識別字
- `metadata.version` MUST 為正整數
- `triggers` MUST 為非空陣列
- `steps` MUST 為非空陣列

### 頂層結構（Agent）

- `apiVersion` MUST 為 `agent/v3`
- `kind` MUST 為 `Agent`
- `metadata.name` MUST 存在且為有效的 dotted namespace 識別字（如 `order.analyzer`）
- `metadata.version` MUST 為正整數
- `model` MUST 存在（string alias 或 object）
  - 若為 string：MUST 為非空字串（alias 名稱）
  - 若為 object：`provider` 和 `name` MUST 存在且為非空字串
  - 若含 `alias` + `config`：`alias` MUST 為非空字串
- `tools`（若存在）MUST 為陣列，每個元素依 [14-agent-tools](14-agent-tools.md) 定義驗證
- `skills`（若存在）MUST 為陣列，每個元素 MUST 為 `type: toolset` 引用
- `config.max_iterations`（若存在）MUST 為正整數
- `config.max_tool_calls`（若存在）MUST 為正整數
- `config.output_format`（若存在）MUST 為 `text` 或 `json`
- `config.persist_history`（若存在）MUST 為 `full`、`summary`、`none` 之一
- `config.stream`（若存在）MUST 為 `token`、`iteration`、`none` 之一
- `loop`（若存在）：`loop.steps` MUST 為非空 step 陣列（詳見 [16-agent-loop](16-agent-loop.md)）

### 頂層結構（Toolset）

- `apiVersion` MUST 為 `toolset/v3`
- `kind` MUST 為 `Toolset`
- `metadata.name` MUST 存在且為有效的 `kebab-case` 識別字
- `metadata.version` MUST 為正整數
- `tools`（若存在）MUST 為陣列，每個元素依 [14-agent-tools](14-agent-tools.md) 定義驗證
- `skills`（若存在）MUST 為陣列
- `includes`（若存在）MUST 為陣列，每個元素為 toolset name 字串

### 頂層結構（Resources）

- `apiVersion` MUST 為 `resource/v3`
- `kind` MUST 為 `Resources`
- `metadata.name` MUST 存在且為有效的 `kebab-case` 識別字
- `metadata.version` MUST 為正整數
- `mcp_servers`（若存在）MUST 為 map，每個 value 須含 `transport`（`stdio` 或 `sse`）
  - `stdio` transport：`command` MUST 存在且為非空字串
  - `sse` transport：`url` MUST 存在且為有效 URL
- `string_templates`（若存在）MUST 為 map，每個 value 為非空字串
- `model_aliases`（若存在）MUST 為 map，每個 value 為 object，含 `provider` 和 `name`

### 頂層結構（Project）

- `apiVersion` MUST 為 `project/v3`
- `kind` MUST 為 `Project`
- `metadata.name` MUST 存在且為有效的 `kebab-case` 識別字
- 資料夾中 MUST 存在 `project.yaml` 檔案（詳見 [20-project](20-project.md)）
- `defaults.labels`（若存在）MUST 為 `map<string, string>`

### Step 結構

- 每個 step MUST 有 `type`；`id` 為非必填（見 Step ID 驗證）
- `type` MUST 為已定義的 13 種類型之一（task、assign、if、switch、foreach、parallel、emit、wait_event、fail、return、sub_workflow、agent、saga）
- 每種 step type 的必填欄位 MUST 存在（如 task 的 `action`、if 的 `condition`、switch 的 `condition`）
- `compensate`（若存在）MUST 為非空 step 陣列，且 compensate 內的 step 須通過標準 step 驗證（詳見 [12-step-saga](12-step-saga.md)）
- 未知欄位 SHOULD 產生警告

---

## Step ID 驗證

- `id` 為非必填；未指定 id 的 step，引擎自動產生內部 ID（`_<type>_<index>`）
- 使用者定義的 `id` MUST NOT 以 `_` 開頭（避免與自動產生的 ID 衝突）
- 所有使用者定義的 step id MUST 在整個 definition 中全域唯一
- 包含巢狀於 if/switch/foreach/parallel/on_error/on_timeout/saga/compensate 內的 steps
- 重複的 id → 驗證失敗
- 若有 `steps.<id>.output` 參照，被參照的 `<id>` 對應的 step MUST 有明確指定 `id`

---

## Expression 驗證

### CEL 語法

- 所有 `${ }` 內的 CEL 表達式 MUST 可被解析（語法正確）
- 語法錯誤 → 驗證失敗

### Namespace 參照

- `steps.<id>.output` 中的 `<id>` MUST 對應一個存在的 step id
- 被參照的 step MUST 保證在當前 step 之前執行（不允許前向參照）
- 前向參照的判定規則：
  - 同一 `steps` 陣列中，只能參照索引較小的 step
  - 巢狀 step 可參照外層已完成的 step
  - `parallel` branches 之間不可互相參照

### 型別檢查

- 引擎 SHOULD 進行 CEL 型別檢查（如 boolean 型別的 `condition` 確認回傳 boolean）
- 型別檢查失敗 SHOULD 產生警告（不阻擋驗證，因為某些情況需要動態型別）

---

## 循環參照偵測

- 資料流中不可有循環依賴
- `assign` 不可參照自身（如 `vars.x` = `${ vars.x + 1 }` 在同一 assign step 中）

### Toolset 循環引用偵測

- Toolset 的 `includes` 陣列形成有向圖，引擎 MUST 在載入時偵測循環引用
- 若 toolset A includes B，B includes C，C includes A → validation error
- 不存在的引用（`includes` 引用不存在或非 PUBLISHED 的 toolset）→ validation error
- 詳見 [18-toolset-definition](18-toolset-definition.md)

---

## Schema 一致性

- `input_schema` 和 `output_schema` MUST 為有效的 JSON Schema 子集
- 支援的型別限於：string、number、integer、boolean、array、object
- `required` 中列出的欄位 MUST 存在於 `properties` 中
- 此規則適用於 Workflow、Task、Agent 的 `input_schema` / `output_schema`

---

## Trigger 驗證

- 每個 trigger MUST 有 `type`
- `type` MUST 為 `manual`、`event`、`scheduled`、`http` 之一
- event trigger MUST 有 `event` 欄位
- `when` 欄位（若存在）MUST 為有效的 CEL 表達式
- `input_mapping` 中的每個值（若存在）MUST 為有效的 CEL 表達式或字面值

### scheduled trigger 驗證

- `cron` 與 `interval` MUST 擇一存在，不可同時存在，不可都不存在
- `cron` MUST 為有效的 5 欄位 cron 表達式
- `interval` MUST 為有效的 duration 格式（如 `5m`、`1h`），且 MUST >= 1s
- `timezone`（若存在）MUST 為有效的 IANA timezone 名稱
- `allow_concurrent`（若存在）MUST 為 boolean
- `input` 與 `input_mapping` 不可同時存在
- `input_mapping` 中的值 MUST 為有效的 CEL 表達式（可用 `schedule` namespace）

### http trigger 驗證

- `path` MUST 存在且為非空字串
- `path` MUST 以 `/` 開頭
- `path` 中的 path parameter MUST 使用 `{param_name}` 格式，`param_name` 為 `snake_case`
- 同一 definition 內不可有兩個 http trigger 的 `method` + `path` 完全相同
- 跨 definitions 的 `method` + `path` 衝突在 PUBLISH 時檢查（而非 VALIDATE 時）
- `method`（若存在）MUST 為 `GET`、`POST`、`PUT`、`PATCH`、`DELETE` 之一
- `response.mode`（若存在）MUST 為 `async` 或 `sync`
- `response.timeout`（若存在）MUST 為有效的 duration 格式
- `response.success_status` 和 `response.error_status`（若存在）MUST 為有效的 HTTP status code（100-599）
- `input_mapping` 中的值 MUST 為有效的 CEL 表達式（可用 `request` namespace）

---

## Artifact 宣告驗證

- `artifacts` 中每個 artifact MUST 有 `kind`（`file` 或 `data`）
- `artifacts` 中每個 artifact MUST 有 `lifecycle`（`input`、`output`、`intermediate`）
- `lifecycle: input` 的 artifact 若 `required: true`，CreateInstance 時 MUST 提供
- artifact 名稱 MUST 為有效的 `snake_case` 識別字
- 同一 definition 內 artifact 名稱 MUST 唯一

---

## Task Step Resources 驗證

- `resources` 中每個 resource MUST 有 `name`、`type`、`ref`、`access`
- `resources[].type` MUST 為 `artifact`
- `resources[].ref` MUST 引用 workflow `artifacts` 中已宣告的 artifact 名稱
- `resources[].access` MUST 為 `read`、`write`、`read_write` 之一
- 同一 task step 內 resource `name` MUST 唯一
- 若 task definition 使用 `backend.type: http`，step MUST NOT 定義 `resources`（HTTP backend 不支援 artifact binding）

---

## Task 參照驗證

- `task` step 的 `action` MUST 為非空字串
- `action` MUST 為兩段式命名格式 `namespace.action`（詳見 [06-step-task](06-step-task.md)）
- `version` 若為 integer，MUST 為正整數
- `version` 若為 string，MUST 為 `"latest"`
- 引擎 SHOULD 檢查 `action` 對應的 task definition 是否存在且為 PUBLISHED
- 若 task definition 有 `input_schema`，引擎 SHOULD 檢查 step 的 `input` 與 schema 的相容性（靜態分析）
- Task definition 自身的驗證規則：
  - `apiVersion` MUST 為 `task/v3`
  - `kind` MUST 為 `Task`
  - `metadata.name` MUST 為有效的 dotted namespace（兩段式命名 `namespace.action`）
  - `backend.type` MUST 為 `stdio` 或 `http`（`builtin` 已移除，改用 `type: tool` 搭配兩段式命名，詳見 [14-agent-tools](14-agent-tools.md)）
  - 各 backend type 的必填欄位 MUST 存在（如 stdio 的 `command`、http 的 `url`）

### 兩段式 Tool 命名驗證

- Tool name MUST 為 `namespace.action` 格式（恰好兩段，以 `.` 分隔）
- 每段 MUST 為有效的 `snake_case` 識別字
- 例：`order.load`、`agent.ask_human`、`payment.create`
- 單段或三段以上命名 → validation error
- 此規則適用於 task definition 的 `metadata.name`、agent tool 的 `action`（type: task）、以及 `type: tool` 的 `name`

---

## Agent Step 驗證

- `agent` step 的 `agent` 欄位 MUST 為非空字串（引用 agent definition name）
- `version` 若為 integer，MUST 為正整數
- `version` 若為 string，MUST 為 `"latest"`
- `prompt` MUST 存在且為非空字串（支援 `${ }` CEL 表達式）
- 引擎 SHOULD 檢查 `agent` 對應的 agent definition 是否存在且為 PUBLISHED

### Agent Tool 合併驗證

- `tools` 和 `tools_override` MUST NOT 同時存在 → 同時設定為 validation error
- `tools`（若存在）：與 agent definition 預設 tools 合併
- `tools_override`（若存在）：完全取代 agent definition 預設 tools
- `tools_exclude`（若存在）MUST 為 string 陣列
- 合併後的 tool 列表中不可有重複的 tool name
- 詳見 [16-agent-loop](16-agent-loop.md)

---

## Sub-workflow 驗證

- `workflow` 欄位 MUST 為非空字串
- `version` 若為 integer，MUST 為正整數
- `version` 若為 string，MUST 為 `"latest"`
- 若 version 為固定整數，引擎 SHOULD 檢查對應的 definition 是否存在且為 PUBLISHED
- 若 version 為 `"latest"`，驗證延遲至執行時

---

## Saga 驗證

- `saga` step 的 `steps` MUST 為非空 step 陣列
- `on_failure`（若存在）MUST 為 `abort` 或 `continue`
- `recovery`（若存在）MUST 為 `backward` 或 `forward`
- Saga MUST NOT 巢狀：saga step 內的 steps 不可包含另一個 `type: saga` step
- Saga 內 step 的 `compensate` 區塊 MUST 為有效 step 陣列（通過標準 step 驗證）
- Compensate 區塊內不可包含 `type: saga` step
- 詳見 [12-step-saga](12-step-saga.md)

---

## Timeout 驗證

- `timeout` 值 MUST 為有效的 duration 格式（如 `10s`、`5m`、`1h`、`2h30m`）
- Step timeout SHOULD 小於 workflow timeout（若兩者都定義）— 違反時產生警告
- `wait_event` 的 timeout SHOULD 被定義 — 缺少時產生警告

---

## Secret Definition 驗證

- `apiVersion` MUST 為 `secret/v3`
- `kind` MUST 為 `Secret`
- `metadata.name` MUST 存在且為有效識別字
- `is_encrypted` MUST 為 boolean
- `data` MUST 為 map（key-value pairs）
- 若 `is_encrypted: true`：
  - `encryption` MUST 存在
  - `encryption.type` MUST 為 `aes`
  - `encryption.config.algorithm` MUST 為 `aes-256-gcm` 或 `aes-256-cbc`
  - `encryption.config.iv` MUST 存在且為有效 base64 字串
  - `encryption.config.salt` MUST 存在且為有效 base64 字串
  - 若 `algorithm` 為 `aes-256-gcm`：`encryption.config.tag` MUST 存在且為有效 base64 字串
- 若 `is_encrypted: false`：`encryption` SHOULD NOT 存在

---

## 控制流程 Step 驗證

### if

- `condition` MUST 存在且為有效的 CEL 表達式
- `condition` SHOULD 回傳 boolean（型別檢查為 warning）
- `then` MUST 存在且為非空 step 陣列
- `else`（若存在）MUST 為非空 step 陣列

### switch

- `condition` MUST 存在且為有效的 CEL 表達式
- `cases` MUST 存在且為非空陣列（至少一個 case）
- 每個 `cases[]` MUST 有 `value` 和 `then`
- `cases[].then` MUST 為非空 step 陣列
- `cases[].value` 的型別 SHOULD 與 `condition` 的推斷回傳型別相容（warning）
- `default`（若存在）MUST 為非空 step 陣列

### foreach

- `items` MUST 存在且為有效的 CEL 表達式（SHOULD 回傳 list）
- `do` MUST 存在且為非空 step 陣列
- `as`（若存在）MUST 為有效的識別字（documentation only，不影響 `loop.item` 行為）

---

## 其他驗證

- `foreach` 的 `concurrency` MUST 為正整數
- `foreach` 的 `failure_policy` MUST 為 `fail_fast`、`continue`、`ignore` 之一
- `parallel` 的 `failure_policy` MUST 為 `fail_fast`、`wait_all` 之一
- `parallel` 的 `branches` MUST 至少有兩個分支
- `task` 的 `execution.policy` MUST 為 `replayable`、`idempotent`、`non_repeatable` 之一
- `agent` 的 `execution.policy` MUST 為 `replayable`、`idempotent`、`non_repeatable` 之一
- `retry.max_attempts` MUST 為正整數
- `retry.backoff` MUST 為 `fixed` 或 `exponential`
- `emit` 的 `delay`（若存在）MUST 為有效的 duration 格式，且 MUST > 0s
- `config.secrets`（若存在）MUST 為 string 陣列，每個元素為 secret definition 的 `metadata.name`
- `sub_workflow` 的 `execution.policy` MUST 為 `replayable`、`idempotent`、`non_repeatable` 之一
- `return` 的 `renew`（若存在）MUST 為 boolean
- `return` 的 `version`（若存在且 `renew: true`）MUST 為正整數或 `"latest"`

---

## 驗證錯誤格式

驗證錯誤 SHOULD 包含：

| 欄位 | 說明 |
|------|------|
| `rule` | 違反的規則名稱 |
| `path` | YAML 中的路徑（如 `steps[2].input.order_id`） |
| `step_id` | 相關的 step id（若適用） |
| `message` | 人類可讀的錯誤說明 |
| `severity` | `error`（阻擋）或 `warning`（不阻擋） |

```json
{
  "rule": "step_id_unique",
  "path": "steps[3].id",
  "step_id": "load_order",
  "message": "duplicate step id: load_order (first defined at steps[0])",
  "severity": "error"
}
```

---

## 交叉參照

| 主題 | 文件 |
|------|------|
| Step 類型總覽 | [05-steps-overview](05-steps-overview.md) |
| Task step 與兩段式命名 | [06-step-task](06-step-task.md) |
| 控制流程 step | [08-step-control-flow](08-step-control-flow.md) |
| Saga 與 compensate | [12-step-saga](12-step-saga.md) |
| Agent Definition | [13-agent-definition](13-agent-definition.md) |
| Agent Tools | [14-agent-tools](14-agent-tools.md) |
| Agent Loop | [16-agent-loop](16-agent-loop.md) |
| Toolset Definition | [18-toolset-definition](18-toolset-definition.md) |
| Resources Definition | [19-resources-definition](19-resources-definition.md) |
| Project Definition | [20-project](20-project.md) |
| Lifecycle 狀態 | [23-lifecycle](23-lifecycle.md) |
| Triggers | [26-triggers](26-triggers.md) |
| 版本管理 | [28-versioning](28-versioning.md) |
