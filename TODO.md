# TODO

## ✅ 已完成 — 已併入 v3 DSL Spec

以下項目已全部決策完畢（見 [DECISION.md](DECISION.md)），並整合至 [v3/dsl-spec/](v3/dsl-spec/)：

- ~~Saga / Compensation 模式~~ → [v3/dsl-spec/12-step-saga.md](v3/dsl-spec/12-step-saga.md)
- ~~JSON Schema 發布~~ → [v3/dsl-spec/31-json-schema-publication.md](v3/dsl-spec/31-json-schema-publication.md)
- ~~Instance Labels + 搜尋 API~~ → [v3/dsl-spec/24-instance-labels.md](v3/dsl-spec/24-instance-labels.md)
- ~~本地測試模式~~ → [v3/dsl-spec/30-testing.md](v3/dsl-spec/30-testing.md)
- ~~Continue-as-new~~ → [v3/dsl-spec/10-step-terminal.md](v3/dsl-spec/10-step-terminal.md)
- ~~Skills 專案資料夾結構~~ → [v3/dsl-spec/20-skill-project.md](v3/dsl-spec/20-skill-project.md)
- ~~Toolset 概念~~ → [v3/dsl-spec/18-toolset-definition.md](v3/dsl-spec/18-toolset-definition.md)
- ~~Agent 功能~~ → [v3/dsl-spec/13](v3/dsl-spec/13-agent-definition.md)–[17](v3/dsl-spec/17-agent-hooks-and-streaming.md)
- ~~DSL 擴充~~ → [v3/dsl-spec/19-resources-definition.md](v3/dsl-spec/19-resources-definition.md)（resources、string template、model alias）、[08](v3/dsl-spec/08-step-control-flow.md)（expr→condition）、[14](v3/dsl-spec/14-agent-tools.md)（tool 命名）

## 待辦 — v3 Runtime Spec & CLI Spec

v3 DSL spec 已完成，尚需更新對應的 runtime-spec 與 cli-spec：

1. **v3 Runtime Spec** — 將 v2/runtime-spec/ 升級至 v3，整合 agent session 管理、saga compensation 執行、MCP server 生命週期、tool 並行執行等
2. **v3 CLI Spec** — 將 v2/cli-spec/ 升級至 v3，新增 `slogan test`、`slogan skill` 相關命令、instance label 操作

## FUTURE.md 延後項目

- [FUTURE.md](FUTURE.md)（Event Replay API、Transactional Outbox/Inbox、JWT Authentication、Pause/Resume、Bulk Operations、Scheduled Trigger DST）
