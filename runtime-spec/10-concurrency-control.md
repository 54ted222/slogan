# 10 — Concurrency Control

本文件定義全域並發控制機制：rate limiting、per-task-type 並發上限、backpressure 策略、以及與 step 級 concurrency 的交互。

---

## 概述

並發控制分為兩個層級：

| 層級 | 位置 | 控制對象 |
|------|------|---------|
| **Step 級** | `foreach.concurrency` | 單一 foreach step 內的迭代並行度 |
| **全域** | Engine 設定 | 跨所有 instances 的 task 執行並行度 |

Step 級設定見 [dsl-spec/v2/08-step-control-flow](../dsl-spec/v2/08-step-control-flow.md)。本文件定義全域層級。

---

## 全域並發設定

引擎 SHOULD 支援以下全域並發設定：

### Engine 級設定

| 設定 | 型別 | 預設 | 說明 |
|------|------|------|------|
| `concurrency.max_running_steps` | integer | 無限制 | 同時 RUNNING 的 step instances 上限（跨所有 workflow instances） |
| `concurrency.max_running_instances` | integer | 無限制 | 同時 RUNNING / WAITING 的 workflow instances 上限 |

### Per-Task-Type 並發設定

引擎 SHOULD 支援對特定 task action 設定並發上限：

```yaml
# 引擎設定（非 DSL 語法，由部署設定提供）
concurrency:
  max_running_steps: 100
  task_limits:
    - action: "payment.create"
      max_concurrent: 5
    - action: "email.send"
      max_concurrent: 10
```

| 設定 | 說明 |
|------|------|
| `action` | Task definition 的 `metadata.name`，支援 `*` 萬用字元（如 `payment.*`） |
| `max_concurrent` | 該 action 同時 RUNNING 的 step instances 上限 |

---

## 排隊機制

當並發上限到達時，新的 step MUST 排隊等待，不可直接失敗。

### 狀態轉換

排隊中的 step 維持 `READY` 狀態（不引入新狀態），與 `dsl-spec/v2/05-steps-overview.md` 及 `runtime-spec/08-storage-schema.md` 的 step lifecycle 一致。

```
READY（排隊等待）──→ READY（獲得執行資格）──→ RUNNING
       │
       └──→ CANCELLED（instance 被取消時）
```

引擎 MUST 在內部區分「等待排隊」與「可立即執行」的 READY step，確保排隊中的 step 不被執行。實作方式由引擎決定（如內部 queue、排隊時間戳等），但 step_instance 的 `state` 欄位 MUST 為 `READY`。

### 排隊規則

| 規則 | 說明 |
|------|------|
| 排隊順序 | FIFO（先到先執行） |
| 排隊上限 | 引擎 SHOULD 設定排隊上限（建議 1000），超過時新 step → FAILED |
| 排隊 timeout | 引擎 SHOULD 支援排隊 timeout（建議預設無限），超過時 step → TIMED_OUT |

### 排隊對 Timeout 的影響

Step 的 `timeout` 計時從 step 進入 **RUNNING** 狀態開始，不包含排隊時間。Workflow 的 `config.timeout` 計時從 instance 建立開始，**包含**排隊時間。

---

## Backpressure 策略

當系統整體負載過高時，引擎 SHOULD 提供 backpressure 機制：

### Instance 建立的 Backpressure

| 條件 | 行為 |
|------|------|
| Running instances 到達 `max_running_instances` | 新建 instance 保持 CREATED 狀態，不轉為 RUNNING。引擎在有空位時自動啟動 |
| Event trigger 造成的 instance 建立 | 同上，instance 建立但延遲啟動 |
| API 呼叫建立 instance | 回傳成功（instance 已建立），但 instance 排隊等待啟動 |

### Step 執行的 Backpressure

| 條件 | 行為 |
|------|------|
| Running steps 到達 `max_running_steps` | READY step 排隊等待 |
| 特定 action 到達 `max_concurrent` | 該 action 的 step 排隊，其他 action 的 step 不受影響 |

---

## 與 `foreach.concurrency` 的交互

`foreach.concurrency` 和全域並發限制**同時生效**，取較嚴格的限制：

```
foreach concurrency = 10
task_limits[action].max_concurrent = 3

→ 雖然 foreach 允許 10 個迭代同時執行，
  但因為全域限制為 3，所以實際最多 3 個同時執行
```

### 交互規則

1. Foreach 產生的迭代 step 先受 `foreach.concurrency` 限制
2. 通過 foreach 限制的 step 再受全域限制
3. 全域限制造成的排隊**不算作** foreach 的並行 slot（foreach 可以繼續派出新迭代進入排隊）

---

## 監控指標

引擎 SHOULD 暴露以下並發相關指標（見 [11-observability](11-observability.md)）：

| 指標 | 說明 |
|------|------|
| `running_steps_total` | 當前 RUNNING 的 step 數量 |
| `queued_steps_total` | 當前排隊中的 step 數量 |
| `running_instances_total` | 當前 RUNNING/WAITING 的 instance 數量 |
| `task_concurrent_usage` | 每個 action 的當前並發使用量 |
