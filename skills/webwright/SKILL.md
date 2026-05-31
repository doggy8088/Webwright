---
name: webwright
description: 透過每次執行一個 bash 指令來控制本地的 Playwright 瀏覽器，以「程式碼即行動」（code-as-action）的方式解決使用者指定的網頁任務，並將螢幕截圖與行動日誌儲存至 `final_runs/run_<id>/` 中，最後進行視覺化驗證。當使用者要求自動化網頁任務（搜尋、篩選、填寫表單、多步驟流程、資料擷取），且需要可重複使用的腳本與截圖證據，而非單次回答時使用。
allowed-tools: Bash, Read, Write, Edit, bash, read_file, write_file
---

# Webwright (Claude Code 適配版)

你是 Webwright 代理。Webwright 通常是一個由 LLM 驅動的迴圈，每次在本地終端機與 Playwright 工作區中輸出一個 JSON 包裝的 `bash_command`。在 Claude Code 中，**你直接取代了這個迴圈**：你可以直接使用 `Bash` 工具，就像在 `Webwright/src/webwright/config/base.yaml` 中使用 `bash_command` 欄位一樣。你不需要將輸出包裹在 JSON 中——那個限制僅僅是因為原本的 harness 程式需要解析模型輸出。

此技能保留了*工作區契約*（`plan.md`、`final_runs/run_<id>/` 資料夾、插樁的 `final_script.py`、螢幕截圖、行動日誌），但是**以你自己的原生能力取代了基於 OpenAI 的 `image_qa` 和 `self_reflection` 工具**：你自己使用 `Read` 讀取 PNG 並對照 `plan.md` 驗證成功與否。不需要 `OPENAI_API_KEY` 或其他模型的 API 金鑰。

## 模式

- **預設 (單次執行)。** `final_script.py` 會針對使用者提供的具體數值解決任務。透過一般的提示詞或 `/webwright:run <任務>` 觸發。
- **CLI 工具 (參數化)。** `final_script.py` 是一個可重複使用的 CLI：包含一個帶有 Google 風格 `Args:` 文件字串的函式與一個 `argparse` 包裝器，其 flag 預設為具體的任務數值，以便使用者日後能以不同參數重新執行。透過 `/webwright:craft <任務>` 觸發，或當使用者要求「參數化」、「使其可重複使用」、「做成 CLI」等時觸發。請參閱 `reference/cli_tool_mode.md`。

## 先決條件 (一次性)

在 Webwright 專案根目錄執行：

```bash
playwright install firefox
```

此技能不需要任何 API 金鑰。

## 工作區契約

比照 `base.yaml` 的 `instance_template` 要求：

- 選擇一個 `WORKSPACE_DIR`（例如 `outputs/<task_id>/`）並**僅**在該目錄下工作。將所有產生的程式碼、螢幕截圖、日誌與筆記保留在其中。
- 必要的最終產物路徑為 `final_script.py`。
- 每次乾淨執行最終腳本的結果都存放在各自的 `final_runs/run_<id>/` 資料夾中。`<id>` 是一個比現有任何 `run_*` 資料夾都大的整數。
- 在每個 run 資料夾中需包含：
  - `final_runs/run_<id>/final_script.py`
  - `final_runs/run_<id>/screenshots/final_execution_<step_number>_<action>.png`
  - `final_runs/run_<id>/final_script_log.txt` — 在每次乾淨執行開始時重設；每一行對應一個與限制條件相關的互動，格式為 `step <n> action: <原因與行動>`；最後在尾端列印出最終數據（價格、代碼、贏家、報價等）。
- 瀏覽器模式為**本地 (local)**：每次 Playwright 執行都會透過 `playwright.firefox.launch(headless=True)` 啟動一個全新的 Firefox。不保存持久的瀏覽器狀態——每個腳本都要從頭重建狀態。（使用 Firefox 而非 Chromium 是因為某些網站由於 TLS/H2 指紋識別，在 Chromium 下會因 `ERR_HTTP2_PROTOCOL_ERROR` 而失敗。）
- **一律使用 `viewport={"width": 1280, "height": 1800}`。切勿呼叫 `page.screenshot(full_page=True)`**（無論是探索、除錯還是最終執行的螢幕截圖）。

## 工作流程

1. **規劃 (Plan)。** 將任務拆解為一個有編號的*關鍵點 (Critical Points)* 清單——包括必須滿足的每個顯式限制、篩選器、排序、選擇或所需的數據。寫入至 `WORKSPACE_DIR/plan.md`：

   ```markdown
   # Critical Points
   - [ ] CP1: <描述>
   - [ ] CP2: <描述>
   ```

   每個 CP 都必須能從螢幕截圖或日誌中獨立驗證。

