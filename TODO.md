# TODO — DSL v2 規格補完清單

1. **Saga / Compensation 模式** — 宣告式補償區塊設計，語言層面決策，越晚加入越難
2. **JSON Schema 發布** — 為 YAML DSL 產出 JSON Schema，提供 IDE autocompletion / linting
3. **Instance Labels + 搜尋 API** — instance 自訂標籤（key-value）與進階查詢（按標籤、時間範圍、workflow name 過濾）
4. **本地測試模式** — `slogan test` 命令，支援本地執行 workflow、mock task backend、replay 歷史紀錄驗證
5. **Continue-as-new** — 長時間 workflow 重置機制，帶新 state 重啟並截斷歷史，避免無限增長

## FUTURE.md 延後項目

- Event Replay API
- Transactional Outbox / Inbox Pattern
- JWT Authentication（公鑰配置）
- Pause / Resume 機制
- Bulk Operations
- Scheduled Trigger DST 轉換行為
