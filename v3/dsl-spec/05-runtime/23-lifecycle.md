# 23 — 生命週期

本文件定義所有 definition kind 與 instance 的狀態機。

---

## Definition Lifecycle

### Workflow / Task / Agent Definition

Workflow definition、task definition 與 agent definition 共用相同的發布流程：

```
DRAFT ──→ VALIDATED ──→ PUBLISHED ──→ DEPRECATED ──→ ARCHIVED
```

| 狀態 | 說明 |
|------|------|
| DRAFT | 可編輯，不可執行 |
| VALIDATED | 已通過結構與表達式驗證，不可編輯 |
| PUBLISHED | 可執行，不可編輯。Workflow: trigger 生效；Task: 可被 workflow step 呼叫；Agent: 可被 workflow step 引用 |
| DEPRECATED | 仍可執行，但不建議使用。Workflow: trigger 停用；Task: 可被呼叫但產生警告；Agent: 同 Task |
| ARCHIVED | 不可執行，保留用於稽核 |

#### 允許的轉換

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

### Toolset / Resources / Project Definition

Toolset（[18-toolset-definition](../04-resources/18-toolset-definition.md)）、Resources（[19-resources-definition](../04-resources/19-resources-definition.md)）與 Project（[20-project](../04-resources/20-project.md)）使用較簡化的發布流程：

```
DRAFT ──→ PUBLISHED ──→ ARCHIVED
```

| 狀態 | 說明 |
|------|------|
| DRAFT | 可編輯，不可被引用 |
| PUBLISHED | 正式發布，可被其他 definition 引用。不可編輯 |
| ARCHIVED | 已封存，不可被新引用。既有引用仍可使用（引擎以 snapshot 保留） |

#### 允許的轉換

| 從 | 到 | 條件 |
|----|-----|------|
| DRAFT | PUBLISHED | 結構驗證通過 |
| PUBLISHED | ARCHIVED | 確認封存 |

不允許的轉換：
- PUBLISHED → DRAFT（需建新版本）
- ARCHIVED → 任何狀態

> 簡化原因：Toolset、Resources、Project 不直接執行，僅作為組裝素材被 Workflow / Agent 引用。不需要 VALIDATED 階段（發布時即驗證）和 DEPRECATED 階段（無 trigger 或獨立執行的概念）。

---

## DRAFT 狀態的可編輯性

DRAFT 狀態的 definition **可修改內容**。VALIDATED / PUBLISHED 以後的狀態不可修改。

| 狀態 | 可修改內容 | 說明 |
|------|-----------|------|
| DRAFT | 是 | 可自由修改 YAML 內容、metadata（name 除外） |
| VALIDATED | 否 | 需退回 DRAFT 才能修改（僅 Workflow / Task / Agent） |
| PUBLISHED | 否 | 不可修改，需建新版本 |
| DEPRECATED | 否 | — |
| ARCHIVED | 否 | — |

DRAFT 狀態修改時：
- `metadata.name` MUST NOT 變更（name 是 definition 的身份識別）
- `metadata.version` MUST NOT 變更
- 其他所有欄位 MAY 修改
- 修改後 `updated_at` 更新

---

## 從 PUBLISHED 建立新版本（Fork）

本規格不提供自動 fork 機制。建立新版本的流程為：

1. 建立新的 definition：同一 `name`，`version` 為下一個可用的正整數
2. 新 definition 初始狀態為 DRAFT
3. 使用者自行複製舊版本的 YAML 內容並修改

引擎 CLI MAY 提供便捷指令簡化此流程。

---

## Running Instance 與新版 Definition 的關係

Running instance **不可升級到新版 definition**。

| 情境 | 行為 |
|------|------|
| Instance 綁定 v3，使用者發布 v4 | Instance 繼續使用 v3 完成執行 |
| Instance 中的 `version: "latest"` 的 task step | 執行時解析到當下最新的 PUBLISHED task version（可能是新版），但 workflow definition 本身不變 |
| Instance 中的 `version: "latest"` 的 agent step | 同 task step — 執行時解析到當下最新的 PUBLISHED agent version |
| Instance 綁定的 definition 被 DEPRECATED | Instance 繼續執行，不受影響 |

「不可升級」的原因：
- 新版 definition 的 step 結構可能完全不同
- Running instance 已有 step instances 的狀態記錄，與新 definition 不相容
- 違反 deterministic replay 原則

---

## Instance Lifecycle

Workflow instance 的執行狀態：

