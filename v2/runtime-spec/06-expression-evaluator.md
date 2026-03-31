# 06 — Expression Evaluator

本文件定義 CEL 表達式求值引擎的行為：namespace 解析、求值時機、非確定性函式記錄。

---

## 職責

Expression Evaluator 接收 CEL 表達式與 execution context，回傳求值結果。所有 `${ }` 定界符內的內容都透過此元件求值。

---

## 求值流程

```
輸入 YAML 值（可能含 ${ }）
     ↓
解析 ${ } 定界符
     ↓
若整個值為單一 ${ } → 回傳原始型別
若混合文字 + ${ } → 各段求值後串接為字串
若無 ${ } → 回傳 YAML 原生值
     ↓
CEL 求值（附 execution context）
     ↓
回傳結果
```

---

## Namespace 解析

Expression Evaluator 根據 step 的 execution context 提供 namespace 變數：

### 建立 Namespace 的資料來源

| Namespace | 資料來源 | 載入方式 |
|-----------|---------|---------|
| `input` | workflow_instance.input | 從 storage 載入 |
| `steps` | 所有 SUCCEEDED step_instances 的 output | 從 storage 載入，以 step_id 為 key 建立 map |
| `vars` | 最新的 vars snapshot | 從 storage 載入 |
| `loop` | foreach 迭代 context | 由 Step Executor 傳入 |
| `env` | OS 環境變數 / .env | 引擎啟動時載入（記憶體快取） |
| `secret` | Secret definitions | 引擎啟動時解密載入（記憶體快取） |
| `artifacts` | artifact_records | 從 storage 載入 |
| `event` | 事件資料（type, data, id, timestamp, source） | 由 Event Router / Step Executor 傳入（trigger `when`、`wait_event` `match` 中可用） |
| `error` | 錯誤資訊 | 由 Step Executor 傳入（僅 on_error handler） |
| `timeout` | timeout 資訊 | 由 Step Executor 傳入（僅 on_timeout handler） |

### Sub-workflow Namespace 隔離

Sub-workflow（child）instance 擁有獨立的 execution context，不繼承 parent 的 namespace：

- `input`：child 的 input（由 parent 的 sub_workflow step 傳入）
- `steps`：僅包含 child 內部的 steps
- `vars`：child 獨立的 vars（不繼承 parent vars）
- `artifacts`：child 獨立宣告的 artifacts（不繼承 parent artifacts）
- `loop`：不存在（除非 child 自己有 foreach）

若需在 parent 與 child 間共享資料，MUST 透過 sub_workflow step 的 `input` mapping 明確傳遞。

### steps Namespace 建立規則

`steps` namespace 為一個 map，key = step_id，value = `{ output: <step_output> }`。

建立規則：

1. 僅包含在當前 step **之前**已到達 terminal 狀態的 steps
2. SUCCEEDED：`steps.<id>.output` = step output
3. SKIPPED：`steps.<id>.output` = `null`
4. FAILED（錯誤已被 handler 處理）：`steps.<id>.output` = `null`
5. 其他 terminal 狀態（TIMED_OUT、CANCELLED）：`steps.<id>.output` = `null`
6. 未到達 terminal 狀態的 steps：不存在於 `steps` namespace

### vars Namespace 持久化

`vars` 為 instance 級的全域命名空間。每次 `assign` step 執行後：

1. 將新設定的變數 merge 至 vars snapshot
2. 持久化整個 vars snapshot 至 storage
3. 後續 steps 從 storage 載入最新 vars

建議實作方式：vars 存為 workflow_instance 的一個 JSON 欄位。

### 並行 assign 的行為

在 `parallel` 的不同 branches 中同時 `assign` 同名變數，結果為**非確定性**（取決於 branch 完成順序，last write wins）。開發者 SHOULD 避免在 parallel branches 中 assign 同名變數。引擎不需要偵測或阻止此情況。

---

## 型別對應

CEL 型別與 YAML / JSON 型別的對應：

