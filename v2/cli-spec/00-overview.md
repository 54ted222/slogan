# 00 — CLI 總覽

本規格定義 `slogan` 命令列工具的介面與行為。CLI 是開發者與 Slogan engine 互動的主要入口，涵蓋 definition 管理、引擎啟動、instance 操作與 secret 管理。

---

## 命令結構

```
slogan <command> [subcommand] [arguments] [options]
```

### 全域選項

| 選項 | 縮寫 | 說明 |
|------|------|------|
| `--config <path>` | `-c` | 指定設定檔路徑（預設：`./slogan.yaml`） |
| `--verbose` | `-v` | 顯示詳細輸出 |
| `--quiet` | `-q` | 僅顯示錯誤 |
| `--format <type>` | `-f` | 輸出格式：`text`（預設）、`json` |
| `--help` | `-h` | 顯示說明 |
| `--version` | | 顯示版本號 |

---

## 命令群組

| 群組 | 說明 | 對應規格 |
|------|------|---------|
| `slogan server` | 啟動與管理引擎 | [01-server](01-server.md) |
| `slogan definition` | 管理 definitions（workflow / task / secret） | [02-definition](02-definition.md) |
| `slogan instance` | 管理 workflow instances | [03-instance](03-instance.md) |
| `slogan event` | 發送事件 | [04-event](04-event.md) |
| `slogan secret` | Secret 加解密工具 | [05-secret](05-secret.md) |
| `slogan validate` | 快速驗證 YAML 檔案 | [06-validate](06-validate.md) |

---

## 設定檔

CLI 在啟動時載入設定檔（預設 `./slogan.yaml`），格式見 [07-configuration](07-configuration.md)。

---

## Exit Code

| Code | 說明 |
|------|------|
| `0` | 成功 |
| `1` | 一般錯誤 |
| `2` | 使用方式錯誤（無效的參數或選項） |
| `3` | 設定檔錯誤 |
| `4` | 連線錯誤（storage 或遠端 engine） |
| `5` | 驗證失敗 |

---

## 輸出格式

- `text`（預設）：人類可讀的表格與訊息
- `json`：機器可讀的 JSON，適合腳本串接

所有成功操作在 `json` 模式下回傳：

```json
{
  "ok": true,
  "data": { ... }
}
```

錯誤回傳：

```json
{
  "ok": false,
  "error": {
    "code": "...",
    "message": "..."
  }
}
```
