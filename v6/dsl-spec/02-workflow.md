# 02 — Workflow Definition

本文件定義 `kind: Workflow` 的頂層 YAML 結構、觸發機制與 workflow 級設定。

---

## 頂層欄位

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `apiVersion` | string | MUST | 固定值 `workflow/v6` |
| `kind` | string | MUST | 固定值 `Workflow` |
| `metadata` | object | MUST | 識別與描述資訊 |
| `triggers` | array | MUST | 啟動機制，至少一個 |
| `input_schema` | object | MAY | 輸入 JSON Schema |
| `output_schema` | object | MAY | 輸出 JSON Schema |
| `config` | object | MAY | workflow 級設定 |
| `steps` | array | MUST | 主要步驟序列（非空；載入期拒絕空陣列） |

---

## triggers

Trigger 的職責是**建立新的 workflow instance**。一個 workflow MAY 定義多個 triggers。

### manual — 手動觸發

```yaml
triggers:
  - type: manual
```

透過 API 或 CLI 手動啟動，呼叫端提供 input 資料。

規則：

- 同一 workflow 的 `triggers[]` 中 **至多一個** `manual` trigger；多個 → 載入失敗，`registry.duplicate_manual_trigger`（manual 無可區分參數，多個沒有語意）
- Manual trigger 不經 event bus；API / CLI 呼叫直接建立 instance
- API 呼叫端可附 `Idempotency-Key` header 實現重試去重（key → instance_id 映射，TTL 綁 instance 生命週期 + retention，見 `runtime-spec/07-event-bus.md` 的「Idempotency 映射的保留期」）；同 key 第二次呼叫回傳既有 instance 而非新建
- `Idempotency-Key` 僅於 manual trigger 有效；event trigger 走 `(workflow_name, workflow_version, event.id)` 去重（見 `runtime-spec/07-event-bus.md`）
- CLI 呼叫省略 `Idempotency-Key` → 每次呼叫建立新 instance（不去重）

### event — 事件觸發

```yaml
triggers:
  - type: event
    event: order.created
    scope: project              # MAY, 預設 project — workflow | project | global
    when: ${ event.data.source == "api" }
    input_mapping:
      order_id: ${ event.data.order_id }
      action: ${ event.data.type }
```

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `event` | string | MUST | 要訂閱的事件類型 |
| `scope` | string | MAY | 僅匹配指定 scope 的事件；`project`（預設）/`global`；`workflow` 明確**拒絕**並於載入期回報 `registry.invalid_trigger_scope` |
| `when` | CEL expression | MAY | 過濾條件，回傳 boolean |
| `input_mapping` | map | MAY | 事件資料到 workflow input 的映射 |

#### 多 trigger 同時匹配

若同一事件（相同 `event.id`）同時匹配**同一 workflow** 的多個 event trigger（不同 `when` / `scope` / `input_mapping`）：

- 引擎**僅建立一個 instance**；選用 `triggers[]` 陣列中**通過 `scope` 與 `when` 過濾後索引最小**的那個 trigger 的 `input_mapping` 與 `scope` 規則執行
- **選取順序**：先依 triggers[] 索引由小至大列出 event / manual 型 trigger；對每個 trigger 依序判定 scope 匹配、`when` 求值 → 第一個通過者即為「被選中 trigger」；其餘不再求值（避免副作用與 CEL 開銷）
- **去重與多匹配的原子性**：去重檢查（步驟 2）以 `(workflow_name, workflow_version, event.id)` 為 key，**per-event 執行一次**（非 per-trigger）。流程為：
  1. 先依上述順序找到「被選中 trigger」（未通過任一 trigger → 事件被丟棄，不進 dedup 表，發 `trigger.skipped` observability 事件）
  2. 找到被選中 trigger 後，以 INSERT `(workflow_name, workflow_version, event.id)` 至 trigger_dedup 表並以 unique 衝突判定去重；INSERT 成功 → 進入 input_mapping；INSERT 衝突 → 視為重複投遞，丟棄事件（ack）、不建立 instance
  - 此順序確保「未被選中的 trigger 不佔用 dedup slot」；後續同 `event.id` 若因 trigger 定義變更匹配到其他 trigger，仍會被 dedup 表擋下（設計意圖：以 event.id 為去重鍵，不因 trigger 版本差異重複觸發）
