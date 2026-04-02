# 11 — Step: sub_workflow

`sub_workflow` 步驟呼叫另一個 workflow definition，建立子 instance 並等待其完成。

---

## 設計動機

- **組合復用**：將常見的子流程（如付款、通知）封裝為獨立的 workflow definition
- **關注分離**：子流程有獨立的 input/output schema、lifecycle、版本
- **獨立測試**：子 workflow 可獨立執行與測試

---

## Schema

```yaml
- id: string                          # MAY — 需要參照 output 時才必填
  type: sub_workflow                   # MUST
  workflow: string                     # MUST — 要呼叫的 workflow definition name
  version: integer | "latest"         # MAY, 預設 "latest"
  input: map | CEL expression         # MAY — 傳給子 workflow 的輸入
  timeout: duration                   # MAY
  execution_policy: replayable | idempotent | non_repeatable  # MAY, 預設 replayable
  retry:
    max_attempts: integer             # MAY
    delay: duration                   # MAY
    backoff: fixed | exponential      # MAY
  catch: [...]                     # MAY, step 陣列
  on_timeout: [...]                   # MAY, step 陣列
  compensate: [...]                   # MAY, step 陣列 — saga 補償邏輯（見 [12-step-saga](12-step-saga.md)）
```

---

## 執行語意

1. Parent step 進入 RUNNING 狀態
2. 引擎建立 **child workflow instance**：
   - Definition = `workflow` 指定的 workflow definition
   - Version = `version` 指定的版本
   - Input = `input` 求值的結果
3. Child instance 獨立執行（擁有自己的 step instances、state）
4. Parent step 等待 child instance 完成
5. Child SUCCEEDED → parent step SUCCEEDED，`steps.<id>.output` = child 的 return output
6. Child FAILED → parent step FAILED（可被 `catch` 捕捉）

---

## 資料隔離

Child workflow **不可** 存取 parent 的任何 namespace：

| Namespace | Parent → Child | Child → Parent |
|-----------|---------------|----------------|
| `input` | 透過 `input` 傳入 ✅ | — |
| `steps` | ❌ | 透過 return output ✅ |
| `vars` | ❌ | ❌ |
| `loop` | ❌ | ❌ |
| `artifacts` | 透過 input 傳遞路徑 ✅ | — |

這是硬性邊界，確保子 workflow 的可組合性與獨立性。

### Artifact 傳遞方式

Parent 與 child 共用同一 workspace 目錄。Parent 透過 `input` 傳遞 artifact 的 workspace 路徑，child 可直接存取：

```yaml
# Parent workflow
- id: process
  type: sub_workflow
  workflow: file_processor
  input:
    file_path: ${ artifacts.order_file.path }
```

Child workflow 透過 `input.file_path` 直接讀寫 parent workspace 中的檔案。

Child workflow 完成後若需將結果傳回 parent，MUST 透過 `return` step 的 output 傳遞路徑或內容。

---

## 版本解析

| 值 | 說明 |
|----|------|
| `"latest"` | 執行時解析為最新的 PUBLISHED 版本 |
| `integer` | 固定版本號 |

- 預設為 `"latest"`
- 在正式環境中 SHOULD 使用固定版本號，確保可重現性
- 若指定版本不存在或非 PUBLISHED 狀態 → step FAILED

---

## Execution Policy 語意

Sub_workflow step 的 `execution.policy` 決定 crash recovery 時 parent step 的行為：

| Policy | Recovery 行為 |
|--------|---------------|
| `replayable`（預設） | 查詢 child instance 狀態：RUNNING → 恢復等待；已完成 → 直接取結果；不存在或狀態不明 → 建立新的 child instance |
| `idempotent` | 同 replayable，但以 idempotency key（`parent_instance_id + step_id + attempt`）確保不重複建立 child instance |
| `non_repeatable` | 若 child instance 狀態不明確（非 terminal）→ parent step 標記 FAILED |

在大多數情況下，child instance 的狀態可從資料庫查詢，因此 recovery 通常只需恢復等待而非重新執行。`non_repeatable` 適用於 child workflow 有不可逆副作用且狀態無法確認的情境。

---

## 巢狀深度限制

- 子 workflow MAY 再呼叫 sub_workflow（遞迴 / 間接遞迴）
- 引擎 MUST 設定最大巢狀深度限制（建議上限 5 層，與 agent step 共用深度計數）
- 超過深度限制 → step FAILED（錯誤碼 `max_depth_exceeded`）

---

## 錯誤傳播

Child workflow 的失敗資訊可在 parent 的 `catch` handler 中存取：

| 變數 | 說明 |
|------|------|
| `error.message` | child workflow 的 fail message |
| `error.code` | child workflow 的 fail code |
| `error.step_id` | parent 中的 sub_workflow step id |

