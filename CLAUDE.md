# CLAUDE.md — ai-image-prompt-studio（AI 繪圖提示詞工廠）

「AI 繪圖提示詞工廠」——單檔前端工具，把一句話的中文畫面概念，優化成適合圖片生成模型使用的提示詞：Google Nano Banana Pro／Gemini、ChatGPT／DALL-E、Midjourney，以及一個不特定模型的通用格式。填一次概念與風格設定，四個目標模型分頁共用；可以純前端組成關鍵字草稿，也可以串接使用者自己的語言模型取得中英對照的正式提示詞。

與 `行銷內容工具/ai-prompt-generator/`（影音框架＋TAG／APE／CO-STAR 文字提示詞產生器）是姊妹專案，同一套 BYOK 呼叫 LLM 的手法、同一種「模板組裝＋可選 AI 優化」兩段式互動，服務對象換成圖片生成提示詞。

## 架構

單一 `index.html`：內嵌 CSS/JS、無外部資源、無建置步驟。視覺主題是深色「暗房／畫布」風格（`--bg #170b12` + 圓點網格背景 + 洋紅色 `--accent #ec4899`），與 `ai-prompt-generator`（青色）、`Prompt`（琥珀色）刻意做出區隔，方便一眼分辨是哪個工具；AI 優化面板用 teal（`--teal #2dd4bf`）區隔於主色。

