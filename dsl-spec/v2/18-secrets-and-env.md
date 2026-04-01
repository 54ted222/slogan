# 18 — Secrets & Environment Variables

本文件定義 workflow 與 task 中存取外部設定值的兩種機制：環境變數（`env`）與加密機密（`secret`）。

---

## 概述

| Namespace | 來源 | 加密 | 適用場景 |
|-----------|------|------|----------|
| `env` | OS 環境變數 / .env 檔 | 否 | 非敏感設定（LOG_LEVEL、API_URL 等） |
| `secret` | `kind: Secret` YAML 檔 | 是 | 敏感資料（API key、密碼、token） |

兩者在 CEL 表達式中透過 `${ }` 定界符存取：

```yaml
input:
  api_url: ${ env.API_BASE_URL }
  api_key: ${ secret.PAYMENT_API_KEY }
```

---

## env — 環境變數

### 來源

引擎啟動時從以下來源載入（優先順序由高至低）：

1. OS 環境變數
2. `.env` 檔案（若存在）

### 在 CEL 中存取

```yaml
# Workflow step 中
- type: task
  action: order.load
  input:
    db_host: ${ env.DATABASE_HOST }
    log_level: ${ env.LOG_LEVEL }

# Task definition 的 http backend 中
backend:
  type: http
  url: ${ env.API_BASE_URL }
```

### 行為

- `env.XXX` 不存在 → 回傳 `null`
- 使用 `default()` 做防禦性存取：`${ default(env.LOG_LEVEL, "info") }`
- `env` 的值一律為 `string` 型別

---

## secret — 加密機密

### Secret Definition

Secret 以獨立的 YAML 檔案定義，具有自己的 `kind`：

```yaml
apiVersion: secret/v2
kind: Secret

metadata:
  name: payment_secrets
  description: "付款相關 API 金鑰"

is_encrypted: false

data:
  PAYMENT_API_KEY: "sk_live_abc123"
  PAYMENT_WEBHOOK_SECRET: "whsec_xyz789"
```

#### 加密後

```yaml
apiVersion: secret/v2
kind: Secret

metadata:
  name: payment_secrets
  description: "付款相關 API 金鑰"

is_encrypted: true
encryption:
  type: aes
  config:
    algorithm: aes-256-gcm
    salt: "base64_encoded_salt"

data:
  PAYMENT_API_KEY: "base64(iv + ciphertext + tag)"
  PAYMENT_WEBHOOK_SECRET: "base64(iv + ciphertext + tag)"
```

### 欄位說明

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `apiVersion` | string | MUST | 固定值 `secret/v2` |
| `kind` | string | MUST | 固定值 `Secret` |
| `metadata` | object | MUST | 識別與描述 |
| `is_encrypted` | boolean | MUST | 是否已加密 |
| `encryption` | object | 加密時 MUST | 加密設定 |
| `data` | map\<string, string\> | MUST | 機密鍵值對 |

### encryption 物件

| 欄位 | 型別 | 說明 |
|------|------|------|
| `type` | string | 加密演算法類型（目前僅支援 `aes`） |
| `config` | object | 演算法專屬設定 |

#### AES 加密設定

| 欄位 | 型別 | 說明 |
|------|------|------|
| `algorithm` | string | `aes-256-gcm`（預設）或 `aes-256-cbc` |
| `salt` | string | Base64 編碼的鹽值（用於 PBKDF2 衍生金鑰） |

每個 `data` value 為獨立加密，格式為 `base64(iv + ciphertext + tag)`：
- `iv`：12 bytes（AES-GCM）或 16 bytes（AES-CBC），每個 value 獨立隨機產生
- `ciphertext`：加密後的密文
- `tag`：16 bytes authentication tag（僅 GCM 模式）

**重要**：每個 value MUST 使用獨立的 IV。AES-GCM 下重用 IV 會破壞認證性與機密性。

### 在 CEL 中存取

```yaml
# Workflow 中
- type: task
  action: payment.create
  input:
    api_key: ${ secret.PAYMENT_API_KEY }

# Task definition 的 http backend 中
backend:
  type: http
  url: "https://api.payment.example.com/v1/charges"
  headers:
    Authorization: "Bearer ${ secret.PAYMENT_API_KEY }"
```

### 行為

