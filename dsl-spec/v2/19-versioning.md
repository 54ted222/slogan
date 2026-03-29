# 19 — Versioning & Migration

本文件定義 definition 版本管理策略、向下相容保證、running instance 與新版 definition 的關係。

---

## 版本模型

每個 definition（Workflow / Task / Secret）以 `(name, version)` 唯一識別。

### 版本號規則

- Version 為正整數，單調遞增
- 同一 `name` 下可存在多個版本，各自擁有獨立的 lifecycle state
- 新版本 MUST 的 version 值大於同 name 下所有已存在的版本
- 引擎 MUST 拒絕建立 version ≤ 已存在最大版本的 definition

### 多版本並存

同一 `name` 下 MAY 有多個版本同時處於 PUBLISHED 狀態：

```
order_fulfillment v3  →  PUBLISHED
order_fulfillment v4  →  PUBLISHED
order_fulfillment v5  →  DRAFT
```

這允許漸進式遷移：新版本發布後，舊版本仍可服務既有的 instance。

---

## `"latest"` 版本解析

`version: "latest"` 在 **執行時**解析為同 name 下 version 最大且 lifecycle_state 為 PUBLISHED 的 definition。

### 解析規則

1. 查詢 `name` 相同、`lifecycle_state` = PUBLISHED 的所有 definitions
2. 取 `version` 最大者
3. 若無 PUBLISHED 版本 → step FAILED（錯誤碼 `task_failed`，message: `no published version found for <name>`）

### 解析時機

- Task step：step 進入 RUNNING 時解析，不在 validation 或 instance 建立時解析
- Sub_workflow step：同上
- 一旦解析完成，該 step 的整個執行週期（含 retry）使用相同的 resolved version

---

## Running Instance 與版本的關係

### Instance 綁定

Workflow instance 在**建立時**綁定到特定的 definition version：

| 時機 | 行為 |
|------|------|
| 建立 instance（manual trigger） | 使用指定版本，或 latest 解析結果 |
| 建立 instance（event trigger） | 使用 trigger subscription 對應的 definition version |

### 不可變原則

Instance 建立後，其綁定的 workflow definition version **不可變更**。即使該 definition 的 lifecycle_state 後續變為 DEPRECATED 或 ARCHIVED，已建立的 instance 仍繼續使用原版本的定義執行至完成。

此原則確保：
- 執行中的 instance 不受 definition 變更影響
- Deterministic replay：相同的 definition + 相同的事件序列 = 相同的結果

### Definition 狀態變更對 Running Instance 的影響

| Definition 狀態變更 | 對 Running Instance 的影響 |
|---------------------|--------------------------|
| PUBLISHED → DEPRECATED | 無影響，instance 繼續執行 |
| DEPRECATED → ARCHIVED | 無影響，instance 繼續執行。但引擎 SHOULD 在 ARCHIVED 前檢查是否有非 terminal instance |
| Definition 內容修改 | 不可能（PUBLISHED 之後不可編輯） |

---

## 新版本發布流程

### 建議流程

```
1. 建立新版本（DRAFT）
     ↓
2. 驗證（DRAFT → VALIDATED）
     ↓
3. 發布（VALIDATED → PUBLISHED）
     ↓
4. 觀察一段時間，確認新版本正常
     ↓
5. 將舊版本標記為棄用（PUBLISHED → DEPRECATED）
     ↓
6. 確認所有舊版本 instance 完成後，封存（DEPRECATED → ARCHIVED）
```

### Trigger Subscription 的更新

當新版本 PUBLISHED 時：

- 新版本的 event triggers 建立新的 trigger subscriptions
- **舊版本的 trigger subscriptions 不自動移除**（除非舊版本被 DEPRECATED）
- 若新舊版本都有相同 event type 的 trigger → 同一事件 MAY 同時觸發新舊版本的 instance
- 若要避免重複觸發，SHOULD 先 DEPRECATE 舊版本（移除其 trigger subscriptions），再 PUBLISH 新版本

引擎 SHOULD 在同一 name 下有多個 PUBLISHED 版本的 event trigger 訂閱相同 event type 時產生警告。

---

## Task Definition 版本更新

### 對 Workflow 的影響

| Workflow step 設定 | Task version 更新影響 |
|-------------------|---------------------|
| `version: 3`（固定版本） | 無影響，始終使用 v3 |
| `version: "latest"` | 下一次 step 執行時解析到新版本 |

### DEPRECATED Task Definition

- `version: "latest"` 不會解析到 DEPRECATED 版本（僅解析 PUBLISHED）
- `version: 3`（固定）若 v3 為 DEPRECATED → 仍可執行，但引擎 SHOULD 在 execution_log 記錄警告
- `version: 3`（固定）若 v3 為 ARCHIVED → step FAILED（錯誤碼 `task_failed`，message: `task version archived`）

---

## Breaking Change 處理

### 定義

Breaking change 指新版 definition 的變更導致既有的呼叫端（workflow step 或外部 API）無法正常運作。常見情境：

| 變更類型 | 是否 Breaking |
|---------|-------------|
| 新增 optional input 欄位 | 否 |
| 新增 required input 欄位 | 是 |
| 移除 output 欄位 | 是 |
| 變更 output 欄位型別 | 是 |
| 新增 output 欄位 | 否 |
| 變更 backend 實作 | 否（若 input/output 不變） |

### 處理策略

本規格**不提供自動相容機制**。Breaking change 的處理由使用者負責：

1. **同時更新**：同時發布新版 task 和參照它的 workflow（兩者都建新版本）
2. **版本固定**：workflow 使用固定 version 號，確保不受 latest 解析影響
3. **漸進遷移**：先發布新 task version（保持舊版 PUBLISHED），更新 workflow 後再 DEPRECATE 舊 task version

### 驗證輔助

引擎 SHOULD 在 workflow validation 階段檢查 task step 的 `input` 與 task definition 的 `input_schema` 的靜態相容性。若 workflow 參照的 task 版本有 breaking change，validation 會捕捉到不匹配。

---

## apiVersion 遷移

`apiVersion`（如 `workflow/v2`、`task/v2`）代表 DSL 語法版本。

### 保證

- 同一 apiVersion 內，引擎 MUST 保持向下相容
- apiVersion 變更（如 v2 → v3）MAY 引入 breaking changes
- 引擎 MUST 支援至少兩個 major apiVersions（如同時支援 v2 和 v3）
- 舊 apiVersion 的支援期限由引擎實作決定，SHOULD 提供至少 6 個月的遷移期

### 多版本引擎

引擎 SHOULD 能同時載入不同 apiVersion 的 definitions：

- v2 的 workflow 呼叫 v2 的 task → 正常
- v3 的 workflow 呼叫 v2 的 task → 正常（引擎負責適配）
- v2 的 workflow 呼叫 v3 的 task → 正常（引擎負責適配）

跨 apiVersion 的 sub_workflow 呼叫同樣由引擎負責適配，因為 sub_workflow 的資料隔離邊界（僅透過 input/output 溝通）天然支援版本解耦。
