# TODO — DSL v2 規格補完清單

## 高優先級

### [x] 版本遷移策略 → `dsl-spec/v2/19-versioning.md`

- 定義 DSL 版本升級路徑（v1 → v2 相容性）
- 定義向下相容保證範圍與 deprecation timeline
- 定義 running instance 遇到定義版本升級時的行為
- 定義 breaking change 的處理流程

### [x] Task 執行語意保證 → `dsl-spec/v2/20-execution-guarantees.md`

- 明確定義 task 執行的 delivery guarantee（exactly-once / at-least-once）
- 定義 idempotency key 在 recovery 場景的使用方式
- 釐清 HTTP retry 語意與 task execution 語意的區分
- 定義 timeout 與 event 同時抵達時的 race condition 處理

### [x] 全域並發控制 → `runtime-spec/10-concurrency-control.md`

- 定義全域 rate limiting 機制（跨 instance）
- 定義 per-task-type 並發上限
- 定義 parallel step 資源耗盡時的 queue / backpressure 策略
- 定義 `foreach.concurrency` 與全域限制的交互關係

### [x] Observability 規格 → `runtime-spec/11-observability.md`

- 定義標準 metrics（execution time、success rate、retry frequency）
- 定義 tracing / correlation ID 跨 instance 傳遞機制
- 定義 engine health check endpoint
- 定義 resource usage tracking 方式

## 中優先級

### [x] Artifact 生命週期補完 → `dsl-spec/v2/13-artifacts.md` 新增章節

- 定義 artifact cleanup 時機（immediate / async / scheduled）
- 定義 concurrent access 的 mutual exclusion 機制
- 定義 storage backend 錯誤處理（upload 失敗、磁碟空間不足）
- 定義 partial write / failure 的回復行為

### [x] Error Code 體系化 → `dsl-spec/v2/12-error-handling.md` 擴充

- 統整散落各文件的 error code 定義至統一清單
- 定義 custom error code namespace 與 extensibility 規則
- 細分 timeout 類型（step timeout / workflow timeout / wait_event timeout）
- 定義 task backend custom error code 向 workflow 傳遞的規則

### [x] Sub-workflow 生命週期 → `dsl-spec/v2/11-step-sub-workflow.md` 新增章節

- 定義 child instance 的 archive / cleanup 時機與 retention policy
- 明確定義 parent cancellation cascade 行為
- 明確定義 parent step timeout 與 child workflow `config.timeout` 同時存在時的優先權

### [x] HTTP Backend 邊界行為 → `dsl-spec/v2/17-task-definition.md` + `runtime-spec/05-task-executor.md`

- 定義 3xx redirect 處理策略
- 定義 response size limit
- 定義非 JSON content-type 的 response 處理
- 定義 chunked transfer encoding 行為

### [x] Stdio Backend 邊界行為 → `dsl-spec/v2/17-task-definition.md` + `runtime-spec/05-task-executor.md`

- 定義大 payload（> 1MB JSON）處理方式
- 定義 SIGTERM → SIGKILL 時序與 signal handling
- 定義 process resource limit（memory、CPU、open files）
- 定義 null byte 等特殊字元處理

### [x] Workflow 原地更新 → `dsl-spec/v2/14-lifecycle.md` 新增章節

- 定義 DRAFT 狀態定義是否可修改內容
- 定義 PUBLISHED 版本 fork 機制
- 定義 running instance 升級至新版定義的策略

## 低優先級

### [x] Pause / Resume 機制 → `FUTURE.md` 新增章節

- 設計有別於 wait_event 的手動暫停 / 恢復語意
- 定義 pause 狀態下的 timeout 行為
- 定義 CLI pause / resume 指令

### [x] Bulk Operations → `FUTURE.md` 新增章節

- 定義批次 instance 建立 API
- 定義批次 cancellation API
- 定義進階 filter 查詢語法

### [x] CEL 表達式邊界行為 → `dsl-spec/v2/04-expressions.md` 新增章節

- 定義 null 上呼叫函式的行為（如 `size()` on null）
- 定義數值精度（IEEE 754 double）
- 定義字串編碼（UTF-8）
- 定義 list / map iteration order 保證

### [x] Event Deduplication 細節 → `runtime-spec/03-event-router.md` 擴充

- 將 deduplication 從 SHOULD 提升為 MUST 或明確定義為 optional
- 定義 deduplication window 大小
- 定義 deduplication key collision 處理
- 定義同一 event 觸發多個 workflow 時的行為

