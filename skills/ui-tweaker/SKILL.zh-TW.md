<!--
  繁體中文版說明文件 — 對應英文版 SKILL.md（同一份行為的中文翻譯）。
  此檔「不含」YAML frontmatter，因此不會被註冊成第二個 skill；
  唯一被當作 skill 載入的是 SKILL.md。修改行為時兩份要同步（見 CONTRIBUTING.md）。
-->

# UI Tweaker（繁體中文）

讓使用者在 Claude 對話中直接用滑桿微調 UI，確認後 AI 自動更新程式碼，不需要來回用文字描述數值。

核心原則：所有調整都以可回填的結構化數值為主；不要只用「感覺」描述差異。

## 分發與更新

Claude Code 是此 skill 主要支援的安裝與更新目標。此 repo 同時包成 Claude Code plugin marketplace（`.claude-plugin/`）與實際 skill 資料夾（`skills/ui-tweaker/`），因此使用者可從 GitHub-backed marketplace 安裝，之後在有新 push 或 tag 後透過重新整理／重新安裝 plugin 取得更新。

其他 AI 工具仍可使用這份 skill 指令，但目前沒有跨 AI 通用的 skill 安裝與更新標準。對 Cursor、Windsurf、ChatGPT 類 custom instructions 或其他 agent，請把 `skills/ui-tweaker/` 視為可攜式指令資料夾，接到該工具自己的 rules／skills 機制。更新建議用 GitHub `git pull`、submodule、symlink，或一支平台專用同步腳本處理。除非宿主 AI 真的支援即時推送更新，否則不要把它描述成 live push updates。

## 觸發條件

以下任一情況觸發此 skill：
- 使用者說「幫我調整」、「微調」、「我想改」＋元件或樣式名稱
- 使用者貼上程式碼並說「調整這個的 XX」
- 使用者說「數值調一下」、「感覺不夠 XX」（不夠立體、太大、太擠等）
- 使用者上傳截圖並指出要改的部分

---

## 流程

### Step 1：確認目標元件與程式碼

收到指令後：
1. 確認使用者要調整的元件名稱
2. 如果使用者已貼上程式碼 → 直接進入 Step 2
3. 如果沒有貼程式碼 → 詢問：
   「請把 [元件名稱] 的程式碼貼過來，我幫你開啟調整面板」

---

### Step 2：同時產生「預覽畫面」＋「控制面板」

#### 一律從固定介面樣板開始（最高優先・禁止重畫 UI）

本 skill 附帶一份固定介面樣板 `template/panel-template.html`（相對於本 SKILL.md 所在資料夾）。**每次開控制板都要複製這份樣板當基底**，不可從零手寫 UI、也不可改動樣板的版面/CSS/builder，確保任何人、任何專案打開的介面都「byte-for-byte 一致」。

流程：
1. 讀 `template/panel-template.html`，複製到使用者專案可由瀏覽器存取的位置（例如專案根目錄，檔名如 `ui-tweaker-<元件>.html`）。
2. 只替換樣板內標記為 `⟦PROJECT-SPECIFIC⟧` 的區塊（其餘所有區塊原封不動）：
   - **#1** `<title>`
   - **#2** 字體 `<link>`（換成該專案實際字體）
   - **#3** 樣式表 `<link href>`（連結該專案真實 CSS，禁止手抄數值進 `<style>`）
   - **#4** `#card-stage` 內預覽 DOM（貼真實元件 markup，每個原子元素加 `data-pick`）
   - **#5** `SEL` map（pick key → widget 內查詢用 selector）
   - **#6** `OUT_SEL` map（pick key → 對外輸出 CSS selector 名稱）
   - **#7** `LAYERS` 樹（依真實 DOM 拆到原子層級，見下方「元件原子化拆解」）
   - **#8** 確認輸出的「元件：」「來源：」字串
   - **#9 元件舞台 reset**：把元件根節點在 `#card-stage` 內從隱藏/定位 reset 成可見/靜態的那條規則（把 selector 換成你的真實根節點；保留 `--tw-card-transform` 變數，卡片層級的 transform 才會生效）。只有當真實元件在 production 是隱藏/絕對定位時才需要動它。
