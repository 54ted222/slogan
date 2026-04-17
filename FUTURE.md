# Future Features

以下功能已納入考量但暫不實作，待核心引擎穩定後再依需求優先順序加入。

> **注意**：Scheduled Trigger、Delayed Event、HTTP Trigger、Authentication & Authorization 已有完整規格（見各 spec 文件），但實作上列為第二階段。

---

## Event Replay API（事件重播）

提供從特定時間點重播事件的能力，用於除錯與災難復原。

設計考量：

- Replay-from-timestamp、replay-range 等 API
- 重播期間的 dedup 處理策略
- 重播進度追蹤與中止機制

---

## Transactional Outbox / Inbox Pattern

將 event publish 與 workflow 狀態變更包在同一筆交易中，確保一致性。

設計考量：

- Outbox table 設計與 polling / CDC 機制
- Inbox table 用於冪等接收外部事件
- 與現有 at-least-once 保證的整合方式

---

## JWT Authentication（JWT 驗證與公鑰配置）

支援 JWT Bearer Token 驗證，允許對接外部 Identity Provider。

設計考量：

- 公鑰載入方式（設定檔、JWKS endpoint）
- 支援 RSA / ECDSA 演算法
- Token 過期與 refresh 處理
- 可選的 OIDC / OAuth2 整合

---

## Pause / Resume（手動暫停與恢復）

有別於 `wait_event` 的系統暫停，提供人工手動暫停與恢復 workflow instance 的能力。

```yaml
# 透過 API 暫停
POST /instances/{id}/pause

# 透過 API 恢復
POST /instances/{id}/resume
```

設計考量：

- 新增 instance 狀態 `PAUSED`（或復用 WAITING + 標記）
- PAUSED 期間所有 step 執行暫停，timeout 計時暫停
- Resume 後從暫停點繼續執行
- PAUSED 期間收到的事件排隊等待 resume
- CLI 支援：`slogan instance pause <id>`、`slogan instance resume <id>`

---

## Bulk Operations（批次操作）

支援批次建立、取消 workflow instances 與進階查詢。

```
POST /instances/bulk-create
POST /instances/bulk-cancel
GET  /instances?filter=state:running,workflow:order_*&sort=created_at:desc
```

設計考量：

- Bulk create 接受 instance 陣列，回傳建立結果陣列
- Bulk cancel 接受 instance ID 陣列，回傳取消結果
- 進階查詢支援 filter 語法（按 state、workflow name、labels、時間範圍）
- 分頁支援（cursor-based pagination）

---

## Label Schema（Instance Label 建議性 Schema）

在 Workflow Definition 頂層宣告 `label_schema`，描述 instance 預期使用的 label keys，用於文件化與 IDE 提示。

```yaml
label_schema:
  customer_id:
    type: string
    description: "客戶 ID"
  region:
    description: "區域代碼"
```

設計考量：

- 純建議性（advisory only），引擎不以此拒絕 instance 建立
- 主要用途為文件化與 IDE 自動補全
- 待確認是否需要更嚴格的驗證模式（如 `strict: true` 選項）

---

## Stream Tool（含生命週期與優雅關閉）

v6 原規劃的 content-level stream 能力（tool 即時串流輸出、`tool.stream` 事件、SSE `event: stream`、`stream.enabled` 等）於 2026-04-17 審閱中發現生命週期與優雅關閉機制有明確缺口（graceful close 訊號、finalizer / on_close hook、`stream.max_duration`、長駐 application 概念），暫不納入 v6。

完整原設計歸檔與未來議題見 [future/v6-stream-lifecycle.md](future/v6-stream-lifecycle.md)。

---

## Scheduled Trigger DST 處理

定義 scheduled trigger 在日光節約時間（DST）轉換時的行為。

設計考量：

- 時鐘前撥（spring forward）：被跳過的觸發時間是否補觸發
- 時鐘後撥（fall back）：重複的觸發時間是否觸發兩次
- 與 cron 表達式的交互（如 `0 2 * * *` 在 DST 轉換日的行為）