### [x] Audit Trail 與合規 → `runtime-spec/08-storage-schema.md` 新增章節

- 定義 execution_log 不可變性保證
- 定義 tamper-proof log 機制
- 定義 retention policy 合規需求

## FUTURE.md 已列項目

### [x] Scheduled Trigger (cron) → `dsl-spec/v2/02-triggers.md` 新增章節

- 撰寫 cron trigger 規格

### [x] Delayed Event → `dsl-spec/v2/09-step-events.md` 新增章節

- 撰寫 delayed event 規格

### [x] HTTP Trigger → `dsl-spec/v2/02-triggers.md` 新增章節

- 撰寫 HTTP trigger 規格

### [x] Authentication & Authorization → `runtime-spec/12-authentication.md`

- 撰寫 authn / authz 規格

### [x] 檢查所有的文件確保文件描述的內容沒有相互矛盾或是缺少實作的時候必要的資訊

- [x] 新增 `delayed_event`、`scheduled_trigger` 表至 `runtime-spec/08-storage-schema.md`
- [x] 新增 `correlation_id` 欄位至 `workflow_instance` 實體
- [x] 擴充 `dsl-spec/v2/16-validation-rules.md` 加入 scheduled / http trigger 驗證規則
- [x] 擴充 `runtime-spec/07-timeout-manager.md` 加入 delayed event 投遞與 scheduled trigger 排程
- [x] 更新 `runtime-spec/04-step-executor.md` emit step 處理 delay 邏輯
- [x] 統一 error code 引用（移除 04-step-executor.md 中過時的 hardcoded 清單）
- [x] 新增 trigger 相關 error code 至 `12-error-handling.md`
- [x] 更新所有 overview 文件索引
- [x] **foreach fail_fast 輸出索引矛盾** — 統一為 array 長度 = items 長度，索引一一對應，失敗/取消迭代為 null
- [x] **QUEUED 狀態未統一** — 移除 QUEUED 狀態，統一使用 READY 表示排隊（內部區分由引擎實作決定）
- [x] **definition 實體結構不一致** — 更新 15-runtime-persistence.md 改用統一 definition 實體
- [x] **event namespace 上下文標註錯誤** — 在 06-expression-evaluator.md 加入 event namespace 正確描述
- [x] **HTTP trigger 無 runtime 規格** — 新增 03-event-router.md HTTP Trigger 處理章節
- [x] **HTTP backend 不支援 artifact 未記載** — 在 05-task-executor.md 和 17-task-definition.md 加入 artifact 限制說明
- [x] **sub_workflow runtime 執行不完整** — 補充 timeout、retry、execution policy、cascade cancellation
- [x] **取消級聯無 runtime 規格** — 在 04-step-executor.md sub_workflow 章節加入 cascade cancellation
- [x] **Scheduled trigger 的 schedule namespace 未在 runtime 定義** — 在 07-timeout-manager.md 加入 namespace 表
- [x] **Instance 狀態 WAITING → RUNNING 轉換未明確** — 在 02-scheduler.md 加入完整流程
- [x] **Secret definition 驗證規則缺失** — 在 16-validation-rules.md 新增 Secret Definition 驗證章節
- [x] **Artifact 宣告驗證規則缺失** — 在 16-validation-rules.md 新增 Artifact 宣告驗證章節
- [x] **Task step resources 驗證規則缺失** — 在 16-validation-rules.md 新增 Task Step Resources 驗證章節
- [x] **if/switch step 驗證規則不完整** — 在 16-validation-rules.md 新增控制流程 Step 驗證章節
- [x] **foreach `as` 欄位無驗證規則** — 在 16-validation-rules.md foreach 驗證中加入 `as` 規則
- [x] **Sub-workflow artifact 傳遞機制不完整** — 在 11-step-sub-workflow.md 加入完整 artifact 傳遞範例
- [x] **15-runtime-persistence.md 欄位不對稱** — 補齊所有缺失欄位與 Secret 管理章節
- [x] **Extension functions 定義不對稱** — 在 06-expression-evaluator.md 新增完整擴充函式表
- [x] **CEL ↔ JSON 型別映射表不對稱** — 在 04-expressions.md 加入映射表與交叉引用
- [x] **Event deduplication 詳細度不對稱** — 更新 02-triggers.md 加入 MUST 級 dedup 規則與交叉引用
- [x] **Delayed event 投遞失敗重試** — 在 09-step-events.md 加入重試說明
