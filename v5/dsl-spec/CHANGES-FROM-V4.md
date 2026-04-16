# v4 → v5 變更摘要

v5 的目標是**收斂與補齊**：把 v4 中分歧的命名統一為單一寫法、把「待補」的協議補完，不新增大功能。所有 `apiVersion` 從 `*/v4` 升為 `*/v5`，各範例與文字同步更新。

分類：
- **Breaking**：v4 的寫法在 v5 不再合法
- **Clarification**：行為在 v4 就是如此，v5 只是明文化
- **Additions**：v5 新增的小功能或協議

---

## Breaking

| # | 項目 | v4 | v5 | 影響檔案 |
|---|------|----|----|----------|
| B1 | apiVersion | `*/v4` | `*/v5` | 全部 |
| B2 | Tool backend type | `exec` / `stdio` / `http` / `builtin` 混用 | `exec` / `http` / `extension`。`builtin` 為內建 tool 類別、不是 backend type；`stdio` 移除 | `05-tool.md`、`99-examples.md` |
| B3 | Agent 角色欄位 | `system:` 與 `system_prompt:` 並存 | 僅保留 `system:` | `06-agent.md`、`07-resources.md`、`99-examples.md` |
| B4 | Wait step 的 step 模式欄位 | `step_id:`（註解寫「陣列」） | `step:`（值為字串或字串陣列） | `03-steps.md` |
| B5 | Agent / toolset `tools` 多型 | 字串、`{action,...}`、`{type: task, action:}`、`{type: tool, name:}`、`{type: mcp, server:}` 並存 | 僅字串簡寫或 `{action, name?, description?, input?}` 物件；MCP 以 `<server>.*` / `<server>.<tool>` wildcard 引用 | `05-tool.md`、`06-agent.md`、`07-resources.md`、`99-examples.md` |
| B6 | Toolset 引用方式 | `{type: toolset, name: ...}` | `"toolset.<name>"` 命名空間字串 | `07-resources.md` |
| B7 | Agent step `tools` 合併衝突 | 僅描述 tools / tools_override 互斥 | 明定衝突錯誤碼 `agent.tools_conflict`，`tools_exclude` 可與任一搭配 | `06-agent.md` |

---

## Clarification

| # | 項目 | v5 明文化內容 | 檔案 |
|---|------|---------------|------|
| C1 | `prev` 指到 SKIPPED step | `prev.status == "SKIPPED"`、`prev.output == null`；欲取「最近 SUCCEEDED」需用具名 `steps.<id>` | `04-expressions.md` |
| C2 | `when` 求值 | `when == false` → SKIPPED；`when` 求值異常 → FAILED | `03-steps.md`、`04-expressions.md` |
| C3 | Saga 並行補償 | 以「進入 SUCCEEDED 時間戳」降序；`parallel` / `foreach` 內不同分支無全序保證 | `03-steps.md` |
| C4 | Lifecycle init / destroy 求值上下文 | 僅 `secret` / `env` / `project` / `artifacts._workspace_path`，無 step context | `04-expressions.md`、`05-tool.md` |
| C5 | Model config 合併優先級 | `alias.config` ← `definition.model.config` ← `step.model.config`（右側覆寫左側） | `06-agent.md` |
| C6 | Agent 自訂 steps 的執行上限 | `config.max_iterations` / `max_tool_calls` 由引擎強制監控，自訂 steps 繞不過；超過時 `error.type == "max_iterations_exceeded"` / `"max_tool_calls_exceeded"` | `06-agent.md` |
| C7 | Auto step ID 格式 | `_<type>_<index>`：`<type>` 為 step `type` 原字串、`<index>` 為 0-based 陣列索引 | `03-steps.md` |
| C8 | Toolset `includes` 循環 | 載入時靜態偵測，錯誤碼 `toolset.circular_include` | `07-resources.md` |
| C9 | Duration 格式 | 正式 EBNF：`duration := segment+`、`segment := <int> ("h"|"m"|"s")`；不支援小數 / 單位重複 | `01-overview.md` |
| C10 | Foreach 巢狀 `loop` 變數 | 內層 `loop.item` / `loop.index` 遮蔽外層；需要外層時以具名 step 或先 `assign` 保留 | `03-steps.md` |
| C11 | Callback step 的 output 存取 | `name` 即 step id，統一用 `steps.<name>.output`；緊接下一步可用 `prev.output`（不帶名稱） | `05b-function.md` |
| C12 | Wait `step` 模式下 async step FAILED 行為 | Wait FAILED，`error.type == "async_step_failed"`、`error.cause` 為原 error；陣列模式以首個失敗者為主、其餘仍跑完 | `03-steps.md` |
| C13 | Saga 補償失敗處理 | Best-effort：補償失敗不中止其他補償；最終 `error.compensation_failures` 列出失敗清單 | `03-steps.md` |
| C14 | Workflow input/output schema 驗證時機 | input 在 instance 建立時、output 在 `return` 求值後；失敗錯誤碼 `workflow.input_schema_violation` / `output_schema_violation` | `02-workflow.md` |
| C15 | Foreach `failure_policy` 與 catch 互動 | Iteration 內部 `catch` 在 `failure_policy` 之前求值；三策略行為明文化 | `03-steps.md` |
| C16 | Callback step `timeout` 涵蓋範圍 | 從發 callback 算起到 handler `return` 為止；handler 內 step timeout 不延長外層 | `05b-function.md` |
| C17 | Agent `system` / `prompt` 合併分隔符 | step 值以 `\n\n` 連接於 definition 尾部 | `06-agent.md` |
| C18 | `${ }` 引用不存在 step 的錯誤碼 | `error.type == "expression_error"` | `04-expressions.md` |
| C19 | Task registry 解析規則 | 第一段命名風格分流：snake_case → action；kebab-case → MCP server；`toolset` → toolset 引用；混用 `-`/`_` → `registry.invalid_action_name` | `01-overview.md` |
| C20 | MCP server `env` / `headers` / `config` 求值上下文 | 同 lifecycle init（僅 `secret` / `env` / `project` / `artifacts._workspace_path`） | `07-resources.md` |

---

## Additions

| # | 項目 | 說明 | 檔案 |
|---|------|------|------|
| A1 | `foreach.count` 參數 | `count: <int|CEL-int>` 作為 `items: range(n)` 的快捷 | `03-steps.md` |
| A2 | Tool callback 協議 | 補齊 v4 標記為「待補」的協議。完整定義 NDJSON framing、`call_id` 對應、`v5-stream` 模式探測、SSE / `X-Callback-URL` 路由，與 `05b-function.md` caller `callback:` 區塊直接對接 | `05-tool.md` |
| A3 | Agent tools 合併錯誤碼 | `tools` 與 `tools_override` 共存 → `agent.tools_conflict`；toolset includes 循環 → `toolset.circular_include`；foreach 非法 count → `invalid_count`；lifecycle init 失敗 → `lifecycle_init_failed` | `03-steps.md`、`06-agent.md`、`07-resources.md` |

---

## 未處理、留到下一版

以下問題審查時有提到，但 v5 未處理（理由於括號）：

- Tool / Function / Agent / MCP 在 `tools` 清單中的來源類別無法從 YAML 一眼看出（這是統一呼叫模型的設計取捨，不屬於矛盾；需要類別資訊時仍可從 registry 查詢）
- Model alias 的 scope 與 lookup 順序（跨 project 引用、同名覆寫）在 resources 層面尚未規範
- `session.*` namespace 在多層巢狀 agent-as-tool 呼叫下的隔離規則
- Extension backend 的第三方協議細節
