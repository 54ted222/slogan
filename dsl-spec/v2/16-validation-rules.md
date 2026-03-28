# 16 — Validation Rules

本文件定義 workflow definition 在發布前 MUST 通過的靜態驗證規則。

---

## 驗證時機

驗證在 definition lifecycle 從 DRAFT → VALIDATED 轉換時執行。所有規則 MUST 通過，否則轉換失敗。

---

## 結構驗證

### 頂層結構

- `apiVersion` MUST 為 `workflow/v2`
- `kind` MUST 為 `Workflow`
- `metadata.name` MUST 存在且為有效的 `snake_case` 識別字
- `metadata.version` MUST 為正整數
- `triggers` MUST 為非空陣列
- `steps` MUST 為非空陣列

### Step 結構

- 每個 step MUST 有 `type`；`id` 為非必填（見 Step ID 驗證）
- `type` MUST 為已定義的 11 種類型之一
- 每種 step type 的必填欄位 MUST 存在（如 task 的 `action`、if 的 `expr`）
- 未知欄位 SHOULD 產生警告

---

## Step ID 驗證

- `id` 為非必填；未指定 id 的 step，引擎自動產生內部 ID（`_<type>_<index>`）
- 使用者定義的 `id` MUST NOT 以 `_` 開頭（避免與自動產生的 ID 衝突）
- 所有使用者定義的 step id MUST 在整個 definition 中全域唯一
- 包含巢狀於 if/switch/foreach/parallel/on_error/on_timeout 內的 steps
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

- 引擎 SHOULD 進行 CEL 型別檢查（如 boolean 型別的 `expr` 確認回傳 boolean）
- 型別檢查失敗 SHOULD 產生警告（不阻擋驗證，因為某些情況需要動態型別）

---

## 循環參照偵測

- 資料流中不可有循環依賴
- `assign` 不可參照自身（如 `vars.x` = `${ vars.x + 1 }` 在同一 assign step 中）

---

## Schema 一致性

- `input.schema` 和 `output.schema` MUST 為有效的 JSON Schema 子集
- 支援的型別限於：string、number、integer、boolean、array、object
- `required` 中列出的欄位 MUST 存在於 `properties` 中

---

## Trigger 驗證

- 每個 trigger MUST 有 `type`
- `type` MUST 為 `manual` 或 `event`
- event trigger MUST 有 `event` 欄位
- `when` 欄位（若存在）MUST 為有效的 CEL 表達式
- `input_mapping` 中的每個值（若存在）MUST 為有效的 CEL 表達式或字面值

---

## Task 參照驗證

- `task` step 的 `action` MUST 為非空字串
- `version` 若為 integer，MUST 為正整數
- `version` 若為 string，MUST 為 `"latest"`
- 引擎 SHOULD 檢查 `action` 對應的 task definition 是否存在且為 PUBLISHED
- 若 task definition 有 `input.schema`，引擎 SHOULD 檢查 step 的 `input` 與 schema 的相容性（靜態分析）
- Task definition 自身的驗證規則：
  - `apiVersion` MUST 為 `task/v2`
  - `kind` MUST 為 `Task`
  - `metadata.name` MUST 為有效的 dotted namespace
  - `backend.type` MUST 為 `bash`、`http`、`builtin`、`sdk` 之一
  - 各 backend type 的必填欄位 MUST 存在（如 bash 的 `command`、http 的 `url`、sdk 的 `module`、builtin 的 `handler`）

---

## Sub-workflow 驗證

- `workflow` 欄位 MUST 為非空字串
- `version` 若為 integer，MUST 為正整數
- `version` 若為 string，MUST 為 `"latest"`
- 若 version 為固定整數，引擎 SHOULD 檢查對應的 definition 是否存在且為 PUBLISHED
- 若 version 為 `"latest"`，驗證延遲至執行時

---

## Timeout 驗證

- `timeout` 值 MUST 為有效的 duration 格式（如 `10s`、`5m`、`1h`、`2h30m`）
- Step timeout SHOULD 小於 workflow timeout（若兩者都定義）— 違反時產生警告
- `wait_event` 的 timeout SHOULD 被定義 — 缺少時產生警告

---

## 其他驗證

- `foreach` 的 `concurrency` MUST 為正整數
- `foreach` 的 `failure_policy` MUST 為 `fail_fast`、`continue`、`ignore` 之一
- `parallel` 的 `failure_policy` MUST 為 `fail_fast`、`wait_all` 之一
- `parallel` 的 `branches` MUST 至少有兩個分支
- `task` 的 `execution.policy` MUST 為 `replayable`、`idempotent`、`non_repeatable` 之一
- `retry.max_attempts` MUST 為正整數
- `retry.backoff` MUST 為 `fixed` 或 `exponential`

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
