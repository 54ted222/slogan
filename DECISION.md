# 決策紀錄

> 所有草稿中的設計選擇，依主題分類。已於 2026-03-31 完成決策。

---

## A. Saga / Compensation（[draft/saga-compensation.md](draft/saga-compensation.md)）

### A1. 補償區塊結構

- [x] **A + B 並存** — Step 層級 `compensate` + `saga` 區塊皆支援
- [ ] ~~**C** — 全域 compensation 策略~~

### A2. 補償失敗行為

- [x] **saga 區塊**：可配置（預設中止）
- [x] **step 層級 compensate**：固定中止

### A3. 補償是否支援重試

- [x] 是

### A4. 是否需要 forward recovery

- [x] **saga 區塊**：可配置 forward / backward
- [x] **step 層級**：僅 backward

---

## B. JSON Schema 發布（[draft/json-schema.md](draft/json-schema.md)）

### B1. 檔案辨識方式

- [x] 以 YAML 中 `kind` 欄位判斷

### B2. IDE 整合方式

- [x] Schema Store 登錄（自動套用）

### B3. Schema 產出方式

- [x] 從引擎程式碼生成

---

## C. Instance Labels（[draft/instance-labels.md](draft/instance-labels.md)）

### C1. Label 值型別

- [x] 僅 string

### C2. `label_schema` 驗證強度

- [x] 僅建議（不驗證，允許任意 label）

### C3. 搜尋語法

- [x] 簡單 query parameter（`?label.region=tw&status=running`）

### C4. Storage 方案

- [x] `workflow_instances` 表的 JSONB 欄位

### C5. 動態設定 label 的方式

- [x] 內建 task `instance.set_labels`

---

## D. 本地測試模式（[draft/local-test.md](draft/local-test.md)）

### D1. Mock 檔案格式

- [x] YAML

### D2. Replay 比對策略

- [x] Strict — 所有 step output 必須完全一致

### D3. Agent step mock 粒度

- [x] 整個 agent（直接回傳 output）

### D4. 是否支援 `--watch` 模式

- [x] 否（v1 不做）

### D5. 是否支援互動式 debug 模式

- [x] 否（v1 不做）

---

## E. Continue-as-New（[draft/continue-as-new.md](draft/continue-as-new.md)）

### E1. Step type 命名

- [x] `renew`

### E2. 語法形式

- [x] **B** — `return` 的變體（`renew: true`）

### E3. 是否允許 continue 到不同版本的 workflow

- [x] 是

### E4. 舊 instance 資料保留策略

- [x] 兩者皆支援（自動清理 + 手動清理）

### E5. `CONTINUED` 是否計入 completed 統計

- [x] 是（視為成功完成的一種）

---

## F. Skills 專案資料夾（[draft/skills-project.md](draft/skills-project.md)）

### F1. Project 的本質

- [x] 獨立 kind（`kind: SkillProject`，有 metadata）

### F2. 是否支援巢狀 project

- [x] 是（`order/domestic/risk-analysis`）

### F3. `project.yaml` 是否必須

- [x] 必須（無此檔案則不視為 project）

### F4. Skill name 是否自動加 project prefix

- [x] 是（`risk-analysis` → `order/risk-analysis`）

---

## G. Toolset 概念（[draft/toolset.md](draft/toolset.md)）

### G1. 是否統一 `kind: Tools` 與 `kind: Toolset`

- [x] 統一為 `kind: Toolset`（取代 `kind: Tools`）

### G2. 引用語法

- [x] **A** — `type: toolset` 作為 tool 的一種引用方式

### G3. Step 層級是否可引用 toolset

- [x] 否（僅 agent definition）

### G4. Skill 是否仍可獨立引用（不透過 toolset）

- [x] 否（必須透過 toolset）

---

## H. DSL 擴充 — Resources（[draft/dsl-extensions.md](draft/dsl-extensions.md) §1）

### H1. `kind: Resources` 定義位置