- 未被選中的 trigger 不執行；不發出警告（由使用者自行避免冗餘 trigger）
- 此選擇寫入 `instance.triggered_by_trigger_index`（便於追蹤）
- 跨**不同 workflow** 的 trigger 匹配（即使同 event）→ 各自獨立建立 instance（見現有「at-least-once」與去重規則）；同一事件在不同 workflow 之間不去重

#### Trigger 處理順序

每個匹配的事件通過以下固定順序處理（任一失敗即停止並進入對應失敗路徑）：

```
1. 選取 trigger：依 triggers[] 索引由小至大，對每個事件型 trigger 判定 scope 匹配與 when CEL；
   第一個通過者為「被選中 trigger」；皆未通過 → 丟棄事件（不進 dedup、不建立 instance）
2. 去重檢查 + insert（trigger_dedup 表以 (workflow_name, workflow_version, event.id) 為 key 做 INSERT；
   衝突 → 丟棄事件 ack）
3. input_mapping CEL 求值（namespace: event / env / secret / project；產出 input 候選值）
4. workflow.input_schema 驗證（對 input 候選值）
5. 建立 instance（寫入 action_pins / trace_id / labels snapshot / triggered_by_trigger_index / 等）
6. 發 instance.created 事件
```

- 步驟 1 的 `when` 與步驟 3 的 `input_mapping` CEL namespace **不含** `input` —— input 在步驟 4 通過驗證後才成為 instance 的 input；`input_mapping` 中引用 `input.*` → `expression_error.identifier_not_found`
- 同一 trigger 內多個 `input_mapping` key 間彼此獨立求值（不可互相引用；需要二階段組裝時請在 workflow 內以 `assign` 處理）
- 步驟 5 若因 size limit 等其他原因失敗 → `workflow.instance_create_failed`；事件已 ack、dedup 已 commit，進 dead letter

#### input_mapping 失敗場景

`input_mapping` 的 CEL 與後續 schema 驗證為 trigger 建立 instance 前的步驟；任一失敗皆**不建立 instance**，但錯誤碼與事件消費行為各異：

| 失敗場景 | 錯誤碼 | 事件 ack 行為 |
|---------|-------|---------------|
| `when` CEL 求值異常 | `trigger.when_eval_failed` | 視為過濾未通過（ack 消費）；對 ops 發 `trigger.skipped` 觀測事件 |
| `input_mapping` 中某 CEL 求值異常 | `trigger.input_mapping_error`、`error.details.field` 為失敗欄位 | 事件 ack 消費（避免 poison-pill 重放）；進 `bus.dead_letter`（`failure_reason: "trigger_input_mapping_error"`） |
| 映射結果不符 `input_schema` | `workflow.input_schema_violation`（既有語意） | 同上（ack 消費 + dead letter） |

- Trigger **無** `catch` 機制；上列失敗皆不建立 instance，無法被 workflow 層攔截。若需對映射失敗做告警，請訂閱 `bus.dead_letter` 事件
- 實作 SHOULD 暴露 metric：`trigger_failed_total{workflow, failure_reason}`
- 事件本身的 `event.id` 會寫入 trigger 去重表（見 `runtime-spec/07-event-bus.md`），避免同一事件反覆因映射失敗而觸發重試

**scope 匹配規則**：

- `scope: workflow` 的事件由發出者 instance 自身消費（透過 short-circuit 路徑），**不進入 trigger 匹配**，因此 trigger 不可宣告 `scope: workflow`
- `scope: project` trigger：只匹配**同 project** 的 `scope: project` 事件（跨 project 即使名稱相同亦不匹配）
- `scope: global` trigger：匹配任何 project 的 `scope: global` 事件
- 預設 `project` 是因為最常見的跨 workflow 協作仍限於同 project；需要跨 project 協作 MUST 顯式宣告 `scope: global` 並在發出端以 `scope: global` emit

---

## input_schema / output_schema

以 JSON Schema 定義 workflow 的輸入輸出結構。驗證時機：