3. 替換後，每個 `LAYERS` 有 key 的節點都要在 `SEL`／`OUT_SEL`／預覽 DOM 的 `data-pick` 三處有對應條目；交付前用腳本交叉檢查 keys 一致、`node --check` 驗證 JS。
4. 樣板已內建固定 UI：左側 Pages/Layers 樹、三段式控制面板、八大分類、可打字數字框、雙欄/四欄格線、Figma section 標題、PS 圓角四角連動、位置偏移對齊圖示、拖曳刷數值、undo/redo + 鍵盤快捷鍵。這些**不要重做或改樣式**。

> 若樣板檔不存在或被刪，才退回「依本文件規格手寫」；但只要樣板在，一律以樣板為準。

用一個 HTML widget 呈現，分左右三欄：
- 左側：Pages／Layers 樹（Figma 式），呈現元件巢狀關聯，點擊選取、Shift 多選
- 中間：元件即時預覽（連結真實樣式表還原），每個可調整原子子元素都能點擊
- 右側：控制面板（八大分類，用折疊區塊排列）

預設展開與目前元件最相關的分類，其餘折疊。

#### 真實樣式還原（最高優先・禁止手抄）

這是本 skill 最重要的規則，凌駕其他所有「還原樣式」字眼。預覽與控制面板的數值**必須由瀏覽器解析真實程式碼得出**，絕不可用眼睛估、也不可把 CSS 規則手抄進 widget。原因：真實畫面常由多層 `!important` override 疊出來（base 規則 + 設計系統層 + pixel polish 層 + 最終覆寫層…），手抄任何一層都會跟實際不符。

1. **連結真實來源，不要重寫**：widget 一律用 `<link rel="stylesheet" href="真實樣式表路徑">`（或 `@import` 真實檔案）載入專案實際的 CSS 檔，並用 `<link>`／同樣 `<head>` 載入專案實際使用的字體。多個來源檔就全部連結。**禁止**把 selector 的數值複製貼進 widget 的 `<style>`。
2. **用真實 DOM 結構與 class**：預覽元素要用元件在專案裡真正的標籤、class、巢狀結構，不要自訂簡化版。若內容由 JS 動態填入（動態 id），要照抄真實 JS 產生的 markup——包含 icon SVG、`<span>` 包法、以及格式化字串（例如日期格式 `今日 14:30` 而非自己編的 `2025/06/02 14:30`）。先去讀產生該 DOM 的 JS 函式再寫。
3. **初始值一律 `getComputedStyle` 讀**：所有控制項的初始數值，必須在「樣式表確定載入完成後」（`window.load` 或輪詢 sheet 就緒）用 `getComputedStyle(真實元素).getPropertyValue(prop)` 讀取真實渲染值，禁止填入基礎規則或估計值。
4. **套用調整要壓過 `!important`**：預覽即時套用一律用 `el.style.setProperty(prop, val, 'important')`，否則會被來源樣式表的 `!important` 蓋掉，導致拖了滑桿畫面卻不動。
5. **處理 JS 預設隱藏的子元素**：有些元素預設 `display:none`，靠 JS 在特定狀態才顯示（例如選單鈕）。預覽要強制還原成「真實出現時」的狀態（如 inline `style="display:flex"`），否則會漏畫元素。
6. **多值屬性要偵測並保護**：多層 `box-shadow`、漸層 `background` 等無法用單層滑桿無損還原的屬性，`getComputedStyle` 也讀不回單一值。偵測到多層/漸層時，該控制項要鎖定或顯示警告，避免單層滑桿把原本的多層值覆寫掉。
7. **確認輸出只含改過的屬性**：送回 `[UI Tweaker 確認]` 時，只輸出使用者實際調整過的屬性；沒動到的一律不進輸出，確保不會誤改使用者原本就滿意的地方。
8. **完成後實機驗證**：用瀏覽器讀 `getComputedStyle` 或截圖，確認預覽解析值與 app 實際值一致（例如挑 2–3 個關鍵屬性比對數字），再交給使用者。

