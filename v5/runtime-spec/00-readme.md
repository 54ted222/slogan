# v5 Runtime Spec

本資料夾定義 v5 workflow engine 的**執行時期行為**，是 [v5/dsl-spec](../dsl-spec/) 的對應實作規格。

DSL spec 描述「YAML 怎麼寫」；runtime spec 描述「引擎怎麼跑」。實作引擎時兩者並讀。

## 文件索引

| 文件 | 內容 |
|------|------|
| [01-architecture](01-architecture.md) | 引擎核心元件、責任分工、資料流 |
| [02-instance-lifecycle](02-instance-lifecycle.md) | Workflow / agent / function instance 的狀態機與持久化 |
| [03-step-execution](03-step-execution.md) | 各 step type 的執行語意（task / wait / foreach / parallel / saga / agent / callback ...） |
| [04-expression-evaluation](04-expression-evaluation.md) | CEL 求值器、namespace 解析、求值時機 |
| [05-task-registry](05-task-registry.md) | Action 名稱解析、tool / function / agent / MCP / toolset 的註冊與查找 |
| [06-tool-backend](06-tool-backend.md) | exec / http / extension backend 的 process 管理、I/O 協議、callback 路由 |
| [07-event-bus](07-event-bus.md) | 事件 routing、scope、wait 解析、emit 投遞、ordering 保證 |
| [08-persistence](08-persistence.md) | Instance state、step output、execution log 的儲存格式與 checkpoint 行為 |
| [09-error-model](09-error-model.md) | 錯誤分類、`error.*` 結構、catch 路徑、saga 補償 |
| [10-concurrency](10-concurrency.md) | Parallel / foreach concurrency、async step、wait fan-in 的 scheduler |
| [11-secret-and-context](11-secret-and-context.md) | Secret 解密、env / project / artifacts 注入、log 遮蔽 |
| [12-determinism-and-replay](12-determinism-and-replay.md) | Replay 語意、idempotency key、時間相關函式（`now()` / `uuid()`）的記錄 |

## 設計原則

1. **可重啟**：所有 instance 狀態 MUST 可從持久化儲存完全還原；engine crash 後 resume 不得導致已執行 step 重複執行（除非 `idempotent: false` 且引擎判定中斷發生在執行中）。
2. **事件驅動**：跨 instance 通訊一律經 event bus；engine 內部以事件迴圈推動 step 進展，無共享記憶體狀態。
3. **協議優先**：tool / agent 介面為協議邊界，引擎內部資料結構不外洩；所有外部互動透過 schema 化的訊息。
4. **可觀測**：每個 step transition、每次 expression 求值、每筆 tool 呼叫 MUST 寫入 execution log，可在 `idempotent: true` 的前提下 replay。
5. **本規格描述行為，不規範實作語言**。
