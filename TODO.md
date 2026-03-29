# TODO — DSL v2 規格補完清單

所有主要規格撰寫與一致性審查已完成。以下為已完成的項目記錄。

## 已完成

- [x] 版本遷移策略 → `dsl-spec/v2/19-versioning.md`
- [x] Task 執行語意保證 → `dsl-spec/v2/20-execution-guarantees.md`
- [x] 全域並發控制 → `runtime-spec/10-concurrency-control.md`
- [x] Observability 規格 → `runtime-spec/11-observability.md`
- [x] Artifact 生命週期補完 → `dsl-spec/v2/13-artifacts.md`
- [x] Error Code 體系化 → `dsl-spec/v2/12-error-handling.md`
- [x] Sub-workflow 生命週期 → `dsl-spec/v2/11-step-sub-workflow.md`
- [x] HTTP / Stdio Backend 邊界行為 → `dsl-spec/v2/17-task-definition.md` + `runtime-spec/05-task-executor.md`
- [x] Workflow 原地更新 → `dsl-spec/v2/14-lifecycle.md`
- [x] CEL 表達式邊界行為 → `dsl-spec/v2/04-expressions.md`
- [x] Event Deduplication → `runtime-spec/03-event-router.md`
- [x] Audit Trail 與合規 → `runtime-spec/08-storage-schema.md`
- [x] Scheduled Trigger → `dsl-spec/v2/02-triggers.md`
- [x] Delayed Event → `dsl-spec/v2/09-step-events.md`
- [x] HTTP Trigger → `dsl-spec/v2/02-triggers.md` + `runtime-spec/03-event-router.md`
- [x] Authentication & Authorization → `runtime-spec/12-authentication.md`
- [x] 全規格一致性審查（矛盾修復、缺失補齊、不對稱修正）

## 評估
transactional outbox/inbox pattern
	2.	event publish state machine
	3.	dead-letter / poison event 策略
	4.	dedup key 保存期限與 replay policy
JWT 配合 設定檔案 公鑰

circuit breaker
	•	connection pool / keepalive policy
	•	per-backend rate limit / quotas
	•	sandbox / container isolation / seccomp 類型規格



## FUTURE.md 延後項目

- Pause / Resume 機制
- Bulk Operations
