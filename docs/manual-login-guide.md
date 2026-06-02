# Webwright 預先手動登入網站實作指南 (Manual Login Guide)

本指南旨在幫助你了解如何利用 Webwright 內建的持久化瀏覽器工具，進行 **預先人工手動登入**（例如繞過 2FA、圖形驗證碼 Captcha、簡訊認證或複雜的企業 SSO 單一登入），並將該登入狀態（Session）交由 Webwright 代理（Agent）接手自動化。

由於所有指令皆使用 **[uv](https://github.com/astral-sh/uv)** 套件管理器，請確保你已安裝 `uv` 並在專案目錄下完成 `uv sync`。

---

## 💡 核心機制與優勢

Webwright 提供了一個 `persistent_local_browser` 工具，能啟動一個獨立的 Chromium 子程序，並與指定的 `userDataDir`（使用者資料目錄）綁定：
1. **人工手動介入：** 以「看得見的實體瀏覽器 (Headed Mode)」啟動它，讓你進行登入與驗證。
2. **狀態持久化：** 所有登入成功後的 Cookies、LocalStorage 與 Session 都會寫入到本機工作區的指定資料夾中。
3. **無縫接手：** Webwright 代理執行時會透過 CDP (Chrome DevTools Protocol) 連上同一個瀏覽器實體，直接以「已登入」的身分接續操作。
4. **狀態重用：** 任務結束時可選擇只關閉瀏覽器而不刪除資料，下次執行同網站任務時即可自動保持登入狀態，不需重複登入！

---

## 🛠️ 實作四大步驟

### 步驟 1：啟動實體「有頭」瀏覽器

在終端機中執行以下指令，在特定的工作區（以 `outputs/my_login_session` 為例）啟動一個看得見的 Chromium 瀏覽器：

```bash
uv run python -m webwright.tools.persistent_local_browser \
    --workspace-dir outputs/my_login_session \
    create \
    --out .lb_session.json \
    --no-headless
```

#### 📌 參數說明：
* `--workspace-dir`：此 Session 所有檔案存放的目錄。
* `--out`：輸出的連線快照 JSON 檔名（預設為 `.lb_session.json`）。
* `--no-headless`：**關鍵旗標**。此參數會強制打開實體 Chromium 瀏覽器視窗，讓它顯示在桌面上。

執行後，你會在螢幕上看到一個乾淨的 Chromium 視窗被開啟。

---

### 步驟 2：在視窗中手動完成登入與驗證

在剛剛彈出的實體瀏覽器視窗中：
1. **導航至目標網站**：在網址列輸入你要登入的網站（例如：`https://github.com/login` 或是你的公司後台、電子信箱等）。
2. **手動登入**：手動輸入你的帳號、密碼，並完成所有安全驗證（2FA、手機簡訊、滑動驗證碼等）。
3. **確認登入成功**：成功進入系統主頁面（Dashboard）後，**請讓該瀏覽器視窗保持在背景開啟（不要關閉它）**。

*(此時，你所有的登入憑證都已經被寫入到了 `outputs/my_login_session/.lb_user_data` 中)*

---

### 步驟 3：使用持久化設定檔讓 Webwright 接手

現在你的瀏覽器已是登入狀態。我們透過載入 `-c persistent_browser.yaml` 這個設定檔，指示 Webwright 代理直接連上剛剛啟動的瀏覽器：

```bash
uv run python -m webwright.run.cli \
    -c persistent_browser.yaml -c model_openai.yaml \
    -t "進入已登入的後台，點擊 OOO 選單並幫我擷取最近五筆交易紀錄" \
    --start-url https://example.com/dashboard \
    --task-id my_logged_in_task \
    -o outputs/my_login_session
```

#### 📌 為什麼這行指令可以接手？
* **`-c persistent_browser.yaml`**：這個設定檔會蓋過預設的全新瀏覽器設定，指示主代理自動讀取剛才在 `outputs/my_login_session` 下建立的 `.lb_session.json` 連線資訊。
* **直接接手操作**：主代理會透過 CDP 發送 async Playwright 指令直接在那個彈出的視窗中操作，它打開網址時便已經是登入完成的狀態。你會親眼看著網頁中的按鈕被自動點擊！

---

### 步驟 4：安全釋放（關閉）瀏覽器

當 Webwright 代理回報 Done，或者你想中止測試時，你必須關閉這個 Chromium 視窗並終止子程序，以免在背景殘留殭屍程序。

#### 情境 A：下次還想重用登入狀態 (推薦)
如果你希望下一次跑任務時**不需要**再次手動登入，請在釋放時帶上 `--no-delete-user-data` 旗標：

```bash
uv run python -m webwright.tools.persistent_local_browser \
    --workspace-dir outputs/my_login_session \
    release \
    --session-file .lb_session.json \
    --no-delete-user-data
```
這會關閉瀏覽器，但把 Cookies 與 Session 保留。下次你要用同個帳號跑任務時，只要**直接重複「步驟 1」**，一開瀏覽器就會是登入好的狀態！

#### 情境 B：徹底清除登入狀態與所有隱私資料
如果你是在公用電腦，或者想徹底清空所有快取、Cookies 與 User Data：

```bash
uv run python -m webwright.tools.persistent_local_browser \
    --workspace-dir outputs/my_login_session \
    release \
    --session-file .lb_session.json \
    --delete-user-data \
    --delete-file
```
這會自動刪除 `.lb_session.json` 以及整個 `.lb_user_data` 資料夾，不留下任何帳密或 Cookie 隱私。

---

## ⚠️ 常見問答與注意事項

1. **Q：執行步驟 1 時，為什麼沒有彈出瀏覽器視窗？**
   * **A**：請檢查你是否有設定 `DISPLAY` 環境變數（在 Linux 上），或確認你在本機上有桌面 GUI。在 headless 伺服器上是無法使用 `--no-headless` 的。
   
2. **Q：我可以把這個手動登入做成自動化 CLI 工具嗎？**
   * **A**：可以。你可以將 Webwright 產出的 `final_script.py` 改寫，使其預設載入指定的 `userDataDir`。只要你沒有徹底 release 刪除 user data，該 CLI 腳本在獨立執行時也會自動讀取保存的 Cookie 來避開登入步驟。
