# 08 — Project Definition + Secret Definition

本文件定義 `kind: Project` 與 `kind: Secret` 的 YAML 結構。

---

## Part A — Project Definition（`kind: Project`）

Project 為所有 definition 檔案提供 project 層級的分組與管理，類似 `package.json` 的角色。

### 頂層結構

```yaml
apiVersion: project/v6
kind: Project

metadata:
  name: order                   # MUST — kebab-case
  description: "訂單相關定義"

owner: order-team               # MAY — 負責團隊

defaults:                       # MAY — project 層級預設
  labels:                       # 自動套用至 project 下所有 definition
    domain: order
    team: order-team
```

### defaults.labels 合併規則

- Project defaults.labels 先展開，definition 自身 `metadata.labels` **覆寫同名 key**（leaf-wins）
- 巢狀 project：由根 project 向葉 project 逐層合併；葉 project defaults 覆寫父 project 同名 key；最終 definition labels 覆寫合併後的 defaults
- 合併為 shallow merge（label value 為 string，無 deep merge 情境）

### 資料夾結構

資料夾中 MUST 包含 `project.yaml` 才被視為 project：

```
projects/
  order/
    project.yaml                    # Project 定義
    workflows/
      order_fulfillment.yaml        # workflow: order/order_fulfillment
    tools/
      order.load.yaml               # tool: order/order.load
    secrets/
      payment.yaml

  customer/
    project.yaml
    workflows/
      customer_onboarding.yaml
```

### Name 自動前綴

Definition name 自動加上 project 路徑前綴：

| 位置 | Definition name | 完整 name |
|------|-----------------|-----------|
| `projects/order/` | `order.load` | `order/order.load` |
| `projects/order/domestic/` | `compliance.check` | `order/domestic/compliance.check` |

引用時 MUST 使用完整 name：

```yaml
- type: task
  action: order/order.load
```

### 巢狀 Project

支援子 project，每一層都需要 `project.yaml`。子 project 繼承父 project 的 defaults，可覆寫。

### 載入順序與 defaults 合併

引擎啟動時對 projects 資料夾深度優先掃描：

1. 依檔案樹由根到葉收集所有 `project.yaml`
2. 按深度升序逐層建立 defaults chain：`[root, level1, level2, ...]`
3. Action 載入時，其所屬 project 的 defaults 已由外向內疊加完成（外層先展開，內層覆寫同 key）
4. 最後將 action 自身的 `metadata.labels` 疊加在合併後 defaults 上（leaf-wins）

**跨 project 引用**：

- Project A 的 action 使用 `action: b/some.action` 引用 Project B 的 action，載入驗證時會觸發 Project B 先完成載入（自動相依）
- 循環依賴（A → B → A）在 `registry.dependency_cycle` 檢查中偵測
- 若 B 的 action 使用 project defaults 的版本預設，B 自身的 defaults 優先，而非呼叫端 A 的 defaults

---

## Part B — Secret Definition（`kind: Secret`）

Secret 儲存加密的機密鍵值對。

### 頂層結構

```yaml
# 明文（加密前）
apiVersion: secret/v6
kind: Secret

metadata:
  name: payment_secrets         # MUST — snake_case
  description: "付款 API 金鑰"

is_encrypted: false

data:
  PAYMENT_API_KEY: "sk_live_abc123"
  PAYMENT_WEBHOOK_SECRET: "whsec_xyz789"
```

```yaml
# 加密後
apiVersion: secret/v6
kind: Secret

metadata:
  name: payment_secrets

is_encrypted: true
encryption:
  type: aes
  config:
    algorithm: aes-256-gcm
    iv: "base64_encoded_iv"
    salt: "base64_encoded_salt"

data:
  PAYMENT_API_KEY: "base64_encrypted_value"
  PAYMENT_WEBHOOK_SECRET: "base64_encrypted_value"
```

### 在 CEL 中存取

`secret` namespace 可在所有 steps、tool definition 中使用：

```yaml
input:
  api_key: ${ secret.PAYMENT_API_KEY }
```

### 跨 project 隔離規則

Secret 依宣告位置自動歸屬於所在 project；CEL 對 `secret.X` 的解析：

| 場景 | 行為 |
|------|------|
| Workflow / Tool 位於 project 內 | 只能讀取**同 project**的 secret；引用其他 project 的 key → `expression_error.identifier_not_found` |
| Workflow 於根目錄（無 project） | 可讀「根目錄 secrets/」下的 key |
| 跨 project 共用 secret | 需明確以 `${ secret["<project>/<KEY>"] }` 存取，且該 secret MUST 標記 `metadata.shared: true`（未標記則仍視為不可見） |

- 引擎載入期對每個 definition 記錄其歸屬 project，並為 `secret` namespace 建立 project-scoped SecretAccessor
- 日誌／trace redaction 對所有 project 的 secret 一致適用

### env — 環境變數

非敏感設定透過 `env` namespace 存取（來自 OS 環境變數或 `.env` 檔）：

```yaml
input:
  db_host: ${ env.DATABASE_HOST }
  log_level: ${ default(env.LOG_LEVEL, "info") }
```

### 安全性要求

- Secret 值 MUST NOT 被寫入 execution_log 或持久化儲存
- Secret 值在 log 中 SHOULD 被遮蔽
- 加密後的 secret 檔案可安全提交至 Git
