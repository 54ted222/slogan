# TODO — DSL v2 規格補完清單

1. **Saga / Compensation 模式** — 宣告式補償區塊設計，語言層面決策，越晚加入越難
2. **JSON Schema 發布** — 為 YAML DSL 產出 JSON Schema，提供 IDE autocompletion / linting
3. **Instance Labels + 搜尋 API** — instance 自訂標籤（key-value）與進階查詢（按標籤、時間範圍、workflow name 過濾）
4. **本地測試模式** — `slogan test` 命令，支援本地執行 workflow、mock task backend、replay 歷史紀錄驗證
5. **Continue-as-new** — 長時間 workflow 重置機制，帶新 state 重啟並截斷歷史，避免無限增長

6. **`kind: tools`** — 工具集合宣告，將多個工具統一定義與管理
7. **`kind: resources`** — 共用資源宣告區域（mcps, string templates, skills, vars, artifact 等）
8. **`expr` 條件關鍵字調整** — 目前 `expr` 命名不直觀，難以推斷用途，需重新命名或改善語意
9. **分析加入 `when` 的好壞** — 評估是否引入 `when` 作為條件判斷語法，分析優缺點
10. **`switch` 加入 `between`** — 在 switch step 中支援 `between` 區間比對
11. **考慮加入 `router`** — 簡化版 switch，以 key → steps 映射簡化路由邏輯

## agent 功能

string template + variable substitution

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