#### 自動解析 JS 動態樣板（Zero-Jargon Rule）

- 若目標元件是寫在 JS 檔案中的動態字串樣板或組件（如 template literals），AI 必須主動在預覽面板中，將其還原／mock 成瀏覽器最終渲染出的真實 HTML DOM 結構。
- 遇到與變數綁定的 inline style 或動態生成的文字節點，需自動解析並實體化到預覽 DOM 上，絕對不可當作純字串擷取。
- AI 需主動辨識並將帶有視覺屬性（如名稱輸入框、地點列、圖示、標籤、徽章、按鈕）的節點原子化，拆分成可獨立點擊的最小子元素，並掛載對應的點擊事件或 key。
- **嚴格禁止**：AI 不得要求使用者使用工程術語（如「還原 DOM」、「mock」、「原子層級」）來下指令。AI 需自行在背景完成上述所有結構轉換，只呈現可直覺點擊微調的預覽畫面給使用者。

#### 預覽區互動規則

- 一律用「連結真實來源 + 真實 DOM + `getComputedStyle`」還原樣式（見上節），不用假資料、不手抄數值。
- 每個可調整的子元素都要有可辨識 key，例如 `data-focus-target="photo"`、`data-focus-target="glass"`，或使用 `onclick="pick('元素id')"`。
- icon + 文字混排的元件（例如徽章 = SVG + 時間字串）要讓兩者可「分別選取、分別調」，不要綁成一組：icon 直接給 selector（如 `.badge svg`）；但裸文字節點無法被選取或單獨套樣式，必須在預覽 DOM 裡把它包成帶 class 的 `<span>`（如 `.badge-text`）才能當獨立 pick 目標。注意此 span 通常是真實 app 沒有的，送 `[UI Tweaker 確認]` 套用到原始碼時，除了改 CSS 還要去產生該 DOM 的模板/JS 補上同樣的 span 包裝，樣式才掛得上；先在確認輸出或回覆裡點明這個前提。
- 點擊預覽子元素後，右側自動展開對應分類，捲動到相關控制項，並短暫標示相關控制項。
- 切換元素時要清除所有舊選取狀態，避免多個元素同時顯示選取。
- 選取提示不能干擾陰影、玻璃、box-shadow 預覽。優先使用低干擾提示，例如控制列左側細色條、分類標題淡色提示、或目標元素外側輕微光暈。
- 禁止在 `applyDOM()` 中把選取框混入元件本身的 `box-shadow`；陰影預覽必須保持純粹。
- 若需要框選目標元素，使用不影響陰影的方式，例如 `outline-offset: -2px` 的內側 outline，或獨立 overlay/hit area。不要用會改變元件陰影感的 inset shadow 當選取框。

#### 預覽寬度與歷史操作

- 預覽區應提供裝置/寬度切換控制，至少包含三種寬度：窄版、手機版、寬版。
- 裝置切換建議使用圖示型 segmented control，不用冗長文字按鈕。
- 預覽寬度切換只影響控制面板預覽，不應改寫使用者程式碼。
- 提供「放大檢視」面板，讓使用者看細節時只放大預覽、不放大整個網頁：用 CSS `zoom` 套在預覽舞台容器上（`zoom` 會同時放大渲染與捲動範圍，配合 `overflow:auto` 可捲動看細節），提供 −/百分比/＋ 按鈕（點百分比回 100%）與 `Ctrl/⌘+滾輪` 縮放。注意：`zoom` 會讓 `getBoundingClientRect` 回傳放大後的座標，所有以量測 rect 推算的位移（對齊、置中）都要除以目前 zoom 倍率才正確。縮放只影響檢視，不寫入使用者程式碼。
- 工具列建議橫跨預覽區滿版、用 `justify-content:space-between` 分三區：裝置寬度切換靠最左、縮放控制靠最右、undo/redo/對齊等置中，避免全部擠在中間。
- 控制面板應提供「上一步 / 下一步」功能，讓使用者可復原與重做滑桿調整。
- undo/redo 按鈕建議放在預覽寬度切換下方或附近，使用圖示按鈕呈現。
- 除了按鈕，也要綁鍵盤快捷鍵：`Ctrl/⌘+Z` 上一步、`Ctrl/⌘+Shift+Z` 或 `Ctrl+Y` 下一步（在 `document` 上 `keydown`，命中時 `preventDefault`）。快捷鍵要走跟按鈕同一套 history 函式，不重整頁面、不清空目前調整值。
- history 應記錄每次完成的控制值變更；拖曳滑桿時可在 `input` 階段即時預覽，但只在 `change` 或同等完成事件時寫入一筆歷史。
- undo/redo 不應重整頁面、不應清空使用者目前調整值。
- 若使用 `localStorage` 保存控制面板狀態，undo/redo 後也要同步保存目前 state。

