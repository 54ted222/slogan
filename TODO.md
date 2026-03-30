# TODO — DSL v2 規格補完清單

1. **Saga / Compensation 模式** — 宣告式補償區塊設計，語言層面決策，越晚加入越難
2. **JSON Schema 發布** — 為 YAML DSL 產出 JSON Schema，提供 IDE autocompletion / linting
3. **Instance Labels + 搜尋 API** — instance 自訂標籤（key-value）與進階查詢（按標籤、時間範圍、workflow name 過濾）
4. **本地測試模式** — `slogan test` 命令，支援本地執行 workflow、mock task backend、replay 歷史紀錄驗證
5. **Continue-as-new** — 長時間 workflow 重置機制，帶新 state 重啟並截斷歷史，避免無限增長

## agent 功能

6. **`kind: resources`** — 共用資源宣告區域（mcps, string templates, skills, vars, artifact 等）
7. **`kind: tools`** — 工具集合宣告，將多個工具統一定義與管理
8. **`expr` 條件關鍵字調整** — 目前 `expr` 命名不直觀，難以推斷用途，需重新命名或改善語意
9. **分析加入 `when` 的好壞** — 評估是否引入 `when` 作為條件判斷語法，分析優缺點
10. **`switch` 加入 `between`** — 在 switch step 中支援 `between` 區間比對
11. **考慮加入 `router`** — 簡化版 switch，以 key → steps 映射簡化路由邏輯
12. **`exists` 檢核機制** — 類似 `exists` 的語法或工具，判斷變數、物件、ID 是否存在
13. 評估 agent 使用時 exclude append or override prompt/tools 功能，因該優先使用 input
14. agent 提供 標準 hook
15. agent loop 可以修改 prompt, system 的修改方式

string template + variable substitution

### 待定義項目

- **Agent built-in 工具清單** — 列出 agent 內建工具（如 `get_tools`、`call_llms` 等），定義各工具的介面與行為
- **History 定義與處理** — 定義 history 的資料結構、操作方式（讀取、截斷、摘要）及持久化方案
- **Sub-agent / Fork agent** — 如何從 agent 中啟動子 agent 或 fork 新 agent，定義父子關係與通訊機制
- **RAG / Memory 機制** — 評估是否透過 tools 方式實現 RAG 與長期記憶，而非內建於 agent 核心
- **Agent Team** — 多 agent 協作機制，包含共用 team tools、message box、任務分派與結果匯集

```yaml
id: "greeting"
type: string_template
template: "Hello, {name}! Today is {day}."
input_schema: object # json schema 定義變數

prompts:
  override:
    - use_template: "greeting"
      input:
        name: "Bob"
        day: "Tuesday"
    - "abcdef ${vars.name}"
  append:
    - use_template: "greeting"
      input:
        name: "Alice"
        day: "Monday"
    - "xyz ${vars.name}"
```

## FUTURE.md 延後項目

- Event Replay API
- Transactional Outbox / Inbox Pattern
- JWT Authentication（公鑰配置）
- Pause / Resume 機制
- Bulk Operations
- Scheduled Trigger DST 轉換行為
