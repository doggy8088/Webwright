---
description: 透過將網頁任務參數化，來打造一個可重複使用的 Webwright CLI 工具。
argument-hint: <包含具體數值的自然語言網頁任務>
---

你目前正以 **CLI 工具模式** 運作 Webwright 代理。請先閱讀 `webwright` 技能（此 `commands/` 資料夾的父目錄）中的 `SKILL.md`，以及旁邊的 `reference/cli_tool_mode.md`，然後將以下任務參數化，以便產生的 `final_script.py` 稍後能使用不同的參數值重新執行：

$ARGUMENTS

步驟：

1. **識別參數。** 提取使用者可能會合理變更的每一項需求（搜尋字詞、地點、日期、篩選值等）。對於網站而言真正固定的項目（起始 URL、網站名稱、定位器策略）則**不是**參數——請保持寫死（hard-coded）狀態。

2. **撰寫 `plan.md`。** 新增一個 `# Parameters` 表格，欄位包括 `name | type | source phrase | default | allowed/format`，以及一般的 `# Critical Points` 檢查清單。預設值必須等於具體的任務數值，以便在無參數執行 `python final_script.py` 時能重現該任務。

3. **在全新的 `final_runs/run_<id>/` 中撰寫 `final_script.py`：**
   - 一個以任務領域命名的可重複使用函式（例如：`def search_<domain>(arg_a, arg_b, ...): ...`）。
   - Google 風格的文件字串，包含摘要、完整的 `Args:` 區塊（名稱、型態、意義、格式／單位、預設值）和 `Returns:`。
   - 在 `if __name__ == "__main__":` 底下實作 `argparse` CLI，其 flag 必須與函式參數完全對應，且預設值等於具體任務的數值。
   - **匯入時不得產生副作用 (Side-effect-free)** — 在模組頂層不得啟動瀏覽器、不得進行網路呼叫、亦不得寫入檔案。
   - 重設後的首行日誌必須是 `step 0 params: <name>=<value> <name>=<value> ...`。
   - 使用與預設模式相同的插樁：檢視區 1280×1800、無頭本地 Firefox、無 `full_page=True`、螢幕截圖與最終數據儲存在該 run 資料夾中。

4. **在無參數下重現任務。** 執行 `python final_runs/run_<id>/final_script.py` 並確認其端到端執行成功。

5. **匯入安全性冒煙測試 (Smoke test)。** 在獨立的 Python 行程中載入該模組，確認不會啟動瀏覽器，且可重複使用的函式是可以被匯入的。

6. **自我驗證。** 對照儲存的螢幕截圖與行動日誌，自我驗證每個關鍵點（取代 `self_reflection`）。如果任何 CP 失敗，請診斷並修正腳本（保留 CLI 形狀），在 `final_runs/run_<id+1>/` 內重新執行，然後重新驗證。

7. **向使用者展示 `--help`。** 最後，執行 `python final_runs/run_<id>/final_script.py --help`，並向使用者回報最終數據與說明文字，讓使用者知道如何以不同的參數再次呼叫此工具。

完整契約請參考 [cli_tool_mode.md](file:///Users/will/projects/webwright/skills/webwright/reference/cli_tool_mode.md)，Playwright 骨架請參考 [playwright_patterns.md](file:///Users/will/projects/webwright/skills/webwright/reference/playwright_patterns.md)。
