# 00 — Runtime 總覽

本規格定義 Slogan workflow engine 的執行時期架構。引擎依據 [DSL v2](../dsl-spec/v2/00-overview.md) 定義的 workflow / task / secret 文件，建立並驅動 workflow instance 的執行。

---

## 與 DSL 規格的關係

| 規格 | 職責 |
|------|------|
| **DSL spec**（`dsl-spec/v2`） | 定義語言語意：YAML 結構、step 類型、表達式、狀態機、錯誤處理模型 |
| **Runtime spec**（本目錄） | 定義引擎行為：元件架構、API 介面、排程流程、事件路由、task 執行協定 |

DSL spec 定義「規則」，runtime spec 定義「如何執行這些規則」。

---

## 架構概覽

```
                          ┌─────────────────────────────┐
                          │        Engine API            │
                          │  (definitions, instances,    │
                          │   events, cancellation)      │
                          └──────────┬──────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
     ┌─────────────┐      ┌──────────────┐      ┌──────────────┐
     │  Event       │      │  Scheduler    │      │  Timeout     │
     │  Router      │      │  Loop         │      │  Manager     │
     └──────┬──────┘      └──────┬───────┘      └──────┬───────┘
            │                    │                      │
            │              ┌─────┴─────┐                │
            │              │           │                │
            ▼              ▼           ▼                ▼
     ┌───────────┐  ┌───────────┐ ┌──────────┐  ┌───────────┐
     │ Trigger   │  │  Step     │ │Expression│  │ Timer     │
     │ Evaluator │  │ Executor  │ │Evaluator │  │ Store     │
     └───────────┘  └─────┬─────┘ └──────────┘  └───────────┘
                          │
              ┌───────────┼───────────┐
              │           │           │
              ▼           ▼           ▼
        ┌──────────┐ ┌─────────┐ ┌────────┐
        │  Task    │ │  Task   │ │ Task   │
        │ Executor │ │Executor │ │Executor│
        │  (bash)  │ │ (http)  │ │ (sdk)  │
        └──────────┘ └─────────┘ └────────┘
                          │
                    ┌─────┴─────┐
                    │  Storage   │
                    │  Layer     │
                    └───────────┘
```

---

## 核心元件

| 元件 | 職責 | 對應規格文件 |
|------|------|-------------|
| **Engine API** | 對外介面：管理 definitions、建立 instances、發送 events、取消 instances | [01-engine-api](01-engine-api.md) |
| **Scheduler** | 驅動 instance 執行：尋找 READY steps、分派執行、狀態轉換 | [02-scheduler](02-scheduler.md) |
| **Event Router** | 事件路由：trigger 訂閱、wait_event 訂閱、事件分發 | [03-event-router](03-event-router.md) |
| **Step Executor** | 執行各類 step：解析 step type、呼叫對應處理邏輯 | [04-step-executor](04-step-executor.md) |
| **Task Executor** | 執行 task backend：bash / http / sdk / builtin 四種協定 | [05-task-executor](05-task-executor.md) |
| **Expression Evaluator** | CEL 表達式求值：namespace 解析、型別檢查、非確定性函式記錄 | [06-expression-evaluator](06-expression-evaluator.md) |
| **Timeout Manager** | Timeout 排程與觸發：step timeout、workflow timeout | [07-timeout-manager](07-timeout-manager.md) |

---

## 設計原則

| 原則 | 說明 |
|------|------|
| 元件邊界清晰 | 每個元件有明確的輸入 / 輸出 / 職責，可獨立測試 |
| Storage 為抽象介面 | 不綁定特定資料庫，僅定義操作語意 |
| 非同步驅動 | Scheduler 以事件驅動模式運作，不使用 blocking poll |
| 冪等操作 | 所有狀態變更操作 MUST 為冪等，支援 crash recovery |
| 水平擴展 | 多 worker 架構下透過 lease 機制分配 instance ownership |

---

## 文件索引

| 文件 | 說明 |
|------|------|
| [01-engine-api](01-engine-api.md) | Engine 對外 API 介面 |
| [02-scheduler](02-scheduler.md) | Scheduler 主迴圈與 step 排程 |
| [03-event-router](03-event-router.md) | 事件路由與訂閱管理 |
| [04-step-executor](04-step-executor.md) | Step 分派與執行邏輯 |
| [05-task-executor](05-task-executor.md) | Task backend 執行協定 |
| [06-expression-evaluator](06-expression-evaluator.md) | CEL 表達式求值引擎 |
| [07-timeout-manager](07-timeout-manager.md) | Timeout 排程與觸發 |
| [08-storage-schema](08-storage-schema.md) | Storage schema、artifact 存取控制、secret 管理、資料保留 |
| [09-engine-lifecycle](09-engine-lifecycle.md) | 引擎啟動、運行、關閉流程 |