- [x] 僅獨立檔案

### H2. 是否支援跨 namespace 引用

- [x] 否

---

## I. DSL 擴充 — String Template（[draft/dsl-extensions.md](draft/dsl-extensions.md) §3）

### I1. Template 變數定界符

- [x] `${ name }`（與 CEL 統一）

### I2. Template 是否支援巢狀引用

- [x] 否

### I3. `override` + `append` 是否可同時存在

- [x] 是（先 override 再 append）

---

## J. DSL 擴充 — Model Alias（[draft/dsl-extensions.md](draft/dsl-extensions.md) §4）

### J1. Alias 定義位置

- [x] 兩者皆可（引擎配置 + `kind: Resources`）

### J2. Agent 是否可覆寫 alias 的部分 config

- [x] 是（如只改 `max_tokens`）

---

## K. DSL 擴充 — 條件與路由（[draft/dsl-extensions.md](draft/dsl-extensions.md) §6-9）

### K1. `expr` 是否改名

- [x] 改為 `condition`

### K2. 是否引入 `when` 條件守衛

- [x] 否（用 `if` step 即可）

### K3. `switch` 是否加入 `between`

- [x] 否（用 CEL 表達式更靈活）

### K4. `between` 區間類型

- [x] N/A（K3 選否，不適用）

### K5. 是否加入 `router` step type

- [x] 否（改善 `switch` 語法即可）

---

## L. DSL 擴充 — `exists` 檢核（[draft/dsl-extensions.md](draft/dsl-extensions.md) §10）

### L1. exists 方案

- [x] **B** — CEL 原生 `has()` + `steps.<id>.status` 欄位

---

## M. Agent — 討論議題（[draft/agent.md](draft/agent.md) §10, §12）

### M1. Agent-as-Tool 的 prompt

- [x] Agent A 總是可透過 tool call input 傳入 prompt

### M2. `agent.llm_call` 的 messages

- [x] 允許傳入任意 messages（如做 summarization）

### M3. 是否需要 `agent.update_system_prompt`

- [x] 否（`agent.set_messages` 已足夠）

### M4. MCP Server 生命週期

- [x] **A** — 每個 agent session 獨立啟動/關閉

### M5. Tool Execution 並行性

- [x] **B** — 預設並行，tool 可標記 `sequential_only`

### M6. Skills 版本管理

- [x] 延後（v1 不做）

### M7. 自訂 loop 是否支援 `sub_workflow` / `agent` step

- [x] 是（移除限制）

### M8. RAG / Memory 機制

- [x] 透過 builtin tools 實現

---

## N. Artifact Workspace 模型（[draft/artifact-workspace.md](draft/artifact-workspace.md)）

> 已於 2026-04-02 完成決策，歸檔至 [history/20260402_artifact_workspace_decision.md](history/20260402_artifact_workspace_decision.md)。

### N1. Workspace 目錄是否支援自訂結構

- [x] **B** — 支援子目錄：artifact 可指定 `workspace_path` 放在子目錄中

### N2. 大檔案是否支援 lazy 下載

- [x] **A** — 所有 artifact 在 instance 建立時全部下載

### N3. 本機 source 是否支援目錄（不僅是檔案）

- [x] **B** — 支援目錄（遞迴複製或 mount）

### N4. 遠端 task 的 artifact API token 注入方式

- [x] **C** — 兩者皆支援

### N5. 同一 artifact 的並行寫入處理

- [x] **C** — 由 artifact 設定決定（`concurrency: lock | last_write_wins`）

---

## O. v6 規格一致性修正（2026-04-16 審閱，待決策）

審閱 v6 規格時發現以下不合理或實作困難之處，部分已直接修正，剩餘待決：

### O1. `if` / `switch` 的條件欄位與共通 `when` 衝突（已修正）

- [x] `if` 改用 `condition`；`switch` 改用 `subject`
- 共通 `when` 統一為前置條件（false → SKIPPED）
- `if.condition` / `switch.subject` 在共通 `when` 通過後才求值

