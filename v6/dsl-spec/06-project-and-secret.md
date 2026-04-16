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

owner: order-team               # MAY — 負責團隊（見下方「owner 用途」）

defaults:                       # MAY — project 層級預設
  labels:                       # 自動套用至 project 下所有 definition
    domain: order
    team: order-team
  action_versions:              # MAY — 同 project 內 step 未帶 @version 時的預設版本
    order.load: 3               # canonical name（含 project 前綴展開後的完整名）
    payment.process: 2

retention_policy:               # MAY — 覆寫 instance 終態保存期（見 runtime-spec/02-instance-lifecycle.md）
  succeeded: 30d                # MAY — SUCCEEDED instance 保存期；預設 7 天
  failed: 90d                   # MAY — FAILED instance 保存期；預設 30 天
  cancelled: 7d                 # MAY — CANCELLED instance 保存期；預設 7 天
```

### retention_policy

- 欄位型別均為 duration string（格式同 `dsl-spec/01-overview.md` Duration 格式）
- Instance 建立時依 **建立時的 project retention_policy 設定** 計算 `retention_until` 並持久化；後續修改 `retention_policy` **不影響**既有 instance
- 未宣告任一欄位 → 沿用 engine 內建預設；未宣告 `retention_policy` 整個物件 → 等同全部沿用預設
- 巢狀 project：子 project 繼承父 project 的 retention_policy，可**部分**覆寫（shallow merge at 欄位層）
- 不提供 API 延長單一 instance 的 retention（見 `runtime-spec/02-instance-lifecycle.md`「Retention 覆寫規則」）

### defaults.action_versions

- Map 的 key 為 action 的 canonical name（若為跨 project 引用，使用完整 `<owning_project>/<body>`；同 project 內亦可省略前綴，載入期自動補齊）
- value 為 integer，MUST 對應 registry 中實際存在的 version；載入時不存在 → `registry.action_not_found`
- 僅影響本 project 下的 workflow / function / tool 作為呼叫端時的解析；不影響其他 project 內的同名 action 呼叫
- 優先權見 `03-steps.md` 的「版本解析優先權」

### defaults.labels 合併規則

- Project defaults.labels 先展開，definition 自身 `metadata.labels` **覆寫同名 key**（leaf-wins）
- 巢狀 project：由根 project 向葉 project 逐層合併；葉 project defaults 覆寫父 project 同名 key；最終 definition labels 覆寫合併後的 defaults
- 合併為 shallow merge（label value 為 string，無 deep merge 情境）

### owner 用途

v6 中 `owner` 為**文件性**欄位，引擎不做 ACL / 路由檢查：

- 自動寫入 project 下所有 definition 的 `metadata.labels.owner`（如 definition 自身未指定同名 label）
- `instance.failed` 事件的 `data.owner` 欄位會帶此值，便於告警系統分派（由消費者實作分派邏輯，引擎不直接通知）
- 未來版本若引入 RBAC / 通知訂閱機制，此欄位將成為識別子；v6 先保留語意空間

### 根目錄 definition（無 project）的語意

所有 `projects/` 外的 definition（直接置於根目錄 `workflows/` / `tools/` / `functions/`）視為「無 project」，適用以下規則：

- `project` namespace：`project.name == ""`、`project.path == ""`、`project.labels == {}`、`project.owner == ""`（見 `04-expressions.md#project`）
- **無自動 owner / labels 注入**：definition 的 `metadata.labels` 僅取自身宣告；不經 defaults 合併（根目錄無 `project.yaml` 可供合併）
- Secret 引用：根目錄 secret 恆為 shared（對所有 project 可見；見上節）
- Action 完整名稱不帶 `/` 前綴（例：`workflows/hello.yaml` 的 action name 為 `hello`，非 `/hello`）
- 跨 project 引用：project 內 workflow 引用根目錄 tool 直接寫 `action: tool_name@<version>`（無前綴即視為根目錄）；歧義場景（同 project 內已有同名 tool）優先 project 內
- 建議生產環境一律以 project 組織 definition；根目錄僅用於 prototyping 或單一 tool / 單一 workflow 的小型部署

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

### 空子目錄規則

- `workflows/` / `tools/` / `functions/` / `secrets/` 皆為**慣例**子目錄，非強制存在；缺失或空目錄皆視為「該類別無 definition」，不報錯
- 載入器以**檔案副檔名**（`.yaml` / `.yml`）遞迴掃描 project 根（含子資料夾），依每個檔案的 `kind` 欄位分類；檔案**不需**放在 `<kind>s/` 目錄下（建議但非強制）
- 根目錄內不含 `project.yaml` 但含 `workflows/` 等子目錄 → 視為**根目錄 definition**（見上「根目錄 definition」），不自動建立虛擬 project
- 空 project（僅有 `project.yaml`，無任何 definition）→ 合法；適合作為 namespace 占位或僅宣告 `defaults.labels` / `owner` 的元件
- 非 YAML 檔案（如 `.md` / `.json` / `.sh`）被忽略；engine 不嘗試解析

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
  shared: false                 # MAY, 預設 false — true 時允許跨 project 引用

is_encrypted: false

data:
  PAYMENT_API_KEY: "sk_live_abc123"
  PAYMENT_WEBHOOK_SECRET: "whsec_xyz789"
```

### metadata.shared

| 值 | 行為 |
|----|------|
| `false`（預設） | 僅同 project 內的 workflow / tool / function 可引用（短路徑 `${ secret.KEY }`）；其他 project 嘗試 `${ secret["<this_project>/KEY"] }` → `expression_error.identifier_not_found` |
| `true` | 其他 project 可用完整路徑 `${ secret["<owning_project>/KEY"] }` 存取；同 project 仍可用短路徑 |

- `shared` 欄位**不加密**；僅為存取控制旗標
- 將 `shared: true` 改回 `false` 屬破壞性變更：已引用該 secret 的跨 project workflow 下次載入時 → `registry.secret_access_denied`（新增錯誤碼）
- 根目錄（無 project）的 secret 恆等同 `shared: true`（對所有 project 可見，以短路徑或 `${ secret.KEY }` 皆可；推薦僅用於共用基礎設施憑證）

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
| 跨 project 共用 secret | 需明確以 `${ secret["<project>/<KEY>"] }` 存取，且該 secret MUST 標記 `metadata.shared: true`（未標記則仍視為不可見；未標記時引擎於載入期拒絕並回報 `registry.secret_access_denied`） |

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
