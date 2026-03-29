# Future Features

以下功能已納入考量但暫不實作，待核心引擎穩定後再依需求優先順序加入。

> **注意**：Scheduled Trigger、Delayed Event、HTTP Trigger、Authentication & Authorization 已有完整規格（見各 spec 文件），但實作上列為第二階段。

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
