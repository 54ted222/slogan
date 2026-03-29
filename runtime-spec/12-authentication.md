# 12 — Authentication & Authorization

本文件定義引擎 API 的身份驗證與授權機制。

---

## 概述

引擎的所有外部介面（Engine API、HTTP trigger endpoint、health check）SHOULD 支援身份驗證與授權。

| 層級 | 說明 |
|------|------|
| **Authentication（驗證）** | 確認呼叫者身份 |
| **Authorization（授權）** | 確認呼叫者是否有權執行該操作 |

---

## Authentication

引擎 MUST 支援以下至少一種驗證方式：

### API Key

最基本的驗證方式，適用於 service-to-service 通訊。

| 項目 | 說明 |
|------|------|
| Header | `Authorization: Bearer <api_key>` 或 `X-API-Key: <api_key>` |
| Key 儲存 | 引擎設定檔或 Secret definition |
| Key 格式 | 不限，建議使用隨機字串（≥ 32 字元） |

### 無驗證模式

引擎 MAY 支援無驗證模式（用於開發與測試）：

```yaml
# 引擎設定
auth:
  enabled: false    # 停用驗證（僅限開發環境）
```

預設 SHOULD 啟用驗證。

---

## Authorization 模型

引擎 SHOULD 支援基於角色的存取控制（RBAC）。

### 角色

| 角色 | 說明 |
|------|------|
| `admin` | 完全存取：管理 definitions、instances、events、設定 |
| `operator` | 操作存取：管理 instances、events、查看 definitions |
| `viewer` | 唯讀存取：查看 definitions、instances、logs |
| `trigger` | 僅限觸發：建立 instances、發送 events |

### 權限矩陣

| 操作 | admin | operator | viewer | trigger |
|------|-------|----------|--------|---------|
| Definition CRUD | ✅ | ❌ | ❌ | ❌ |
| Definition 查看 | ✅ | ✅ | ✅ | ❌ |
| Definition publish/deprecate | ✅ | ❌ | ❌ | ❌ |
| Instance 建立 | ✅ | ✅ | ❌ | ✅ |
| Instance 取消 | ✅ | ✅ | ❌ | ❌ |
| Instance 查看 | ✅ | ✅ | ✅ | ❌ |
| Event 發送 | ✅ | ✅ | ❌ | ✅ |
| Secret 管理 | ✅ | ❌ | ❌ | ❌ |
| Engine 設定 | ✅ | ❌ | ❌ | ❌ |
| Health check | ✅ | ✅ | ✅ | ✅ |
| Execution log 查看 | ✅ | ✅ | ✅ | ❌ |

### API Key 與角色的綁定

```yaml
# 引擎設定
auth:
  enabled: true
  keys:
    - key: "sk_admin_abc123"
      role: admin
      description: "Admin service"
    - key: "sk_trigger_def456"
      role: trigger
      description: "External event source"
      allowed_workflows: ["order_*"]    # 可選：限制可觸發的 workflows
```

### Workflow 級權限

API key MAY 限制為僅能存取特定的 workflows：

| 設定 | 說明 |
|------|------|
| `allowed_workflows` | String array，支援 `*` 萬用字元 |
| 未設定 | 可存取所有 workflows |

---

## HTTP Trigger 的驗證

HTTP trigger endpoint 的驗證獨立於 Engine API：

| 設定 | 說明 |
|------|------|
| 使用 Engine API 的驗證 | HTTP trigger endpoint 使用相同的 API key 驗證 |
| 獨立驗證 | 每個 HTTP trigger MAY 定義自己的驗證方式 |
| 無驗證 | HTTP trigger MAY 設為 public（無驗證） |

```yaml
triggers:
  - type: http
    path: /api/orders
    auth:
      mode: api_key           # api_key | none
```

---

## 驗證失敗的 HTTP Response

| 情境 | HTTP Status | Response |
|------|-------------|----------|
| 無 API key | 401 Unauthorized | `{"error": "authentication required"}` |
| 無效 API key | 401 Unauthorized | `{"error": "invalid api key"}` |
| 權限不足 | 403 Forbidden | `{"error": "insufficient permissions", "required_role": "admin"}` |
| Workflow 不在 allowed list | 403 Forbidden | `{"error": "workflow not allowed"}` |

---

## 安全建議

| 建議 | 說明 |
|------|------|
| HTTPS | 生產環境 MUST 使用 HTTPS（引擎本身 MAY 不處理 TLS，由 reverse proxy 負責） |
| Key rotation | 引擎 SHOULD 支援不停機的 API key 輪換（同時接受新舊 key） |
| Rate limiting | 引擎 SHOULD 對驗證失敗的請求做 rate limiting（防暴力破解） |
| Audit | 所有驗證失敗的請求 SHOULD 記錄至 log |
