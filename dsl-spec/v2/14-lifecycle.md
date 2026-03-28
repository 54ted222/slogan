# 14 — Lifecycle

本文件定義 workflow definition、task definition、workflow instance、step instance 的狀態機。

---

## Definition Lifecycle

Workflow definition 與 task definition 共用相同的發布流程：

```
DRAFT ──→ VALIDATED ──→ PUBLISHED ──→ DEPRECATED ──→ ARCHIVED
```

| 狀態 | 說明 |
|------|------|
| DRAFT | 可編輯，不可執行 |
| VALIDATED | 已通過結構與表達式驗證（見 [16-validation-rules](16-validation-rules.md)），不可編輯 |
| PUBLISHED | 可執行，不可編輯。Workflow: trigger 生效；Task: 可被 workflow step 呼叫 |
| DEPRECATED | 仍可執行，但不建議使用。Workflow: trigger 停用；Task: 可被呼叫但產生警告 |
| ARCHIVED | 不可執行，保留用於稽核 |

### 允許的轉換

| 從 | 到 | 條件 |
|----|-----|------|
| DRAFT | VALIDATED | 通過所有驗證規則 |
| VALIDATED | DRAFT | 需要修改時退回 |
| VALIDATED | PUBLISHED | 確認發布 |
| PUBLISHED | DEPRECATED | 標記為棄用 |
| DEPRECATED | ARCHIVED | 確認所有 instance 都已完成或取消 |

不允許的轉換：
- PUBLISHED → DRAFT（已發布的版本不可修改，需建新版本）
- ARCHIVED → 任何狀態

---

## Instance Lifecycle

Workflow instance 的執行狀態：

```
CREATED ──→ RUNNING ──⇄── WAITING
               │                │
               ├──→ SUCCEEDED   │
               ├──→ FAILED      │
               └──→ CANCELLED   │
                                │
               WAITING ────→ FAILED
               WAITING ────→ CANCELLED
```

| 狀態 | 說明 |
|------|------|
| CREATED | Instance 已建立，尚未開始執行 |
| RUNNING | 正在執行 steps |
| WAITING | 暫停中，等待外部事件（`wait_event`） |
| SUCCEEDED | 正常完成（透過 `return` 或所有 steps 完成） |
| FAILED | 異常終止（透過 `fail`、未處理的錯誤、或 timeout） |
| CANCELLED | 被外部取消（API / parent workflow） |

### 允許的轉換

| 從 | 到 | 觸發條件 |
|----|-----|----------|
| CREATED | RUNNING | 排程器開始執行 |
| RUNNING | WAITING | 遇到 `wait_event` step |
| RUNNING | SUCCEEDED | 遇到 `return` 或所有 steps 完成 |
| RUNNING | FAILED | 遇到 `fail`、未處理錯誤、或 workflow timeout |
| RUNNING | CANCELLED | 外部取消請求 |
| WAITING | RUNNING | 收到匹配事件，或 wait_event timeout 觸發且有 handler 需執行 |
| WAITING | FAILED | wait_event timeout 且所有層級均無 handler（on_timeout 及 on_error） |
| WAITING | CANCELLED | 外部取消請求 |

SUCCEEDED、FAILED、CANCELLED 為 terminal 狀態，MUST NOT 再轉換。

---

## Step Lifecycle

Step instance 的執行狀態：

```
PENDING ──→ READY ──→ RUNNING ──→ SUCCEEDED
                │        │   │
                │        │   ├──→ FAILED
                │        │   ├──→ WAITING    （僅 wait_event）
                │        │   │      ├──→ TIMED_OUT （等待逾時）
                │        │   │      └──→ CANCELLED
                │        │   ├──→ TIMED_OUT
                │        │   └──→ CANCELLED  （外部取消）
                │
                └──→ SKIPPED
```

| 狀態 | 說明 |
|------|------|
| PENDING | 尚未到達（前置 step 還在執行） |
| READY | 前置條件滿足，等待排程 |
| RUNNING | 正在執行中 |
| SUCCEEDED | 執行成功 |
| FAILED | 執行失敗（retry 用盡，且 on_error 未處理或不存在） |
| WAITING | 等待外部事件（僅 `wait_event`） |
| TIMED_OUT | 執行超時 |
| CANCELLED | 被外部取消（workflow timeout、parent 取消、API 取消） |
| SKIPPED | condition 為 false 或所在分支未被選中 |

### 允許的轉換

| 從 | 到 | 觸發條件 |
|----|-----|----------|
| PENDING | READY | 前一個 step 完成（SUCCEEDED、SKIPPED、或 FAILED / TIMED_OUT 但錯誤已被 handler 處理） |
| READY | RUNNING | 排程器開始執行 |
| READY | SKIPPED | condition 為 false |
| RUNNING | SUCCEEDED | 執行完成 |
| RUNNING | FAILED | 執行失敗 |
| RUNNING | WAITING | wait_event 進入等待 |
| RUNNING | TIMED_OUT | 超過 timeout |
| RUNNING | CANCELLED | 外部取消（workflow timeout 或取消請求） |
| WAITING | RUNNING | 收到匹配事件 |
| WAITING | TIMED_OUT | `wait_event` 等待逾時 |
| WAITING | CANCELLED | 外部取消（workflow timeout 或取消請求） |

SUCCEEDED、FAILED、TIMED_OUT、CANCELLED、SKIPPED 為 terminal 狀態。

---

## 狀態持久化

所有狀態轉換 MUST 被持久化至資料庫。引擎 MUST 確保：

1. 狀態轉換是原子操作
2. 不允許非法的狀態轉換
3. Terminal 狀態不可被覆寫
4. crash 後可從最後的持久化狀態恢復
