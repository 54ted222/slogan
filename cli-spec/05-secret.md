# 05 — Secret 命令

管理 secret 檔案的加解密。本文件整合 [dsl-spec/v2/18-secrets-and-env](../dsl-spec/v2/18-secrets-and-env.md) 中定義的 CLI 行為。

---

## 密碼來源

所有需要密碼的操作，依以下優先順序取得密碼：

1. 環境變數 `SLOGAN_SECRET_KEY`
2. 互動提示（stdin 為 TTY 時）
3. 若以上皆不可用 → 錯誤退出

---

## slogan secret encode

將明文 secret 檔案加密。

```bash
slogan secret encode <file> [options]
```

### 選項

| 選項 | 縮寫 | 說明 |
|------|------|------|
| `--output <path>` | `-o` | 輸出至指定檔案（預設覆寫原檔） |
| `--algorithm <name>` | | 加密演算法（預設 `aes-256-gcm`） |

### 行為

1. 讀取明文 YAML（`is_encrypted: false`）
2. 取得密碼
3. 產生隨機 salt 與 IV
4. 使用 PBKDF2 從密碼衍生 256-bit 加密金鑰
5. 以 AES-256-GCM 加密每個 `data` 中的值
6. 將加密結果以 Base64 編碼寫回 `data`
7. 設定 `is_encrypted: true` 並填入 `encryption` 區塊
8. 覆寫原檔或寫入 `-o` 指定的檔案

```bash
$ slogan secret encode secrets/payment.yaml
Password: ****
Confirm: ****
✓ Encrypted: secrets/payment.yaml
```

---

## slogan secret decode

將加密 secret 檔案解密。

```bash
slogan secret decode <file> [options]
```

### 選項

| 選項 | 縮寫 | 說明 |
|------|------|------|
| `--output <path>` | `-o` | 輸出至指定檔案（預設覆寫原檔） |
| `--stdout` | | 僅顯示 key=value 至 stdout，不修改檔案 |

```bash
$ slogan secret decode secrets/payment.yaml
Password: ****
✓ Decrypted: secrets/payment.yaml

$ slogan secret decode secrets/payment.yaml --stdout
Password: ****
PAYMENT_API_KEY=sk_live_abc123
PAYMENT_WEBHOOK_SECRET=whsec_xyz789
```

---

## slogan secret view

查看 secret 的 key 列表（不解密值）。

```bash
slogan secret view <file>
```

不需要密碼。

```bash
$ slogan secret view secrets/payment.yaml
Name:       payment_secrets
Encrypted:  true
Algorithm:  aes-256-gcm
Keys:
  - PAYMENT_API_KEY
  - PAYMENT_WEBHOOK_SECRET
```

---

## slogan secret set

新增或更新單一 secret 值。

```bash
slogan secret set <file> <key> [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--stdin` | 從 stdin 讀取值（適合管線） |

### 行為

1. 讀取 secret 檔案
2. 解密（需密碼）
3. 設定 / 更新指定 key 的值
4. 重新加密
5. 寫回檔案

```bash
$ slogan secret set secrets/payment.yaml NEW_KEY
Password: ****
Value: ****
✓ Set NEW_KEY in secrets/payment.yaml

$ echo "new_value" | slogan secret set secrets/payment.yaml NEW_KEY --stdin
Password: ****
✓ Set NEW_KEY in secrets/payment.yaml
```

---

## slogan secret delete

刪除單一 secret key。

```bash
slogan secret delete <file> <key>
```

```bash
$ slogan secret delete secrets/payment.yaml OLD_KEY
Password: ****
✓ Deleted OLD_KEY from secrets/payment.yaml
```