### O2. `error.compensation_failures` 路徑不一致（已修正）

- [x] 統一為 `error.details.compensation_failures`（與 runtime error model 對齊）

### O3. wait event signals 缺少 scope 過濾（已修正）

- [x] 於 signals 事件訊號新增 `scope` 欄位（`workflow` / `project` / `global`，預設不限）
- 訂閱模型同步更新（07-event-bus.md / 08-persistence.md）

### O4. `type: callback` step 的 `name` 與 `id` 衝突（已修正）

- [x] 預設 `name` 作為 step 識別子；迴圈中多次呼叫同一 callback 時 MUST 顯式指定 `id`

### O5. Task registry resolve 無法處理 project prefix（已修正）

- [x] resolve 算法切分 project prefix（kebab-case）與 action body（snake_case + dotted）

### O6. 殘留 v5 namespace 引用（已修正）

- [x] 移除 `session` / `config` / `templates` 於 Runtime 04 的 write rules
- [x] 更新 lifecycle hooks 範圍說明
- [x] 修正不存在的 `12-determinism-and-replay` 引用

### O7. Artifact 寫入操作未定義（待決策）

- `artifacts.<name>.path` 等 namespace 已出現於 DSL spec
- 但 05-task-registry.md 目前 builtin 僅宣告 `artifact.list`，`read` / `write` 標為未來
- **選項 A**：v6 納入 `artifact.read` / `artifact.write` 作為核心 builtin
- **選項 B**：維持現狀，v6 只允許 tool 透過 `artifacts._workspace_path` 讀寫檔案
- **選項 C**：延後至 v2，移除 DSL spec 中 `artifacts.<name>` 用例

### O8. Workflow `config.catch` 允許的 step 類型（待決策）

- Runtime 09 目前以 SHOULD 建議「僅 emit/fail/return/assign」
- **選項 A**：升級為 MUST，載入時驗證（拒絕含 wait/task 等 I/O 型 step）
- **選項 B**：維持 SHOULD，由實作決定
- 考量：config.catch 執行在 instance 即將 FAILED 的終結流程中，若允許長 I/O 會延長終結時間並可能再度失敗

### O9. foreach 內 vars 並行寫入 race（待決策）

- 目前文件說「建議避免」，但 atomic 寫入僅保證單 key 級別
- **選項 A**：foreach iteration 內 `type: assign` 為編譯期禁止
- **選項 B**：維持現狀，由使用者自行確保 key 不衝突（以 `loop.index` 區分）
- **選項 C**：引入 iteration-local vars namespace（如 `iter.*`）

---

## P. v6 第二輪審閱（2026-04-16，待決策）

第二輪審閱發現 10 個問題，6 個已直接修正（wait step 訊號 race、saga compensate 狀態機、stdout.mapping 失敗處理、idempotency_key 範圍、async foreach 取消傳播、NDJSON 超長行）。剩餘 4 個需決策：

### P1. foreach / parallel `failure_policy: continue` 的 output 型別

目前 output 為陣列，失敗位置填 null。這使下游 CEL 無法區分「未執行」vs「執行成功但輸出為 null」。

- **選項 A**：output 結構改為 `{results: [...], failed_indices: [...]}`，失敗位置仍為 null 但有明確清單
- **選項 B**：output 每項為 `{status, output?, error?}` 物件（型別安全，但使用者需多一層解構）
- **選項 C**：維持現狀，使用者以 `has()` 或 `default()` 防禦

### P2. emit scope=workflow 在 function instance 內的傳播範圍

Function instance 內 emit scope=workflow 的事件，是否能被父 workflow 的 wait 訂閱？

- **選項 A**：scope=workflow 嚴格隔離於該 instance（function instance 內外互不相通）
- **選項 B**：scope=workflow 涵蓋「該 instance 及其所有後代 function instance」（即 instance tree）
- **選項 C**：引入 `scope: tree` 作為新選項，`workflow` 維持嚴格隔離

