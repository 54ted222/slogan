# 05 — Task Registry

本文件定義 Task Registry 的結構、解析規則、載入時驗證。對應 [dsl-spec/01-overview.md](../dsl-spec/01-overview.md) 的「Task registry 解析規則」。

---

## 角色

Task Registry 是 engine 唯一的 action 解析權威：

- 載入時將所有 `kind: Tool` / `kind: Function` definition 註冊進來。
- 提供 `resolve(action_name) → Action`；`Action` 是引擎內部的執行單元抽象。

---

## Action 物件

```
Action {
  kind:      "tool" | "function" | "builtin",
  name:      string,            # 完整 action name（projects 前綴展開後）
  source:    Definition | BuiltinRef,
  metadata:  { description, version, ... },
  input_schema:  JSONSchema | null,
  output_schema: JSONSchema | null,
  callbacks: map<name, {input_schema, output_schema}> | null,  # function 才有
  compensate: { action_name, input_template } | null,
  idempotent: bool,
  lifecycle:  { init?, destroy? } | null,
  registered_at: timestamp,
  loader:     LoaderInfo,        # 來源檔案 / project
}
```

`resolve()` 回傳同一物件實例的引用；engine 不複製 Action（避免雙寫狀態）。

---

## 解析演算法

```
def resolve(action_name: str) -> Action:
    # 1. 若含 '/'，切出 project prefix 與 action body
    if "/" in action_name:
        prefix, body = action_name.rsplit("/", 1)
        # prefix 每一段 MUST 為 kebab 風格 project name
        for seg in prefix.split("/"):
            if not is_kebab(seg):
                raise InvalidActionName(action_name)
    else:
        body = action_name

    # 2. action body MUST 為 snake_case + dotted
    first = body.split(".", 1)[0]
    if not is_snake(first):
        raise InvalidActionName(action_name)
    for seg in body.split("."):
        if not is_snake(seg):
            raise InvalidActionName(action_name)

    # 3. 以完整 action_name 查表
    if action_name in registry.actions:
        return registry.actions[action_name]
    raise NotFound("action", action_name)
```

- `is_snake(s)`：`^[a-z][a-z0-9_]*$`
- `is_kebab(s)`：`^[a-z][a-z0-9-]*$`

純單字（如 `git`）同時符合 snake / kebab；引擎以註冊表是否存在區分 namespace。

---

## 載入時驗證

引擎啟動 / project 載入時：

1. 全域唯一性：
   - 兩個 action 完整 name 相同 → `registry.duplicate_action`
   - 同類型同名 → `registry.name_conflict`
2. Schema 一致性：
   - 同名不同版本：MAY 並存；引用時必須帶 version 或預設取 `version` 最大者（建議由 project 配置）
3. Lifecycle：
   - tool `lifecycle.init` / `destroy` 中的 backend 也透過 registry 驗證引用（其 backend 不可呼叫其他 tool，僅可 spawn process / http）

驗證失敗導致 engine 拒絕啟動或拒絕載入該 project。

---

## Action 執行入口

`Action.execute(input, context) → ActionOutcome`

- 對 Tool：呼叫 Backend Driver
- 對 Function：建立 Function Instance；返回 promise
- 對 Builtin：直接呼叫引擎內部實作

ActionOutcome：

```
{
  status: "SUCCEEDED" | "FAILED",
  output: any,
  error:  ErrorObject | null,
  metadata: { duration_ms, attempts, ... }
}
```

---

## Builtin Actions

引擎預先註冊；命名空間：

| Namespace | 用途 |
|-----------|------|
| `artifact.*` | artifact 操作：`artifact.list`、（未來）`artifact.read` / `artifact.write` |
| `builtin.*` | 通用：`builtin.echo`、`builtin.sleep`（測試用） |

Builtin 的 input/output schema 由引擎內建，不需 YAML 定義。

---

## Project 前綴

`projects/<project_name>/...` 下的定義在註冊時自動加上 `<project_name>/` 前綴：

- `projects/order/tools/order.load.yaml` → action name `order/order.load`
- 引用時：`type: task` 的 `action: order/order.load`

跨 project 引用 MUST 寫完整路徑；同 project 內 MAY 省略前綴（loader 自動補齊）。

巢狀 project：路徑以 `/` 分隔層層展開（`order/domestic/compliance.check`）。

---

## 解析快取

- `resolve()` 結果 SHOULD 被 instance 級快取（key: `(action_name, instance_id)`）；避免重複命名解析。

---

## 觀測性

Registry MUST 暴露：

- 載入後的所有 action 清單（含來源檔案）
- 每次 `resolve()` 的命中與 miss（log）
