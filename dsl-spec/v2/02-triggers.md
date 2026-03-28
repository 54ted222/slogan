# 02 — Triggers

本文件定義 workflow instance 的啟動機制。

---

## 觸發模型

Trigger 的唯一職責是 **建立新的 workflow instance**。

- 一個 workflow definition MAY 定義多個 triggers
- 每個 trigger 獨立運作，各自可建立 instance
- `triggers` 陣列 MUST 至少包含一個 trigger

```yaml
triggers:
  - type: manual
  - type: event
    event: order.created
    when: ${ event.data.source == "api" }
```

---

## manual Trigger

透過 API 或 CLI 手動啟動 workflow instance。

### Schema

```yaml
- type: manual
```

無額外欄位。

### 行為

- 呼叫端提供 input 資料（需符合 `input_schema`）
- 引擎建立 instance，狀態設為 CREATED，隨即轉為 RUNNING

---

## event Trigger

當系統收到匹配的事件時，自動建立 workflow instance。

### Schema

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `type` | string | MUST | 固定值 `event` |
| `event` | string | MUST | 要訂閱的事件類型 |
| `when` | CEL expression | MAY | 過濾條件，MUST 回傳 boolean |
| `input_mapping` | map | MAY | 事件資料到 workflow input 的映射 |

```yaml
- type: event
  event: order.created
  when: ${ event.data.source == "api" && event.data.amount > 100 }
  input_mapping:
    order_id: ${ event.data.order_id }
    action: ${ event.data.type }
    notify_customer: ${ default(event.data.notify, true) }
```

### 行為

1. 引擎訂閱指定的 event type
2. 收到事件時，若有 `when` 條件，以事件資料求值 CEL 表達式
3. 條件為 `true`（或無 `when`）→ 建立新 instance
4. 條件為 `false` → 忽略該事件
5. `when` 求值失敗 → 忽略該事件並記錄錯誤

### when 的求值上下文

`when` 表達式中可用的 namespace：

| Namespace | 說明 |
|-----------|------|
| `event.type` | 事件類型 |
| `event.data` | 事件酬載 |
| `event.id` | 事件唯一 ID |
| `event.timestamp` | 事件時間戳 |
| `event.source` | 事件來源 |

注意：`input`、`steps`、`vars` 在 trigger 的 `when` 中不可用（instance 尚未建立）。

---

## Trigger 與 Input 的關係

### manual trigger

Input 由呼叫端明確提供。

### event trigger

Event trigger 透過 `input_mapping` 將事件資料轉換為 workflow input：

#### 有 input_mapping（建議）

- 每個 key 為 workflow input 的欄位名
- 每個 value 為 CEL 表達式，在 `event` namespace 下求值
- 映射結果 MUST 符合 `input_schema`（若有定義）
- 映射表達式求值失敗 → instance 不建立，記錄錯誤

```yaml
- type: event
  event: order.created
  input_mapping:
    order_id: ${ event.data.order_id }
    action: "pay"                          # 字面值
    notify_customer: ${ has(event.data.notify) ? event.data.notify : true }
    items: ${ event.data.line_items }
```

#### 無 input_mapping（隱式 passthrough）

- `event.data` 整體作為 workflow input
- `event.data` MUST 直接符合 `input_schema`
- 適用於事件結構與 input schema 完全一致的情況

---

## 多 Trigger 範例

```yaml
triggers:
  # 手動啟動（測試或管理用途）
  - type: manual

  # 新訂單事件（僅 API 來源），帶 input_mapping
  - type: event
    event: order.created
    when: ${ event.data.source == "api" }
    input_mapping:
      order_id: ${ event.data.order_id }
      action: "pay"
      notify_customer: ${ has(event.data.notify) ? event.data.notify : true }
      items: ${ event.data.line_items }

  # 新訂單事件（僅金額 > 1000），passthrough
  - type: event
    event: order.created
    when: ${ event.data.amount > 1000 }
```

以上設定表示：同一事件可能同時觸發多個 trigger（若都符合條件），各自建立獨立的 instance。引擎 SHOULD 提供機制避免非預期的重複建立（如 deduplication key）。
