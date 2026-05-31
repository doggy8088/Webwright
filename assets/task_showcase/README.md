# 任務展示頁 (Task Showcase) —— 可重複執行之網頁代理執行的儀表板

一個極簡的 Flask 應用程式，它將**可重複執行**的 odyssey 任務（答案會隨時間變動的任務——如優惠、庫存、物件列表、求職看板、天氣等）的 Webwright 執行結果整合至單一個儀表板中。

每個任務只包含兩個檔案：

```
tasks/<short_id>/
  task.json     – 中繼資料（標題、主題、週期、級別、提示詞、網站）
  report.json   – 整理後的結構化代理輸出：{sources, result.sections}
```

此 Flask 應用程式完全是通用的：模板會直接迭代 `sources` 與 `result.sections`，不需要針對特定任務編寫分支邏輯，因此要新增任務，只需將包含這兩個檔案的新資料夾放入即可。

## 執行

```bash
pip install flask
python app.py            # 啟動服務於 http://127.0.0.1:5005
```

若要直接渲染 Webwright 執行產生的 JSON 而不將其複製回此資料夾，請將應用程式指向該執行產生的任務目錄：

```bash
python app.py --tasks-dir ../../outputs/default/<run>/task_showcase/tasks
```

`task_showcase.yaml` 執行期 overlay 會在執行的工作區下寫入相同的目錄結構，接著對此渲染器進行冒煙測試。

## report.json 格式

```jsonc
{
  "sources": [
    {"name": "Slickdeals", "url": "https://slickdeals.net/", "note": "front-page best deal"}
  ],
  "result": {
    "headline": "Today's bargain roundup",
    "sections": [
      {"type": "summary", "title": "...", "body": "..."},
      {"type": "table",   "title": "...", "columns": [...], "rows": [[...], ...]},
      {"type": "list",    "title": "...", "entries": ["..."]},
      {"type": "kv",      "title": "...", "entries": [["k", "v"], ...]},
      {"type": "cards",   "title": "...", "entries": [
        {"title": "...", "subtitle": "...", "fields": [["k","v"]], "url": "..."}
      ]}
    ]
  }
}
```

內建的 6 個任務（Slickdeals、神奇寶貝 TCG、奧斯汀公寓、匹茲堡 The Ophelia、紐澳法律職缺、本週行車天氣）分別對應到 odyssey 基準測試任務，這些任務的底層網頁都會定期更新，因此非常適合透過儀表板反覆重新執行並檢視。