#### 位置偏移與對齊參考線

- X/Y 位移滑桿在接近 0 時應支援 snap：建議在 `±3px` 或等比例單位內自動歸零。
- snap 到 0 時，數值顯示要清楚標示「置中」或以藍色狀態顯示。
- X 歸零時可顯示垂直參考線與「垂直置中」標籤。
- Y 歸零時可顯示水平參考線與「水平置中」標籤。
- 參考線應短暫顯示後淡出，建議約 `0.7s`。
- 支援 PS 式多選相互對齊：`Shift+點擊`（預覽或選取列皆可）切換加入/移出多選集合（最後一個不可清空），`cur` 仍為最後互動的主要元素、控制面板顯示它。選到 ≥2 個時顯示一排對齊鈕（左/水平置中/右、頂/垂直置中/底），放在 undo/redo 同一排、<2 個時隱藏。對齊以「選取集合的外框（bbox）」為基準，靠位移屬性（translate `_tx`/`_ty`）達成並記進 history。注意：被選元素必須是「可被 transform 位移」的元素（block／inline-block／flex 或 grid 子項等）；純 `display:inline` 元素 transform 無效，需先確保它是 flex 子項或改 inline-block。同列（已被 flex 對齊）的元素做垂直對齊時位移可能極小、看起來像沒反應，屬正常。
- 對齊控制要依容器 `display` 選對屬性，否則套了畫面不動：
  - **水平對齊**對 flex／inline-flex 容器套 `text-align` 對其子元素無效，必須改套主軸屬性；非 flex 元素才用 `text-align`。
  - **垂直對齊**單靠 `text-align` 永遠做不到，必須用交叉軸屬性（flex 容器）；非 flex 元素要垂直置中時，以最小副作用包成 `display:flex; align-items:center`。對齊面板除了水平排（靠左/置中/靠右），也要提供垂直排（頂端/置中/底部）。
  - **軸要看 `flex-direction`**：`row` 時水平=`justify-content`、垂直=`align-items`；`column` 時兩者對調。先用 `getComputedStyle(el)` 讀 `display` 與 `flex-direction` 再決定屬性。值對應：起點→`flex-start`、置中→`center`、終點→`flex-end`。
  - 對齊輸出要帶上「實際生效」的屬性（flex 容器輸出 `justify-content`／`align-items`），確認時才不會誤改成無效的 `text-align`。

#### 固定介面設計（Figma／PS 風・由樣板內建）

以下是樣板已實作、必須維持一致的介面設計。換新元件時照用，不要改樣式或重排：