- **共用欄位 + 多目標模型分頁**是這個工具與 `ai-prompt-generator` 最大的結構差異：`ai-prompt-generator` 的 `FRAMEWORKS` 是每個分頁各自獨立的欄位，本工具則是**單一組共用欄位**（`state.fields`：概念描述／風格／構圖／光線／長寬比／畫質強化詞（可複選）／常見負面提示詞（可複選，`negativeTags`）／其他負面提示詞自由文字（`negative`））搭配四個只改變「組裝格式」的目標模型分頁（`TARGETS`：`gemini`/`chatgpt`/`midjourney`/`generic`）。使用者填一次欄位，四個分頁共用，只有下方「組成提示詞」「送給 AI 優化」的輸出格式規則依分頁不同。
- 風格／構圖／光線／畫質強化詞／常見負面提示詞的下拉與複選選項（`STYLE_OPTIONS`／`COMPOSITION_OPTIONS`／`LIGHTING_OPTIONS`／`QUALITY_OPTIONS`／`NEGATIVE_OPTIONS`）每個選項都同時定義中文標籤（`zh`，UI 顯示）與英文提示詞片語（`en`，組裝時使用）；新增選項時兩者都要補。`renderCheckboxGroup(elId, options, fieldKey)` 是複選類欄位（畫質強化詞／常見負面提示詞）共用的渲染／勾選邏輯，新增另一組複選欄位時直接重用這個函式，不要複製貼上。
- `STYLE_OPTIONS` 裡有四個以**電商銷售為目的**的選項（`ecommerce-white`/`ecommerce-lifestyle`/`ecommerce-flatlay`/`ecommerce-banner`），這幾個選項的 `en` 片語本身就包含「optimized for online store listing to drive sales」之類的銷售導向描述，不是單純視覺風格——新增類似「以特定用途為目的」的風格選項時，維持這個做法（把用途/目的寫進 `en` 片語本身），不需要另外開一個欄位維度。
- `COLOR_TONE_OPTIONS`（色調／色彩風格）是**刻意獨立於 `STYLE_OPTIONS` 的第二個維度**：`STYLE_OPTIONS` 管的是攝影/繪畫技法（寫實攝影、電影感、2D插畫…），`COLOR_TONE_OPTIONS` 管的是整體色彩氛圍（暖色調、黃白調、莫蘭迪色系、黑白單色…），兩者可自由搭配、不互斥。新增這類「正交的視覺維度」時比照這個模式開新的 `_OPTIONS` 陣列＋新的 `<select>`，不要塞進既有維度的選項列表裡稀釋掉原本的分類意義。`buildTagsEn()`/`buildTagsZh()` 已把 `colortone` 併入標籤陣列，`state.fields.colortone` 走跟 `style`/`composition`/`lighting` 一樣的單選欄位模式（不是像 `quality`/`negativeTags` 那樣的複選陣列）。
- 「負面提示詞」是**複選勾選（`negativeTags`，常見項目）+ 自由文字（`negative`，其他自訂內容）** 兩段合併，由 `buildNegativeList(v)` 統一組合成 `{en:[], zh:[]}`（`negative` 自由文字視為未翻譯內容，同時併入 `en`／`zh` 兩個陣列——這是刻意的權宜作法，因為使用者輸入的自由文字語言不固定，client-only 組裝階段沒有翻譯能力）；`assembleForTarget()`／`aiPromptForTarget()` 都呼叫這個函式取得負面提示詞內容，不要各自重新拼接。
- **從商品網址抓取重點**（`fetchUrlBtn`，位於「概念描述」欄位上方）：兩段式流程——(1) 用免金鑰、支援 CORS 的第三方網頁擷取服務 `r.jina.ai`（`READER_PREFIX = 'https://r.jina.ai/'`，直接把目標網址接在後面即可，不需 `encodeURIComponent`）把網頁轉成純文字，這一步**不需要 LLM**；(2) 把擷取到的純文字（截斷前 6000 字元避免過度消耗 token）丟給使用者設定的 BYOK LLM，請它整理成商品名稱＋賣點、輸出一段適合當概念描述的中文文字，成功後**詢問是否覆蓋**既有的「概念描述」內容（非空才問）。**這是本工具唯一會把使用者資料送到「非使用者自選 LLM 服務商」的第三方服務**，manual.html／index.html 的隱私與警語區塊都要提到這件事。`r.jina.ai` 已實測會依請求的 `Origin` header 動態回傳 `Access-Control-Allow-Origin`（見 2026-08-12 的 curl 驗證），瀏覽器 `fetch()` 可直接呼叫；未帶金鑰時有速率限制（約 20 次/分鐘），一般互動使用足夠，不需要另外申請 API 金鑰。抓取失敗（403／需要登入／JS 動態渲染導致內容過短）時走 fail-soft：顯示清楚的錯誤訊息並建議使用者手動填寫概念描述，不影響其他功能。`state.fields.productUrl` 只是持久化這個輸入框的值本身，不會儲存抓取到的原始網頁全文。**`r.jina.ai` 回傳非 2xx 時，錯誤 body 是結構化 JSON**（`{message, readableMessage, code, ...}`，例如網域無法解析會回 422 + `readableMessage: "SubmittedDataMalformedError: Domain '...' could not be resolved"`）——一定要解析這個 JSON 顯示 `readableMessage`/`message`，不要只顯示 HTTP 狀態碼，否則使用者完全不知道實際原因（2026-08-12 修過這個坑：一開始只顯示「狀態碼 422」，使用者回報看不懂）。