| 階段 | 時機 | 失敗處理 |
|------|------|----------|
| `input_schema` | trigger 欲建立 instance 時、入庫前 | 拒絕建立 instance；trigger 回報錯誤 `workflow.input_schema_violation`，詳情含違反欄位路徑 |
| `output_schema` | `type: return` 求值其 `output` 後、寫回前 | workflow 以 FAILED 結束，`error.type == "output_schema_violation"` |

驗證器 MUST 使用嚴格模式（未定義欄位視為違反 `additionalProperties: false`，除非 schema 顯式允許）。

### JSON Schema 版本與子集

- Engine 採 **JSON Schema Draft 2020-12** 為基準
- 支援關鍵字：`type` / `properties` / `required` / `additionalProperties` / `items` / `enum` / `const` / `minimum` / `maximum` / `minLength` / `maxLength` / `pattern` / `default` / `format`（僅 `date-time` / `email` / `uri` 常用幾種，engine 文件 SHOULD 列出完整支援清單）
- **不支援**（v6 延後）：`$ref` / `$defs` / `allOf` / `anyOf` / `oneOf` / `not` / dynamic schema composition；使用者若於 schema 中宣告這些 keyword → 載入期拒絕，`error.type: "registry.unsupported_schema_keyword"`、`details.keyword: "<keyword_name>"`
- 不支援的 `format` 值：engine 實作 SHOULD 記錄 warning 但不 fail（draft 2020-12 中 format 預設為 annotation 而非驗證）
- 未來版本 MAY 放寬至完整 Draft 2020-12 或擴充 `$ref` 支援；v6 以簡單 schema 為主

### JSON Schema `default` 套用規則

- 驗證器 MUST 在驗證 **之前** 以遞迴方式套用 `default`：若 input 缺少某欄位（key 不存在）**且** schema 對該欄位宣告 `default`，則補入 default 值
- `null` 被視為**顯式值**，不觸發 default 套用（即 `{"x": null}` 不會被補為 default；若 schema 要求 non-null，驗證失敗）
- Default 套用於整個巢狀結構：例如 schema `{type: object, properties: {cfg: {type: object, properties: {retries: {type: integer, default: 3}}, default: {}}}}`，input `{}` 套用後為 `{cfg: {retries: 3}}`
- 套用順序：先於當前層級套用 default 補齊，再遞迴對 children 套用
- 套用後的 input **即為 instance 的最終 input**（寫入 `instances.input`；後續 CEL 讀 `input.cfg.retries` 取得 `3`）
- Tool `input_schema` 套用 default 的規則相同；套用在「task step 進入 RUNNING、CEL 求值後、backend 驗證前」。CEL 求 null 視為顯式值，不套 default

### 驗證層級分工

Schema 驗證分三個獨立層級，各司其職、不互相代勞：

| 層級 | Schema 來源 | 驗證時機 | 失敗行為 |
|------|-------------|----------|----------|
| Workflow instance input | `workflow.input_schema` | trigger 建立 instance 前 | 拒絕建立 |
| Workflow instance output | `workflow.output_schema` | 終結 `return` 前 | instance FAILED |
| Function instance input / output | `function.input_schema` / `function.output_schema` | input：function instance `PENDING → RUNNING` 前（與 workflow input 驗證同層級，由子 instance 自己執行，**不**由 caller task step 代驗）；output：子 instance `return` 前 | 子 instance FAILED，`error.type == "schema_violation"`；呼叫端 task step 以此 failure 呈現並可被 catch 捕捉。caller task step 的 `input:` map CEL 求值失敗 `expression_error` 於建立 function instance **前** 即返回，不進入 function 層驗證 |
| Task step 的 tool input | 被呼叫的 Tool definition 的 `input_schema` | step 進入 RUNNING、CEL 求值後、呼叫 backend 前 | step FAILED，`error.type == "schema_violation"`、`error.details.direction == "input"` |
| Task step 的 tool output | 被呼叫的 Tool definition 的 `output_schema` | backend 回傳、寫入 checkpoint 前 | step FAILED，`error.type == "schema_violation"`、`error.details.direction == "output"` |
| Callback input / output | `callback.input_schema` / `callback.output_schema`（見 `05b-function.md`） | callback 傳入 caller 前 / handler return 後 | callback_result 以 `error.type == "schema_violation"` 回傳 |