- **三欄版面**：`grid-template-columns: 230px 1fr 360px`。左＝Layers/Pages 樹、中＝預覽舞台、右＝控制面板。
- **三段式控制面板**（解決「確認列浮在清單中間」破圖）：`.tw-right` 為 `display:flex; flex-direction:column; height:100vh; overflow:hidden`；固定 header（`flex:none`）＋ 可捲動 `#panel`（`flex:1; min-height:0; overflow-y:auto`）＋ 固定底部確認列（`flex:none`）。**不要用 `position:sticky` 的底部確認列**。
- **Figma section 標題**：八大分類用折疊區塊（`cat(title, open, [...])`），標題無數字前綴（純文字，不要「① ⑥」之類）。
- **可打字數字框**（取代滑桿）：`numBox(prop,label,min,max,step,unit,init,snap)` 產生 `<input type=text inputmode=decimal>`，方向鍵 ↑↓ 步進（Shift×10）、Enter 失焦、聚焦全選、X/Y 近 0 snap。
- **多欄格線**：相關欄位用 `fieldGrid(cols, fields)` 併成兩欄/四欄（例如內距上下左右、陰影 XYBlur）。
- **拖曳刷數值（Figma scrub）**：`attachScrub(...)`，在 label/icon 上水平拖曳改值（1px≈1 step），拖曳中即時預覽、放開才寫一筆 history。游標 `ew-resize`。**不放滑桿**。
- **PS 式圓角**：`radiusControl(key)` 為 2×2 四角長手寫欄位（tl/tr/bl/br）＋中央鏈結鈕；連動開啟時，調一角直接把值套到其餘三角（直接呼叫 `applyNum`，**不要用 dispatch event**，避免綁定順序 bug）。
- **位置偏移用圖示對齊鈕**：`alignTools(key)` 三排（框位置／框內文字／垂直對齊），用圖示按鈕（非文字）；依容器 `display`/`flex-direction` 選對屬性（flex 用 `justify-content`/`align-items`，非 flex 才 `text-align`；垂直置中不可只靠 `text-align`）。
- **左側 Layers/Pages 樹**：`LAYERS` 巢狀結構鏡射真實 DOM；`renderLayerNode` 遞迴渲染、有 key 可點選、Shift 多選、分組節點可摺疊；`refreshSelectionUI` 同步 `.on` 狀態與 `#cur-name`。
- **變形框（選取框＝可變形）**：選取框是疊在 `#tw-left` 上的純 overlay（`#tw-tbox`，**不混入 box-shadow**），單選時顯示 8 向手把＋頂部旋轉手把＋即時 px 尺寸標籤。語意：**拖四角＝等比縮放**（寫 `_scl`，以中心為原點，比例抵銷 zoom）、**拖四邊＝改 `width`/`height`**（螢幕位移 ÷zoom 換 CSS px）、**框內拖移＝改 `_tx`/`_ty`**（近 0 snap；未拖動的單擊用 `elementFromPoint` 選底下元素，支援 Shift 多選）、**旋轉手把＝改 `_rot`**。多選時只顯示群組外框＋整組移動，隱藏縮放/旋轉手把。所有拖曳即時 `applyEl`＋同步對應數字框（`i-width`/`i-height`/`i-_scl`/`i-_rot`/`i-_tx`/`i-_ty`），放開才 `commit()` 寫一筆 history，並進確認輸出。位置由 `updateTBox()`（`getBoundingClientRect` 相對 `#tw-left` 換算，`scheduleTBox()` 於 `applyEl`／zoom／寬度切換／捲動／resize 觸發）。注意：純 `display:inline` 元素（如文字、tag span）transform 與 width/height 皆無效，變形框對它們不會動——與位置偏移控制同一限制。`_rot` 數字框範圍需放寬到 `-180..180` 以配合旋轉手把。**`#tw-tbox` 的 `z-index` 必須高於預覽元件本身的 `z-index`**（例如某些卡片容器自身帶 `z-index:100`，故 overlay 預設用 `z-index:99999`）；否則 overlay 會被元件內容蓋住，只剩最外緣那幾根 px 露出來——表現為「只有最外層容器看得到藍框，內部元件都沒有框」。換新專案時，若元件樹有更高的 `z-index`，overlay 要再往上加。
- **undo/redo**：按鈕在預覽寬度切換附近＋鍵盤 `Ctrl/⌘+Z` / `Ctrl/⌘+Shift+Z`；走同一套 history 函式，不重整、不清空目前值。
- **選取提示低干擾**：用內側 `outline-offset:-2px` 或外側光暈，**禁止把選取框混入 `box-shadow`**，陰影/玻璃預覽必須保持純粹。