**已知還會遇到的失敗模式**（真實案例：使用者拿政府機關網站 `ojt.wda.gov.tw` 測試時遇到，實測驗證過）：
1. `r.jina.ai` 本身回非 2xx，例如 `422 AssertionFailureError: Failed to goto ...: net::ERR_HTTP_RESPONSE_CODE_FAILURE`——這是 `r.jina.ai` 內部無頭瀏覽器導航到目標網站失敗，不是我們的請求格式錯，已被上面那條規則涵蓋（會顯示 `readableMessage`）。
2. **`r.jina.ai` 回 HTTP 200，但內容其實是「目標網站本身出錯」的說明頁**（例如目標站 502/504），內容裡會固定出現一行 `Warning: Target URL returned error <code>: <text>`——這種情況原本會被 `pageText.length >= 30` 的檢查誤判成「抓到正常內容」，直接把錯誤頁餵給 LLM 產生垃圾結果，是個真的 bug，已修：`runUrlFetch()` 現在用 `/Warning: Target URL returned error (\d+)/` 這個 regex 偵測並視為失敗。
3. 上述兩種失敗都**可能是暫時性的**（同一個政府機關網址實測過，第一次是網路層失敗、第二次改用不同 header 重試變成 504——顯示該站對自動化流量不穩定，不是永久封鎖），所以失敗訊息下方一律會附一顆「🔄 重試」按鈕（`renderUrlFetchError()`，重用同一顆 `runUrlFetch()` 函式），不需要使用者重新貼網址。這是刻意不做「自動重試」而是「使用者手動點重試」的設計——因為失敗有時要等網站自己恢復，自動連續打好幾次沒有意義，也可能撞到 `r.jina.ai` 的匿名速率限制（約 20 次/分鐘）。
- 狀態存 `localStorage`（key: `imgPromptState`）：`{activeTarget, fields:{concept,style,composition,lighting,ratio,quality[],negative}, assembled:{gemini,chatgpt,midjourney,generic}, aiOutput:{...}}`——欄位是單一物件（不像 `ai-prompt-generator` 按分頁各自存一份），但組成結果與 AI 結果仍按分頁各自保留，切換分頁不會互相覆蓋。
- **核心互動模型**（與 `ai-prompt-generator` 相同精神）：「🔧 組成提示詞」是純前端字串組裝（不需金鑰、不連網）——概念描述**保留原始語言、不會被翻譯**，只有風格／構圖／光線／畫質標籤是已經對照好的英文片語；「🚀 送給 AI 優化」把概念與風格設定送給 BYOK LLM，要求輸出「中文說明＋分隔線＋English Prompt」的中英對照結果，這一步才會真的把概念翻譯成英文。Midjourney 分頁的 `aiPromptForTarget()` 額外要求逗號分隔關鍵詞＋`--ar --v` 參數語法；其餘三個分頁要求自然語言描述句。
- **BYOK AI 串接**：與 `ai-prompt-generator/index.html`、`Prompt/index.html` 同一套 `callLLM()` 模式（改動時互相參照）——全部走瀏覽器直連 `fetch()`：Claude 需 `anthropic-dangerous-direct-browser-access: true` header；Gemini 金鑰放 `x-goog-api-key` header；OpenAI/OpenRouter 用 Bearer。設定（provider/model/apiKey）存 `localStorage`（key: `imgPromptApiConfig`）——**金鑰只落在使用者本機瀏覽器，絕不可寫進程式碼**。逾時 180 秒；429/500/503/529 自動重試最多 2 次（間隔 8、16 秒）。
- **已儲存的提示詞**：`localStorage`（key: `imgPromptSavedItems`），每筆 `{id, target, name, savedAt, fieldValues, assembledPrompt, aiOutput, videoScript}`。載入時會同時還原共用欄位＋切換回對應的目標模型分頁。事件委派的單一 click listener（`#savedList`）處理載入／複製／下載／刪除。
- **五秒影片腳本參考**（`videoScriptBtn`，位於 AI 優化面板之後、金色 `.video-panel`）：以 `state.aiOutput[activeTarget]`（AI 優化結果，不是原始概念）為輸入，請 LLM 產出 3 個「5 秒動態影片」方案（鏡頭運動／動態元素／一句英文動作提示詞），目的是給 Kling／Runway／Google Veo 這類圖生影片工具或短影音剪輯腳本用。**依賴 AI 優化結果先存在**——按鈕點下去若 `state.aiOutput[activeTarget]` 是空字串會直接擋下並提示「請先在上方送給 AI 優化」，不會另外呼叫 API。狀態存在 `state.videoScripts`（結構與 `aiOutput` 對稱，每個目標模型分頁各自獨立保留，切換分頁互不覆蓋）。複用同一組 `AI_PROVIDERS`／`callLLM()`／API 連線設定（不需要另外填一次金鑰）。`videoScriptPromptForTarget(target, aiOptimizedText)` 與 `aiPromptForTarget()` 是兩個獨立的 prompt builder，改其中一個不會互相影響。
- `manual.html` 操作手冊：四目標模型分頁介紹／操作步驟／「組成」與「AI優化」差異說明／已儲存的提示詞／AI 串接說明／授權序號說明／隱私說明／使用警語／創作者資料／授權限制。**創作者經歷內容與 `icap-generator/manual.html`、`sbir-generator/manual.html`、`phoenix-loan-generator/manual.html`、`Prompt/manual.html`、`ai-prompt-generator/manual.html` 為同一份，更新其中一邊時同步其餘各邊。**