驗證原則：

- **CEL 求值結果即視為最終型別**；驗證器不對 CEL 輸出做自動強制轉換（`int(x)` 之類的強制轉換須由使用者顯式呼叫）。例：若 schema 要求 `integer` 而 CEL 傳入 `3.0`（double），驗證失敗而非靜默轉型。
- Task step 的 `input:` map 不通過 workflow 級 `input_schema`，只受 Tool 的 `input_schema` 檢查。
- Workflow 的 `input_schema` **不** 連鎖套用到任何 step；子 step 對 `input.*` 的引用只保證「CEL namespace 可解析」，不額外驗證。
- Tool output 僅在 task step 層驗證，不再於 workflow `output_schema` 重驗（除非該 output 被 `type: return` 引用）。

```yaml
input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

output_schema:
  type: object
  properties:
    status:
      type: string
```

---

## config

Workflow 級設定，所有欄位皆為可選。

| 欄位 | 型別 | 說明 |
|------|------|------|
| `timeout` | duration \| CEL→duration string | workflow instance 的最長執行時間；支援 CEL，於 `PENDING → RUNNING` 前求值一次並持久化（見下） |
| `max_step_executions` | integer | step 執行次數上限（防止無限迴圈） |
| `cancellation_policy` | enum | `graceful`（預設）/ `hard`；見 `runtime-spec/02-instance-lifecycle.md` |
| `cancel_grace_period` | duration | graceful 模式的寬限期，預設 30s |
| `catch` | catch 陣列 | workflow 級錯誤處理（含 timeout） |
| `secrets` | string 陣列 | 依賴的 secret 名稱列表；載入期驗證所有名稱存在於當前 project scope，缺失 → 載入失敗 `registry.missing_secret`；運行時不額外檢查（CEL 引用未宣告的 secret 仍可解析，但建議開發者盡量宣告以便 CI/CD 檢查） |

#### config.timeout / cancel_grace_period 的 CEL 求值

- 兩欄位皆支援 CEL（如 `timeout: ${ input.max_duration }`）；載入期僅做語法檢查，不求值
- 求值時機：`PENDING → RUNNING` transition 前，與其他 instance-level 欄位（`cancellation_policy`）同一批次
- 求值 namespace **限** `input` / `env` / `secret` / `project`（instance 尚未執行任何 step，無 `steps` / `vars` / `prev` / `loop`）；引用受限 namespace → instance 直接 FAILED，`error.type == "expression_error.identifier_not_found"`
- 求值結果 MUST 為符合 Duration 格式的 string；否則 instance FAILED，`error.type == "invalid_duration_format"`
- 結果寫入 instance checkpoint（`timeout_resolved_ms` 欄位）；engine restart 後不重求值，沿用 checkpoint 值
- Function instance 的 `config.timeout` 求值時機相同（建立 function instance 前）；namespace 為子 instance 的 `input` 與 inherited `secret` / `project`

```yaml
config:
  timeout: 2h
  max_step_executions: 1000
  catch:
    - type: fail
      when: ${ error.type == "timeout" }
      message: "workflow timeout exceeded"
    - type: emit
      event: workflow.failed
      data:
        error: ${ error.message }
```

---

## 完整結構範例

### 最小結構

```yaml
apiVersion: workflow/v6
kind: Workflow

metadata:
  name: hello
  version: 1

triggers:
  - type: manual

steps:
  - type: task
    action: system.echo
    input:
      message: "hello world"
```

### 完整結構

```yaml
apiVersion: workflow/v6
kind: Workflow

metadata:
  name: order_fulfillment
  version: 3
  description: "訂單履行流程"
  labels:
    team: commerce

triggers:
  - type: manual
  - type: event
    event: order.created
    when: ${ event.data.source == "api" }

input_schema:
  type: object
  properties:
    order_id:
      type: string
  required: [order_id]

output_schema:
  type: object
  properties:
    status:
      type: string

config:
  timeout: 2h

steps:
  - id: load_order
    type: task
    action: order.load
    input:
      order_id: ${ input.order_id }

  - type: return
    output:
      status: "completed"
```
