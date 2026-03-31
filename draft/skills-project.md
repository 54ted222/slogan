# Skills 專案資料夾結構

> **狀態**：草稿
> **範圍**：skill 數量增多時的資料夾（project）層級組織概念

---

## 設計動機

目前 skill 以扁平結構存放，隨著 skill 數量增多會出現問題：

- 大量 YAML 散落無組織，難以找到目標 skill
- 不同業務領域的 skill 混雜
- 缺乏分組層級的共用配置（如相同 model、相同 tools）
- 團隊協作時缺乏 ownership 劃分

---

## 資料夾結構

### 提案：Project 層級

```
skills/
  order/                         # project: order
    project.yaml                 # project metadata
    risk-analysis/               # skill
      SKILL.md
      scripts/
      references/
    fraud-detection/             # skill
      SKILL.md
      scripts/
    compliance-review/           # skill
      SKILL.md
      references/

  customer/                      # project: customer
    project.yaml
    sentiment-analysis/
      SKILL.md
    churn-prediction/
      SKILL.md

  shared/                        # project: shared（跨領域共用）
    project.yaml
    general-knowledge/
      SKILL.md
```

### `project.yaml` 格式

```yaml
name: order
description: "訂單相關的 agent skills"
owner: order-team

# project 層級的預設配置（skill 可覆寫）
defaults:
  labels:
    domain: order
    team: order-team
```

---

## 引用方式

### 完整路徑引用

```yaml
skills:
  - name: order/risk-analysis        # project/skill-name
  - name: order/fraud-detection
  - name: customer/sentiment-analysis
```

### 通配符

```yaml
skills:
  - name: "order/*"                  # 載入 order project 下所有 skills
  - name: "*/risk-*"                 # 載入所有 project 中 risk- 開頭的 skills
```

### 向後相容

不在任何 project 下的 skill 維持原有引用方式：

```yaml
skills:
  - name: risk-analysis              # 扁平結構（無 project）
  - name: order/risk-analysis        # project 結構
```

---

## 搜尋與列表

### CLI

```bash
# 列出所有 projects
slogan skill list-projects

# 列出某 project 下的 skills
slogan skill list --project order

# 搜尋 skill
slogan skill search "risk"
```

### API

```
GET /skills?project=order
GET /skills?search=risk
GET /projects
```

---

## 待決定

- Project 是否僅為資料夾組織，或有獨立的 kind 定義
- 是否支援巢狀 project（`order/domestic/risk-analysis`）
- `project.yaml` 是否必須存在，或可省略
- Skill 的 name 是否自動加上 project prefix
- 跨 project 的 skill 依賴關係