## 序號授權（鎖定整個工具，12 個月）

比照 `ai-prompt-generator/index.html`（2026-08-12 最新模式）「單一工具、整個鎖住」的做法，而非 `sbir-generator`/`icap-generator` 那種主版免費、另開子資料夾才鎖的舊模式：`#licenseGate` 全螢幕遮罩預設鎖定，驗證通過才加上 `.hidden`；載入時一律對後端即時重驗（不只信任 localStorage 快取），背景每 20 分鐘重驗一次，過期會自動重新鎖住整個頁面。`localStorage` key：`imgPromptSerial`。

- `Code.gs` — 部署到 Google Sheet 的 Apps Script 原始碼：`doPost` 只做序號驗證＋首次自動啟用，`doGet` 供部署後測試。`VALID_AMOUNT = 12`（月）。這不是這個資料夾裡的檔案在跑，是使用者手動複製貼到 Google Sheet 的「擴充功能 → Apps Script」編輯器裡部署成 Web App，取得網址後回填到 `index.html` 的 `LICENSE_CHECK_URL`。部署步驟見 `SETUP-授權伺服器設定.md`。
- **這支後端只做序號驗證，不代理任何付費 API**（本工具的 LLM 串接是 BYOK，前端直連使用者自己的服務商 API，跟序號系統無關），也**不處理跑馬燈**（見下）。
- **綁定的 Google Sheet 是使用者指定沿用的既有表**（非本專案新開）：<https://docs.google.com/spreadsheets/d/1lwvqRNqpF5Z11jiYhS0Eo1_PRsN789N01zjPEgSarYk/edit>。原始欄位為「任務／優先順序／負責人／狀態／序號／開始日期／結束日期／交件／附註」（含一筆測試列 `xyz-9991`），比純驗證表多了任務追蹤用欄位；`Code.gs` 依表頭文字比對「序號」「開始日期」「結束日期」三個欄位，其餘欄位不影響驗證邏輯，可保留給使用者自己的任務追蹤用途。
- **已完成部署（2026-08-12）**。`index.html` 的 `LICENSE_CHECK_URL` 已填入實際部署網址：`https://script.google.com/macros/s/AKfycbwRQJLIQrKj99TUB3oxch1yuytQI32HD0B9eF965Pf3krYfEMMmrwMRLFvL76tPmdDO/exec`。`doGet`／`doPost` 皆已用 curl／Node `fetch()` 驗證正常（假序號正確回傳 `serial_not_found`；Sheet 上的測試序號 `mark0131` 正確回傳 `valid:true` 及對應的啟用/到期日期）。
- **部署過程**：複製貼上 `Code.gs` 到 Apps Script 編輯器容易出現語法錯誤是已知踩坑，這次改用 `clasp login`（使用者自行在獨立終端機完成 OAuth，Claude 無法代勞）→ 使用者手動開啟 Sheet 的「擴充功能 → Apps Script」建立綁定腳本專案並提供 Script ID → `clasp clone` → 覆蓋 `Code.js` → `clasp push --force` 一次成功，跳過複製貼上；部署為 Web App（新增部署，非更新既有部署，因為這是這個腳本專案第一次真正部署成 Web App）仍由使用者手動完成（涉及 OAuth 同意畫面，無法自動化）。**驗證 `doPost` 時遇到的坑**：用單一 Node 腳本先後打 GET 再 POST，第二次 POST 的結果被污染成回傳了 `doGet` 的健康檢查訊息——不是部署壞了，是同一 Node 行程裡連續 `fetch()` 呼叫時的連線/重導向快取造成的假象；改成各自獨立的 Node 呼叫、且用 `redirect:'manual'` 先確認第一段 302 沒問題，再手動 `fetch()` 那組不重複使用的 `user_content_key` echo 網址，就能穩定拿到正確結果。

