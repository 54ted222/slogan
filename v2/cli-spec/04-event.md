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

---

## slogan event dlq

管理 dead-letter queue 中無法處理的事件。

### 子命令

#### slogan event dlq list

列出 DLQ 中的事件。

```bash
slogan event dlq list [options]
```

| 選項 | 說明 |
|------|------|
| `--since <time>` | 只顯示此時間之後的記錄 |
| `--reason <reason>` | 篩選原因（`no_match` / `routing_failed` / `poison`） |
| `--limit <n>` | 最多顯示筆數（預設 20） |

```bash
$ slogan event dlq list --reason poison --limit 5
ID          EVENT_ID     TYPE              REASON  ATTEMPTS  CREATED_AT
dlq_001     evt_abc123   order.created     poison  3         2024-01-15T10:30:00Z
dlq_002     evt_def456   payment.confirm   poison  3         2024-01-15T11:00:00Z
```

#### slogan event dlq inspect

查看 DLQ 事件詳情。

```bash
slogan event dlq inspect <dlq_id>
```

```bash
$ slogan event dlq inspect dlq_001
ID:           dlq_001
Event ID:     evt_abc123
Type:         order.created
Reason:       poison
Attempts:     3
Error:        CEL expression evaluation failed: undefined field 'customer_id'
Created At:   2024-01-15T10:30:00Z

Data:
  {"order_id": "ORD-999", "amount": 50}
```

#### slogan event dlq retry

重新投遞事件至 Event Router。

```bash
slogan event dlq retry <dlq_id>
```

成功投遞後從 DLQ 移除。若再次失敗則保留在 DLQ 並更新 `attempt_count`。

#### slogan event dlq drop

從 DLQ 移除事件（放棄處理）。

```bash
slogan event dlq drop <dlq_id>
```
