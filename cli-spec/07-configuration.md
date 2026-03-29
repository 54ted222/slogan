# 07 — Configuration

引擎設定檔格式與載入規則。

---

## 設定檔格式

設定檔為 YAML 格式，預設路徑 `./slogan.yaml`。

```yaml
# slogan.yaml

server:
  host: 127.0.0.1
  port: 3000

storage:
  dsn: sqlite://./slogan.db

workers: 1

log:
  level: info            # debug | info | warn | error

env_file: .env

shutdown_timeout: 30s

# 引擎行為設定
engine:
  max_step_executions: 10000     # 單一 instance 最大 step 執行次數
  max_sub_workflow_depth: 10     # sub_workflow 最大巢狀深度
  lease_ttl: 30s                 # Instance lease 有效期
  lease_heartbeat: 10s           # Lease 續租間隔
  timeout_poll_interval: 1s     # Timeout Manager 輪詢間隔

# 並發控制
concurrency:
  max_running_steps: 0           # 同時 RUNNING 的 step 上限（0 = 無限制）
  max_running_instances: 0       # 同時 RUNNING/WAITING 的 instance 上限（0 = 無限制）
  queue_limit: 1000              # 排隊 step 上限
  task_limits: []                # Per-action 並發限制（見下方說明）
  # task_limits:
  #   - action: "payment.create"
  #     max_concurrent: 5
  #   - action: "email.*"
  #     max_concurrent: 10

# Dead-letter queue
dead_letter:
  max_route_attempts: 3          # 事件最大路由嘗試次數

# Payload 大小限制
limits:
  max_event_data_bytes: 1048576    # Event data 最大大小（預設 1MB）
  max_step_output_bytes: 1048576   # Step output 最大大小（預設 1MB）

# Artifact storage
artifact_storage:
  backend: local                   # local | s3
  local:
    base_path: ./artifacts         # 本地儲存根目錄
  # s3:
  #   bucket: my-bucket
  #   prefix: slogan/artifacts
  #   region: ap-northeast-1

# 資料保留
retention:
  instances: 90d          # Workflow instance 保留期限
  execution_logs: 90d     # Execution log 保留期限
  events: 7d              # Event record 保留期限
  dead_letter_events: 30d  # DLQ 事件保留期限
```

---

## 載入優先順序

設定值依以下優先順序決定（高優先覆蓋低優先）：

1. **命令列選項**（如 `--port 8080`）
2. **環境變數**（如 `SLOGAN_PORT=8080`）
3. **設定檔**（`slogan.yaml`）
4. **內建預設值**

---

## 環境變數對應

| 環境變數 | 對應設定 | 說明 |
|---------|---------|------|
| `SLOGAN_HOST` | `server.host` | 監聽地址 |
| `SLOGAN_PORT` | `server.port` | 監聽埠 |
| `SLOGAN_STORAGE_DSN` | `storage.dsn` | Storage 連線字串 |
| `SLOGAN_WORKERS` | `workers` | Worker 數量 |
| `SLOGAN_LOG_LEVEL` | `log.level` | Log 層級 |
| `SLOGAN_SECRET_KEY` | — | Secret 解密密碼（不寫入設定檔） |
| `SLOGAN_ENV_FILE` | `env_file` | .env 檔案路徑 |
| `SLOGAN_SHUTDOWN_TIMEOUT` | `shutdown_timeout` | Graceful shutdown 等待 |

---

## 預設值

| 設定 | 預設值 |
|------|--------|
| `server.host` | `127.0.0.1` |
| `server.port` | `3000` |
| `storage.dsn` | `sqlite://./slogan.db` |
| `workers` | `1` |
| `log.level` | `info` |
| `env_file` | `.env` |
| `shutdown_timeout` | `30s` |
| `engine.max_step_executions` | `10000` |
| `engine.max_sub_workflow_depth` | `10` |
| `engine.lease_ttl` | `30s` |
| `engine.lease_heartbeat` | `10s` |
| `engine.timeout_poll_interval` | `1s` |
| `concurrency.max_running_steps` | `0`（無限制） |
| `concurrency.max_running_instances` | `0`（無限制） |
| `concurrency.queue_limit` | `1000` |
| `dead_letter.max_route_attempts` | `3` |
| `limits.max_event_data_bytes` | `1048576`（1MB） |
| `limits.max_step_output_bytes` | `1048576`（1MB） |
| `artifact_storage.backend` | `local` |
| `artifact_storage.local.base_path` | `./artifacts` |
| `retention.instances` | `90d` |
| `retention.execution_logs` | `90d` |
| `retention.events` | `7d` |
| `retention.dead_letter_events` | `30d` |

---

## 零配置啟動

無設定檔時，引擎以所有預設值啟動（SQLite、單 worker、port 3000）。適用於開發與快速測試：

```bash
$ slogan server start
# 等同於：
# storage: sqlite://./slogan.db
# port: 3000
# workers: 1
```

---

## 驗證

引擎啟動時 MUST 驗證設定檔：

| 檢查項目 | 說明 |
|---------|------|
| YAML 語法 | 設定檔可解析 |
| DSN 格式 | `storage.dsn` 為合法的連線字串 |
| 數值範圍 | `workers` >= 1、`port` 1-65535 |
| Duration 格式 | 所有 duration 欄位為合法格式（如 `30s`、`5m`、`2h`） |
| SQLite + workers | `workers` > 1 且使用 SQLite 時發出警告（不阻止啟動） |
| Artifact backend | `artifact_storage.backend` 為 `local` 或 `s3` |
| S3 設定 | backend 為 `s3` 時，`bucket` 與 `region` MUST 存在 |
| 大小限制 | `limits.*` 值 > 0 |

驗證失敗 → 引擎不啟動，exit code = 3。
