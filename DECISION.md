# 待決策清單

> 所有草稿中尚未定案的設計選擇，依主題分類。
> 在選項後方標記你的決定（如 `← 選這個`），或加入備註。

---

## A. Saga / Compensation（[draft/saga-compensation.md](draft/saga-compensation.md)）

### A1. 補償區塊結構

- [ ] **A** — Step 層級 `compensate`（每個 step 各自宣告補償）
- [ ] **B** — `saga` 區塊包裹一組 step
- [ ] **C** — 全域 compensation 策略（workflow 層級集中定義）

### A2. 補償失敗行為

- [ ] **Best-effort** — 個別補償失敗不中止，繼續執行其餘補償
- [ ] **中止** — 任一補償失敗即停止

### A3. 補償是否支援重試

- [ ] 是
- [ ] 否

### A4. 是否需要 forward recovery

- [ ] 是（支援重試前進，不只回滾）
- [ ] 否（僅支援 backward compensation）

---

## B. JSON Schema 發布（[draft/json-schema.md](draft/json-schema.md)）

### B1. 檔案辨識方式

- [ ] 檔案命名慣例（`*.workflow.yaml`、`*.agent.yaml`）
- [ ] 以 YAML 中 `kind` 欄位判斷
- [ ] 兩者皆支援

### B2. IDE 整合方式

- [ ] Schema Store 登錄（自動套用）
- [ ] YAML modeline（使用者手動加）
- [ ] 專用 VS Code extension
- [ ] 以上皆做

### B3. Schema 產出方式

- [ ] 手動撰寫
- [ ] 從 spec markdown 自動生成
- [ ] 從引擎程式碼生成

---

## C. Instance Labels（[draft/instance-labels.md](draft/instance-labels.md)）

### C1. Label 值型別

- [ ] 僅 string
- [ ] 支援 string、integer、boolean

### C2. `label_schema` 驗證強度

- [ ] 僅建議（不驗證，允許任意 label）
- [ ] 強制（缺少 required label 時拒絕建立 instance）
- [ ] 可配置（definition 層級選擇 strict / lenient）

### C3. 搜尋語法

- [ ] 簡單 query parameter（`?label.region=tw&status=running`）
- [ ] 進階 filter DSL（`?filter=label.region:tw AND status:running`）
- [ ] 兩者皆支援（簡單 = 語法糖，底層統一）

### C4. Storage 方案

- [ ] 獨立 `instance_labels` 表
- [ ] `workflow_instances` 表的 JSONB 欄位

### C5. 動態設定 label 的方式

- [ ] 專用 `set_labels` step type
- [ ] 內建 task `instance.set_labels`
- [ ] 兩者皆支援

---

## D. 本地測試模式（[draft/local-test.md](draft/local-test.md)）

### D1. Mock 檔案格式

- [ ] YAML
- [ ] JSON
- [ ] 兩者皆支援

### D2. Replay 比對策略

- [ ] Strict — 所有 step output 必須完全一致
- [ ] Loose — 僅驗證控制流（分支選擇、最終 output）

### D3. Agent step mock 粒度

- [ ] 整個 agent（直接回傳 output）
- [ ] 個別 tool call（模擬 agentic loop 每一步）

### D4. 是否支援 `--watch` 模式

- [ ] 是
- [ ] 否（v1 不做）

### D5. 是否支援互動式 debug 模式

- [ ] 是（step-by-step 暫停 + 檢視 state）
- [ ] 否（v1 不做）

---

## E. Continue-as-New（[draft/continue-as-new.md](draft/continue-as-new.md)）

### E1. Step type 命名

- [ ] `continue_as_new`
- [ ] `restart`
- [ ] `renew`

### E2. 語法形式

- [ ] **A** — 專用 step type
- [ ] **B** — `return` 的變體（`continue_as_new: true`）

### E3. 是否允許 continue 到不同版本的 workflow

- [ ] 是
- [ ] 否（僅限相同 name + version）

### E4. 舊 instance 資料保留策略

- [ ] 自動清理（可配置保留天數）
- [ ] 手動清理
- [ ] 兩者皆支援

### E5. `CONTINUED` 是否計入 completed 統計

- [ ] 是（視為成功完成的一種）
- [ ] 否（獨立統計）

---

## F. Skills 專案資料夾（[draft/skills-project.md](draft/skills-project.md)）

### F1. Project 的本質

- [ ] 僅資料夾組織（無 kind 定義）
- [ ] 獨立 kind（`kind: SkillProject`，有 metadata）

### F2. 是否支援巢狀 project

- [ ] 是（`order/domestic/risk-analysis`）
- [ ] 否（僅一層）

### F3. `project.yaml` 是否必須

- [ ] 必須（無此檔案則不視為 project）
- [ ] 可省略（資料夾即 project）

### F4. Skill name 是否自動加 project prefix

- [ ] 是（`risk-analysis` → `order/risk-analysis`）
- [ ] 否（name 維持原樣，project 僅為分類）

---

## G. Toolset 概念（[draft/toolset.md](draft/toolset.md)）

### G1. 是否統一 `kind: Tools` 與 `kind: Toolset`