## 序號剩餘天數持續顯示（2026-08-13）

原本「剩餘 N 天可用」只出現在 `#gateStatus` 裡、驗證通過那一瞬間，遮罩一加上 `.hidden` 整段文字就跟著消失，使用者解鎖後完全看不到還剩幾天。改法：`.topbar` 內 `nav` 前新增常駐徽章 `#licenseBadge`（🔑 剩餘 N 天，hover 顯示到期日；`.license-badge`／`.license-badge.warn` 用 `--accent-soft`/`--accent` 與 `rgba(251,191,36,.14)`/`#fbbf24` 兩色），`unlock()` 時同步寫入、`lock()`（含每 20 分鐘背景重驗失敗時）隱藏；剩餘 ≤7 天變色。已用 Chrome 對真實部署端點實測，序號 `mark0131` 解鎖後正確顯示「🔑 剩餘 505 天」。這個修法同步套用到 `ai-prompt-generator`、`ai-music-prompt-studio`（同一套序號授權骨架的姊妹專案，發現這是系統性缺口後應使用者要求一併補上）。

## 頂部共用跑馬燈

`#marqueeBar` 內容抓自工作區既有的共用授權伺服器（`https://script.google.com/macros/s/AKfycbwKX0.../exec`，與 `Prompt/index.html`、`ai-prompt-generator`、`ai-video-studio` 系列共用同一個 Google Sheet），做法完全比照 `ai-prompt-generator/index.html` 的獨立跑馬燈邏輯——**跟本工具自己的序號授權後端是兩個互不相干的系統**：頁面載入時直接 POST 一個空序號給共用端點，`localStorage` key `imgPromptMarquee`，每 20 分鐘背景重抓一次。改跑馬燈內容直接編輯共用 Sheet 即可，不需要重新部署任何 Apps Script。

**2026-08-20 更新（`Code.gs` 未改動、不需重新部署）**：`render()` 新增 `lastKey`（`JSON.stringify(items)`）比對，內容沒變就不重繪，CSS animation 不再被重置歸零重跑；新增 `appendParsedText()`／`buildTrackContent()` 支援 `[文字](https://...)` 連結語法（`createTextNode` 組 DOM，避免 XSS），資料格式仍是純字串陣列，向下相容。已 commit＋push（GitHub Pages 自動重新部署）。

## 響應式設計（桌機／平板／手機）

版型從一開始就採用流體設計（`main`/`.hero p.lead` 皆為 `max-width` 而非固定寬、按鈕與 chip 皆 `flex-wrap`），本次針對桌機／平板／手機三種寬度做過完整稽核（Playwright 在此環境的 `browser_resize` 實際套用的視窗寬度會被畫面縮放係數影響、跟請求的數字不一致——不要假設 resize 後 `window.innerWidth` 等於你傳入的寬度，稽核時務必用 `window.innerWidth` 實際讀值，不要用請求值）：