---

## Timeout 行為

- Parent step 的 `timeout` 是 sub_workflow 的最長執行時間
- 若 parent step timeout 觸發 → child instance 被 CANCELLED
- Child 的 workflow 級 `on_timeout` 不會被觸發（因為是被外部取消，非自身 timeout）
- Parent 的 `on_timeout` handler 觸發

### Parent Step Timeout 與 Child Workflow Timeout 的優先權

當兩者同時設定時：

| 情境 | 行為 |
|------|------|
| Parent step timeout < child `config.timeout` | Parent step timeout 先觸發 → child 被 CANCELLED |
| Parent step timeout > child `config.timeout` | Child `config.timeout` 先觸發 → child 自身 FAILED → parent step FAILED |
| 兩者相等 | 哪個先觸發視為 race condition，引擎 MUST 保證只有一個贏家 |

Parent step timeout 是**外部取消**，child `config.timeout` 是**自身 timeout**。差異在於 child 的 `config.on_timeout` handler：

- 外部取消（parent timeout）→ child `config.on_timeout` **不觸發**
- 自身 timeout（child config.timeout）→ child `config.on_timeout` **觸發**（若有）

---

## Child Instance 生命週期管理

### 取消級聯

Parent instance 被取消時，引擎 MUST 級聯取消所有 child instances：

```
Parent CANCELLED
  ↓
查詢 parent_instance_id = parent.id 的所有非 terminal child instances
  ↓
逐一取消（與 API cancel 相同行為）
  ↓
Child instances → CANCELLED
  ↓
Child 的 child（孫 instance）→ 遞迴取消
```

### 級聯規則

| 規則 | 說明 |
|------|------|
| 深度 | 遞迴取消所有後代 instances，不限深度 |
| 順序 | 先取消最深層（leaf），再向上取消 |
| 原子性 | 單一 child 取消失敗不影響其他 child 的取消 |
| 已完成的 child | 已為 terminal 狀態的 child 不受影響 |

### Child Instance 清理與保留

Child instance 的保留策略與一般 instance 相同（見 [runtime-spec/08-storage-schema](../../runtime-spec/08-storage-schema.md) 資料保留策略）。

| 情境 | 保留行為 |
|------|---------|
| Parent SUCCEEDED | Child instances 保留至保留期限到期 |
| Parent FAILED | 同上 |
| Parent CANCELLED | 同上（child 也已被級聯取消） |

引擎 MAY 將 child instance 的保留期限與 parent 綁定：parent 被清理時，一併清理所有 child instances。

### 孤兒 Instance

若 parent instance 被清理（超過保留期限刪除）但 child instance 尚存在，child instance 成為「孤兒」。引擎 SHOULD 定期掃描並清理孤兒 instances。

判斷孤兒的條件：`parent_instance_id` 不為 null，且對應的 parent instance 已不存在。

---

## 範例

### 基本呼叫

```yaml
- id: process_payment
  type: sub_workflow
  workflow: payment_processing
  version: 3
  input:
    order_id: ${ steps.load_order.output.id }
    amount: ${ steps.load_order.output.amount }
  timeout: 5m
```

### 帶錯誤處理

```yaml
- id: process_payment
  type: sub_workflow
  workflow: payment_processing
  input:
    order_id: ${ steps.load_order.output.id }
    amount: ${ steps.load_order.output.amount }
  timeout: 5m
  execution_policy: non_repeatable
  catch:
    - type: emit
      event: payment.failed
      data:
        order_id: ${ steps.load_order.output.id }
        error: ${ error.message }
    - type: fail
      message: ${ "payment failed: " + error.message }
  on_timeout:
    - type: fail
      message: "payment sub-workflow timeout"
```

### 被呼叫的子 workflow

```yaml
apiVersion: workflow/v3
kind: Workflow

metadata:
  name: payment_processing
  version: 3

triggers:
  - type: manual

input_schema:
  type: object
  properties:
    order_id:
      type: string
    amount:
      type: number
  required: [order_id, amount]

output_schema:
  type: object
  properties:
    payment_id:
      type: string
    status:
      type: string
  required: [payment_id, status]

steps:
  - id: create_payment
    type: task
    action: payment.create
    input:
      order_id: ${ input.order_id }
      amount: ${ input.amount }
    execution_policy: non_repeatable
    timeout: 30s

  - id: wait_confirmed
    type: wait
    event: payment.confirmed
    match: ${ event.data.payment_id == steps.create_payment.output.payment_id }
    timeout: 30m
    on_timeout:
      - type: fail
        message: "payment confirmation timeout"

  - type: return
    output:
      payment_id: ${ steps.create_payment.output.payment_id }
      status: "confirmed"
```