- 引擎在啟動時載入並解密 secret 檔案
- 解密後的值僅存在於記憶體中，不持久化至資料庫
- `secret.XXX` 不存在 → 回傳 `null`
- Secret 值一律為 `string` 型別
- 引擎 MUST 確保 secret 值不被寫入 execution_log、step output、或任何持久化儲存
- Secret 值在 log 輸出中 SHOULD 被遮蔽（如 `***`）

### Secret 與 Workflow 的綁定

Workflow definition MAY 在 `config` 中宣告依賴的 secret：

```yaml
config:
  secrets:
    - payment_secrets       # metadata.name of the secret file
    - notification_secrets
```

宣告後，引擎在 instance 啟動時 SHOULD 檢查所需的 secret 是否已載入。未宣告時，引擎仍可存取所有已載入的 secret（但不做啟動時檢查）。

---

## CLI 工具

### slogan secret encode

將明文 secret 檔案加密：

```bash
$ slogan secret encode <file> [options]

# 互動模式（提示輸入密碼）
$ slogan secret encode secrets/payment.yaml
Password: ****
Confirm: ****
✓ Encrypted: secrets/payment.yaml

# 指定演算法
$ slogan secret encode secrets/payment.yaml --algorithm aes-256-gcm

# 從環境變數讀取密碼
$ SLOGAN_SECRET_KEY=mykey slogan secret encode secrets/payment.yaml

# 輸出至另一個檔案
$ slogan secret encode secrets/payment.yaml -o secrets/payment.enc.yaml
```

#### 行為

1. 讀取明文 YAML（`is_encrypted: false`）
2. 提示輸入密碼（或從 `SLOGAN_SECRET_KEY` 環境變數讀取）
3. 產生隨機 salt
4. 使用 PBKDF2 從密碼衍生 256-bit 加密金鑰
5. 對每個 `data` 中的值，產生獨立隨機 IV，以 AES-256-GCM 加密
6. 將每個加密結果以 `base64(iv + ciphertext + tag)` 格式寫回 `data`
7. 設定 `is_encrypted: true` 並填入 `encryption` 區塊（含 `salt`）
8. 覆寫原檔（或寫入 `-o` 指定的檔案）

### slogan secret decode

將加密 secret 檔案解密：

```bash
$ slogan secret decode <file> [options]

# 互動模式
$ slogan secret decode secrets/payment.yaml
Password: ****
✓ Decrypted: secrets/payment.yaml

# 僅顯示不覆寫（stdout）
$ slogan secret decode secrets/payment.yaml --stdout
Password: ****
PAYMENT_API_KEY=sk_live_abc123
PAYMENT_WEBHOOK_SECRET=whsec_xyz789

# 輸出至另一個檔案
$ slogan secret decode secrets/payment.yaml -o secrets/payment.plain.yaml
```

### slogan secret view

查看 secret 的 key 列表（不解密值）：

```bash
$ slogan secret view secrets/payment.yaml
Name: payment_secrets
Encrypted: true
Algorithm: aes-256-gcm
Keys:
  - PAYMENT_API_KEY
  - PAYMENT_WEBHOOK_SECRET
```

### slogan secret set

新增或更新單一 secret 值：

```bash
# 互動模式
$ slogan secret set secrets/payment.yaml NEW_KEY
Password: ****
Value: ****
✓ Set NEW_KEY in secrets/payment.yaml

# 從 stdin
$ echo "new_value" | slogan secret set secrets/payment.yaml NEW_KEY --stdin
```

---

## 安全性要求

1. **密碼不存檔**：密碼僅在加解密時使用，不寫入任何檔案
2. **記憶體保護**：解密後的 secret 在使用完畢後 SHOULD 從記憶體中清除
3. **Log 遮蔽**：secret 值在任何 log 輸出中 MUST 被遮蔽
4. **持久化排除**：secret 值 MUST NOT 被寫入 execution_log、step_instance output、或其他資料庫表
5. **版本控制**：加密後的 secret 檔案可安全提交至版本控制（Git）
6. **Key rotation**：更換密碼時，先 decode 再以新密碼 encode

---

## 建議的檔案結構

```
project/
  workflows/
    order_fulfillment.yaml
    payment_processing.yaml
  tasks/
    order.load.yaml
    payment.create.yaml
  secrets/
    payment.yaml              # is_encrypted: true
    notification.yaml         # is_encrypted: true
  .env                        # 非敏感設定
```

`.gitignore` SHOULD 包含：

```
.env
secrets/*.plain.yaml
```

加密的 `secrets/*.yaml` 可安全提交。
