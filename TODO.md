# TODO

## 待辦 — v3 Runtime Spec & CLI Spec

v3 DSL spec 已完成，尚需更新對應的 runtime-spec 與 cli-spec：

1. **v3 Runtime Spec** — 將 v2/runtime-spec/ 升級至 v3，整合 agent session 管理、saga compensation 執行、MCP server 生命週期、tool 並行執行等
2. **v3 CLI Spec** — 將 v2/cli-spec/ 升級至 v3，新增 `slogan test`、`slogan skill` 相關命令、instance label 操作

## 待辦 — DSL Spec 補充

3. **分析 Task toolset 是否合併** — 評估 task step 與 toolset 概念是否應合併，分析合併與不合併的優缺點
4. **Agent stream 處理方式** — 定義 agent step 的 streaming 輸出如何傳遞與消費
5. **wait 同時等待多個事件** — 定義 wait step 可以同時等待多個不同事件的語法與行為（例如同時等 signal + timeout + webhook 等）
6. **sub_workflow 改名為 function** — 將 `sub_workflow` step 重新命名為 `function`，更新所有相關規格文件中的名稱與範例

## 大調整

- Workflow 啟動進入點
- Task 改為叫做 Tool，定義一套工具 ex: memory.read memory.write_line
- Agent 定義一個 Agent
- Toolset 降級為 Resources 一個屬性，定義 工具集， 方便 agent 引用
- Resources 多種設定變數集合
- Project 專案定義 (像是 package.json)
- Secret 放密鑰的地方

## FUTURE.md 延後項目

- [FUTURE.md](FUTURE.md)（Event Replay API、Transactional Outbox/Inbox、JWT Authentication、Pause/Resume、Bulk Operations、Scheduled Trigger DST）
