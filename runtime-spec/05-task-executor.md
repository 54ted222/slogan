# 05 — Task Executor

本文件定義 Task Executor 的行為：根據 task definition 的 backend type 執行實際工作。

---

## 職責

Task Executor 接收已驗證的 task input，依據 task definition 的 `backend` 設定選擇對應的執行策略，回傳 output 或 error。

```
Step Executor 呼叫 Task Executor
     ↓
載入 task definition
     ↓
依 backend.type 分派
     ├── bash → Shell Executor
     ├── http → HTTP Executor
     ├── sdk → SDK Executor
     └── builtin → Builtin Executor
     ↓
回傳 output 或 error
```

---

## 共通行為

所有 backend types 共享以下行為：

### Idempotency Key

若 step 的 `execution.policy` 為 `idempotent`，Task Executor MUST 產生 idempotency key 並傳遞給 backend：

```
key = hash(workflow_instance_id + step_id + attempt)
```

傳遞方式依 backend type 而異（見各 backend 節）。

### Timeout

若 step 有 `timeout` 設定，Task Executor MUST 在 timeout 到期時終止 backend 執行。

### Secret 解析

Task definition 中的 `${ secret.XXX }` 表達式在 Task Executor 階段求值。Secret 值 MUST NOT 被記錄至 execution_log 或持久化至 step output。

---

## bash Backend

### 執行流程

1. 組合環境變數：
   - 系統 env + task definition 的 `backend.env`
   - 若有 idempotency key → `SLOGAN_IDEMPOTENCY_KEY` 環境變數
   - `SLOGAN_INSTANCE_ID` = workflow instance ID
   - `SLOGAN_STEP_ID` = step ID
2. 求值 `backend.env` 中的 CEL 表達式
3. 啟動 shell process（使用 `backend.shell`，預設 `/bin/sh`）
4. 將 task input 以 JSON 格式寫入 **stdin**
5. 執行 `backend.command`
6. 等待 process 完成（受 step timeout 限制）

### 結果處理

| 條件 | 結果 |
|------|------|
| Exit code = 0 | 成功：從 **stdout** 讀取 JSON 作為 output |
| Exit code ≠ 0 | 失敗：error message = stderr 內容（截取前 4KB） |
| stdout 非有效 JSON | 失敗：error code = `task_failed` |
| Process timeout | 失敗：先發 SIGTERM，等待 `timeout_kill_after`（預設 5s），再發 SIGKILL |

### stderr 處理

- stderr 內容寫入 execution_log（作為 task 的 log 輸出）
- 不影響成功 / 失敗判定（僅 exit code 決定）
- 記錄上限：建議 64KB，超過部分截斷

---

## http Backend

### 執行流程

1. 求值 `backend.url` 中的 CEL 表達式
2. 求值 `backend.headers` 中的 CEL 表達式
3. 組合 HTTP 請求：
   - Method = `backend.method`（預設 POST）
   - URL = 求值後的 url
   - Headers = 求值後的 headers + `Content-Type: application/json`
   - Body = task input（依 `body_mapping` 規則）
4. 若有 idempotency key → 加入 `Idempotency-Key` header
5. 發送 HTTP 請求（受 `backend.timeout` 或 step timeout 限制）

### Body Mapping

| `body_mapping` 值 | Request Body |
|-------------------|-------------|
| `full`（預設） | 整個 input map 序列化為 JSON |
| `none` | 不傳送 body |

### 結果處理

| 條件 | 結果 |
|------|------|
| HTTP 2xx | 成功：response body 解析為 JSON 作為 output |
| HTTP 非 2xx 且 status 在 `retry_on_status` 中 | 可重試失敗：觸發 step 的 retry 機制 |
| HTTP 非 2xx 且 status 不在 `retry_on_status` 中 | 永久性失敗 |
| Response body 非有效 JSON | 失敗：error code = `task_failed` |
| 連線失敗 / DNS 解析失敗 | 可重試失敗 |
| 請求 timeout | 可重試失敗 |

### Response Mapping

若有 `response_mapping`：

- `body`：CEL 表達式從 `response.body` 取子欄位作為 output
- `status_code`：將 HTTP status code 存入 output 的指定欄位

若無 `response_mapping`：response body 整體作為 output。

---

## sdk Backend

### 執行流程

1. 載入 module（`backend.module`）
2. 呼叫 entry function（`backend.entry`，預設 `"execute"`）
3. 傳入 task input（JSON 序列化）
4. 若有 idempotency key → 作為 context 參數傳入
5. 等待回傳（受 step timeout 限制）

### SDK Protocol

Task SDK 模組 MUST 實作以下介面：

```
function execute(context: TaskContext): TaskResult
```

#### TaskContext

| 欄位 | 型別 | 說明 |
|------|------|------|
| `input` | map | task input |
| `config` | map | `backend.config`（若有） |
| `idempotency_key` | string \| null | 冪等 key（若有） |
| `instance_id` | string | workflow instance ID |
| `step_id` | string | step ID |
| `attempt` | integer | 當前嘗試次數 |

#### TaskResult

| 欄位 | 型別 | 說明 |
|------|------|------|
| `success` | boolean | 是否成功 |
| `output` | map \| null | 成功時的輸出 |
| `error` | object \| null | 失敗時的錯誤（含 message、code） |

### Runtime 隔離

- 各 runtime（node、python 等）的模組載入與執行方式由實作決定
- 引擎 SHOULD 支援 process-level 隔離（子 process 執行模組）
- 模組 crash 不應影響引擎主 process

---

## builtin Backend

### 執行流程

1. 查找 `backend.handler` 對應的內建函式
2. 以 task input 和 `backend.config` 呼叫
3. 回傳 output

### 內建 Handler

引擎 MUST 提供以下內建 handler：

| Handler | 說明 |
|---------|------|
| `echo` | 回傳 input 作為 output（用於測試） |
| `noop` | 不做任何事，回傳空 output |
| `log` | 將 input 寫入 execution_log，回傳空 output |

引擎 MAY 提供額外的內建 handlers。

### 註冊機制

引擎 SHOULD 支援在啟動時註冊自訂 builtin handlers，供 task definition 使用。