| CEL 型別 | JSON 型別 | YAML 型別 |
|---------|----------|----------|
| `int` / `uint` | number | integer |
| `double` | number | float |
| `bool` | boolean | boolean |
| `string` | string | string |
| `list` | array | array |
| `map` | object | map |
| `null_type` | null | null |
| `timestamp` | string (ISO 8601) | string |
| `duration` | string (e.g. "30m") | string |

---

## 擴充函式

Expression Evaluator MUST 支援以下 DSL 擴充函式（完整語意定義見 [dsl-spec/v2/04-expressions](../dsl-spec/v2/04-expressions.md)）：

| 函式 | 回傳 | 說明 | 確定性 |
|------|------|------|--------|
| `now()` | timestamp | 當前 UTC 時間 | 非確定性 |
| `uuid()` | string | UUID v4 | 非確定性 |
| `json_encode(value)` | string | 將值編碼為 JSON 字串 | 確定性 |
| `json_decode(string)` | any | 將 JSON 字串解碼為值 | 確定性 |
| `coalesce(a, b, ...)` | any | 回傳第一個非 null 的值 | 確定性 |
| `default(value, fallback)` | any | 若 value 為 null 則回傳 fallback | 確定性 |

確定性函式不需要記錄（replay 時重新求值即可）。非確定性函式需要記錄以確保 replay 一致性（見下方）。

---

## 非確定性函式記錄

### 記錄對象

以下擴充函式為非確定性（每次呼叫可能回傳不同值），MUST 記錄求值結果：

| 函式 | 回傳 | 說明 |
|------|------|------|
| `now()` | timestamp | 當前 UTC 時間 |
| `uuid()` | string | UUID v4 |

### 記錄流程

```
首次執行：
  1. 正常求值 now() / uuid()
  2. 記錄：{ expression_position: result_value }
  3. 與 step_instance 一同持久化

Sub-workflow 隔離：child instance 擁有獨立的 step_instances，
recorded_values 儲存於各自的 step_instance 中。Parent step
的 recorded_values 僅記錄 parent 層級的表達式（如 sub_workflow
step 的 input CEL），不包含 child 內部的記錄。Crash recovery
時 parent 與 child 各自從自己的 step_instances 恢復。

Replay（recovery）：
  1. 檢查是否有記錄
  2. 有 → 使用記錄的值
  3. 無 → 正常求值（首次執行）
```

### Expression Position

Expression position 為表達式在 YAML 中的唯一位置識別：

```
step_id + "." + field_path + "[" + expression_index + "]"
```

例如：

```yaml
- id: load_order
  type: task
  input:
    created_at: ${ now() }      # position: "load_order.input.created_at[0]"
    id: ${ uuid() }             # position: "load_order.input.id[0]"
```

### 儲存格式

在 step_instance 中新增 `recorded_values` 欄位：

```json
{
  "load_order.input.created_at[0]": "2024-01-15T10:30:00Z",
  "load_order.input.id[0]": "550e8400-e29b-41d4-a716-446655440000"
}
```

---

## 求值失敗處理

| 失敗原因 | 結果 |
|---------|------|
| 語法錯誤（應在 validation 時攔截） | step FAILED，error code: `expression_error` |
| 型別錯誤（如 string + int） | step FAILED，error code: `expression_error` |
| Null reference（如 `null.field`） | step FAILED，error code: `expression_error` |
| 不存在的 namespace 變數 | 回傳 `null`（不失敗） |

### 防禦性存取

Expression Evaluator MUST 支援以下防禦性存取模式：

- `has(field)` — 檢查欄位是否存在
- `default(value, fallback)` — null 時回傳 fallback
- `coalesce(a, b, ...)` — 回傳第一個非 null

---

## 效能考量

- Expression Evaluator SHOULD 快取 CEL 程式的編譯結果（parsed AST）
- 同一 workflow definition 的表達式在多個 instances 間可重用編譯結果
- Namespace 資料 SHOULD 延遲載入（僅在被存取時才從 storage 讀取）
