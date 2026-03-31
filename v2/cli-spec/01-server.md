# 01 — Server 命令

管理 Slogan engine 的啟動與運行。

---

## slogan server start

啟動引擎。

```bash
slogan server start [options]
```

### 選項

| 選項 | 縮寫 | 預設 | 說明 |
|------|------|------|------|
| `--port <number>` | `-p` | `3000` | HTTP API 監聽埠 |
| `--host <address>` | | `127.0.0.1` | 監聽地址 |
| `--storage <dsn>` | | `sqlite://./slogan.db` | Storage 連線字串 |
| `--workers <number>` | `-w` | `1` | Worker 執行緒數量 |
| `--env-file <path>` | | `.env` | 環境變數檔案路徑 |
| `--secret-key` | | | 從環境變數 `SLOGAN_SECRET_KEY` 讀取（或互動提示） |
| `--shutdown-timeout <duration>` | | `30s` | Graceful shutdown 等待時間 |
| `--log-level <level>` | | `info` | Log 層級：`debug`、`info`、`warn`、`error` |
| `--daemon` | `-d` | | 背景執行 |

### Storage DSN 格式

| 格式 | 說明 |
|------|------|
| `sqlite://./slogan.db` | SQLite（預設） |
| `postgres://user:pass@host:5432/dbname` | PostgreSQL |

### 行為

1. 載入設定檔（`--config` 或預設路徑）
2. 命令列選項覆蓋設定檔中的對應值
3. 依 [runtime-spec/09-engine-lifecycle](../runtime-spec/09-engine-lifecycle.md) 定義的順序啟動
4. 啟動完成後輸出：

```
✓ Storage: sqlite://./slogan.db
✓ Secrets: 2 loaded
✓ Definitions: 5 published
✓ Recovery: 0 instances recovered
✓ Engine started on http://127.0.0.1:3000
```

### 信號處理

| 信號 | 行為 |
|------|------|
| `SIGTERM` / `SIGINT` | Graceful shutdown |
| 再次收到 `SIGTERM` / `SIGINT` | 強制關閉 |

---

## slogan server status

查詢引擎狀態（需引擎運行中）。

```bash
slogan server status [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--url <address>` | 引擎位址（預設：`http://127.0.0.1:3000`） |

### 輸出

```
Engine: running
Uptime: 2h 15m
Storage: sqlite://./slogan.db
Workers: 1
Active instances: 3
Published definitions: 5
```

---

## slogan server migrate

初始化或升級 storage schema。

```bash
slogan server migrate [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--storage <dsn>` | Storage 連線字串 |
| `--dry-run` | 僅顯示待執行的 migration，不實際執行 |

### 行為

1. 連線至 storage
2. 檢查當前 schema 版本
3. 執行待套用的 migrations
4. 輸出結果

```
Current version: 3
Applying migration 4... done
Applying migration 5... done
✓ Schema up to date (version 5)
```

首次執行時自動建立所有表。`slogan server start` 啟動時若偵測到 schema 不存在，會自動執行 migrate。
