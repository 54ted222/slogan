# 20 — Project Definition（`kind: Project`）

本文件定義 `kind: Project` 的完整結構、資料夾組織、引用方式與 CLI 操作。Project 為所有 definition 檔案（Workflow、Task、Agent、Toolset、Resources、Skill、Secret）提供 project 層級的分組與管理，解決 definition 數量增多時的組織問題。

---

## 1. 設計原則

1. **獨立 kind**（決策 F1）— Project 是一個獨立的 kind，具有完整 metadata，不僅僅是資料夾慣例。
2. **`project.yaml` 必須**（決策 F3）— 資料夾中必須包含 `project.yaml` 才被視為 project；無此檔案的資料夾不具備 project 語意。
3. **支援巢狀**（決策 F2）— Project 可包含子 project，形成階層結構（如 `order/domestic`）。
4. **自動 prefix**（決策 F4）— Definition name 自動加上 project 路徑作為前綴（如 `risk-analysis` → `order/risk-analysis`）。
5. **統一管理** — Project 管理其下所有類型的 definition 檔案，而非僅限於 skill。

---

## 2. Top-level Schema

`project.yaml` 的結構：

```yaml
apiVersion: project/v3
kind: Project

metadata:
  name: string               # MUST — project 名稱（kebab-case）
  description: string         # MAY — 人類可讀說明
  labels: map<string, string> # MAY — 分類鍵值對

owner: string                 # MAY — 負責團隊或個人

defaults:                     # MAY — project 層級的預設配置（definition 可覆寫）
  labels: map<string, string> # MAY — 預設 labels，自動套用至 project 下所有 definition
```

---

## 3. 資料夾結構

### 3.1 基本結構

```
projects/
  order/                           # project: order
    project.yaml                   # Project 定義（必須）
    workflows/
      order_fulfillment.yaml       # workflow: order/order_fulfillment
      order_cancellation.yaml      # workflow: order/order_cancellation
    tasks/
      order.load.yaml              # task: order/order.load
      order.validate.yaml          # task: order/order.validate
    agents/
      order.analyzer.yaml          # agent: order/order.analyzer
    skills/
      risk-analysis/               # skill: order/risk-analysis
        SKILL.md
        scripts/
        references/
      fraud-detection/             # skill: order/fraud-detection
        SKILL.md
    toolsets/
      order-processing.yaml        # toolset: order/order-processing
    resources/
      shared-resources.yaml        # resources: order/shared-resources

  customer/                        # project: customer
    project.yaml                   # Project 定義（必須）
    workflows/
      customer_onboarding.yaml     # workflow: customer/customer_onboarding
    skills/
      sentiment-analysis/          # skill: customer/sentiment-analysis
        SKILL.md
      churn-prediction/            # skill: customer/churn-prediction
        SKILL.md
```

### 3.2 巢狀 project

Project 可包含子 project，每一層都需要 `project.yaml`：

```
projects/
  order/                           # project: order
    project.yaml
    workflows/
      order_fulfillment.yaml       # workflow: order/order_fulfillment
    domestic/                      # sub-project: order/domestic
      project.yaml
      workflows/
        domestic_fulfillment.yaml  # workflow: order/domestic/domestic_fulfillment
      skills/
        compliance-check/          # skill: order/domestic/compliance-check
          SKILL.md
    international/                 # sub-project: order/international
      project.yaml
      workflows/
        export_fulfillment.yaml    # workflow: order/international/export_fulfillment
      skills/
        sanctions-screening/       # skill: order/international/sanctions-screening
          SKILL.md
```

巢狀規則：

| 規則 | 說明 |
|------|------|
| `project.yaml` 必須 | 每一層 project 資料夾 MUST 包含 `project.yaml` |
| name 前綴累積 | 子 project 的 definition name 包含完整路徑（`order/domestic/compliance-check`） |
| defaults 繼承 | 子 project 繼承父 project 的 defaults，可覆寫 |

---

## 4. Definition Name 自動前綴

根據決策 F4，definition name 自動加上所屬 project 路徑作為前綴：

| 資料夾位置 | Definition 名稱 | 完整 name |
|-----------|------------------|-----------|
| `projects/order/` | `order_fulfillment` | `order/order_fulfillment` |
| `projects/order/` | `risk-analysis` | `order/risk-analysis` |
| `projects/order/domestic/` | `compliance-check` | `order/domestic/compliance-check` |

