# 02 — Definition 命令

管理 workflow、task、secret definitions 的完整生命週期。

---

## slogan definition apply

建立或更新 definition（從 YAML 檔案）。

```bash
slogan definition apply <file|directory> [options]
```

### 引數

| 引數 | 說明 |
|------|------|
| `<file>` | 單一 YAML 檔案 |
| `<directory>` | 目錄（遞迴掃描所有 `.yaml` / `.yml` 檔案） |

### 選項

| 選項 | 說明 |
|------|------|
| `--recursive` / `-r` | 遞迴掃描子目錄（僅目錄模式） |
| `--dry-run` | 僅驗證，不實際建立 |

### 行為

1. 解析 YAML 檔案，識別 `kind`（Workflow / Task / Secret）
2. 呼叫 Engine API `CreateDefinition`
3. 若同名同版本已存在 → 顯示跳過訊息
4. 輸出結果

```bash
$ slogan definition apply workflows/order_fulfillment.yaml
✓ Created: Workflow/order_fulfillment@1 (DRAFT)

$ slogan definition apply tasks/
✓ Created: Task/order.load@1 (DRAFT)
✓ Created: Task/payment.create@1 (DRAFT)
✗ Skipped: Task/order.load@1 (already exists)
```

---

## slogan definition validate

驗證 definition（DRAFT → VALIDATED）。

```bash
slogan definition validate <identifier> [options]
```

### 引數

| 格式 | 說明 |
|------|------|
| `<name>@<version>` | 以名稱 + 版本指定 |
| `<definition_id>` | 以 ID 指定 |

### 行為

1. 呼叫 Engine API `ValidateDefinition`
2. 成功 → 顯示 VALIDATED
3. 失敗 → 顯示驗證錯誤列表，exit code = 5

```bash
$ slogan definition validate order_fulfillment@1
✓ Validated: Workflow/order_fulfillment@1

$ slogan definition validate broken_workflow@1
✗ Validation failed: Workflow/broken_workflow@1
  - [E001] steps[2].action: task definition "order.unknown" not found
  - [E002] steps[3].expr: CEL syntax error at position 15
```

---

## slogan definition publish

發布 definition（VALIDATED → PUBLISHED）。

```bash
slogan definition publish <identifier>
```

### 行為

1. 呼叫 Engine API `PublishDefinition`
2. Workflow definition → 自動建立 trigger subscriptions

```bash
$ slogan definition publish order_fulfillment@1
✓ Published: Workflow/order_fulfillment@1
  Triggers: 1 event subscription created
```

---

## slogan definition deprecate

棄用 definition（PUBLISHED → DEPRECATED）。

```bash
slogan definition deprecate <identifier>
```

```bash
$ slogan definition deprecate order_fulfillment@1
✓ Deprecated: Workflow/order_fulfillment@1
  Note: existing running instances will not be affected
```

---

## slogan definition archive

封存 definition（DEPRECATED → ARCHIVED）。

```bash
slogan definition archive <identifier>
```

### 前置條件

所有 instances MUST 為 terminal 狀態，否則失敗。

```bash
$ slogan definition archive order_fulfillment@1
✓ Archived: Workflow/order_fulfillment@1

$ slogan definition archive payment_processing@2
✗ Cannot archive: 2 instances still running
```

---

## slogan definition revert

退回 definition（VALIDATED → DRAFT）。

```bash
slogan definition revert <identifier>
```

---

## slogan definition get

查詢單一 definition。

```bash
slogan definition get <identifier> [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--output-yaml` | 輸出 YAML 定義內容 |

```bash
$ slogan definition get order_fulfillment@1
ID:        def_abc123
Kind:      Workflow
Name:      order_fulfillment
Version:   1
State:     PUBLISHED
Created:   2024-01-15 10:30:00

$ slogan definition get order_fulfillment@1 --output-yaml
apiVersion: workflow/v2
kind: Workflow
metadata:
  name: order_fulfillment
  version: 1
...
```

---

## slogan definition list

列出 definitions。

```bash
slogan definition list [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--kind <type>` | 篩選：`workflow`、`task`、`secret` |
| `--state <state>` | 篩選 lifecycle 狀態 |
| `--name <prefix>` | 名稱前綴篩選 |
| `--limit <number>` | 每頁筆數（預設 20） |

```bash
$ slogan definition list --kind workflow --state published
KIND      NAME                   VERSION  STATE      CREATED
Workflow  order_fulfillment      1        PUBLISHED  2024-01-15
Workflow  payment_processing     3        PUBLISHED  2024-01-20
```

---

## 快捷流程

### slogan definition deploy

一次完成 apply → validate → publish。

```bash
slogan definition deploy <file|directory> [options]
```

### 選項

| 選項 | 說明 |
|------|------|
| `--recursive` / `-r` | 遞迴掃描子目錄 |

### 行為

1. 掃描檔案，建立 definitions（apply）
2. 逐一驗證（validate）
3. 驗證通過者逐一發布（publish）
4. 任一驗證失敗 → 停止，已建立的 definitions 保留為 DRAFT

```bash
$ slogan definition deploy workflows/ tasks/
✓ Created: Workflow/order_fulfillment@1 (DRAFT)
✓ Created: Task/order.load@1 (DRAFT)
✓ Created: Task/payment.create@1 (DRAFT)
✓ Validated: Task/order.load@1
✓ Validated: Task/payment.create@1
✓ Validated: Workflow/order_fulfillment@1
✓ Published: Task/order.load@1
✓ Published: Task/payment.create@1
✓ Published: Workflow/order_fulfillment@1
  3 definitions deployed
```