#### 元件原子化拆解（Layers 要拆到拆不了為止）

`LAYERS`（樣板 `⟦PROJECT-SPECIFIC #7⟧`）要**依真實 code 把每個元件拆到原子層級**，不可只列大區塊：

- 先讀「產生該 DOM 的真實 JSX/JS/HTML」列舉所有原子元素，再建樹。
- 把混排元素拆開分別選取：icon vs 文字（如徽章＝時鐘 SVG＋時間文字、地點＝圖釘 SVG＋地點文字）、按鈕 vs 其圖示（選單鈕＋三點 SVG）、座標鈕框 vs 框內文字。
- 重複清單每一項各算獨立元件（如標籤一/標籤二/標籤三，用 `:nth-of-type(n)`），且每個標籤再拆 emoji（`.theme-emoji`）與文字（`.theme-label-text`）。
- 純分組容器（無對應可調樣式）用無 key 的節點當分組標題即可。
- **裸文字節點**（真實 code 中文字直接放在容器裡、無獨立元素，如徽章時間、座標文字）：在預覽 DOM 包成帶 class 的 `<span>` 並加 `data-pick` 才能單獨選取/套樣式；此 span 真實 app 沒有，送 `[UI Tweaker 確認]` 套用時要在輸出點明「需於真實模板/JS（產生該 DOM 的函式）補上同名 span」，否則樣式掛不上。
- 範例（一張資訊卡片，拆成 ~24 個原子層）：整張卡片 ▸ 圖片區﹙圖片、時間徽章﹙時鐘圖示、時間文字﹚﹚▸ 資訊區﹙選單按鈕﹙三點圖示﹚、標題、地點﹙圖釘圖示、地點文字﹚、標籤區﹙標籤一/二/三，各﹙emoji、文字﹚﹚、底部鈕框﹙框內文字﹚﹚。

#### 預覽 DOM 必須忠於實際 code（防止自創樣式／結構・開面板前強制核對）

控制板的預覽 DOM（樣板 `⟦PROJECT-SPECIFIC #4⟧`）只准「重現」真實 code，不准「發明」。為避免憑感覺加上真實 code 沒有的屬性/樣式/結構（例如曾誤加 `style="flex:none"`），每次產生後、開面板前都要過下面這關：

- **鐵則**：preview DOM 內每個元素的 tag、class、id、屬性、子結構、文字節點，都要能在「產生該 DOM 的真實來源檔」對應到出處（檔名:行號）。對不上來源的東西，除白名單外一律刪除。
- **唯一允許偏離真實 code 的白名單**（每一項都要在預覽 DOM 就近加註解標明）：
  1. `data-pick`（選取標記）；
  2. 為選取補的 `id`（真實沒有但選取需要時）；
  3. 裸文字節點的 synthetic `<span>`（見「原子化拆解」；須註明真實沒有、確認時要補進 app）；
  4. **必要可見性覆寫**：真實預設 `display:none`／需互動才出現的元件（如選單鈕），為了能預覽才覆寫成可見——註解要寫出「真實預設值＋為何覆寫」。
- **明確禁止**：用自創 inline `style="…"`（如 `flex:none`、`width:…`、`margin:…`）去「補」排版或樣式。一切樣式由連結的真實樣式表決定；覺得需要 inline style 時，先回真實 code 確認——幾乎都不該加（白名單第 4 項除外）。佔位圖片/文字只供顯示，不得改變結構或屬性。
- **產生後強制自我核對（三步，做完才開面板）**：
  1. 對 preview DOM 內每個 `class=`／`id=`，回真實來源 `grep` 確認存在；查無 → 自創，刪除或修正。
  2. 搜尋 preview DOM 內所有 `style="`，逐一對照白名單第 4 項；對不上的 inline style 一律移除（真實 code 沒有就不該有）。
  3. 實機用 `getComputedStyle` 抽查 2–3 個元素（尺寸／`flex`／顏色），確認與真實卡片渲染一致。