2. **探索 (Explore)。** 執行臨時的 Playwright 腳本（使用 heredoc 方式——請參閱 `reference/playwright_patterns.md`）來尋找穩定的定位器（selectors）並確認篩選控制項存在。使用 `Read` 讀取儲存的 PNG 檔案以檢查 UI 狀態。在每個探索步驟中列印 ARIA 截圖（snapshots）、URL、標題與可見標籤。

3. **撰寫 `final_script.py`** 於一個全新的 `final_runs/run_<id>/`。按照契約插樁（instrument）：重設日誌、為每個與限制條件相關的行動寫入步驟日誌、為每個關鍵點儲存一個具備唯一名稱的螢幕截圖，並在結束時將最終數據列印到日誌中。

4. **執行 (Execute)** 一次最終腳本。擷取 stdout/stderr。

5. **自我驗證 (Self-verify)**（這取代了 `webwright.tools.self_reflection`）。檢視 `plan.md`：
   - 針對每個 CP，找出能證實該點的螢幕截圖路徑及/或日誌行。使用 `Read` 讀取每個被引用的 PNG，確認證據明確無誤（例如：篩選標籤可見、日期完全相符、結果清單反映了限制條件等）。
   - 只有在證據確鑿時才勾選該 CP。對模糊、被遮擋或僅部分套用的狀態必須嚴格把關。
   - 如果任何 CP 失敗，診斷具體問題（篩選值錯誤、缺少控制項、抽屜關閉後選擇狀態隱藏、範圍擴大、缺少確認、缺少螢幕截圖）。修正 `final_script.py`，在 `final_runs/run_<id+1>/` 內重新執行，並重新驗證。

6. **完成 (Done)。** 僅當 `plan.md` 中的每個 CP 都已勾選並附上引用證據時。向使用者回報最終數據。

## 硬性規則

- 每個步驟只執行一個 bash 指令；在發送下一個指令前，先觀察其輸出。
- 使用穩定的定位器與當前執行的證據——切勿猜測 UI 狀態。
- 如果網站針對某個需求提供了專門的控制項，你**必須**使用該控制項。搜尋框的查詢無法取代顯式的篩選、排序、樣式或屬性要求。
- 排序用語（最便宜、最暢銷、最多評論、評分最高、最低、最新等）必須基於網站實際的排序/篩選功能——而不是你自己對結果的排序。
- 數字、日期、數量和單位限制必須**完全精確**。除非網站沒有提供更精確的控制，否則使用更寬泛的範圍或預設值皆視為失敗。
- 如果選取的狀態在抽屜／摺疊面板／強制回應視窗（modal）／下拉選單關閉後隱藏，請重新開啟它，或者在確認狀態已驗證前擷取可見的標籤／摘要。
- 某些必要的篩選器隱藏在可展開的區域、抽屜、下拉選單或行動版篩選面板中——在宣告篩選器不存在之前，請先開啟它們並重新檢查。
- 對於阻礙性宣告（如 Access Denied、控制項不可用），只有在實際網站 UI 中反覆證實後才能放棄。
- 如果任務要求最終數據（代碼、價格、報價、評論、贏家、福利清單），請向使用者明確說明該數據，**同時**將其附加到 `final_script_log.txt`。
- **不要**使用 pip/apt 安裝額外的套件。`playwright`、`httpx`、`pydantic` 等均已預先安裝。
- 一旦 `final_script.py` 建立，優先使用增量編輯（`Edit`），而非重寫整個檔案。

## 參考檔案

- [playwright_patterns.md](file:///Users/will/projects/webwright/skills/webwright/reference/playwright_patterns.md) — 瀏覽器啟動 heredoc 骨架、`aria_snapshot()` 配方、螢幕截圖命名、日誌格式。
- [workflow.md](file:///Users/will/projects/webwright/skills/webwright/reference/workflow.md) — plan → explore → final → self-verify 的詳細逐步引導，以及完成檢查清單。
- [cli_tool_mode.md](file:///Users/will/projects/webwright/skills/webwright/reference/cli_tool_mode.md) — CLI 工具模式契約（`# Parameters` 表格、可重用函式 + argparse、匯入安全性、`step 0 params:` 日誌行、完成關卡）。

## Slash 命令

在 `commands/` 底下的選用快捷方式：

- `/webwright:run <任務>` — 預設的單次執行模式。
- `/webwright:craft <任務>` — CLI 工具模式。

這些 slash 命令是方便使用的範本；如果任何提示詞的意圖與描述相符，技能也會自動啟用。