- [ ] 統一為 `kind: Toolset`（取代 `kind: Tools`）
- [ ] 兩者共存（Tools = 純工具集合，Toolset = tools + skills）

### G2. 引用語法

- [ ] **A** — `type: toolset` 作為 tool 的一種引用方式
- [ ] **B** — `capabilities` 統一欄位取代 `tools` + `skills`

### G3. Step 層級是否可引用 toolset

- [ ] 是（agent definition + step 皆可）
- [ ] 否（僅 agent definition）

### G4. Skill 是否仍可獨立引用（不透過 toolset）

- [ ] 是（向後相容）
- [ ] 否（必須透過 toolset）

---

## H. DSL 擴充 — Resources（[draft/dsl-extensions.md](draft/dsl-extensions.md) §1）

### H1. `kind: Resources` 定義位置

- [ ] 僅獨立檔案
- [ ] 可嵌入 workflow/agent definition
- [ ] 兩者皆可

### H2. 是否支援跨 namespace 引用

- [ ] 是
- [ ] 否

---

## I. DSL 擴充 — String Template（[draft/dsl-extensions.md](draft/dsl-extensions.md) §3）

### I1. Template 變數定界符

- [ ] `{name}`（與 CEL `${ }` 區分）
- [ ] `${ name }`（與 CEL 統一）
- [ ] `{{ name }}`（雙括號）

### I2. Template 是否支援巢狀引用

- [ ] 是（template 可引用另一個 template）
- [ ] 否

### I3. `override` + `append` 是否可同時存在

- [ ] 是（先 override 再 append）
- [ ] 否（互斥）

---

## J. DSL 擴充 — Model Alias（[draft/dsl-extensions.md](draft/dsl-extensions.md) §4）

### J1. Alias 定義位置

- [ ] 引擎配置
- [ ] `kind: Resources`
- [ ] 兩者皆可

### J2. Agent 是否可覆寫 alias 的部分 config

- [ ] 是（如只改 `max_tokens`）
- [ ] 否（alias 為不可覆寫的完整定義）

---

## K. DSL 擴充 — 條件與路由（[draft/dsl-extensions.md](draft/dsl-extensions.md) §6-9）

### K1. `expr` 是否改名

- [ ] 改為 `condition`
- [ ] 改為 `test`
- [ ] 改為 `when`
- [ ] 維持 `expr`（不值得 breaking change）

### K2. 是否引入 `when` 條件守衛

- [ ] 是（`when` 附加在任意 step 上，簡化單一 step 條件執行）
- [ ] 否（用 `if` step 即可）

### K3. `switch` 是否加入 `between`

- [ ] 是（閉區間 `[min, max]`）
- [ ] 否（用 CEL 表達式更靈活）

### K4. `between` 區間類型

> 僅 K3 選「是」時需決定

- [ ] 僅閉區間 `[min, max]`
- [ ] 支援開區間（`(0, 30)`、`[0, 30)`）
- [ ] 支援無上界/無下界（`>= 71`）

### K5. 是否加入 `router` step type

- [ ] 是（簡化版 switch，key → steps 映射）
- [ ] 否（改善 `switch` 語法即可）

---

## L. DSL 擴充 — `exists` 檢核（[draft/dsl-extensions.md](draft/dsl-extensions.md) §10）

### L1. exists 方案

- [ ] **A** — 自訂 CEL 函式 `exists()`
- [ ] **B** — CEL 原生 `has()` + `steps.<id>.status` 欄位
- [ ] **C** — 未執行 step 回傳 `null`，用 `!= null` 判斷

---

## M. Agent — 討論議題（[draft/agent.md](draft/agent.md) §10, §12）

### M1. Agent-as-Tool 的 prompt

- [ ] Agent A 總是可透過 tool call input 傳入 prompt
- [ ] 僅在 Agent B 沒有預設 prompt 時才允許
- [ ] 強制由引擎從 input 自動組合

### M2. `agent.llm_call` 的 messages

- [ ] 強制使用 `session.messages`
- [ ] 允許傳入任意 messages（如做 summarization）

### M3. 是否需要 `agent.update_system_prompt`

- [ ] 是（提供便捷 task，不需手動操作 messages 陣列）
- [ ] 否（`agent.set_messages` 已足夠）

### M4. MCP Server 生命週期

- [ ] **A** — 每個 agent session 獨立啟動/關閉
- [ ] **B** — 引擎層級 connection pool，多 session 共用
- [ ] **C** — 可配置（per-session vs shared）

### M5. Tool Execution 並行性

- [ ] **A** — 總是循序執行
- [ ] **B** — 預設並行，tool 可標記 `sequential_only`
- [ ] **C** — 可配置

### M6. Skills 版本管理

- [ ] 僅本地路徑（目前）
- [ ] 支援遠端 skill（git URL）+ 版本鎖定
- [ ] 延後（v1 不做）

### M7. 自訂 loop 是否支援 `sub_workflow` / `agent` step

- [ ] 是（移除限制）
- [ ] 否（維持 agent-as-tool 機制）

### M8. RAG / Memory 機制

- [ ] 透過 MCP tools 實現（保持 agent 核心簡潔）
- [ ] 透過 builtin tools 實現
- [ ] 內建於 agent 核心
- [ ] 延後設計