- 心裡（或輸出）保留一份「來源對應表」：每個 pick key → 真實檔:行（例：`loc → 產生該 DOM 的檔案:行號`、`menu → 模板檔:行號`）。對不上來源的元素不准進控制板。

#### 控制面板技術規範

- 每個 input 要有穩定且唯一的識別 id，例如 `id="i-{rowId}"`；數值顯示可用 `id="v-{rowId}"`。
- 所有 input 更新應走統一處理函式，例如 `ch(id, value, pos)` 或同等集中式 handler。
- 預覽更新應集中在 `applyDOM()` 或同等函式，依目前選取元素 key（例如 `cur`）套用樣式。
- 陰影只在樣式套用函式中寫入真正的 `box-shadow` / `filter` 數值，不混入選取狀態。
- 控制項應可從目前 CSS 初始值還原；若屬性缺失，使用明確預設值並在輸出中保留欄位。
- 對使用者正在調好的數值，新增控制項時不得重設既有數值；若可行，使用 localStorage 或明確的確認輸出保存狀態。

控制面板八大分類：

① 字型
   字級（px）、字重（100–900）、行高（em）、字距（em）、字體（select）、顏色

② 間距
   內距上/下/左/右（px）、外距上/下/左/右（px）、子元素間距（gap）

③ 尺寸
   寬度（px/%）、高度（px/auto）、最大寬度、比例（aspect-ratio）

④ 圓角
   整體圓角（px/%/cqw）、個別四角（左上/右上/左下/右下）、切換整體/個別模式

⑤ 位置偏移
   X 位移（translateX/left/right）、Y 位移（translateY/top/bottom）、旋轉（deg）、縮放（scale）、snap + 參考線

⑥ 陰影
   X offset、Y offset、Blur、Spread（均 px/cqw）、透明度（0–1）、必要時分拆前景/背景/互相遮蔽陰影

⑦ 磨砂玻璃
   模糊強度（blur px）、白色透明度（0–1）、頂部漸層差（%）、
   懸浮陰影（px）、頂部高光邊（0–1）、內層光感（0–1）

⑧ 顏色
   背景色、文字色、邊框色（均支援 rgba）、邊框寬度（px）

字體 select 建議選項：
- `inherit`
- `system-ui`
- `-apple-system`
- `Arial`
- `Georgia`
- `Noto Sans TC`
- `PingFang TC`

---

### Step 3：確認按鈕觸發 AI 更新

控制面板底部「確認，更新程式碼」按鈕使用 sendPrompt() 將所有當下數值以結構化格式送回 AI assistant。

若元件包含多個子元素，優先使用分元素格式：

格式範例：
```
[UI Tweaker 確認]
元件：首頁資料夾
來源：/path/to/source.css

[folder-wrapper]
  - width %: 114
  - left %: -7.5

[folder-photo]
  - white overlay alpha: 0.18
  - box-shadow blur cqw: 6

[folder-glass-panel]
  - backdrop-filter blur px: 4
  - filter photo shadow alpha: 0.34

請直接更新對應程式碼
```

若是簡單單元素，也可以使用平鋪格式：

格式範例：
```
[UI Tweaker 確認]
元件：FolderCard 的 .glass-card
更新以下數值：
- width: 160px
- border-radius: 20px
- backdrop-filter: blur(16px) saturate(1.4)
- background: linear-gradient(160deg, rgba(255,255,255,0.72), rgba(255,255,255,0.55))
- border: 1px solid rgba(255,255,255,0.85)
- box-shadow: inset 0 1px 0 rgba(255,255,255,0.9), 0 18px 36px rgba(0,0,0,0.18)
請直接更新對應的程式碼
```

AI assistant 收到 `[UI Tweaker 確認]`（或其英文形式 `[UI Tweaker Confirm]`，預設樣板送出的就是英文版）開頭的訊息後：
1. 解析所有數值
2. 找到對應 CSS class 或 inline style
3. 輸出更新後的完整程式碼區塊
4. 列出本次變更的行
5. 更新後執行可用的驗證流程，例如專案有 `npm run build` 就要跑 build；必要時用瀏覽器或 computed style 檢查實際套用值