在任何引用處，MUST 使用完整的 name：

```yaml
# 引用 project 下的 definition
steps:
  - name: load_order
    type: task
    task: order/order.load

  - name: analyze
    type: agent
    agent: order/order.analyzer
```

---

## 5. 引用語法

### 5.1 完整路徑引用

```yaml
# 在 toolset 中引用 project 下的 skill
skills:
  - name: order/risk-analysis
  - name: order/fraud-detection
  - name: customer/sentiment-analysis
```

### 5.2 通配符引用

```yaml
skills:
  - name: "order/*"                # 載入 order project 下所有 skills（不含子 project）
  - name: "order/**"               # 載入 order project 下所有 skills（含子 project）
  - name: "*/risk-*"               # 載入所有 project 中 risk- 開頭的 skills
```

| Pattern | 說明 |
|---------|------|
| `order/*` | order project 下的直接項目（不遞迴） |
| `order/**` | order project 下所有項目（遞迴含子 project） |
| `*/risk-*` | 所有 project 中名稱以 `risk-` 開頭的項目 |
| `*/*` | 所有 project 下的直接項目 |

### 5.3 向後相容

不在任何 project 下的 definition 維持原有的扁平引用方式：

```yaml
skills:
  - name: risk-analysis              # 扁平結構（無 project），維持相容
  - name: order/risk-analysis        # project 結構
```

引擎解析順序：

1. 先在 project 結構中查找完整路徑
2. 若未找到，在扁平結構中查找
3. 若仍未找到，回傳錯誤

---

## 6. project.yaml 範例

### 6.1 基礎 project

```yaml
apiVersion: project/v3
kind: Project

metadata:
  name: order
  description: "訂單相關的 workflows、agents 與 skills"
  labels:
    domain: order
    team: order-team

owner: order-team

defaults:
  labels:
    domain: order
    team: order-team
```

### 6.2 巢狀子 project

```yaml
apiVersion: project/v3
kind: Project

metadata:
  name: domestic
  description: "國內訂單專用"
  labels:
    region: domestic

owner: domestic-order-team

defaults:
  labels:
    region: tw
```

---

## 7. 生命週期

與其他 definition 相同：

```
DRAFT → PUBLISHED → ARCHIVED
```

| 狀態 | 說明 |
|------|------|
| DRAFT | 開發中，project 下的 definitions 可使用但標記為 draft |
| PUBLISHED | 正式發布，definitions 可被引用 |
| ARCHIVED | project 下所有 definitions 不可被引用 |

---

## 8. CLI 指令

### 8.1 list-projects

列出所有已註冊的 Project：

```bash
slogan project list

# 輸出範例
# NAME         OWNER          DEFINITIONS  STATUS
# order        order-team     12           PUBLISHED
# customer     support-team   5            PUBLISHED
# shared       platform-team  8            PUBLISHED
```

### 8.2 list --project

列出指定 project 下的所有 definitions：

```bash
slogan project list order

# 輸出範例
# NAME                              KIND       STATUS
# order/order_fulfillment           Workflow   PUBLISHED
# order/order.load                  Task       PUBLISHED
# order/order.analyzer              Agent      PUBLISHED
# order/risk-analysis               Skill      PUBLISHED
# order/order-processing            Toolset    PUBLISHED
# order/domestic/compliance-check   Skill      PUBLISHED
```

### 8.3 search

搜尋 definition（跨所有 project）：

```bash
slogan project search "risk"

# 輸出範例
# NAME                                PROJECT             KIND    STATUS
# order/risk-analysis                 order               Skill   PUBLISHED
# order/domestic/risk-analysis        order/domestic      Skill   PUBLISHED
```

---

## 9. 驗證規則

| 規則 | 說明 |
|------|------|
| `project.yaml` 必須 | 資料夾中無 `project.yaml` 則不視為 project |
| name 唯一性 | 同一層級下 project name MUST 唯一 |
| definition name 唯一性 | 完整 name（含 project prefix）MUST 全域唯一 |
| 巢狀 project name | 子 project 的 `metadata.name` MUST 與資料夾名稱一致 |
| owner | 建議填寫，但非必須 |
