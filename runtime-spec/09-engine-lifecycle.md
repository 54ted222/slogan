# 09 — Engine Lifecycle

本文件定義引擎的啟動、運行與關閉流程。

---

## 啟動流程

引擎啟動 MUST 依以下順序執行：

```
1. 初始化 Storage 連線
     ↓
2. 載入 Secret definitions（解密至記憶體）
     ↓
3. 載入環境變數（OS env + .env）
     ↓
4. 載入 Trigger subscriptions（PUBLISHED workflow definitions）
     ↓
5. 重建 Timeout schedules（非 terminal instances）
     ↓
6. 執行 Crash Recovery（恢復中斷的 instances）
     ↓
7. 啟動 Scheduler 主迴圈
     ↓
8. 啟動 Timeout Manager 監控
     ↓
9. 開放 API 接口（接受外部請求）
```

### 啟動順序保證

- API 接口 MUST 在所有內部元件初始化完成後才開放
- Crash Recovery MUST 在 Scheduler 主迴圈啟動前完成
- Secret 解密失敗 → 引擎啟動失敗（MUST log 錯誤的 secret name）
- Storage 連線失敗 → 引擎啟動失敗

### Crash Recovery

詳見 [dsl-spec/v2/15-runtime-persistence](../dsl-spec/v2/15-runtime-persistence.md) 的 Recovery 機制。

引擎重啟時：

1. 查詢所有非 terminal 狀態的 instances（CREATED / RUNNING / WAITING）
2. 對每個 instance：
   a. CREATED → 轉為 RUNNING，開始執行
   b. RUNNING → 檢查各 step instance 狀態，依 `execution.policy` 決定 recovery 行為
   c. WAITING → 重建 wait_subscription，繼續等待
3. 重建所有未到期的 timeout_schedules
4. 對已過期的 timeout_schedules（`fires_at` <= 當前時間）→ 立即觸發處理

---

## 運行狀態

引擎運行期間維護以下記憶體狀態：

| 狀態 | 說明 |
|------|------|
| Secret 快取 | 解密後的 secret key-value pairs |
| 環境變數快取 | OS env + .env 的合併結果 |
| Trigger subscriptions | PUBLISHED definitions 的事件訂閱 |
| CEL AST 快取 | 編譯過的 CEL 表達式（按 definition 快取） |
| Instance leases | 當前 worker 持有的 instance leases |

記憶體狀態僅為快取，crash 後從 Storage 恢復。DB 為唯一 source of truth。

---

## 關閉流程（Graceful Shutdown）

引擎收到關閉信號時 SHOULD 執行 graceful shutdown：

```
1. 停止接受新的 API 請求
     ↓
2. 停止接受新的 trigger 事件
     ↓
3. 等待進行中的 step 執行完成（受 shutdown timeout 限制）
     ↓
4. 持久化所有待寫入的狀態
     ↓
5. 釋放所有 instance leases
     ↓
6. 關閉 Storage 連線
```

### Shutdown Timeout

- 引擎 SHOULD 提供可設定的 shutdown timeout（建議預設 30s）
- 超過 timeout 後，仍在執行的 step 不做特殊處理（下次啟動時 crash recovery 會處理）
- 進行中的 bash/stdio tasks 收到 SIGTERM

### 強制關閉

- 第二次收到關閉信號 → 立即停止
- Instance leases 未釋放時，其他 workers 在 lease 過期後可重新取得
