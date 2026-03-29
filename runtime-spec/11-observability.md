# 11 — Observability

本文件定義引擎的可觀測性規格：metrics、distributed tracing、health check、resource tracking。

---

## 設計原則

| 原則 | 說明 |
|------|------|
| 標準介面 | 使用業界標準格式（OpenTelemetry / Prometheus），不自創格式 |
| 零配置可用 | 預設啟用基本 metrics 與 health check，進階功能（tracing export）為 opt-in |
| 低開銷 | Observability 不應顯著影響引擎效能 |

---

## Metrics

引擎 MUST 暴露以下 metrics。格式 SHOULD 相容 Prometheus / OpenTelemetry。

### Instance Metrics

| Metric | 型別 | Labels | 說明 |
|--------|------|--------|------|
| `slogan_instances_created_total` | Counter | `workflow_name` | 建立的 instance 累計數 |
| `slogan_instances_completed_total` | Counter | `workflow_name`, `state` | 完成的 instance 累計數（state = succeeded / failed / cancelled） |
| `slogan_instances_active` | Gauge | `workflow_name`, `state` | 當前非 terminal 的 instance 數量（state = created / running / waiting） |
| `slogan_instance_duration_seconds` | Histogram | `workflow_name`, `state` | Instance 從建立到完成的時長 |

### Step Metrics

| Metric | 型別 | Labels | 說明 |
|--------|------|--------|------|
| `slogan_steps_executed_total` | Counter | `workflow_name`, `step_type`, `state` | 執行的 step 累計數 |
| `slogan_step_duration_seconds` | Histogram | `workflow_name`, `step_type` | Step 執行時長（RUNNING 到 terminal） |
| `slogan_step_retries_total` | Counter | `workflow_name`, `action` | Step retry 累計次數 |

### Task Metrics

| Metric | 型別 | Labels | 說明 |
|--------|------|--------|------|
| `slogan_task_executions_total` | Counter | `action`, `backend_type`, `success` | Task 執行累計數 |
| `slogan_task_duration_seconds` | Histogram | `action`, `backend_type` | Task 執行時長（不含排隊時間） |

### Event Metrics

| Metric | 型別 | Labels | 說明 |
|--------|------|--------|------|
| `slogan_events_received_total` | Counter | `event_type` | 收到的事件累計數 |
| `slogan_events_matched_total` | Counter | `event_type`, `match_type` | 匹配成功的事件數（match_type = trigger / wait） |

### Engine Metrics

| Metric | 型別 | Labels | 說明 |
|--------|------|--------|------|
| `slogan_scheduler_cycles_total` | Counter | — | Scheduler 主迴圈執行次數 |
| `slogan_running_steps` | Gauge | — | 當前 RUNNING 的 step 數量 |
| `slogan_queued_steps` | Gauge | — | 當前排隊中的 step 數量 |
| `slogan_active_leases` | Gauge | `worker_id` | 當前持有的 instance leases 數量 |

---

## Distributed Tracing

引擎 SHOULD 支援 OpenTelemetry trace propagation。

### Trace 結構

一個 workflow instance 的執行對應一個 trace。Span 階層如下：

```
Trace: workflow instance {instance_id}
  └─ Span: workflow.run (workflow_name, version)
       ├─ Span: step.execute (step_id, step_type)
       │    └─ Span: task.execute (action, backend_type)
       │         └─ Span: http.request / stdio.process (backend 專屬)
       ├─ Span: step.execute (step_id, step_type)
       │    └─ Span: wait_event (event_type)
       └─ Span: step.execute (step_id, step_type)
            └─ Span: sub_workflow.execute (child_workflow_name)
                 └─ Trace Link → child instance trace
```

### Correlation ID

每個 workflow instance MUST 擁有一個 correlation ID，用於串連：

| 場景 | Correlation ID 來源 |
|------|-------------------|
| Manual trigger | 引擎自動產生 UUID |
| Event trigger | 使用 `event.id`，若無則自動產生 |
| Sub_workflow | 繼承 parent instance 的 correlation ID |

Correlation ID 存於 `workflow_instance` 實體中（新增 `correlation_id` 欄位）。

### 傳遞至 Task Handler

| Backend | 傳遞方式 |
|---------|---------|
| stdio | `context.correlation_id`（request JSON 欄位）+ `SLOGAN_CORRELATION_ID` 環境變數 |
| http | `X-Correlation-Id` HTTP header |
| builtin | 引擎內部存取 |

### 跨 Instance Trace Link

Sub_workflow 建立 child instance 時，引擎 SHOULD 在 parent span 中加入 trace link 指向 child instance 的 trace。

---

## Health Check

引擎 MUST 提供 health check endpoint。

### Endpoint

```
GET /health
```

### Response

```json
{
  "status": "healthy",
  "components": {
    "storage": { "status": "healthy" },
    "scheduler": { "status": "healthy" },
    "event_router": { "status": "healthy" },
    "timeout_manager": { "status": "healthy" }
  },
  "uptime_seconds": 3600,
  "version": "0.1.0"
}
```

### Status 定義

| Status | 說明 |
|--------|------|
| `healthy` | 所有元件正常運作 |
| `degraded` | 部分元件異常但仍可服務（如 storage 延遲偏高） |
| `unhealthy` | 無法正常服務 |

### 元件健康檢查

| 元件 | 檢查方式 |
|------|---------|
| Storage | 執行簡單查詢（如 `SELECT 1`），確認連線正常 |
| Scheduler | 確認主迴圈在預期時間內有執行（無 deadlock） |
| Event Router | 確認 trigger subscriptions 已載入 |
| Timeout Manager | 確認 timer 監控正在運行 |

### HTTP Status Code

| 整體 Status | HTTP Code |
|------------|-----------|
| healthy | 200 |
| degraded | 200 |
| unhealthy | 503 |

---

## Resource Tracking

引擎 SHOULD 追蹤以下資源使用情況：

### Process 資源

| 指標 | 說明 |
|------|------|
| `slogan_process_memory_bytes` | 引擎 process 記憶體使用量 |
| `slogan_process_cpu_seconds_total` | 引擎 process CPU 使用時間 |
| `slogan_open_file_descriptors` | 開啟的 file descriptors 數量 |

### Storage 資源

| 指標 | 說明 |
|------|------|
| `slogan_storage_query_duration_seconds` | Storage 查詢時長（Histogram） |
| `slogan_storage_connections_active` | 當前活躍的 storage 連線數 |

### Task 資源

| 指標 | 說明 |
|------|------|
| `slogan_stdio_processes_active` | 當前活躍的 stdio 子 process 數 |
| `slogan_http_requests_active` | 當前進行中的 HTTP 請求數 |

---

## Execution Log 與 Observability 的關係

Execution log（見 [08-storage-schema](08-storage-schema.md)）是 **持久化的稽核記錄**，metrics 和 tracing 是 **即時的可觀測性信號**。兩者互補：

| 面向 | Execution Log | Metrics / Tracing |
|------|--------------|-------------------|
| 持久化 | 是（DB） | 否（記憶體 / 外部系統） |
| 查詢粒度 | 單一 instance / step | 聚合統計 |
| 用途 | 稽核、除錯、replay | 監控、告警、效能分析 |
| 效能影響 | 寫入 DB | 低（計數器遞增） |
