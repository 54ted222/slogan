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
  definition_hash: string,      # 規範化 YAML 的 SHA256（見下「Hot reload」）；builtin 為固定哨兵值（如 "builtin:<name>"）
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

    # 2. 切出 @version（若有）
    if "@" in body:
        body, version_str = body.rsplit("@", 1)
        try:
            version = int(version_str)
        except ValueError:
            raise InvalidActionName(action_name)
    else:
        version = None  # 由版本解析優先權決定

    # 3. action body MUST 為 snake_case + dotted
    first = body.split(".", 1)[0]
    if not is_snake(first):
        raise InvalidActionName(action_name)
    for seg in body.split("."):
        if not is_snake(seg):
            raise InvalidActionName(action_name)

    # 4. 組合最終 key 查表
    canonical = f"{prefix}/{body}" if "/" in action_name else body
    if version is None:
        version = resolve_version(canonical, caller_instance)
    key = (canonical, version)
    if key in registry.actions:
        return registry.actions[key]
    raise NotFound("action", action_name)
```

### `resolve_version` 演算法

當 step 未指定 `@version` 時，引擎依以下順序決定版本（每一步找到即回傳）：

1. **Instance action pin**：若 caller_instance 已為該 `canonical` 名稱鎖定版本（見下），直接使用該版本
2. **Project default**：action 所屬 project 的 `project.yaml` 中 `defaults.action_versions.<canonical>: <version>`（若有）
3. **Global default**：引擎 config 的 `registry.default_version_policy`：`highest`（預設）/ `lowest` / `require_explicit`（後者直接 raise `registry.version_not_specified`）

### Instance 建立時的 action pin

為避免「step 執行時 registry 剛好熱載入新版本，同一 instance 前後 step 解析到不同版本」的漂移，instance **首次** 解析到某 `canonical` 的實際 version 時 MUST 將 `(canonical → version)` 記錄至 instance 的 `action_pins` map（持久化於 Instance Store 的 `instances.action_pins` 欄位）；該 instance 後續所有同名解析**沿用**此版本，即使 registry 已更新。

- `action_pins` 不跨 instance 共享；子 function instance 獨立解析並持有自己的 pins
- 顯式 `action: foo@3` 不寫入 pin（已是確定版本）；但會驗證 registry 中存在該 `(foo, 3)`；不存在 → `registry.action_not_found`
- Replay 時引擎 MUST 依 `action_pins` 還原版本；若對應 `(canonical, version)` 在 registry 中已被刪除 → `workflow_version_deleted`（同既有語意）

此規則與既有「workflow / function definition 版本鎖定」（見 `02-instance-lifecycle.md`）**分層一致**：workflow 自身版本在 instance 建立時鎖定；其呼叫的 action 在首次解析時鎖定。

### Hot reload 下的定義一致性

Registry 於 engine 運行期支援「熱載入新版本」（新 version 加入、既有 version **不可覆寫內容**）。配合 instance `action_pins` 的保護：

1. **Definition hash**：載入時 engine MUST 對每個 action 的規範化 YAML（移除 comment / 空白、key 排序）計算 SHA256，存入 registry 的 `Action.definition_hash`
2. **既有版本不可變**：若 hot reload 時同 `(canonical, version)` 出現但 `definition_hash` 不同 → `registry.version_content_mismatch`（拒絕載入）；使用者必須分配新 version 號。此規則防「instance 基於 version 2 執行到一半、DevOps 修改 version 2 內容」造成語意漂移
3. **版本刪除處置**：若熱載入時某 `(canonical, version)` 從 registry 消失且有 instance pin 指向此版本：
   - 該 instance 若未來執行該 action 的 step（含 compensate） → step FAILED，`error.type == "action_pin_version_not_found"`、`error.details = { action: canonical, pinned_version: N, available_versions: [...] }`
   - 該 step 可由 catch 接住；使用者可決定是否升級 pin（需以新 instance 重新啟動）或標記為最終失敗
   - 未引用該 action 的 step 不受影響（正常繼續）
4. **Compensate 的遞迴 pin**：當原 step 首次 resolve 並 pin 時，若其 tool definition 含 `compensate.action`（無 `@version`），engine 一併 resolve 並 pin `(compensate_canonical, version)` 至同一 `action_pins` 結構。為保 JSON schema 一致，**所有 pin 的 value 統一為 object 形式** `{canonical: string, version: int}`；key 為 lookup 名（原 step pin 以 canonical name 本身為 key，compensate pin 以 `"<origin_canonical>::compensate"` 區分）：
   ```json
   {
     "action_pins": {
       "order/payment": { "canonical": "order/payment", "version": 2 },
       "order/payment::compensate": { "canonical": "order/payment.refund", "version": 2 }
     }
   }
   ```
   - 一般 pin 的 `canonical` 恆等於 key；compensate pin 的 `canonical` 為補償 action 的實際名稱（可能與 origin 不同）
   - 歷史資料相容：engine 於讀取 `action_pins` 時 MAY 容忍純 integer value（視為 `{canonical: <key>, version: <int>}`）以便向前讀取早期 checkpoint；寫入一律採 object 形式
   - Compensate 執行時以 pin 內 `canonical` + `version` 為準；若 compensate action 自身已 `@version` 顯式 → 不寫入 pin（版本已確定）
5. **Replay 驗證**：replay 既有 instance 時重新比對 `definition_hash`；若當前 registry 中相同版本之 hash 與 pin 時的 hash 不符（僅可能於規則 2 被違反時發生）→ `workflow_version_deleted.hash_mismatch`，replay 失敗

- `is_snake(s)`：`^[a-z][a-z0-9_]*$`
- `is_kebab(s)`：`^[a-z][a-z0-9-]*$`

純單字（如 `git`）同時符合 snake / kebab；引擎以註冊表是否存在區分 namespace。

---

## 載入時驗證

引擎啟動 / project 載入時：

1. 全域唯一性：
   - 兩個 action 完整 name（含 version）相同 → `registry.duplicate_action`
   - 同 name + 同 version 但 kind 不同（Tool vs Function）→ `registry.name_conflict`
     - v6 **禁止** Tool 與 Function 共用同一 `name@version`（即使 kind 不同）；registry 為 action name 的單一真相來源，避免呼叫端猜 kind
     - 同 name 不同 version 共存時，所有 version 必須為同一 kind；混 kind → 載入失敗
2. Schema 一致性：
   - 同名不同版本：MAY 並存；引用時必須帶 version 或預設取 `version` 最大者（建議由 project 配置）
3. Lifecycle：
   - tool `lifecycle.init` / `destroy` 中的 backend 也透過 registry 驗證引用（其 backend 不可呼叫其他 tool，僅可 spawn process / http）
4. Function 依賴圖：
   - 對所有 Function 構造「function A 呼叫 function B」的有向圖（掃 `type: task` step 的 action）
   - 執行 Tarjan SCC 偵測強連通元件；發現循環（含自呼叫）→ `registry.dependency_cycle`，error.details 列出循環路徑
   - 允許使用者顯式標示 `recursion_allowed: true` 於 Function metadata 以跳過該 function 的循環檢查（v6 預設關閉）

驗證失敗導致 engine 拒絕啟動或拒絕載入該 project。

## 遞迴深度限制

即便通過載入驗證（`recursion_allowed: true`），運行時仍有深度上限：

- 每個 Function instance 維護 `call_depth`（從父 instance 繼承 +1；根 workflow instance 為 0）
- 預設上限 128；超過 → function instance FAILED，`error.type: "max_recursion_depth_exceeded"`
- 上限可由 `engine.max_function_call_depth` 覆寫（見 `10-concurrency.md` 的限制表）

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

### Builtin 內建 schema

| Action | input_schema | output_schema | 說明 |
|--------|--------------|---------------|------|
| `builtin.echo` | `{type: object}`（任意 map） | 與 input 相同 | 回傳 input；便於測試 |
| `builtin.sleep` | `{type: object, properties: {duration: {type: string}}, required: [duration]}` | `{type: object, properties: {slept_ms: {type: integer}}}` | duration 依 duration format 解析；engine 以 non-blocking timer 實現（不阻塞 main loop） |
| `artifact.list` | `{type: object, properties: {artifact: {type: string}}, required: [artifact]}` | `{type: array, items: {type: object, properties: {name: string, size: integer, modified_at: string(RFC3339)}}}` | 列出指定 artifact 子目錄下所有檔案；`artifact` 指 artifact 名稱；不遞迴 |

未來擴充（v6 保留，**暫不實作**）：

- `artifact.read` / `artifact.write`：透過 builtin 讀寫 artifact 檔案；v6 建議使用 tool backend 自行讀寫
- `builtin.uuid` / `builtin.hash` / `builtin.random`：純值產生器；v6 已由 CEL 函式提供（`uuid()` / `sha256()` 等）

所有 builtin action 的執行時機與 error handling 與 Tool action 一致（input_schema 驗證、schema_violation 錯誤碼、idempotent 保證等）。

### Builtin 版本規則

- Builtin action **不支援** `@version` 語法；引擎將 builtin 視為「恆定版本」且不暴露 version 欄位
- Step 宣告 `action: builtin.echo@1` / `action: artifact.list@1` 等帶版本號的形式（任何 `builtin.*` / `artifact.*` 前綴）→ **載入期拒絕**，`error.type: "registry.invalid_action_version"`、`details.hint: "builtin actions (builtin.* / artifact.*) do not support @version"`
- Builtin action 的行為於 engine 版本升級時可能改變；使用者若需版本化，應以 Tool wrapper 封裝
- Builtin input 驗證時機同 Tool（step 進入 RUNNING、CEL 求值後、執行前）；失敗 → step FAILED，`error.type == "schema_violation"`（不特別區分 tool / builtin）

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
