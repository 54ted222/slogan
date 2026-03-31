# 20 — SkillProject Definition（`kind: SkillProject`）

本文件定義 `kind: SkillProject` 的完整結構、資料夾組織、引用方式與 CLI 操作。SkillProject 為 skill 提供 project 層級的分組與管理，解決 skill 數量增多時的組織問題。

---

## 1. 設計原則

1. **獨立 kind**（決策 F1）— SkillProject 是一個獨立的 kind，具有完整 metadata，不僅僅是資料夾慣例。
2. **`project.yaml` 必須**（決策 F3）— 資料夾中必須包含 `project.yaml` 才被視為 project；無此檔案的資料夾不具備 project 語意。
3. **支援巢狀**（決策 F2）— Project 可包含子 project，形成階層結構（如 `order/domestic/risk-analysis`）。
4. **自動 prefix**（決策 F4）— Skill name 自動加上 project 路徑作為前綴（如 `risk-analysis` → `order/risk-analysis`）。

---

## 2. Top-level Schema

`project.yaml` 的結構：

```yaml
apiVersion: skillproject/v3
kind: SkillProject

metadata:
  name: string               # MUST — project 名稱（kebab-case）
  description: string         # MAY — 人類可讀說明
  labels: map<string, string> # MAY — 分類鍵值對

owner: string                 # MAY — 負責團隊或個人

defaults:                     # MAY — project 層級的預設配置（skill 可覆寫）
  labels: map<string, string> # MAY — 預設 labels，自動套用至 project 下所有 skill
```

---

## 3. 資料夾結構

### 3.1 基本結構

```
skills/
  order/                           # project: order
    project.yaml                   # SkillProject 定義（必須）
    risk-analysis/                 # skill: order/risk-analysis
      SKILL.md
      scripts/
      references/
    fraud-detection/               # skill: order/fraud-detection
      SKILL.md
      scripts/
    compliance-review/             # skill: order/compliance-review
      SKILL.md
      references/

  customer/                        # project: customer
    project.yaml                   # SkillProject 定義（必須）
    sentiment-analysis/            # skill: customer/sentiment-analysis
      SKILL.md
    churn-prediction/              # skill: customer/churn-prediction
      SKILL.md

  shared/                          # project: shared（跨領域共用）
    project.yaml
    general-knowledge/             # skill: shared/general-knowledge
      SKILL.md
    escalation-procedures/         # skill: shared/escalation-procedures
      SKILL.md
```

### 3.2 巢狀 project

Project 可包含子 project，每一層都需要 `project.yaml`：

```
skills/
  order/                           # project: order
    project.yaml
    risk-analysis/                 # skill: order/risk-analysis
      SKILL.md
    domestic/                      # sub-project: order/domestic
      project.yaml
      risk-analysis/               # skill: order/domestic/risk-analysis
        SKILL.md
      compliance-check/            # skill: order/domestic/compliance-check
        SKILL.md
    international/                 # sub-project: order/international
      project.yaml
      sanctions-screening/         # skill: order/international/sanctions-screening
        SKILL.md
```

巢狀規則：

| 規則 | 說明 |
|------|------|
| `project.yaml` 必須 | 每一層 project 資料夾 MUST 包含 `project.yaml` |
| name 前綴累積 | 子 project 的 skill name 包含完整路徑（`order/domestic/risk-analysis`） |
| defaults 繼承 | 子 project 繼承父 project 的 defaults，可覆寫 |

---

## 4. Skill Name 自動前綴

根據決策 F4，skill name 自動加上所屬 project 路徑作為前綴：

| 資料夾位置 | Skill 資料夾名稱 | 完整 skill name |
|-----------|------------------|-----------------|
| `skills/order/` | `risk-analysis` | `order/risk-analysis` |
| `skills/customer/` | `sentiment-analysis` | `customer/sentiment-analysis` |
| `skills/order/domestic/` | `risk-analysis` | `order/domestic/risk-analysis` |

在 toolset 或任何引用處，MUST 使用完整的 skill name：

```yaml
# kind: Toolset
skills:
  - name: order/risk-analysis              # 完整名稱
  - name: order/domestic/risk-analysis     # 巢狀 project 的完整名稱
```

---

## 5. 引用語法

### 5.1 完整路徑引用

```yaml
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
| `order/*` | order project 下的直接 skills（不遞迴） |
| `order/**` | order project 下所有 skills（遞迴含子 project） |
| `*/risk-*` | 所有 project 中名稱以 `risk-` 開頭的 skills |
| `*/*` | 所有 project 下的直接 skills |

### 5.3 向後相容

不在任何 project 下的 skill 維持原有的扁平引用方式：

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
apiVersion: skillproject/v3
kind: SkillProject

metadata:
  name: order
  description: "訂單相關的 agent skills"
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
apiVersion: skillproject/v3
kind: SkillProject

metadata:
  name: domestic
  description: "國內訂單專用 skills"
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
DRAFT → VALIDATED → PUBLISHED → DEPRECATED → ARCHIVED
```

| 狀態 | 說明 |
|------|------|
| DRAFT | 開發中，project 下的 skills 可使用但標記為 draft |
| VALIDATED | 通過結構與命名驗證 |
| PUBLISHED | 正式發布，skills 可被 toolset 引用 |
| DEPRECATED | 仍可運作但不建議使用 |
| ARCHIVED | project 下所有 skills 不可被引用 |

---

## 8. CLI 指令

### 8.1 list-projects

列出所有已註冊的 SkillProject：

```bash
slogan skill list-projects

# 輸出範例
# NAME         OWNER          SKILLS  STATUS
# order        order-team     3       PUBLISHED
# customer     support-team   2       PUBLISHED
# shared       platform-team  2       PUBLISHED
```

### 8.2 list --project

列出指定 project 下的所有 skills：

```bash
slogan skill list --project order

# 輸出範例
# NAME                          PROJECT  STATUS
# order/risk-analysis           order    PUBLISHED
# order/fraud-detection         order    PUBLISHED
# order/compliance-review       order    PUBLISHED
# order/domestic/risk-analysis  order    PUBLISHED
```

### 8.3 search

搜尋 skill（跨所有 project）：

```bash
slogan skill search "risk"

# 輸出範例
# NAME                                PROJECT             STATUS
# order/risk-analysis                 order               PUBLISHED
# order/domestic/risk-analysis        order/domestic      PUBLISHED
```

---

## 9. 驗證規則

| 規則 | 說明 |
|------|------|
| `project.yaml` 必須 | 資料夾中無 `project.yaml` 則不視為 project |
| name 唯一性 | 同一層級下 project name MUST 唯一 |
| skill name 唯一性 | 完整 skill name（含 project prefix）MUST 全域唯一 |
| 巢狀 project name | 子 project 的 `metadata.name` MUST 與資料夾名稱一致 |
| owner | 建議填寫，但非必須 |
