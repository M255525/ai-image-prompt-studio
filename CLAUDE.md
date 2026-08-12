# CLAUDE.md — ai-image-prompt-studio（AI 繪圖提示詞工廠）

「AI 繪圖提示詞工廠」——單檔前端工具，把一句話的中文畫面概念，優化成適合圖片生成模型使用的提示詞：Google Nano Banana Pro／Gemini、ChatGPT／DALL-E、Midjourney，以及一個不特定模型的通用格式。填一次概念與風格設定，四個目標模型分頁共用；可以純前端組成關鍵字草稿，也可以串接使用者自己的語言模型取得中英對照的正式提示詞。

與 `行銷內容工具/ai-prompt-generator/`（影音框架＋TAG／APE／CO-STAR 文字提示詞產生器）是姊妹專案，同一套 BYOK 呼叫 LLM 的手法、同一種「模板組裝＋可選 AI 優化」兩段式互動，服務對象換成圖片生成提示詞。

## 架構

單一 `index.html`：內嵌 CSS/JS、無外部資源、無建置步驟。視覺主題是深色「暗房／畫布」風格（`--bg #170b12` + 圓點網格背景 + 洋紅色 `--accent #ec4899`），與 `ai-prompt-generator`（青色）、`Prompt`（琥珀色）刻意做出區隔，方便一眼分辨是哪個工具；AI 優化面板用 teal（`--teal #2dd4bf`）區隔於主色。

- **共用欄位 + 多目標模型分頁**是這個工具與 `ai-prompt-generator` 最大的結構差異：`ai-prompt-generator` 的 `FRAMEWORKS` 是每個分頁各自獨立的欄位，本工具則是**單一組共用欄位**（`state.fields`：概念描述／風格／構圖／光線／長寬比／畫質強化詞（可複選）／負面提示詞）搭配四個只改變「組裝格式」的目標模型分頁（`TARGETS`：`gemini`/`chatgpt`/`midjourney`/`generic`）。使用者填一次欄位，四個分頁共用，只有下方「組成提示詞」「送給 AI 優化」的輸出格式規則依分頁不同。
- 風格／構圖／光線／畫質強化詞的下拉與複選選項（`STYLE_OPTIONS`／`COMPOSITION_OPTIONS`／`LIGHTING_OPTIONS`／`QUALITY_OPTIONS`）每個選項都同時定義中文標籤（`zh`，UI 顯示）與英文提示詞片語（`en`，組裝時使用）；新增選項時兩者都要補。
- 狀態存 `localStorage`（key: `imgPromptState`）：`{activeTarget, fields:{concept,style,composition,lighting,ratio,quality[],negative}, assembled:{gemini,chatgpt,midjourney,generic}, aiOutput:{...}}`——欄位是單一物件（不像 `ai-prompt-generator` 按分頁各自存一份），但組成結果與 AI 結果仍按分頁各自保留，切換分頁不會互相覆蓋。
- **核心互動模型**（與 `ai-prompt-generator` 相同精神）：「🔧 組成提示詞」是純前端字串組裝（不需金鑰、不連網）——概念描述**保留原始語言、不會被翻譯**，只有風格／構圖／光線／畫質標籤是已經對照好的英文片語；「🚀 送給 AI 優化」把概念與風格設定送給 BYOK LLM，要求輸出「中文說明＋分隔線＋English Prompt」的中英對照結果，這一步才會真的把概念翻譯成英文。Midjourney 分頁的 `aiPromptForTarget()` 額外要求逗號分隔關鍵詞＋`--ar --v` 參數語法；其餘三個分頁要求自然語言描述句。
- **BYOK AI 串接**：與 `ai-prompt-generator/index.html`、`Prompt/index.html` 同一套 `callLLM()` 模式（改動時互相參照）——全部走瀏覽器直連 `fetch()`：Claude 需 `anthropic-dangerous-direct-browser-access: true` header；Gemini 金鑰放 `x-goog-api-key` header；OpenAI/OpenRouter 用 Bearer。設定（provider/model/apiKey）存 `localStorage`（key: `imgPromptApiConfig`）——**金鑰只落在使用者本機瀏覽器，絕不可寫進程式碼**。逾時 180 秒；429/500/503/529 自動重試最多 2 次（間隔 8、16 秒）。
- **已儲存的提示詞**：`localStorage`（key: `imgPromptSavedItems`），每筆 `{id, target, name, savedAt, fieldValues, assembledPrompt, aiOutput}`。載入時會同時還原共用欄位＋切換回對應的目標模型分頁。事件委派的單一 click listener（`#savedList`）處理載入／複製／下載／刪除。
- `manual.html` 操作手冊：四目標模型分頁介紹／操作步驟／「組成」與「AI優化」差異說明／已儲存的提示詞／AI 串接說明／授權序號說明／隱私說明／使用警語／創作者資料／授權限制。**創作者經歷內容與 `icap-generator/manual.html`、`sbir-generator/manual.html`、`phoenix-loan-generator/manual.html`、`Prompt/manual.html`、`ai-prompt-generator/manual.html` 為同一份，更新其中一邊時同步其餘各邊。**

