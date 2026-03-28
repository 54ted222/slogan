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

# 資料保留
retention:
  instances: 90d          # Workflow instance 保留期限
  execution_logs: 90d     # Execution log 保留期限
  events: 7d              # Event record 保留期限
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
| `retention.instances` | `90d` |
| `retention.execution_logs` | `90d` |
| `retention.events` | `7d` |

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

驗證失敗 → 引擎不啟動，exit code = 3。
