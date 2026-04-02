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