## 序號授權（鎖定整個工具，12 個月）

比照 `ai-prompt-generator/index.html`（2026-08-12 最新模式）「單一工具、整個鎖住」的做法，而非 `sbir-generator`/`icap-generator` 那種主版免費、另開子資料夾才鎖的舊模式：`#licenseGate` 全螢幕遮罩預設鎖定，驗證通過才加上 `.hidden`；載入時一律對後端即時重驗（不只信任 localStorage 快取），背景每 20 分鐘重驗一次，過期會自動重新鎖住整個頁面。`localStorage` key：`imgPromptSerial`。

- `Code.gs` — 部署到 Google Sheet 的 Apps Script 原始碼：`doPost` 只做序號驗證＋首次自動啟用，`doGet` 供部署後測試。`VALID_AMOUNT = 12`（月）。這不是這個資料夾裡的檔案在跑，是使用者手動複製貼到 Google Sheet 的「擴充功能 → Apps Script」編輯器裡部署成 Web App，取得網址後回填到 `index.html` 的 `LICENSE_CHECK_URL`。部署步驟見 `SETUP-授權伺服器設定.md`。
- **這支後端只做序號驗證，不代理任何付費 API**（本工具的 LLM 串接是 BYOK，前端直連使用者自己的服務商 API，跟序號系統無關），也**不處理跑馬燈**（見下）。
- **綁定的 Google Sheet 是使用者指定沿用的既有表**（非本專案新開）：<https://docs.google.com/spreadsheets/d/1lwvqRNqpF5Z11jiYhS0Eo1_PRsN789N01zjPEgSarYk/edit>。原始欄位為「任務／優先順序／負責人／狀態／序號／開始日期／結束日期／交件／附註」（含一筆測試列 `xyz-9991`），比純驗證表多了任務追蹤用欄位；`Code.gs` 依表頭文字比對「序號」「開始日期」「結束日期」三個欄位，其餘欄位不影響驗證邏輯，可保留給使用者自己的任務追蹤用途。
- **尚未部署（`LICENSE_CHECK_URL` 目前是空字串）**：需依 `SETUP-授權伺服器設定.md` 由使用者手動在 Apps Script 編輯器貼上 `Code.gs` 並部署為 Web App，取得網址後回填。部署前頁面會 fail-closed 卡在鎖定畫面並顯示「尚未設定授權伺服器網址」。

## 頂部共用跑馬燈

`#marqueeBar` 內容抓自工作區既有的共用授權伺服器（`https://script.google.com/macros/s/AKfycbwKX0.../exec`，與 `Prompt/index.html`、`ai-prompt-generator`、`ai-video-studio` 系列共用同一個 Google Sheet），做法完全比照 `ai-prompt-generator/index.html` 的獨立跑馬燈邏輯——**跟本工具自己的序號授權後端是兩個互不相干的系統**：頁面載入時直接 POST 一個空序號給共用端點，`localStorage` key `imgPromptMarquee`，每 20 分鐘背景重抓一次。改跑馬燈內容直接編輯共用 Sheet 即可，不需要重新部署任何 Apps Script。

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

已建置（2026-08-12），用 `python launcher.py` 直接測試過 `/index.html`／`/manual.html` 皆回應 200；exe 本身因 Windows Smart App Control 對新編譯未簽章二進位檔的已知延遲封鎖（見全域記憶 `windows-smart-app-control-dll-blocks`），尚未實機雙擊驗證。exe 未簽章，首次執行可能遇 SmartScreen 或 Smart App Control 警告；若被硬擋，可比照 `Prompt_Eng/啟動提示詞控制台.bat` 的做法，改用已簽章的系統 `python.exe` 執行 `launcher.py` 繞過。測試 exe 時注意：PyInstaller onefile 會有父子兩個程序，需要 `taskkill //IM ImagePromptStudio.exe //F` 才殺得乾淨（**不要用不帶 `//IM <名稱>` 的 `taskkill //IM python.exe //F`，會誤殺系統上所有 python.exe 行程**）。

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

## 本次未做（後續視需要再處理）

- `Code.gs` 尚未由使用者部署到 Apps Script，`LICENSE_CHECK_URL` 待回填。
- exe 尚未實機雙擊驗證（Smart App Control 延遲封鎖，見上）。
- 根目錄 `專案目錄.docx` 尚未加入本專案的列（若未同步請檢查）。
