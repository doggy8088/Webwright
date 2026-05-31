# 工作流程

針對 Claude Code 進行調整之 Webwright 六步驟迴圈的詳細展開。原本的迴圈依賴 `webwright.tools.image_qa` 進行視覺 QA，以及依賴 `webwright.tools.self_reflection` 來判定最終結果。兩者在此處都被你自己的原生能力所取代（使用 `Read` 讀取 PNG 檔案 + 對照 `plan.md` 進行推理）。不需要任何 `OPENAI_API_KEY`。

## 1. 規劃 (Plan)

將任務拆解為關鍵點（Critical Points, CP），並寫入 `WORKSPACE_DIR/plan.md`：

```markdown
# Task
<原封不動的任務描述>

# Critical Points
- [ ] CP1: <限制條件 / 篩選器 / 排序 / 選擇 / 所需數據>
- [ ] CP2: ...
```

關鍵點 (CP) 規則：

- 每一項可獨立驗證的要求對應一個 CP。
- 數字、日期、數量和單位 CP 必須精確無誤。
- 排序 CP（最便宜、最暢銷、評分最高等）必須引用網站實際的排序／篩選控制項。
- 如果任務要求最終數據，將其設為獨立的 CP（例如 `CP5: 記錄顯示的最便宜經濟艙票價`）。

## 2. 探索 (Explore)

目標：尋找穩定的定位器（selectors）、確認所有必要的篩選控制項存在，並找出如何為每個 CP 擷取證據。

- 在 `WORKSPACE_DIR/` 內執行臨時的 Playwright 腳本（請參閱 [playwright_patterns.md](file:///Users/will/projects/webwright/skills/webwright/reference/playwright_patterns.md)）。將臨時 PNG 螢幕截圖儲存在 `WORKSPACE_DIR/screenshots/` 中（與 `final_runs/` 分開）。
- 在每個步驟列印感興趣區域的 URL、標題與 `aria_snapshot()`。
- 當 ARIA 證據不明確時，使用 `Read` 讀取儲存的 PNG 以確認 UI 狀態。
- 如果某個篩選器看起來不可用，請在判定其不存在前，先展開抽屜／摺疊面板／行動版篩選面板並重新檢查。
- 搜尋框的查詢絕不能用作專用篩選控制項的替代品。

## 3. 撰寫 `final_script.py`

建立一個全新的 `final_runs/run_<id>/`（使用比現存任何 `run_*` 大的下一個整數），並將 `final_script.py` 存放在其中。按照 [playwright_patterns.md](file:///Users/will/projects/webwright/skills/webwright/reference/playwright_patterns.md) 的要求插樁：

- 檢視區 1280×1800、無頭本地 Firefox、無 `full_page=True`；
- 每個 CP 對應一個 `final_execution_<step>_<action>.png` 螢幕截圖；
- 每個與限制條件相關的互動對應一行 `step <n> action: <原因與行動>` 日誌；
- 結束時將最終數據列印到 `final_script_log.txt` 中。

每個螢幕截圖都應該對應到 `plan.md` 中的一個 CP，以便於驗證。

## 4. 執行 (Execute)

執行一次腳本。如果崩潰，在同一個 run 資料夾中修正並重新執行——但如果部分執行已經產生了與修正後的流程不符的螢幕截圖，請將其刪除，以確保該 run 資料夾反映的是單次乾淨的執行。

## 5. 自我驗證 (Self-verify) (取代 `self_reflection`)

對於 `plan.md` 中的每個 CP：

1. 找出能提供證據的螢幕截圖及／或日誌行。
2. 使用 `Read` 讀取每個被引用的 PNG。
3. 確認證據**明確無誤**：
   - 篩選標籤／選擇狀態已明顯套用（沒有隱藏在已關閉的抽屜後）；
   - 數值／日期完全相符（沒有被放寬）；
   - 排序已透過網站的控制項套用（而非僅從結果順序暗示）；
   - 必要的提交／搜尋／套用動作已明顯執行；
   - 最終數據清晰顯示。
4. 只有在證據確鑿時才勾選該 CP。對部分套用、被遮擋或模糊的狀態必須嚴格審查。

若有任何 CP 失敗，請診斷出**具體**問題——篩選值錯誤、缺少控制項、隱藏的標籤、放寬的範圍、缺少確認、缺少螢幕截圖等。修正 `final_script.py`，並在 `final_runs/run_<id+1>/` 內重新執行，然後對照 `plan.md` 重新驗證。

若已證實套用了正確的篩選器，空結果集（empty result sets）是可以接受的。

## 6. 完成 (Done)

僅當符合以下**所有**條件時才停止：

1. `plan.md` 存在且每個 CP 均已列入檢查清單。
2. `final_runs/run_<id>/final_script.py` 從頭到尾乾淨地執行，並產生了 `final_script_log.txt` 以及所有 CP 螢幕截圖。
3. 每個 CP 都已勾選，並附上引用的螢幕截圖及／或日誌行。
4. 最終數據（如果任務有要求）已原封不動地回報給使用者，並且也存在於 `final_script_log.txt` 中。
5. `ls -R final_runs/run_<id>` 和 `cat final_runs/run_<id>/final_script_log.txt` 顯示預期的產物。

如果其中有任何一項為假，請勿宣告完成——請診斷、修正並在新的 `run_<id+1>/` 中重新執行。
