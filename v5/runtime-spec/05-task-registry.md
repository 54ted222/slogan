# 05 — Task Registry

本文件定義 Task Registry 的結構、解析規則、載入時驗證與動態 (re)registration。對應 [dsl-spec/01-overview.md](../dsl-spec/01-overview.md) 的「Task registry 解析規則」與 [dsl-spec/06-agent.md](../dsl-spec/06-agent.md) 的 tools 機制。

---

## 角色

Task Registry 是 engine 唯一的 action 解析權威：

- 載入時將所有 `kind: Tool` / `kind: Function` / `kind: Agent` definition、所有 MCP server tools、所有 toolset 條目註冊進來。
- 提供 `resolve(action_name) → Action`；`Action` 是引擎內部的執行單元抽象。
- 維持 wildcard 索引：`namespace.*` 可枚舉。

---

## Action 物件

```
Action {
  kind:      "tool" | "function" | "agent" | "mcp" | "builtin",
  name:      string,            # 完整 action name（projects 前綴展開後）
  source:    Definition | MCPRef | BuiltinRef,
  metadata:  { description, version, ... },
  input_schema:  JSONSchema | null,
  output_schema: JSONSchema | null,
  callbacks: map<name, {input_schema, output_schema}> | null,  # function 才有
  compensate: { action_name, input_template } | null,
  idempotent: bool,
  lifecycle:  { init?, destroy? } | null,
  registered_at: timestamp,
  loader:     LoaderInfo,        # 來源檔案 / project / mcp_server
}
```

`resolve()` 回傳同一物件實例的引用；engine 不複製 Action（避免雙寫狀態）。

---

## 解析演算法

```
def resolve(action_name: str, requester_namespace: str|None = None) -> Action:
    # 1. 命名風格判定
    first = action_name.split(".", 1)[0]

    # 2. 字面 toolset 引用
    if first == "toolset":
        path = action_name[len("toolset."):]                 # e.g. "order-processing.order.load"
        toolset_name, _, leaf = path.partition(".")
        toolset = registry.toolsets.get(toolset_name)
        if not toolset: raise NotFound("toolset", toolset_name)
        return toolset.expand(leaf)                          # leaf 為空 → 回傳 ToolsetGroup

    # 3. snake_case 第一段 → 一般 action
    if is_snake(first):
        if action_name in registry.actions:
            return registry.actions[action_name]
        raise NotFound("action", action_name)

    # 4. kebab-case 第一段 → MCP server
    if is_kebab(first):
        server = registry.mcp_servers.get(first)
        if not server: raise NotFound("mcp_server", first)
        tool_name = action_name[len(first)+1:]
        return server.resolve_tool(tool_name)                # tool_name == "*" 時回傳 ServerGroup

    # 5. 不合法
    raise InvalidActionName(action_name)
```

`is_snake(s)` / `is_kebab(s)`：

- snake：`^[a-z][a-z0-9_]*$`
- kebab：`^[a-z][a-z0-9-]*$` 且包含至少一個 `-`（用 `-` 來與 snake 區別；純小寫無分隔的單詞屬 snake）

純單字（如 `git`）視為 snake；引擎以註冊表是否存在區分 namespace。

---

## 載入時驗證

引擎啟動 / project 載入時：

1. 全域唯一性：
   - 兩個 action 完整 name 相同 → `registry.duplicate_action`
   - 名稱第一段同時是 snake 與 kebab 形式合法的衝突（如同時存在 `order-mcp` 與 `order` 動作 namespace）→ 若有歧義，以註冊類型不同為前提允許，但 `resolve` 時走第一段命名風格分流（不會發生實質衝突）；若同類型同名 → `registry.name_conflict`。
2. Toolset 完整性：
   - `tools[]` 與 `includes[]` 中所引用的 action / 子 toolset MUST 已註冊
   - 缺失 → `toolset.unresolved_reference`
3. Toolset 循環：
   - `includes` 形成有向圖，做 DFS；發現環 → `toolset.circular_include`
4. Schema 一致性：
   - 同名不同版本：MAY 並存；引用時必須帶 version 或預設取 `version` 最大者（建議由 project 配置）
5. Lifecycle：
   - tool `lifecycle.init` / `destroy` 中的 backend 也透過 registry 驗證引用（其 backend 不可呼叫其他 tool，僅可 spawn process / http）

驗證失敗導致 engine 拒絕啟動或拒絕載入該 project。

---

## Toolset 展開

Toolset 不是 action，而是「action 集合」。展開時機：

