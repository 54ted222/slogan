# 04 — Event 命令

發送事件至引擎。

---

## slogan event send

發送事件。

```bash
slogan event send <event_type> [options]
```

### 選項

| 選項 | 縮寫 | 說明 |
|------|------|------|
| `--data <json>` | `-d` | JSON 格式的事件酬載 |
| `--data-file <path>` | | 從 JSON 檔案讀取酬載 |
| `--id <event_id>` | | 指定事件 ID（預設自動產生） |
| `--source <source>` | `-s` | 事件來源 |

### 行為

1. 呼叫 Engine API `SendEvent`
2. 回傳事件 ID、觸發的 instances、恢復的 instances

```bash
$ slogan event send order.created -d '{"order_id": "ORD-001", "amount": 100}'
✓ Event sent: evt_xyz789
  Triggered: 1 new instance (inst_abc123)
  Resumed:   0 instances

$ slogan event send payment.confirmed -d '{"payment_id": "PAY-789"}'
✓ Event sent: evt_xyz790
  Triggered: 0 new instances
  Resumed:   1 instance (inst_abc123)
```