```
CREATED ──→ RUNNING ──⇄── WAITING
               │                │
               ├──→ SUCCEEDED   │
               ├──→ FAILED ─────┤
               ├──→ CANCELLED   │
               ├──→ CONTINUED   │
               │                │
               ├──→ COMPENSATING ──→ COMPENSATED
               │         │
               │         └──→ COMPENSATION_FAILED
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
| CONTINUED | 由 `continue_as_new` 產生新 instance 接續，原 instance 視為成功完成（見 [10-step-terminal](../02-steps/10-step-terminal.md)） |
| COMPENSATING | 正在執行 saga 補償流程（見 [12-step-saga](../02-steps/12-step-saga.md)） |
| COMPENSATED | 補償流程已成功完成（所有 compensate 皆 SUCCEEDED） |
| COMPENSATION_FAILED | 補償流程中有 compensate step 失敗 |

### 允許的轉換

| 從 | 到 | 觸發條件 |
|----|-----|----------|
| CREATED | RUNNING | 排程器開始執行 |
| RUNNING | WAITING | 遇到 `wait_event` step |
| RUNNING | SUCCEEDED | 遇到 `return` 或所有 steps 完成 |
| RUNNING | FAILED | 遇到 `fail`、未處理錯誤、或 workflow timeout |
| RUNNING | CANCELLED | 外部取消請求 |
| RUNNING | CONTINUED | 遇到 `continue_as_new`，新 instance 已建立 |
| RUNNING | COMPENSATING | Saga 範圍內 step 失敗，觸發補償流程 |
| WAITING | RUNNING | 收到匹配事件，或 wait_event timeout 觸發且有 handler 需執行 |
| WAITING | FAILED | wait_event timeout 且所有層級均無 handler（on_timeout 及 on_error） |
| WAITING | CANCELLED | 外部取消請求 |
| COMPENSATING | COMPENSATED | 所有補償 step 成功完成 |
| COMPENSATING | COMPENSATION_FAILED | 補償 step 失敗（abort 模式中止 / continue 模式部分失敗） |

### Terminal 狀態

SUCCEEDED、FAILED、CANCELLED、CONTINUED、COMPENSATED、COMPENSATION_FAILED 為 terminal 狀態，MUST NOT 再轉換。

---

## Agent Session Lifecycle

Agent session（`type: agent` step 產生的 session）具有獨立的執行狀態：

```
RUNNING ──→ SUCCEEDED
   │  ↑
   │  │
   ↓  │
WAITING ──→ FAILED
```

| 狀態 | 說明 |
|------|------|
| RUNNING | Agent 正在執行 agentic loop（LLM 推理或 tool 執行中） |
| WAITING | Agent 等待外部輸入（如 `agent.ask_human` builtin tool 被呼叫，等待人類回覆） |
| SUCCEEDED | Agent 正常完成，產生 final output |
| FAILED | Agent 異常終止（達到上限、LLM 錯誤、output 驗證失敗等） |

### 允許的轉換

| 從 | 到 | 觸發條件 |
|----|-----|----------|
| RUNNING | WAITING | Agent 呼叫 `agent.ask_human`，等待人類回覆 |
| RUNNING | SUCCEEDED | Agent 產生 final output 且通過 output_schema 驗證 |
| RUNNING | FAILED | 達到 `max_iterations` / `max_tool_calls`、LLM 錯誤、output 驗證失敗 |
| WAITING | RUNNING | 收到人類回覆，繼續執行 |
| WAITING | FAILED | 等待逾時（若有設定）或外部取消 |

### 與 Step / Instance 狀態的關係

Agent session 狀態與其所屬的 step instance 狀態對應：

| Agent Session 狀態 | Step Instance 狀態 | Workflow Instance 狀態 |
|-------------------|-------------------|----------------------|
| RUNNING | RUNNING | RUNNING |
| WAITING | WAITING | WAITING |
| SUCCEEDED | SUCCEEDED | 繼續執行（或 SUCCEEDED 若為最後一個 step） |
| FAILED | FAILED | 進入錯誤處理流程（見 [21-error-handling](21-error-handling.md)） |

> Agent session 的 storage schema 詳見 [13-agent-definition](../03-agent/13-agent-definition.md) 的「對話歷史持久化」章節。

---

## Step Lifecycle

Step instance 的執行狀態：

```
PENDING ──→ READY ──→ RUNNING ──→ SUCCEEDED
                │        │   │
                │        │   ├──→ FAILED
                │        │   ├──→ SKIPPED    （if/switch 無匹配分支）
                │        │   ├──→ WAITING    （僅 wait_event / agent）
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
| WAITING | 等待外部事件（`wait_event`）或人類回覆（`agent` 呼叫 `ask_human`） |
| TIMED_OUT | 執行超時 |
| CANCELLED | 被外部取消（workflow timeout、parent 取消、API 取消） |
| SKIPPED | when 為 false、所在分支未被選中、或控制流程 step 無匹配的執行分支（if 無 else 且 when 為 false、switch 無匹配且無 default） |

### 允許的轉換

| 從 | 到 | 觸發條件 |
|----|-----|----------|
| PENDING | READY | 前一個 step 完成（SUCCEEDED、SKIPPED、或 FAILED / TIMED_OUT 但錯誤已被 handler 處理） |
| READY | RUNNING | 排程器開始執行 |
| READY | SKIPPED | when 為 false |
| RUNNING | SKIPPED | 控制流程 step 無匹配的執行分支（if 無 else 且 when 為 false、switch 無匹配且無 default） |
| RUNNING | SUCCEEDED | 執行完成 |
| RUNNING | FAILED | 執行失敗 |
| RUNNING | WAITING | wait_event 進入等待；agent 呼叫 `ask_human` |
| RUNNING | TIMED_OUT | 超過 timeout |
| RUNNING | CANCELLED | 外部取消（workflow timeout 或取消請求） |
| WAITING | RUNNING | 收到匹配事件或人類回覆 |
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