- **agent / tools 引用**：在 agent definition 載入後、agent step 啟動前，由 registry 展開為 action 列表。
- **toolset 內 includes**：遞迴展開（已驗證無循環），結果為扁平 action 集合。
- **MCP wildcard**：在展開時連接 MCP server（lazy），列出其 tools。連線失敗 → 該 wildcard 展開為空，發 `registry.warning.mcp_unavailable`，不阻斷 agent 啟動（但 LLM 看不到該批 tools）。

展開結果按 action name 去重；同名以**最後出現**為準（後加入覆蓋）。

```
ExpandedToolset {
  actions: [Action],        # 已 resolve、無重複
  skills:  [SkillRef],
}
```

`steps.<id>` 不會出現 ToolsetGroup；ToolsetGroup 僅在 agent.tools 中作為展開源。

---

## Agent tools 合併

由 Step Executor 在 agent step 啟動時調用：

```
def merge_tools(definition_tools, step_tools, step_override, step_exclude) -> [ToolBinding]:
    if step_tools and step_override:
        raise AgentToolsConflict()
    base = expand(step_override) if step_override else expand(definition_tools)
    if step_tools:
        added = expand(step_tools)
        for a in added:
            base.upsert(a)        # 同 action name 以 step 覆寫
    if step_exclude:
        base = [b for b in base if b.action_name not in step_exclude]
    return base
```

`ToolBinding` 是 Action + 覆寫資訊：

```
ToolBinding {
  action: Action,
  exposed_name: string,            # LLM 看到的 function 名（預設 = action.name）
  exposed_description: string,
  hidden_input: map,               # input pre-bind，LLM 看不到、執行時合併
  visible_schema: JSONSchema,      # action.input_schema 移除 hidden_input keys
}
```

LLM 呼叫 ToolBinding 時，引擎將 LLM 提供的 args + `hidden_input` deep-merge（LLM args 優先）後傳給對應 Action。

---

## Action 執行入口

`Action.execute(input, context) → ActionOutcome`

- 對 Tool：呼叫 Backend Driver
- 對 Function：建立 Function Instance；返回 promise
- 對 Agent：建立 Agent Session；返回 promise
- 對 MCP tool：透過 MCP 連線發送請求
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
| `agent.*` | agent loop 控制：`agent.llm_call`、`agent.exec_tool_calls`、`agent.set_messages`、`agent.ask_human`、`agent.activate_skill`、`agent.read_skill_file` |
| `artifact.*` | artifact 操作：`artifact.list`、（未來）`artifact.read` / `artifact.write` |
| `registry.*` | registry 自省：`registry.search`、`registry.list`、`registry.activate` |
| `builtin.*` | 通用：`builtin.echo`、`builtin.sleep`（測試用） |

Builtin 的 input/output schema 由引擎內建，不需 YAML 定義。

`registry.activate` 動態加入一個 action 至當前 agent session 的可用 tools；僅影響該 session，session 結束後失效。

---

## Project 前綴

`projects/<project_name>/...` 下的定義在註冊時自動加上 `<project_name>/` 前綴：

- `projects/order/tools/order.load.yaml` → action name `order/order.load`
- 引用時：`type: task` 的 `action: order/order.load`

跨 project 引用 MUST 寫完整路徑；同 project 內 MAY 省略前綴（loader 自動補齊）。

巢狀 project：路徑以 `/` 分隔層層展開（`order/domestic/compliance.check`）。

---

## Wildcard 索引

`namespace.*` 在 agent.tools / toolset.tools 中可用。

- 註冊時為每個 action 維護 namespace prefix index：`order` → [order.load, order.update, ...]
- `*` 不包含子 namespace（`order.*` 不展開為 `order.shipping.*`）；如需深度展開可寫 `order.shipping.*` 另一條
- MCP server 的 wildcard 由 server 提供 tool 列表 API；engine 連線並列舉

---

## 動態註冊

執行期透過 `registry.activate` 引入新 action 的條件：

- 必須已存在於 registry（無法執行期定義新 tool）
- 僅影響呼叫此 builtin 的 agent session 的可見 tool 集
- 不修改 instance / project 級的合併結果

若需要熱加載新 tool definition（非 agent session 範圍），實作可提供 admin API；reload 時 MUST 取得全域 lock，避免進行中的 instance 看到不一致 registry。

---

## 解析快取

- `resolve()` 結果 SHOULD 被 instance 級快取（key: `(action_name, instance_id)`）；避免重複命名解析。
- Toolset 展開在 agent step 開始時進行一次，結果 cache 至 session 結束。
- MCP wildcard 展開的結果**不快取**（每次 session 啟動時重新列舉），確保新增 / 移除 tool 即時反映。

---

## 觀測性

Registry MUST 暴露：

- 載入後的所有 action 清單（含來源檔案）
- 每次 `resolve()` 的命中與 miss（log）
- Wildcard 展開結果（debug log）
- Toolset 展開結果（debug log）
- MCP server 連線狀態
