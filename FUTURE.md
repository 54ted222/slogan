# Future Features

以下功能已納入考量但暫不實作，待核心引擎穩定後再依需求優先順序加入。

---

## Scheduled Trigger（定時觸發）

支援 cron 表達式或固定間隔的定時觸發，自動建立 workflow instance。

```yaml
triggers:
  - type: scheduled
    cron: "0 9 * * MON-FRI"
```

---

## Delayed Event（延遲事件）

支援事件延遲發送，在指定時間後才將事件投遞至 Event Router。

```yaml
- type: emit
  event: order.reminder
  delay: 30m
```

---

## HTTP Trigger（HTTP 觸發）

透過 HTTP endpoint 直接觸發 workflow instance，引擎自動產生對應的 REST API。

```yaml
triggers:
  - type: http
    method: POST
    path: /api/orders
```

---

## Authentication & Authorization（調用權限控制）

針對 API 呼叫與 trigger 端點的身份驗證與授權機制，包含：

- API key / Bearer token 驗證
- 基於角色的存取控制（RBAC）
- Definition 與 instance 層級的權限管理
- Trigger endpoint 的存取限制