設計系統作用範圍確認：
- 套用可重用樣式前，先判斷選取數值來源是 design token、shared utility class、component-level selector，還是 local override。
- 如果樣式看起來會被重複使用，寫入前要詢問使用者選擇更新範圍：
  - **Global**：更新 design system token / shared class，讓所有相同引用一起變更。
  - **Component**：只更新這個可重用元件家族。
  - **Local**：只更新目前頁面／目前選取 instance，以 scoped override 處理。
- 作用範圍不明時預設 **Local**。不要把 Local 說成「解除綁定」；應描述為「局部覆寫」或 scoped override。
- 若使用者選 **Global**，先搜尋引用處並摘要可能受影響的檔案／selector，再進行編輯。

結構化更新規則：
- 若確認數值超過少量欄位，或同時涉及多個 selector / 子元素，應使用結構化解析與更新流程，不要手抄數值逐行修改。
- 建議流程：先 parse 確認文字 → 檢查 `missing / unknown / changed` → dry-run 列出將修改的 selector/property → 確認 `skipped=0` 或清楚處理 skipped 原因 → 再寫入檔案。
- 可用專案內腳本、臨時 updater、AST/CSS parser、或其他可重跑的結構化方法完成；不強制固定檔名。
- dry-run 輸出應包含數量，例如 `parsed`、`changes`、`skipped`，方便用數值確認更新完整性。
- 寫入後再次 dry-run 或解析一次，確認基準值與檔案狀態一致。

commit / push 邊界：
- 使用者貼上 `[UI Tweaker 確認]`，或明確表示「調整好了 / 套用這組數值 / 直接更新」時，視為已確認的 UI 調整；套用、驗證成功後直接 commit/push。
- 使用者明確要求 `commit`、`push`、`提交`、`推上去` 時，也要在驗證成功後 commit/push。
- 還在探索、詢問、開控制板、或使用者尚未送出確認數值時，只修改控制板/準備工具，不自動 commit/push。
- commit 前確認工作區狀態，只提交本次 UI 調整相關檔案，避免提交不相關變更。

---

### Step 4：輸出更新後的程式碼

格式：
✅ 已更新 [元件名稱] — [class 名稱]

[完整程式碼區塊]

本次變更：
- border-radius: 16px → 20px
- backdrop-filter: blur(12px) → blur(16px)
- box-shadow 加強懸浮陰影

---

## 注意事項

- 預覽左側要盡量還原使用者實際樣式，不用假資料
- 預覽 DOM 只准重現真實 code、不准自創：開面板前 `grep` 核對所有 `class`/`id` 確實存在、移除非白名單的 inline `style`（見「預覽 DOM 必須忠於實際 code」）
- 控制面板初始數值從使用者貼的程式碼讀取，不用預設值
- 若某屬性在程式碼中未定義，初始值設為 CSS 預設值
- 子元素可點擊切換控制項；點擊後清除舊選取狀態，只保留目前目標
- 選取提示不得污染 `box-shadow`、`filter`、磨砂玻璃或陰影預覽
- 位置偏移應支援接近 0 的 snap 與短暫對齊參考線
- 磨砂玻璃分類：元件若無 backdrop-filter，預設折疊
- 確認按鈕送出前，面板顯示即將更新的 CSS 摘要供確認
- 多欄位或多 selector 更新時，用結構化解析/dry-run/寫入流程，避免手抄數值
- 更新完成後要驗證；有 build 指令時優先跑 build，視覺或瀏覽器行為重要時再做瀏覽器/computed style 檢查
- 收到 `[UI Tweaker 確認]` 這類已調好數值時，更新與驗證成功後直接 commit/push；未確認的探索階段不自動提交
- 收到 [UI Tweaker 確認] 訊息後，直接輸出程式碼，不再詢問；但若偵測到可重用的設計系統樣式且尚未指定更新範圍，需先確認作用範圍