- **900px 以下**（平板）：`main`/`.panel` 內距略縮，`.field-grid`／`.api-grid` 仍維持雙欄（欄寬還很充裕，不需要提早收成單欄）。
- **600px 以下**（手機）：`.field-grid`／`.api-grid` 收成單欄、`.type-tabs` 收成直向堆疊；同時**放大觸控熱區**——`.btn` 至少 44px 高（WCAG 建議值）、`.chip`／`.checkbox-pill` 至少 38px 高，且 `.output-actions .btn` 改用 `flex:1 1 auto` 讓兩顆按鈕平均撐滿寬度，比原本各自窄窄的兩顆更好點按。
- 從 260px（極窄手機）到約 1600px 都稽核過 `document.documentElement.scrollWidth` 沒有超出 `window.innerWidth`（無橫向捲動），topbar 的 brand／nav 在任何寬度都沒有互相覆蓋。
- **測試時的快取坑**：用同一個 `python -m http.server` 常駐服務改完 CSS 後直接用 Playwright 重新 `navigate` 同一個網址，瀏覽器可能吃到 304 快取、看不到最新樣式（改的東西明明存檔了，量出來的尺寸卻沒變）——用 `?v=數字` 這種隨便加的查詢參數強制重新抓取即可，不要誤以為是 CSS 沒生效。

## 隱私與警語

無伺服器端經手使用者資料；欄位內容、組成結果、AI 優化結果、已儲存清單皆只存在使用者瀏覽器的 localStorage。序號驗證只會傳送序號本身給授權伺服器，不會傳送任何概念或提示詞內容。首頁與手冊皆明列使用警語：AI 優化結果需自行查核、請勿輸入真實個資或機密資料、僅供教學與個人使用禁止商業化。修改功能時這些警語需一併檢視是否仍準確。

## 桌面版 exe（ImagePromptStudio/）

`ImagePromptStudio/ImagePromptStudio.exe` 是可攜式單檔桌面版（做法比照 `Prompt/Prompt_Eng/`、`icap-generator/icap/`）：`launcher.py` 把 index/manual 打包進 exe，執行時於 `127.0.0.1:8792` 起本機伺服器並開預設瀏覽器（**固定 8792 埠**——工作區埠號分配見根目錄 `CLAUDE.md`／`Prompt/CLAUDE.md`，8792 為建置時確認未使用的最低空號）。**修改 index.html／manual.html 後 exe 不會自動更新，需重建**（PowerShell、絕對路徑，`--add-data` 的相對路徑會以 specpath 為準而踩雷）：

```powershell
$proj = "C:\Users\mark_\AI Test\行銷內容工具\ai-image-prompt-studio"
cd $proj
python -m PyInstaller --onefile --console --name ImagePromptStudio `
  --distpath "$proj\ImagePromptStudio" --workpath "$env:TEMP\pyi-build-imgprompt" --specpath "$env:TEMP" `
  --add-data "$proj\index.html;." --add-data "$proj\manual.html;." `
  launcher.py
```

已建置兩次（2026-08-12：首次建置＋回填 `LICENSE_CHECK_URL` 後重建），用 `python launcher.py` 直接測試過 `/index.html`／`/manual.html` 皆回應 200；exe 本身因 Windows Smart App Control 對新編譯未簽章二進位檔的已知延遲封鎖（見全域記憶 `windows-smart-app-control-dll-blocks`），尚未實機雙擊驗證。exe 未簽章，首次執行可能遇 SmartScreen 或 Smart App Control 警告；若被硬擋，可比照 `Prompt_Eng/啟動提示詞控制台.bat` 的做法，改用已簽章的系統 `python.exe` 執行 `launcher.py` 繞過。測試 exe 時注意：PyInstaller onefile 會有父子兩個程序，需要 `taskkill //IM ImagePromptStudio.exe //F` 才殺得乾淨（**不要用不帶 `//IM <名稱>` 的 `taskkill //IM python.exe //F`，會誤殺系統上所有 python.exe 行程**）。

## 指令

