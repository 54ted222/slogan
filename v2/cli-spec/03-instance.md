# 03 — Instance 命令

管理 workflow instances 的建立、查詢與取消。

---

## slogan instance create

建立新的 workflow instance（manual trigger）。

```bash
slogan instance create <workflow_name> [options]
```

### 選項

| 選項 | 縮寫 | 說明 |
|------|------|------|
| `--version <number>` | | 指定版本（預設 latest） |
| `--input <json>` | `-i` | JSON 格式的輸入資料 |
| `--input-file <path>` | | 從 JSON 檔案讀取輸入 |
| `--wait` | `-w` | 等待 instance 完成後才返回 |
| `--timeout <duration>` | | `--wait` 的等待上限（預設 5m） |

### 行為

1. 呼叫 Engine API `CreateInstance`
2. 回傳 instance ID
3. 若使用 `--wait`，輪詢 instance 狀態直到 terminal

```bash
$ slogan instance create order_fulfillment --input '{"order_id": "ORD-001"}'
✓ Created: inst_abc123
  Workflow: order_fulfillment@1
  State:    CREATED

$ slogan instance create order_fulfillment -i '{"order_id": "ORD-001"}' --wait
✓ Created: inst_abc123
⠋ Running...
✓ Completed: inst_abc123
  State:  SUCCEEDED
  Output: {"status": "completed", "payment_id": "PAY-789"}
```

---

## slogan instance get

查詢 instance 詳細資訊。

```bash
slogan instance get <instance_id> [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--steps` | 同時顯示所有 step instances |

```bash
$ slogan instance get inst_abc123
ID:         inst_abc123
Workflow:   order_fulfillment@1
State:      RUNNING
Input:      {"order_id": "ORD-001"}
Created:    2024-01-15 10:30:00
Started:    2024-01-15 10:30:01

$ slogan instance get inst_abc123 --steps
...
STEP ID         TYPE    STATE      STARTED              COMPLETED
load_order      task    SUCCEEDED  2024-01-15 10:30:01  2024-01-15 10:30:02
validate_order  task    RUNNING    2024-01-15 10:30:02  -
send_receipt    task    PENDING    -                    -
```

---

## slogan instance list

列出 instances。

```bash
slogan instance list [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--workflow <name>` | 篩選 workflow 名稱 |
| `--state <state>` | 篩選狀態 |
| `--since <duration>` | 篩選建立時間（如 `1h`、`7d`） |
| `--limit <number>` | 每頁筆數（預設 20） |

```bash
$ slogan instance list --workflow order_fulfillment --state running
ID            WORKFLOW             STATE    CREATED
inst_abc123   order_fulfillment@1  RUNNING  2024-01-15 10:30:00
inst_def456   order_fulfillment@1  RUNNING  2024-01-15 10:35:00
```

---

## slogan instance cancel

取消正在執行的 instance。

```bash
slogan instance cancel <instance_id> [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--reason <text>` | 取消原因 |

```bash
$ slogan instance cancel inst_abc123 --reason "manual intervention"
✓ Cancelled: inst_abc123
```

---

## slogan instance logs

查看 instance 的 execution log。

```bash
slogan instance logs <instance_id> [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--follow` / `-f` | 持續輸出新的 log（instance 運行中時） |
| `--step <step_id>` | 篩選特定 step |
| `--type <event_type>` | 篩選事件類型 |

```bash
$ slogan instance logs inst_abc123
TIME                  STEP           TYPE            DATA
10:30:01.123  load_order     state_changed   PENDING → READY
10:30:01.124  load_order     state_changed   READY → RUNNING
10:30:02.456  load_order     state_changed   RUNNING → SUCCEEDED
10:30:02.457  validate       state_changed   PENDING → READY
...
```

---

## slogan instance output

取得已完成 instance 的 output。

```bash
slogan instance output <instance_id>
```

```bash
$ slogan instance output inst_abc123
{"status": "completed", "payment_id": "PAY-789"}
```

Instance 非 SUCCEEDED 狀態時回傳錯誤。
