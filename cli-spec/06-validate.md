# 06 — Validate 命令

快速驗證 YAML 檔案，不需要連線至引擎。

---

## slogan validate

離線驗證 definition YAML 的語法與結構。

```bash
slogan validate <file|directory> [options]
```

### 選項

| 選項 | 縮寫 | 說明 |
|------|------|------|
| `--recursive` | `-r` | 遞迴掃描子目錄 |

### 驗證範圍

此命令執行**靜態驗證**，不需要引擎運行：

| 檢查項目 | 說明 |
|---------|------|
| YAML 語法 | 解析是否成功 |
| 頂層結構 | `apiVersion`、`kind`、`metadata` 必填欄位 |
| Step 結構 | `type` 合法性、各 type 必填欄位 |
| Step ID 唯一性 | 全域唯一 |
| CEL 語法 | `${ }` 內的表達式語法（不求值） |
| Schema 格式 | `input.schema` / `output.schema` 為合法 JSON Schema 子集 |

### 不檢查項目

以下需要引擎狀態的檢查**不**在此命令中執行：

- Task definition 是否存在（`action` 引用）
- Sub-workflow definition 是否存在
- Secret definition 是否存在
- 跨檔案引用完整性

這些檢查由 `slogan definition validate` 在引擎端執行。

### 輸出

```bash
$ slogan validate workflows/order_fulfillment.yaml
✓ Workflow/order_fulfillment@1: valid

$ slogan validate workflows/ tasks/ --recursive
✓ Workflow/order_fulfillment@1: valid
✓ Task/order.load@1: valid
✗ Task/payment.create@1: 2 errors
  - [E001] steps[0].input.amount: CEL syntax error
  - [W001] steps[1]: unknown field "retries" (did you mean "retry"?)

Results: 2 passed, 1 failed
```

### Exit Code

- `0`：所有檔案驗證通過
- `5`：至少一個檔案驗證失敗
