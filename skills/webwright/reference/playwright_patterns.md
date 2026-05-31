# Playwright 設計模式

這些是 Webwright 代理使用的標準 heredoc 模式。在 Claude Code 中，你可以透過 `Bash` 工具直接執行它們——不需要 JSON 包裝，不需要繁瑣的跳脫字元，每次只執行一個 bash 指令。

## 瀏覽器啟動骨架 (本地模式)

Webwright 技能使用 **Playwright Firefox** 作為預設引擎。某些網站（例如 cars.com 或其他受到 Akamai 保護的網站）會因為 TLS/H2 指紋識別而以 `ERR_HTTP2_PROTOCOL_ERROR` 拒絕 Playwright Chromium，但在 Firefox 下能正常載入。在執行第一個任務前，請先執行一次 `playwright install firefox`。

```bash
python - <<'PY'
import asyncio
import os
from pathlib import Path

from playwright.async_api import async_playwright

WORKSPACE = Path(os.environ.get("WORKSPACE_DIR", "."))
SCREENSHOTS = WORKSPACE / "screenshots"
SCREENSHOTS.mkdir(parents=True, exist_ok=True)

async def main():
    async with async_playwright() as playwright:
        browser = await playwright.firefox.launch(headless=True)
        context = await browser.new_context(viewport={"width": 1280, "height": 1800})
        page = await context.new_page()

        await page.goto("<START_URL>", wait_until="domcontentloaded")
        await page.screenshot(path=str(SCREENSHOTS / "explore_1_start.png"))

        print("URL:", page.url)
        print("TITLE:", await page.title())

        # 用 ARIA 截圖檢查你關心的區域
        snapshot = await page.locator("body").aria_snapshot()
        print("ARIA:", snapshot)

        await browser.close()

asyncio.run(main())
PY
```

規則：

- **一律**設定 `viewport={"width": 1280, "height": 1800}`。
- **切勿**呼叫 `page.screenshot(full_page=True)` —— 無論是探索、除錯還是最終執行的螢幕截圖。
- 每次 Playwright 執行都是全新的：從起始 URL 開始導覽、重新套用篩選器，並在程式碼中重建狀態。不存在持久的 session。

## 透過角色與名稱定位元素

```python
await page.get_by_role("button", name="Filters").click()
await asyncio.sleep(1)

# 取得該控制項的「父節點」截圖以檢視同層元素／選項
panel = page.get_by_role("button", name="Filters").first.locator("..")
print(await panel.aria_snapshot())

await page.get_by_role("checkbox", name="BMW").check()
await asyncio.sleep(1)
```

如果選取的狀態在抽屜／下拉選單關閉後隱藏，請在擷取驗證螢幕截圖前重新開啟它。

## 優先選擇互動式填寫表單，而非深層連結 URL

當任務需要將搜尋參數化（地點、日期、篩選器、查詢字串）時，**請在頁面上以互動方式操作表單**，而不是直接建構一個將參數嵌入在查詢字串（query string）中的深層連結（deep-link）URL。深層連結雖然對代理探索的某個特定案例很方便，但作為 CLI 的介面時卻非常脆弱：

- 網站會靜默丟棄無法解析的參數，導致下游欄位留空。
- URL 解析器會因語系、A/B 測試分流與登入狀態而異。
- 某個輸入值組合能用的深層連結，並不能保證另一個輸入值組合也能正常載入。

透過模擬人類點擊的控制項進行互動式填寫，是應對輸入變更最可靠的策略。請將其作為最終腳本中的**主要**路徑；僅將深層連結視為一種機會主義的快捷方式，並且在之後務必驗證表單狀態，當任何欄位為空或錯誤時，退回使用互動式填寫。

```python
# 導覽之後，讀取可見的表單狀態並做決定。
form_state = await page.locator("input[aria-label]").evaluate_all(
    "els => els.map(e => ({label: e.getAttribute('aria-label'), "
    "value: e.value, hidden: e.offsetParent === null}))"
)
if not form_is_fully_populated(form_state, expected):
    # 輸入到每個欄位中、從建議清單中選擇、透過共享的強制回應視窗填寫群組輸入
    # （在同層元素之間按 Tab 以保持強制回應視窗開啟），然後點擊提交控制項。
    await fill_form_interactively(page, expected)
```

互動式路徑的指引：

- 使用 `get_by_role` / `aria-label` 定位器，而不是脆弱的 CSS class。
- 輸入數值，等待建議清單彈出，然後點擊文字包含該輸入項標準標記的選項。
- 對於在單一強制回應視窗中呈現的成對欄位（日期範圍選擇器、步進器群組等），**僅開啟該視窗一次**，並在欄位之間按 `Tab` 切換，而不是分別點擊每個輸入框——在視窗開啟時點擊第二個輸入框往往會被該視窗本身的遮罩層阻擋。
- 填寫完成後，點擊明確的提交控制項，而不要依賴自動提交。
- 在繼續進行結果提取之前，重新讀取表單狀態並斷言（assert）每個檢查點（CP1..CPn）。

## 最終腳本的插樁要求

`final_runs/run_<id>/final_script.py` 必須：

- 寫入至 `final_runs/run_<id>/screenshots/final_execution_<step>_<action>.png`，
- 重設並附加日誌至 `final_runs/run_<id>/final_script_log.txt`，
- 在日誌的最末端下列印出最終數據。

```python
import asyncio, os
from pathlib import Path
from playwright.async_api import async_playwright

RUN_DIR = Path(__file__).parent
SCREENSHOTS = RUN_DIR / "screenshots"
SCREENSHOTS.mkdir(parents=True, exist_ok=True)
LOG = RUN_DIR / "final_script_log.txt"
LOG.write_text("")  # 重設

def log(step: int, msg: str) -> None:
    line = f"step {step} action: {msg}\n"
    LOG.open("a").write(line)
    print(line, end="")

async def main():
    async with async_playwright() as playwright:
        browser = await playwright.firefox.launch(headless=True)
        context = await browser.new_context(viewport={"width": 1280, "height": 1800})
        page = await context.new_page()

        await page.goto("<START_URL>", wait_until="domcontentloaded")
        await page.screenshot(path=str(SCREENSHOTS / "final_execution_1_open_start_page.png"))
        log(1, "open start page")

        # ... 套用 CP1, 螢幕截圖, log ...
        # ... 套用 CP2, 螢幕截圖, log ...

        # 執行結束：在 UI 上和日誌中擷取最終數據
        final_value = "<提取的價格 / 代碼 / 贏家>"
        with LOG.open("a") as f:
            f.write(f"\nFINAL_RESPONSE: {final_value}\n")

        await browser.close()

asyncio.run(main())
```

## 檢查指令

```bash
# 最新一次執行的目錄樹與日誌
ls -R final_runs/run_<id>
cat final_runs/run_<id>/final_script_log.txt

# 快速讀取檔案
sed -n '1,220p' final_runs/run_<id>/final_script.py
```

若要進行視覺化檢查，請直接使用 `Read` 工具讀取 `final_runs/run_<id>/screenshots/` 底下的個別 PNG 檔案，而不是呼叫外部的圖像問答服務。