### P3. lifecycle destroy 失敗的 instance 終態處理

目前「destroy 失敗僅記錄，不影響 instance 最終狀態」，但若涉及關鍵資源（DB 連線池）洩漏，instance 宣稱成功但系統層有問題。

- **選項 A**：維持現狀（destroy 失敗僅警告）
- **選項 B**：instance 保留 SUCCEEDED 狀態，但加上 `destroy_failed: true` 欄位於 observability metadata
- **選項 C**：Tool definition 可標記 `lifecycle.destroy.critical: true`；critical destroy 失敗 → instance FAILED

### P4. wait signals 預設 scope 過濾

目前 scope 欄位 MAY，預設接受任一 scope（workflow/project/global）。多 project 環境中可能誤訂閱。

- **選項 A**：維持現狀（預設寬鬆，由使用者顯式收緊）
- **選項 B**：預設 scope=project（同 project 及以下），顯式寫 `scope: global` 才跨 project
- **選項 C**：MUST 欄位，強制使用者明確指定

---

## Q. v6 第三輪審閱（2026-04-16，待決策）

第三輪審閱新發現 8 項，4 項已直接修正（compensate_state 三態對齊、switch 型別相容規則、callback handler FAILED 漣漪、metric 定義補充）。剩餘 4 項：

### Q1. CLI / HTTP API 邊界規格

v6 DSL / Runtime 完整定義了內部執行，但未定義外部系統如何與 engine 互動：

- 如何觸發 manual workflow（input 傳輸格式、同步/非同步語意、返回值）
- 如何查詢 instance 狀態 / output / 執行日誌
- 如何 cancel instance / list instances / query by labels
- 認證授權模型（JWT 於 FUTURE.md 標示 v2）

**選項 A**：v6 新增 `11-external-interface.md` 定義 REST API（POST /workflows/.../trigger, GET /instances/...）
**選項 B**：v6 僅定義內部模型，CLI / API 延後至獨立 `cli-spec/` 規格
**選項 C**：僅定義 SPI（Service Provider Interface），由實作自行選擇 HTTP / gRPC / CLI

### Q2. Instance 查詢 API 介面

與 Q1 相關但獨立。至少需要：

- `GET /instances/{id}` — instance 完整狀態
- `GET /instances/{id}/steps/{step_id}` — 個別 step 狀態與 output
- `GET /instances?workflow=&state=&labels=` — 條件查詢
- `GET /instances/{id}/execution_log` — 執行日誌（for debug / replay）

是否隨 Q1 一併入 v6，或延後？

### Q3. CHANGES-FROM-V4.md 是否持續記錄 v6 內部修訂

當前 CHANGES-FROM-V4.md 只記錄 v4 → v6 差異。三輪審閱的修正（21+4 項）散落於 git log 與 DECISION.md。

**選項 A**：新增 `v6/CHANGELOG.md` 持續記錄 v6 內部修訂
**選項 B**：維持現狀，以 DECISION.md 的 O/P/Q 區為權威紀錄
**選項 C**：將 DECISION.md 已完成項目遷移至 history/ 並在 v6 README 加 changelog 章節

### Q4. config.catch 執行模型的運行資源上限

runtime-spec/09-error-model.md 提到 config.catch「建議僅允許 emit/fail/return/assign」，但未定義：

- config.catch 執行有無獨立 timeout？
- 若 config.catch 內的 emit 失敗（Event Bus 不可用），instance 最終狀態？

**選項 A**：config.catch 繼承 workflow.config.timeout 剩餘值；若 workflow 已 timeout 則 config.catch 有固定 30s 額外寬限
**選項 B**：config.catch 無 timeout；由實作以短 default（如 60s）限制
**選項 C**：config.catch 內失敗視為「double failure」，instance 標記 FAILED 且 error.details.catch_error 記錄
