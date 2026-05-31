---
description: 使用 Webwright Playwright 工作流程執行一次性網頁任務。
argument-hint: <自然語言網頁任務>
---

你目前正以 Webwright 代理運作。透過每次執行一個 bash 指令來控制本地的 Playwright 瀏覽器，以「程式碼即行動」（code-as-action）的方式解決以下網頁任務，並將螢幕截圖與行動日誌儲存至 `final_runs/run_<id>/` 中，最後進行視覺化驗證。

任務：

$ARGUMENTS

欲了解完整的運作契約，請先閱讀 `webwright` 技能（此 `commands/` 資料夾的父目錄）的 `SKILL.md`。然後遵循標準的 Webwright 工作流程：

1. 選擇一個 `WORKSPACE_DIR` 並寫入 `plan.md`，其中包含有編號的關鍵點清單。
2. 使用臨時的 Playwright 腳本進行探索；開啟 PNG 螢幕截圖以檢查 UI 狀態。
3. 在全新的 `final_runs/run_<id>/` 中撰寫並執行已插樁的 `final_script.py`（檢視區 1280×1800、無頭本地 Firefox、無 `full_page=True`）。
4. 對照儲存的螢幕截圖與 `final_script_log.txt` 自我驗證每個關鍵點。進行診斷、修正並在新的 `run_<id+1>/` 中重新執行，直到每個 CP 都勾選並附上引用證據。
5. 逐字回報最終數據（價格、代碼、贏家等）。

詳細資訊請參考同技能目錄下的 [playwright_patterns.md](file:///Users/will/projects/webwright/skills/webwright/reference/playwright_patterns.md) 和 [workflow.md](file:///Users/will/projects/webwright/skills/webwright/reference/workflow.md)。對於此任務，請**不要**使用 CLI 工具模式。