無建置/測試指令。修改 `index.html` 或 `manual.html` 後直接用瀏覽器開啟驗證，或暫起 `python -m http.server <port>` 測完關閉。修改內嵌 `<script>` 後可用以下方式快速檢查語法（把 `<script>...</script>` 內容抽出存成 `.js` 再跑 `node --check`）：

```bash
python -c "
import re
html = open('index.html', encoding='utf-8').read()
open('_check.js','w',encoding='utf-8').write(re.findall(r'<script>(.*?)</script>', html, re.S)[0])
"
node --check _check.js
```

**測試序號授權邏輯前，需先照 `SETUP-授權伺服器設定.md` 部署好 Apps Script 並回填 `LICENSE_CHECK_URL`**，否則會顯示「尚未設定授權伺服器網址」的 fail-closed 錯誤訊息並停留在鎖定畫面；開發階段要測試欄位/分頁/AI/儲存清單等其他功能，可在瀏覽器 devtools 手動對 `#licenseGate` 加上 `hidden` class 暫時繞過。

## GitHub 與線上部署

公開 repo：<https://github.com/M255525/ai-image-prompt-studio>（與 `ai-prompt-generator` 同樣模式；使用者已確認公開 repo 沒問題，即使 `CLAUDE.md`／`SETUP-授權伺服器設定.md` 裡含有授權後端 Google Sheet 的網址——該連結本身不代表任何人能編輯/檢視，取決於 Sheet 自己的分享設定）。已啟用 GitHub Pages（`gh api repos/.../pages` 建立，source 為 `master` 分支根目錄），線上網址：<https://m255525.github.io/ai-image-prompt-studio/>。`README.md` 是給 GitHub repo 首頁看的說明文件，與 `CLAUDE.md`（給 Claude Code 的開發筆記）分工不同，兩者都要在功能變動時同步更新。

## 加入主畫面（PWA，2026-08-14 新增）

比照 `expense-tracker-pwa`／`ai-prompt-generator`（同一次一併加上）的做法：`manifest.json`＋`icons/`（洋紅 `#ec4899` 背景「圖」字圖示）＋`service-worker.js`（network-first＋同源快取備援，跨網域請求略過，不需要像 `expense-tracker-pwa` 那樣每次改動升版 `CACHE_NAME`）。頁尾 `.footer-meta` 新增「📲 加入主畫面」按鈕（`#installBtn`），獨立 IIFE，跟序號授權閘門互不相依。已用 Playwright 實測 Chromium 觸發 `beforeinstallprompt`、SW 成功註冊。


**iOS／iPadOS／macOS 相容性補強（2026-08-14 同日追加）**：Safari（含 iOS 上的 Chrome/Firefox，底層都是 WebKit）**永遠不會觸發 `beforeinstallprompt`**，原本的按鈕邏輯在這些瀏覽器上一律落入「瀏覽器不支援」這句話，其實是誤導——蘋果裝置本來就能加入主畫面，只是要透過分享選單手動操作，不像 Chrome/Edge 有自動彈窗。修法：安裝腳本新增 `isIOSDevice`（`/iPad|iPhone|iPod/` 或 `navigator.platform==='MacIntel' && maxTouchPoints>1`——後者是因為 iPadOS 13+ 預設偽裝成 Mac 桌面版 UA，要用觸控點數才分得出來是 iPad 還是真的 Mac）與 `isMacDesktop && isSafariEngine`（macOS 桌面版 Safari 17+ 是「檔案→加入 Dock」，跟手機的分享選單操作不同）兩種判斷，各自顯示對應的操作指引文字，不再顯示「不支援」；`isStandalone`（`matchMedia('(display-mode: standalone)')` 或 iOS 專有的 `navigator.standalone`）為真時直接隱藏按鈕（已經是安裝後開啟，不需要再顯示安裝按鈕）。`<head>` 同步補上 `apple-touch-icon`（180×180 專用尺寸，`icons/apple-touch-icon.png`，純色不透明背景）＋ `apple-mobile-web-app-capable`／`mobile-web-app-capable`（兩個都要，前者給 Safari、後者是 Chrome 主張的新標準，只寫一個 Chrome 會在主控台噴 deprecation warning）＋ `apple-mobile-web-app-status-bar-style`／`apple-mobile-web-app-title`。這五個判斷/訊息字串在全部 9 個已加裝 PWA 的專案裡是逐字複製的同一段邏輯，日後若要調整任一處的措辭或判斷式，建議九個一起改，避免各專案的安裝體驗不一致。

