# 02 — Scheduler

本文件定義 Scheduler 的行為：驅動 workflow instance 的執行、step 排程、狀態轉換。

---

## 職責

Scheduler 是引擎的核心驅動迴圈，負責：

1. 接收觸發信號（新 instance、事件到達、timeout 觸發）
2. 找到可執行的 steps（READY 狀態）
3. 分派 step 至 Step Executor 執行
4. 根據執行結果更新狀態並推進後續 steps

---

## 觸發信號

Scheduler 接收以下信號開始工作：

| 信號 | 來源 | 說明 |
|------|------|------|
| `instance_created` | Engine API / Event Router | 新 instance 建立，需開始排程 |
| `event_matched` | Event Router | 等待中的 instance 收到匹配事件 |
| `timeout_fired` | Timeout Manager | Step 或 workflow timeout 觸發 |
| `step_completed` | Step Executor | Step 執行完成，需推進後續 steps |
| `recovery` | Engine 啟動 | Crash recovery，需恢復中斷的 instances |

---

## 主迴圈

```
接收觸發信號
     ↓
載入 instance 狀態（從 DB）
     ↓
Acquire instance lease（多 worker 時）
     ↓
判斷下一步動作
     ├── instance CREATED → 初始化 step instances，推進第一個 step
     ├── step 完成 → 推進後續 step
     ├── event 匹配 → 恢復 wait_event step
     ├── timeout 觸發 → 執行 timeout 處理
     └── recovery → 依 recovery 規則恢復
     ↓
執行動作（透過 Step Executor）
     ↓
更新狀態（原子寫入 DB）
     ↓
釋放 instance lease
     ↓
觸發後續信號（若有）
```

---

## Instance 初始化

當 instance 從 CREATED 轉為 RUNNING 時，Scheduler MUST：

1. 將 instance state 更新為 RUNNING，記錄 `started_at`
2. 解析 workflow definition，為 `steps` 陣列中的每個頂層 step 建立 step_instance record：
   - 第一個 step：state = READY
   - 其餘 steps：state = PENDING
3. 為省略 `id` 的 steps 產生自動 ID（格式：`_<type>_<index>`）
4. 將第一個 step 交由 Step Executor 處理

### 巢狀 Step Instances

巢狀 steps（if/switch/foreach/parallel 內部）的 step_instance 在**父容器 step 進入 RUNNING 時**才建立，不在初始化階段建立。這確保：

- SKIPPED 分支不會建立無用的 step instances
- foreach 的迭代次數在執行前未知

---

## Step 推進規則

當一個 step 到達 terminal 狀態後，Scheduler 判斷下一步：

### 正常推進

| 當前 step 結果 | 動作 |
|---------------|------|
| SUCCEEDED | 同序列中下一個 step → READY |
| SKIPPED | 同序列中下一個 step → READY |
| FAILED（已被 handler 處理） | 按繼續點規則推進（見 [dsl-spec/v2/12-error-handling](../dsl-spec/v2/12-error-handling.md)） |
| TIMED_OUT（已被 handler 處理） | 同上 |

### 序列結束

| 情況 | 動作 |
|------|------|
| 頂層 `steps` 最後一個 step 完成 | 隱式完成：若有 `output.schema` 且含 `required` 欄位 → Instance → FAILED（`schema_validation_error`）；否則 Instance → SUCCEEDED，output = `null` |
| 遇到 `return` step | Instance → SUCCEEDED，記錄 output |
| 遇到 `fail` step（非 handler 內） | Instance → FAILED，記錄 error |

### 錯誤推進

| 情況 | 動作 |
|------|------|
| Step FAILED，有 step-level `on_error` | 建立 handler step instances 並執行 |
| Step TIMED_OUT，有 `on_timeout` | 建立 handler step instances 並執行 |
| 錯誤未被處理 | 向上層尋找 handler（block → workflow） |
| 所有層級無 handler | Instance → FAILED |

---

## 容器 Step 排程

### if / switch

1. 父 step → RUNNING
2. 求值 `expr`
3. 根據結果選擇分支
4. 為選中分支的 steps 建立 step instances（state = READY / PENDING）
5. 未選中分支的 steps 建立 step instances（state = SKIPPED）
6. 開始執行選中分支
7. 分支執行完畢 → 父 step → SUCCEEDED（或 SKIPPED，若無匹配分支）

### foreach

1. 父 step → RUNNING
2. 求值 `items`
3. 若 items 為空 → 父 step → SUCCEEDED，output = `[]`
4. 根據 `concurrency` 設定，批次建立迭代的 step instances
5. 每個迭代擁有獨立的 `loop.item` / `loop.index` 上下文
6. 迭代完成 → 收集 output array
7. 按 `failure_policy` 判斷父 step 最終狀態

### parallel

1. 父 step → RUNNING
2. 為每個 branch 建立 step instances
3. 同時啟動所有 branches（各 branch 第一個 step → READY）
4. 各 branch 獨立執行
5. 按 `failure_policy` 判斷父 step 最終狀態

---

## 多 Worker 架構

### Instance Lease

在多 worker 部署下，引擎 MUST 確保同一 instance 同一時間僅由一個 worker 處理：

| 欄位 | 說明 |
|------|------|
| `instance_id` | 被租借的 instance |
| `worker_id` | 持有 lease 的 worker |
| `acquired_at` | 取得時間 |
| `expires_at` | 過期時間（heartbeat 續租） |

### Lease 規則

- Worker 在處理 instance 前 MUST 取得 lease
- Lease 有效期間，其他 worker MUST NOT 處理該 instance
- Worker SHOULD 定期 heartbeat 續租（建議間隔：lease 時長的 1/3）
- Lease 過期未續租 → 其他 worker 可重新取得
- Worker 處理完成後 MUST 主動釋放 lease

### Work Distribution

引擎 SHOULD 支援以下工作分配策略：

| 策略 | 說明 |
|------|------|
| Polling | Workers 定期查詢待處理的 instances |
| Queue-based | 使用訊息佇列分發觸發信號 |
| Notification | 使用 DB 通知（如 PostgreSQL LISTEN/NOTIFY） |

具體策略由實作決定，本規格僅要求 lease-based 互斥。

---

## 狀態轉換的原子性

所有狀態轉換 MUST 在單一 storage 交易中完成。一次 step 推進通常包含：

1. 更新當前 step instance 狀態（→ SUCCEEDED / FAILED / ...）
2. 記錄 step output
3. 更新下一個 step instance 狀態（→ READY）
4. 更新 instance 狀態（若需要，如 → WAITING / SUCCEEDED / FAILED）
5. 寫入 execution_log

這些操作 MUST 作為原子交易，防止部分更新導致不一致。
