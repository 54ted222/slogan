# Artifact Workspace 模型決策紀錄

> 決策日期：2026-04-02

## 背景

Artifact 系統從 URI 模型調整為 workspace 目錄模型。引擎負責實體化檔案到暫存目錄，task/agent 直接操作檔案；遠端 task 透過 CRUD API 操作。

草案文件：[draft/artifact-workspace.md](../draft/artifact-workspace.md)

---

## 決策結果

### N1. Workspace 目錄是否支援自訂結構

- [x] **B** — 支援子目錄：artifact 可指定 `workspace_path` 放在子目錄中
- [ ] ~~**A** — 扁平結構：所有 artifact 直接放在 workspace 根目錄~~

### N2. 大檔案是否支援 lazy 下載

- [x] **A** — 所有 artifact 在 instance 建立時全部下載
- [ ] ~~**B** — 支援 `lazy: true` 標記，首次存取時才下載~~

### N3. 本機 source 是否支援目錄（不僅是檔案）

- [x] **B** — 支援目錄（遞迴複製或 mount）
- [ ] ~~**A** — 僅支援單一檔案~~

### N4. 遠端 task 的 artifact API token 注入方式

- [x] **C** — 兩者皆支援（task input 注入 + HTTP header 傳遞）
- [ ] ~~**A** — 僅引擎自動注入到 task input~~
- [ ] ~~**B** — 僅透過 HTTP header 傳遞~~

### N5. 同一 artifact 的並行寫入處理

- [x] **C** — 由 artifact 設定決定（`concurrency: lock | last_write_wins`）
- [ ] ~~**A** — 引擎以鎖機制防止同一 artifact 同時被多個 step 寫入~~
- [ ] ~~**B** — 允許並行寫入，last-write-wins~~