**回饋機制與快取踩坑修正（2026-08-14，使用者實測回報「加入主畫面沒有功能」才發現兩層問題）**：(1) 原本無 `showToast` 時用「暫時置換按鈕文字」當提示，在工具列裡太不明顯，使用者完全沒注意到訊息出現過——改成 `window.alert(fallbackMessage())`，`deferredPrompt.prompt()` 也包 try/catch。(2) 改完使用者仍回報沒反應，追查發現 `service-worker.js` 的 `fetch(event.request)` 沒有繞過瀏覽器 HTTP 快取——GitHub Pages 對回應下 `Cache-Control: max-age=600`，10 分鐘內「network-first」名不符實，可能吃到舊版內容重新存進 Cache Storage。改成 `fetch(event.request, {cache:'reload'})` 強制略過 HTTP 快取，`CACHE_NAME` 同步升版 v1→v2 清掉已污染的快取。這是跟 `expense-tracker-pwa` 那次「install 階段 `cache.addAll()` 忘記加 `{cache:'reload'}`」同一個 bug class 的 runtime 版本，細節見 [[pwa-install-rollout]]。

**回饋機制第二個潛藏 bug（2026-08-16，使用者回報 SocialPost「按鈕鍵但沒有對應功能」，排查後發現本工具與另外兩個姊妹專案都中招）**：上面「回饋機制與快取踩坑修正」那次只修了**沒有** `showToast()` 的 6 個專案（改用 `alert()`）；本工具**有** `showToast()`，被歸類為「沒有這個問題」而跳過——但這個判斷本身是錯的。PWA 安裝腳本是**獨立的 `<script>` 區塊／獨立 IIFE**，而 `showToast()` 是宣告在**另一個**（主程式的）IIFE 裡，函式作用域不會跨 `<script>` 區塊或跨 IIFE 共享，所以 `typeof showToast === 'function'` 在安裝腳本裡永遠是 `false`——`deferredPrompt` 為 `null`（例如瀏覽器沒有觸發 `beforeinstallprompt`，或使用者之前已經安裝過/略過安裝提示）時，點按鈕**完全沒有任何回饋、也沒有任何主控台錯誤**，跟「按鈕鍵但沒有對應功能」的回報完全吻合。修法：安裝腳本不再依賴外部 `showToast`，改成自己實作一個 `notify(msg)`（直接抓 `#toast` 這個 DOM 元素、手動套用 `.show` class），`deferredPrompt.prompt()` 也補上 try/catch。**教訓：判斷「這個專案有沒有回饋機制的 bug」不能只看「有沒有 `showToast` 這個函式存在於檔案裡」，要看安裝腳本能不能**真的**在執行當下呼叫到它——`<script>` 區塊邊界和 IIFE 邊界都會擋住跨區塊的函式可見性，這個坑之後只要是「主程式定義一個工具函式、獨立掛載的小功能想沿用」的情境都可能重踩。**已同步修正的姊妹專案：`ai-prompt-generator`、`ai-music-prompt-studio`（同一套安裝腳本、同一個 bug、同一次修正）。

## 本次未做（後續視需要再處理）

- exe 尚未實機雙擊驗證（Smart App Control 延遲封鎖，見上）。
- 根目錄 `專案目錄.docx` 尚未加入本專案的列（建立時檔案被 Word 開啟中鎖定無法寫入；同時發現 `ai-prompt-generator` 也漏加，需一併補上）。
