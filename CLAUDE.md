# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案目的

本儲存庫用於撰寫 YAML workflow engine 的規格文件（spec）。所有規格文件以 Markdown（`.md`）格式撰寫。

## 儲存庫結構

- 規格文件為 `.md` 檔案
- 無建置系統或測試執行器 — 本儲存庫僅包含文件

## project workflow

```
human -> todo.md -> decision.md -> do it in some version specific file -> todo.md to history/yyyyymmdd_hhmmss_decision.md
						|
						v
						future.md -> future/ -> future/topic.md
```

## 特殊文件

- [TODO.md](TODO.md) : 待辦事項列表，包含未完成的規格項目與待決策事項
- [DECISION.md](DECISION.md) : 整理需要由團隊決策的議題與選項，如果已完成決策就遷移到 history/yyyyymmdd_hhmmss_decision.md
- [draft/](draft/) : 歸檔後的草案文件夾，用於撰寫尚未定案的規格草案，命名為 draft/topic.md
- [FUTURE.md](FUTURE.md) : 未歸檔未來功能列表，包含已納入考量但暫不實作的功能與設計考量
- [future.md](future/) : 歸檔後的未來功能草案文件夾，用於撰寫尚未定案的未來功能設計草案，命名為 future/topic.md

## 撰寫慣例

- 語言：繁體中文撰寫說明內容；技術術語、YAML 鍵值與程式碼識別字使用英文
- YAML 範例使用 `yaml` 語法的 fenced code block
- 每份規格文件開頭須清楚說明其涵蓋範圍
- 每一次完整改動都要 commit & push，並在 commit message 中簡要說明變更內容
