# TODO — DSL v2 規格補完清單

1. **Saga / Compensation 模式** — 宣告式補償區塊設計，語言層面決策，越晚加入越難
2. **JSON Schema 發布** — 為 YAML DSL 產出 JSON Schema，提供 IDE autocompletion / linting
3. **Instance Labels + 搜尋 API** — instance 自訂標籤（key-value）與進階查詢（按標籤、時間範圍、workflow name 過濾）
4. **本地測試模式** — `slogan test` 命令，支援本地執行 workflow、mock task backend、replay 歷史紀錄驗證
5. **Continue-as-new** — 長時間 workflow 重置機制，帶新 state 重啟並截斷歷史，避免無限增長

## agent 功能

詳見 [draft/agent.md](draft/agent.md)（第十節「待設計功能」、第十一節「待定義項目」）

### 新增待設計項目

16. **Session ID 概念** — 定義 agent session ID 的產生、用途、與 workflow instance 的關聯
17. **Agent 引用 Artifact** — agent 如何讀取、寫入、引用 workflow artifact
18. **Model Alias** — 定義 model alias 名稱機制，避免在多處硬編碼 model name
19. **Tool 倉庫搜尋工具** — 一種 tools 讓 agent 能在 tool registry 中搜尋與發現可用工具
20. **`ask_human` 格式擴充** — 支援提供選項（選擇題）與一次問多個問題
21. **Tool 名稱兩段式** — 採用 `namespace.action` 格式（如 `history.write`），以點號分隔
22. **Agent 支援 `sub_workflow` 與 `agent` step** — 自訂 loop 中允許使用 `sub_workflow` 和 `agent` step 類型
23. **Agent Stream 支援** — 定義 agent 如何支援 streaming 輸出（即時回傳部分結果）

### 待修正項目

- **移除 `type: builtin`** — 內建 tool 統一使用 `type: tool`，不另設 `builtin` 類型
- **Skills 引用改為 name** — `skills` 改為指定 name 而非 path，路徑在 `kind: resources` 宣告處定義

## FUTURE.md 延後項目

- Event Replay API
- Transactional Outbox / Inbox Pattern
- JWT Authentication（公鑰配置）
- Pause / Resume 機制
- Bulk Operations
- Scheduled Trigger DST 轉換行為
