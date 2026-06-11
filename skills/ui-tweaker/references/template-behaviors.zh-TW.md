# 樣板內建行為與控制面板規格（ui-tweaker 參考文件）

> 以下內容固定樣板**已全部內建**——禁止重做或改樣式。只在這些情況才讀：(a) 處理專案特例（替換 demo root key、stage reset 與 z-index、VISUAL_RECTS/EDIT_TARGETS、SVG icon 來源遷移、預設 stage 寬度）；(b) 驗證／除錯樣板行為；(c) 樣板檔遺失需要照規格手刻。

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
- **預設舞台寬度必須與 active 的 segmented button 一致**：若 widget 開場使用專案自訂寬度，active 裝置按鈕的 `data-w` 必須是同一個值；點擊已啟用的按鈕時不應讓 stage 尺寸跳動。不要在換成專案 root key 或自訂預設寬度後，仍殘留樣板預設如 `390px` / `pick('card')`。
- **寬度切換必須驅動內部 responsive 子元素，而不只是改外框**：變更 `#card-stage` 寬度後，專案專屬 stage reset 必須讓內層 layout container 以 stage 為基準（`width:100%; max-width:100%; box-sizing:border-box`），並壓過 production 裡依瀏覽器 viewport 生效的 desktop media-query 寬度（例如真實 app 在 `min-width:768px` 把 grid 設成 `820px`，放進隔離 stage 就會錯）。使用 `min()`、`max()`、`clamp()`、`%`、`calc()` 等 container-derived sizing，讓子元件跟著目前 stage 寬度縮放。
- **以 viewport 為準的 `@media` 不會被 stage 寬度觸發——要用 `@container` 規則鏡像覆寫**：若專案的 responsive 規則寫在以瀏覽器 viewport 為準的 `@media` 裡（例如 `@media (min-width:768px)` 才有多欄格線），舞台寬度切換不會觸發它，因為瀏覽器視窗寬度沒變。解法：在 `⟦PROJECT-SPECIFIC #9⟧` stage reset 區塊給 `#card-stage` 加 `container-type: inline-size`，再用相同斷點的 `@container` 規則覆寫格線欄數等屬性（含對應的 max-width fallback 規則），讓欄數跟著 stage 寬度走，而不是跟著視窗。
- **responsive 子元素不可無條件寫死尺寸**：固定 cell 寬度只能當最大值，不可當唯一值。兩欄 grid 建議用類似 `repeat(2, minmax(0, min(MAX_CELL, calc((100% - GAP) / 2))))` 的模式，窄版 stage 才不會裁掉第二欄。
- 提供「放大檢視」面板，讓使用者看細節時只放大預覽、不放大整個網頁：用 CSS `zoom` 套在預覽舞台容器上（`zoom` 會同時放大渲染與捲動範圍，配合 `overflow:auto` 可捲動看細節），提供 −/百分比/＋ 按鈕（點百分比回 100%）與 `Ctrl/⌘+滾輪`、`Option/Alt+滾輪` 縮放。放大後，一般滾輪／觸控板捲動仍必須平移預覽區可視範圍；只有帶 modifier 的滾輪縮放手勢可以縮放或攔截原本滾輪行為。注意：`zoom` 會讓 `getBoundingClientRect` 回傳放大後的座標，所有以量測 rect 推算的位移（對齊、置中）都要除以目前 zoom 倍率才正確。縮放只影響檢視，不寫入使用者程式碼。
- 工具列建議橫跨預覽區滿版、用 `justify-content:space-between` 分三區：裝置寬度切換靠最左、縮放控制靠最右、undo/redo/對齊等置中，避免全部擠在中間。
- 控制面板應提供「上一步 / 下一步」功能，讓使用者可復原與重做滑桿調整。
- undo/redo 按鈕建議放在預覽寬度切換下方或附近，使用圖示按鈕呈現。
- 除了按鈕，也要綁鍵盤快捷鍵：`Ctrl/⌘+Z` 上一步、`Ctrl/⌘+Shift+Z` 或 `Ctrl+Y` 下一步（在 `document` 上 `keydown`，命中時 `preventDefault`）。快捷鍵要走跟按鈕同一套 history 函式，不重整頁面、不清空目前調整值。
- history 應記錄每次完成的控制值變更；拖曳滑桿時可在 `input` 階段即時預覽，但只在 `change` 或同等完成事件時寫入一筆歷史。
- undo/redo 不應重整頁面、不應清空使用者目前調整值。
- 若使用 `localStorage` 保存控制面板狀態，undo/redo 後也要同步保存目前 state。

#### 位置偏移與對齊參考線

- X/Y 位移滑桿在接近 0 時應支援 snap：建議在 `±3px` 或等比例單位內自動歸零。
- snap 到 0 時，數值顯示要清楚標示「置中」或以控制版 accent 狀態顯示。
- X 歸零時可顯示垂直參考線與「垂直置中」標籤。
- Y 歸零時可顯示水平參考線與「水平置中」標籤。
- 參考線應短暫顯示後淡出，建議約 `0.7s`。
- **移動時的智慧型參考線**：拖曳變形框本體時，必須自動顯示動態對齊參考線，不需要使用者手動從尺標拉出輔助線。拖曳期間，比對移動中選取物件的 left/center/right 與 top/middle/bottom，對上預覽舞台與其他所有可選元素的 left/center/right、top/middle/bottom。當距離落在小範圍內（約螢幕 `6px`）時，自動把拖曳位移磁吸到最近參考線，繪製約 50% 不透明度的綠色垂直／水平參考線，並顯示即時移動數值，例如 `X 12 px   Y -4 px`。這些數值與參考線只屬於操作 overlay；滑鼠放開時必須清除，且不得輸出到確認結果。
- **標尺＋手動參考線**：工具列上的參考線按鈕必須改為「顯示／隱藏標尺」，不可再作為置中線釘選按鈕。綁定 `Ctrl/⌘+R` 切換標尺，並 `preventDefault()` 避免瀏覽器重新整理。標尺外觀要像設計軟體的邊緣標尺：淺色中性底、深色短刻度沿水平標尺下緣與垂直標尺右緣排列，不可做成一般格線填滿；標尺工具列圖示的視覺粗細也要與鄰近工具列圖示一致。標尺顯示時，水平與垂直標尺要固定在預覽 viewport 邊緣，不隨畫布內容漂走；標尺條本身厚度固定，但刻度間距要跟著 preview `zoom` 比例更新，讓 170%、220% 時仍然是在量 stage 像素。使用者可從水平標尺拖出水平手動參考線、從垂直標尺拖出垂直手動參考線。點擊手動參考線會選取它；選取狀態用淺綠色線條表示，但不可替線加上粗外框或 box-shadow 包覆。點擊預覽背景或一般預覽元件時，要取消手動參考線選取。焦點不在輸入框時，按 `X`、`Delete` 或 `Backspace` 可刪除目前選取的手動參考線。手動參考線座標必須存成未縮放的 stage 座標，再以 `stage origin + 參考線位置 × zoom` 渲染；不可存 viewport 原始像素，否則畫布縮放時參考線會漂移。手動參考線與像素讀數只屬於操作 overlay，在控制面板開啟期間保留，並在捲動、縮放、resize 時同步更新；靠近上方時讀數必須避開工具列按鈕，顯示在工具列下方，不得被按鈕遮住，也不得輸出到確認結果。
- **手動參考線也要參與吸附**：拉出或移動手動參考線時，要用與智慧型參考線相同的小範圍 threshold，自動吸附到預覽舞台的邊緣／中心與附近可選元素的邊緣／中心／中線，並在像素讀數上顯示命中的對齊目標。拖曳已選元件時，也要把既有手動參考線納入智慧型參考線候選，讓元件可以吸附到使用者自己拉出的參考線。
- 支援 PS 式多選相互對齊：`Shift+點擊`（預覽或選取列皆可）切換加入/移出多選集合（最後一個不可清空），`cur` 仍為最後互動的主要元素、控制面板顯示它。選到 ≥2 個時顯示一排對齊鈕（左/水平置中/右、頂/垂直置中/底），放在 undo/redo 同一排、<2 個時隱藏。對齊以「選取集合的外框（bbox）」為基準，靠位移屬性（translate `_tx`/`_ty`）達成並記進 history。注意：被選元素必須是「可被 transform 位移」的元素（block／inline-block／flex 或 grid 子項等）；純 `display:inline` 元素 transform 無效，需先確保它是 flex 子項或改 inline-block。同列（已被 flex 對齊）的元素做垂直對齊時位移可能極小、看起來像沒反應，屬正常。
- 對齊控制要依容器 `display` 選對屬性，否則套了畫面不動：
  - **水平對齊**對 flex／inline-flex 容器套 `text-align` 對其子元素無效，必須改套主軸屬性；非 flex 元素才用 `text-align`。
  - **垂直對齊**單靠 `text-align` 永遠做不到，必須用交叉軸屬性（flex 容器）；非 flex 元素要垂直置中時，以最小副作用包成 `display:flex; align-items:center`。對齊面板除了水平排（靠左/置中/靠右），也要提供垂直排（頂端/置中/底部）。
  - **軸要看 `flex-direction`**：`row` 時水平=`justify-content`、垂直=`align-items`；`column` 時兩者對調。先用 `getComputedStyle(el)` 讀 `display` 與 `flex-direction` 再決定屬性。值對應：起點→`flex-start`、置中→`center`、終點→`flex-end`。
  - 對齊輸出要帶上「實際生效」的屬性（flex 容器輸出 `justify-content`／`align-items`），確認時才不會誤改成無效的 `text-align`。

#### 固定介面設計（Figma／PS 風・由樣板內建）

以下是樣板已實作、必須維持一致的介面設計。換新元件時照用，不要改樣式或重排：

- **控制版 accent 色**：固定 shell 使用紅橘 accent 系統，不再使用 Google blue。`--tw-accent:#e04120`、`--tw-accent-strong:#b93419`、`--tw-accent-soft:#fde9e4`、`--tw-accent-softer:#fdf2ef`、`--tw-accent-glow:rgba(224,65,32,.15)` 是 active button、選取 layer row、hover/selection frame、focus ring、range/checkbox accent、ruler readout、confirm action 的唯一來源。生成控制版時不可再引入 `#1a73e8`、`#1765cc`、`#e8f0fe` 或其他藍色 accent hard-code。綠色 smart guide / manual guide 只保留給對齊回饋語意使用。
- **帶拉寬手把的三欄版面**：外殼使用 `grid-template-columns: var(--tw-left-w, 230px) 6px 1fr 360px`。左＝Layers/Pages 樹、6px 軌道＝可拖曳拉寬手把、中＝預覽舞台、右＝控制面板。左欄必須能垂直捲動且可拉寬，拉寬後的寬度要用 `localStorage` 記住，深層圖層名稱才可讀。左欄過窄時，圖層名稱用 `text-overflow:ellipsis` 顯示 `...`；眼睛等 row 控制要作為 hover 浮層出現在 row 內，不要永久佔用文字寬度。
- **Pages 是真實可切換目標**：若同一個控制板包含多個頁面／畫面／狀態，左側 Pages 區必須把每個目標列成可選項。點擊 page 時要一起切換 preview DOM/root、Layers tree、`SEL`/`OUT_SEL`、目前選取與確認輸出 context。不要放不能切換的裝飾性 page row。若多頁拆成不同 HTML 檔，必須明確標示/連結，不要暗示可在同一面板切換。
- **三段式控制面板**（解決「確認列浮在清單中間」破圖）：`.tw-right` 為 `display:flex; flex-direction:column; height:100vh; overflow:hidden`；固定 header（`flex:none`）＋ 可捲動 `#panel`（`flex:1; min-height:0; overflow-y:auto`）＋ 固定底部確認列（`flex:none`）。**不要用 `position:sticky` 的底部確認列**。
- **Figma section 標題**：九大分類用折疊區塊（`cat(title, open, [...])`），標題無數字前綴（純文字，不要「① ⑥」之類）。
- **可打字數字框**（取代滑桿）：`numBox(prop,label,min,max,step,unit,init,snap)` 產生 `<input type=text inputmode=decimal>`，方向鍵 ↑↓ 步進（Shift×10）、Enter 失焦、聚焦全選。X/Y 欄位只能在精準 `0` 時顯示 Center 狀態，絕對不可把使用者輸入的 `1`、`2`、`3` 這類小數值自動歸零。
- **多欄格線**：相關欄位用 `fieldGrid(cols, fields)` 併成兩欄/四欄（例如內距上下左右、陰影 XYBlur）。
- **拖曳刷數值（Figma scrub）**：`attachScrub(...)`，在 label/icon 上水平拖曳改值（1px≈1 step），拖曳中即時預覽、放開才寫一筆 history。游標 `ew-resize`。**不放滑桿**。
- **PS 式圓角**：`radiusControl(key)` 為 2×2 四角長手寫欄位（tl/tr/bl/br）＋中央鏈結鈕；連動開啟時，調一角直接把值套到其餘三角（直接呼叫 `applyNum`，**不要用 dispatch event**，避免綁定順序 bug）。右欄角落欄位要鏡射角落位置：數字在左、圓角圖示在右。
- **位置偏移用圖示對齊鈕**：`alignTools(key)` 三排（框位置／框內文字／垂直對齊），用圖示按鈕（非文字）；依容器 `display`/`flex-direction` 選對屬性（flex 用 `justify-content`/`align-items`，非 flex 才 `text-align`；垂直置中不可只靠 `text-align`）。
- **多選時樣式控制要套到所有選取 key**：當 `multiKeys().length > 1` 時，一般樣式控制（字型、間距、尺寸、圓角、位置、顏色等共用 CSS 屬性）必須更新全部選取 key，不可只更新 `cur`。文字內容編輯預設仍維持單一 key，避免多選標籤時不小心把所有文案改成同一段字。
- **右鍵 context menu／格式複製**：在預覽元件上按右鍵時，要開一個精簡深色選單。必備動作：`Copy style`、`Paste style here`、`Paste style to same kind`、`Select deepest layer`、`Select next outer layer`。這是格式刷功能：要從目標目前**實際渲染後的 computed style** 複製字型、填色、邊框、圓角、陰影/effects、SVG paint 等樣式/控制格式，不可只複製控制版 state 裡已經被拖過的值。不得複製文字內容、圖層名稱、選取狀態、位置偏移、transform、寬高、responsive frame size，除非使用者明確要求複製 layout。貼上時要把格式合併進目標現有 state，不可整包取代 state，避免抹掉目標原本的位置、文字或尺寸調整。`Paste style to same kind` 要把複製格式套到等價重複元件（同語意 suffix，如 `flowerBack` → `mushroomBack` / `hiddenBack` / `seedlingBack`；否則用相同 layer label），讓使用者調好一個重複元件後可一次套到同類，不必逐一調整。自訂選單也必須攔截選取 overlay（`#tw-tbox`、選取框 overlay、控制點）上的右鍵，不可只綁真實預覽 DOM；否則已選取元件上按右鍵會跳出瀏覽器原生選單。
- **computed 顏色解析必須保留透明度**：顏色、陰影、格式複製 helper 必須同時解析舊式逗號 RGB（`rgba(0,0,0,.12)`）、新版空白／斜線 RGB（`rgb(0 0 0 / .12)` 或百分比）、8 位 hex，以及瀏覽器回傳的 `color(srgb ... / alpha)`。遇到有效的新式顏色字串時，絕對不可 fallback 成不透明黑色；否則使用者第一次拖陰影或貼上格式時，畫面會爆成大片黑影。
- **左側 Layers/Pages 樹**：`LAYERS` 巢狀結構鏡射真實 DOM；`renderLayerNode` 遞迴渲染、有 key 可點選、Shift 多選、分組節點可摺疊；帶 key 的 row click 必須先 `e.stopPropagation()` 再 `pick(...)`，避免子圖層點擊冒泡到父圖層、覆蓋使用者真正想選的 layer；`refreshSelectionUI` 同步 `.on` 狀態與 `#cur-name`。滑過任一有 key 的 layer row 時，必須用 `rectFor(key)` 在預覽中顯示對應元素的 accent hover 框（`#tw-hover`）；hover 不得選取該圖層、不得改 `cur`、不得寫 history、也不得污染元素自身 `box-shadow`。hover overlay 要獨立於 `#tw-tbox`，`z-index` 略低於選取框、`pointer-events:none`，並在 scroll、resize、zoom、寬度切換、空白鍵拖動畫布時同步更新位置。當使用者在預覽中選取元素時，必須自動展開所有父層 layer group，並把對應 layer row 捲到可見範圍。每個帶 key 的 layer row 必須有眼睛 toggle，只用 preview-only class 隱藏／顯示該預覽元素；眼睛按鈕預設隱藏，只在滑過 row 或鍵盤 focus 時顯示，避免圖層樹過度雜訊。眼睛是 row 內的 hover 浮層，不是永久佔位的 flex 欄位；hover 時可以蓋住長名稱尾端，但閒置時不得固定保留寬度、不得提早壓縮文字。layer 可視性切換不得輸出到確認結果。
- **Photoshop/Figma 式畫布平移**：預覽區必須支援按住 `Space` 後拖曳來平移可捲動的預覽畫布，特別是 zoom 放大後用來查看被擋住或超出視窗的部分。在 input/select/textarea/contenteditable 中輸入時要忽略空白鍵捷徑，toolbar 仍可點擊；平移期間暫時停用 preview hit-testing，並同步呼叫 overlay 更新函式，讓選取框/hover 框位置不漂移。空白鍵平移優先於變形框/選取 overlay：`.space-pan` 啟用時，`#tw-tbox`、`#tw-selected`、hover overlay、parent overlay 應 `pointer-events:none`，讓使用者即使按在已選取元素或選取框上，也是在平移畫布，不會變成移動或重新選取元件。scroll、resize、空白鍵拖動畫布都必須走同一個 `requestAnimationFrame` 排程的 overlay 同步函式，等 scroll 位置更新後再重畫選取框/hover 框/標尺，不要用舊座標直接畫框。
- **變形框（選取框＝可變形）**：選取框是疊在 `#tw-left` 上的純 overlay（`#tw-tbox`，**不混入 box-shadow**），單選時顯示 8 向手把＋頂部旋轉手把＋即時 px 尺寸標籤。語意：**拖四角＝等比縮放**（寫 `_scl`，以中心為原點，比例抵銷 zoom）、**拖四邊＝改 `width`/`height`**（螢幕位移 ÷zoom 換 CSS px）、**框內拖移＝改 `_tx`/`_ty`**（螢幕位移 ÷zoom 換 CSS px；不可把小位移自動 snap 成 0；未拖動的單擊用 `elementFromPoint` 選底下元素，支援 Shift 多選）、**旋轉手把＝改 `_rot`**。多選時只顯示群組外框＋整組移動，隱藏縮放/旋轉手把。所有拖曳即時 `applyEl`＋同步對應數字框（`i-width`/`i-height`/`i-_scl`/`i-_rot`/`i-_tx`/`i-_ty`），放開才 `commit()` 寫一筆 history，並進確認輸出。位置由 `updateTBox()` 統一使用 `rectFor(key)` 相對 `#tw-left` 換算；`rectFor` 必須過濾 0 尺寸 rect，並提供 SVG/icon fallback（例如 `getBBox()` + `getScreenCTM()`），讓被選到的 SVG、mask icon、純視覺 span 都能顯示選取框。`scheduleTBox()` 於 `applyEl`／zoom／寬度切換／捲動／resize 觸發。加入全域方向鍵微調：焦點不在 input/select/textarea/contenteditable 時，Left/Right/Up/Down 讓目前選取 key 移動 1px，Shift 為 10px；上鍵減少 `_ty`、下鍵增加 `_ty`，讓移動方向符合畫面直覺。注意：純 `display:inline` 元素（如文字、tag span）transform 與 width/height 皆無效，變形框對它們不會動——與位置偏移控制同一限制。`_rot` 數字框範圍需放寬到 `-180..180` 以配合旋轉手把。**`#tw-tbox` 的 `z-index` 必須高於預覽元件本身的 `z-index`**（例如某些卡片容器自身帶 `z-index:100`，故 overlay 預設用 `z-index:99999`）；否則 overlay 會被元件內容蓋住，只剩最外緣那幾根 px 露出來——表現為「只有最外層容器看得到 accent 框，內部元件都沒有框」。換新專案時，若元件樹有更高的 `z-index`，overlay 要再往上加。**特別注意 stage reset（#9）中和 production overlay 時要連 `z-index` 一起歸零**：很多全螢幕 overlay／彈窗在 production 帶著極高的 `z-index`（例如 `z-index:100300`），這個高值配上 `position` 會自成一個 stacking context，把整個元件疊到 `#tw-tbox`（即使是 99999）之上，導致 accent 框整個被蓋住。在 #9 把 overlay reset 成舞台元素時，除了中和 `position`/`display`/`background`，也要加 `z-index: auto !important;`（或一個低值），讓它在舞台內正常堆疊；那個 production 的高 z-index 在隔離舞台上沒有意義。
- **隱藏 DOM / pseudo-element 視覺 rect**：若某個 layer key 指向真實 DOM，但該 DOM 在正式 CSS 中被刻意隱藏（`display:none`、`visibility:hidden`、`0×0`），而實際可見畫面是由父層、`::before`/`::after`、`mask`、或 `background-image` 畫出，不能讓該 layer 沒有 accent 框。必須新增專案專屬 `VISUAL_RECTS` map，並在真實 rect 無效時讓 `rectFor(key)` 改走 `visualRectFor(key)`；若可見父節點本身有效但要代表 pseudo-element 子區域，允許用 `force:true` 強制走視覺 rect，讓 accent 框框住 pseudo 像素而不是整個父層。隱藏節點可用 `type:'alias'` 指到另一個可見 key；pseudo-element 子區域則用 `type:'rect'` 或專案自訂規則計算。若語意 key 的 DOM 是隱藏或 pseudo-rendered，也必須加 `EDIT_TARGETS` 或 pseudo-element 寫入路徑，讓控制數值真的作用到可見表面；「有 accent 框但控制寫到 `display:none` DOM 或錯誤父層表面」是不合格狀態。若可見 pseudo/composite layer 使用 `filter: drop-shadow(...)` 而不是 `box-shadow`，陰影控制必須從真正渲染來源解析，例如 `getComputedStyle(wrapper, '::before'/'::after').filter` 或實際繪製層，再透過 pseudo 寫入路徑回寫 `drop-shadow(...)` 堆疊（例如 scoped CSS 變數 `--tw-back-filter`）。不可從父元素空白/預設的 `filter` 初始化 pseudo 陰影控制，否則使用者第一次拖 X/Y/Blur 就會把原本多層陰影整組抹掉。空的 pseudo 陰影狀態（`_shadows: []`）不代表可見 pseudo layer 沒有陰影；除非另有明確的使用者關閉/刪除意圖旗標，否則面板渲染或 live apply 前都必須從原始/預設 pseudo 陰影堆疊補回來。若 pseudo 子層自己的陰影移除後仍看得到陰影，是因為父層 wrapper 也有 `filter: drop-shadow(...)`，控制版必須在同一個固定 `Shadow` 類別內用子段明確暴露父層陰影（例如 `Backboard` 與 `Artwork`），不可新增額外頂層類別；使用者不用回圖層樹猜父層，但右側面板類別結構仍要穩定。陰影層 row 必須可收合、可用眼睛暫時隱藏、可用圖示按鈕刪除；隱藏只是不輸出該層，不等於刪除，刪除則必須先還原 target 原始 style 再重播剩餘 active state，避免舊陰影殘留。Layers 樹可以保留語意名稱，但 selection / hover / click-to-drill 必須框住並編輯實際看得見的像素。
- **工具列固定**：預覽工具列（版位切換、undo/redo、reset/zoom）必須固定在 viewport 上方，並高於所有預覽 overlay，但左右邊界只能落在中間 preview 欄內。不可用整個視窗的 `left:0; right:0`，否則會蓋到 Layers 欄與右側控制欄。標準 shell 用 `left:calc(var(--tw-left-w, 230px) + 6px); right:360px;`，搭配高於 `#tw-tbox` 的 `z-index`（例如 `100002`），wrapper 設 `pointer-events:none`、工具列群組設 `pointer-events:auto`。toolbar 的 click/mousedown 必須 `stopPropagation()`，避免版位、縮放、undo、多選對齊按鈕冒泡到 preview 背景而觸發取消選取。
- **undo/redo**：按鈕在預覽寬度切換附近＋鍵盤 `Ctrl/⌘+Z` / `Ctrl/⌘+Shift+Z`；走同一套 history 函式，不重整、不清空目前值。
- **專案 root key 必須取代所有樣板 root 特例**：若把樣板 demo root key（`card`）換成 `homeScreen`、`modalRoot` 等，所有相關特殊邏輯也要同步替換：初始 `cur`、啟動時 `pick(...)`、root/card transform var 處理、Size/Radius 分類是否顯示、對齊時的 parent 邏輯、以及啟動時 stage 寬度。殘留 demo key 會讓 reset／transform／selection 行為悄悄壞掉。
- **fixed / production-positioned 元件要改成 stage-relative**：production 裡的 `position:fixed` 或全域 overlay（bottom nav、modal、dock）在 `#card-stage` 內必須用 scoped project CSS reset。使用 stage-relative 定位，例如 `position:absolute; left:50%; transform:translateX(-50%); width:min(MAX_WIDTH, calc(100% - SIDE_PADDING));`，不要用只在單一 stage 尺寸成立的固定 `left + width`。該元件在每個裝置寬度都要置中且完整可見。
- **SVG/icon 可視性保護**：若 icon 的顯示仰賴真實 stylesheet，stage 預覽要保留同樣的渲染表面。依 CSS 顯示的 inline SVG 要補 scoped fallback：`width`/`height`，以及正確的 inactive/active `stroke`/`fill` 行為（線型 icon 用 `stroke:currentColor`，active 填滿 icon 用 `fill:currentColor`，cutout 也要復原）。不要讓 reset/undo 把 SVG 原本的 inline `width`/`height` style 清掉。
- **SVG 來源遷移必須同步更新 Layers / selector**：若真實專案把 icon 從 CSS pseudo、background、mask 或佔位 `<span>` 改成 inline SVG，控制面板必須立刻鏡射新的實際渲染 DOM。不可沿用舊 pseudo layer 名稱、隱藏 span 目標、`PSEUDO_BADGES`、`PSEUDO_BADGE_BACKS`、`PSEUDO_BADGE_ICONS`、`PSEUDO_*` 輔助映射、`NON_PICKABLE`，或仍指向舊實作的 `SEL` / `OUT_SEL` selector。要把真正的 `<svg>` 作為 `LAYERS` 裡獨立原子子層、允許選取，讓 `SEL` / `OUT_SEL` 指向 SVG 本體，並使用 SVG paint 控制（`fill` / `stroke` / `stroke-width`），確保改色會作用在可見圖示上。
- **遷移成 inline SVG 後不可保留 mask icon class**：若舊圖示是 `<span class="cat-icon ...">`，靠 CSS `background-color` + `mask` 渲染，而預覽或專案已改成 inline SVG，SVG 不應繼續保留舊的 mask 驅動 class，除非已完整中和該 class。優先使用新的 SVG 專用 class（例如 `.folder-category-svg`），並把 `SEL` / `OUT_SEL` 一起改指向該 class。若為了命名必須保留舊 class，preview scoped CSS 必須對該 SVG 設定 `background:transparent`、`background-color:transparent`、`mask:none`、`mask-image:none`、`-webkit-mask:none`、`-webkit-mask-image:none`；否則 SVG path 與舊 mask background 會同時渲染，造成圖示重疊/重複。
- **SVG paint 要同時寫 style 與 attribute**：SVG 子節點可能帶硬寫的 `fill` / `stroke` attribute，所以即時改色必須同時對 root SVG 與可上色子節點寫 `node.style.setProperty(prop, value, 'important')` 與 `node.setAttribute(prop, value)`。reset/undo 必須快取並還原子節點原始 inline style，以及原始 `fill`/`stroke` attribute。
- **SVG 格式複製要讀可見子節點 paint，不讀根節點預設值**：複製 SVG 樣式時，`fill` / `stroke` 必須從實際可上色子節點（`path`、`circle`、`rect` 等）解析後的 computed style 取得。忽略 `none`、透明、以及 cutout 這類非主色節點。除非根 SVG 的預設黑色就是實際可見子節點顏色，否則不可把根節點預設 computed black 當成格式複製來源；不然彩色圖示貼到另一個圖示會變黑。
- **選取提示低干擾**：被選到的 key（單選或多選）可以在目標元素本身加低干擾視覺備援，例如 `.tw-pick { outline:2px solid var(--tw-accent-frame); outline-offset:-2px; }`，但 **SVG 目標或目前有 CSS `filter` 的目標不可加真實元素 outline**。CSS filter 會把 outline 一起納入濾鏡渲染，所以 SVG drop-shadow 可能讓選取框看起來也被加陰影或重複。每個 selected key 仍必須用 `rectFor(key)` 額外畫一層獨立 overlay（`#tw-selected` 內的 `.tw-sel-frame`）；這是必要的，因為 SVG、masked icon、inline span、很小的視覺節點不一定可靠顯示 CSS `outline`。`#tw-tbox` 是變形手把／群組外框，不是唯一可視選取提示；若它的 rect 太小、被裁切或暫時算不到，被選元素仍必須看得出來。**禁止把選取框混入 `box-shadow` 或 `filter`**，陰影/玻璃預覽必須保持純粹。
- **點擊鑽選＋點背景取消（Figma 風選取，取代 `closest('[data-pick]')`）**：用 `closest('[data-pick]')` 選取有兩個盲點——(1) 合成圖層（如文字 Hug、裸文字 span 不存在時）沒有 DOM 節點可命中；(2) 子元素填滿父元素（如圖示填滿按鈕）時，永遠只命中最內層，選不到父層；使用者只能靠圖層樹選，很不直觀。正解：**改用矩形包含＋面積排序**。`clickStackAt(x,y)` 只掃描同時存在於 `SEL` 與 `LAYERS`（`LAYER_LABEL`）且未標記 `NON_PICKABLE` 的可選 key，用 `rectFor(key)`（文字角色用 hug 矩形）取得矩形，收集所有「包含該點」的 key，依面積由小到大排序（最小＝最深）。只為 selector、visual rect、舊 DOM 或 pseudo 實作細節保留的輔助 key 必須標記 `NON_PICKABLE` 或不要放進 `LAYERS`，否則使用者會選到沒有對應 layer row 的隱形框。`pickAt(x,y)`：若 `cur` 已經是游標下最深層（`stack[0]`），才選下一個外層；否則直接選 `stack[0]`。不可依目前已選外層在 stack 內的位置往外循環，否則使用者選著父層時點小 SVG 會跳到更外層，反而很難點到圖示。效果：第一次點圖示/文字就選最深層；在同一點重複點擊才逐層往外鑽選。`shift+click` 走加選（`stack[0]`）。
- **點背景空白取消選取**：點到元件以外的畫布（不在 `#card-stage`、也不在變形框 `#tw-tbox` 上）要 `deselectAll()`：`multi={}; cur=null;` 後 `refreshSelectionUI()`＋`buildPanel()`。配套必須讓「無選取」狀態安全：`buildPanel` 開頭 `if(!key){ 顯示提示文字 return; }`（否則 `getComputedStyle(elFor(null))` 會丟錯）；`refreshSelectionUI` 的 `cur-name` 要處理 `cur===null`；`updateTBox` 在 `multiKeys()` 為空時本來就會隱藏變形框與父框。`#card-stage` 的 click handler 用 capture＋`stopPropagation`，所以元件內的點擊不會冒泡到背景；只有點到外層 `#tw-left` 灰底才觸發取消。
- **巢狀框＋文字 Hug（單一元素拆框/文字時的選取框，Figma 風）**：當「框」與「文字」是同一個元素的兩個邏輯圖層（見「元件原子化拆解」的單一 `<input>` 拆分），文字層的選取框**不能**直接用 `el.getBoundingClientRect()`——那會跟框層一樣大。文字沒有自己的 DOM 節點，要**量測實際渲染文字**來貼齊（Figma 的「Hug」）：用一個 off-stage 鏡像 `<span>`（`position:fixed` 移到畫面外、`white-space:pre`、複製 `font-family/size/weight/style/letter-spacing/line-height`），塞進目前顯示文字（`input` 取 `value||placeholder`、其他取 `textContent`）量 `offsetWidth/Height`，再依元素的 padding/border、`text-align`（left/center/right）與垂直置中算出文字在元素內的矩形；尺寸都要乘 `zoom`（`getComputedStyle` 回傳未縮放 CSS px，`getBoundingClientRect` 回傳已縮放螢幕 px）。把變形框統一改走一個 `rectFor(key)`：文字角色→量測矩形，其餘→元素矩形。配套：選到文字層時，把**父元素**用虛線框 overlay（`#tw-parentbox`）標出（Figma 巢狀提示）；hover 任一含文字子層的元素時，同時顯示「框實線 overlay＋文字虛線 overlay」（`#tw-hover`／`#tw-htext`，皆 `pointer-events:none`、不污染 box-shadow）。文字 Hug 框因寬度由內容決定，要**隱藏邊手把**（只留四角＋旋轉）；**四角拖曳改 `font-size`**（不是 `_scl`，因為 `_scl` 會縮放整個元素），邊手把停用。注意：框與文字共用同一元素，所以位移/旋轉/縮放這類 transform 本就無法各自獨立（同一個元素），文字層真正獨立的只有字型；Hug 框的角落手把對應到 `font-size` 才合理。
#### 控制面板技術規範

- 每個 input 要有穩定且唯一的識別 id，例如 `id="i-{rowId}"`；數值顯示可用 `id="v-{rowId}"`。
- 所有 input 更新應走統一處理函式，例如 `ch(id, value, pos)` 或同等集中式 handler。
- 預覽更新應集中在 `applyDOM()` 或同等函式，依目前選取元素 key（例如 `cur`）套用樣式。
- 陰影只在樣式套用函式中寫入真正的 `box-shadow` / `filter` 數值，不混入選取狀態。
- 控制項應可從目前 CSS 初始值還原；若屬性缺失，使用明確預設值並在輸出中保留欄位。
- 對使用者正在調好的數值，新增控制項時不得重設既有數值；若可行，使用 localStorage 或明確的確認輸出保存狀態。
- **undo/redo/reset 要還原「到原始 inline style」，不可清空**：忠實還原常讓 preview 元素帶著**真實的 inline style**（例如 `.batch-loc-row` 從 JS 模板帶 `display:flex;gap:4px;…`）。`restore()` 切忌用 `el.style.cssText=''` 把整串清掉——那會連真實 inline 一起抹除，導致 undo/reset 後版面跑掉（要重新整理才復原）。正確做法：開場（init，且在套任何 override 前）用 `captureOrigStyle()` 快取每個 keyed 元素的 `getAttribute('style')` 到 `ORIG_STYLE`，`restore()` 改成 `el.style.cssText = ORIG_STYLE[k] || ''` 再重套 state override。與文字內容的 `ORIG_TEXT` 同一套「先快取原值、還原回原值」模式。
- **每次 live apply 都必須可逆**：套用某個 key 的目前 state 前，先把該 key 的 edit target 還原到快取的原始 style，再把共用同一個 edit target 的其他 key 目前 state 補套回去。這可避免使用者刪除 effect、移除漸層、切換無填色、或調 SVG paint 後留下舊值。不要用全域亂清屬性解 stale effect；正確流程是先還原原始 target，再重播仍然有效的 state。
- **SVG 改色的 undo/reset 也要還原子節點 inline style**：若 `applySvgPaint()` 會把 `fill`/`stroke` 寫到 SVG 子節點（`path`、`circle`、`rect`、`line` 等），init 時也要快取每個子節點原始 `style`，並在 `restoreBaseStyle()` 內一起還原。只還原 keyed `<svg>` 本身，會留下子節點 inline paint override，表現為 undo/reset 後顏色回不去。

控制面板九大分類：

① 字型
   字級（px）、字重（100–900）、行高（em）、字距（em）、字體（select）、顏色

② 間距
   內距上/下/左/右（px）、外距上/下（px）

③ 尺寸
   寬度與高度各自提供 Fixed / Fill / Hug / Auto 模式。Fixed 寫 px；Fill 依父層 flex 軸向映射成 `width:100%`、`height:100%`、`flex:1 1 0` 或 `align-self:stretch`；Hug 寫 `max-content`；Auto 寫 `auto`。另包含最大寬度、比例（aspect-ratio）、等比 Scale % 控制（寫入與四角拖曳／變形框縮放相同的 `_scl` transform scale 狀態，讓使用者能用單一欄位放大或縮小，不必同時改 W/H）

④ 排版
   display（`block` / `flex` / `inline-flex`）、flex-direction（`row` / `column`）、flex-wrap、gap、align-items、justify-content。調整 direction / alignment / justification 時，若容器不是 flex，應自動補 `display:flex`，讓預覽立即生效；確認輸出必須列出實際 CSS declarations，不輸出抽象標籤。

⑤ 圓角
   整體圓角（px/%/cqw）、個別四角（左上/右上/左下/右下）、切換整體/個別模式

⑥ 位置偏移
   X 位移（translateX/left/right）、Y 位移（translateY/top/bottom）、旋轉（deg）、縮放（scale）、snap + 參考線

⑦ 陰影
   X offset、Y offset、Blur、Spread（均 px/cqw）、透明度（0–1）、必要時分拆前景/背景/互相遮蔽陰影

⑦b Effects
   可堆疊效果層，包含眼睛開關、效果類型下拉、刪除按鈕：Drop shadow、Inner shadow、Layer blur、Background blur、Glass。Effects 要使用控制版自己的簡化線性圖示，不要照抄其他軟體或截圖的圖示。Drop/Inner 寫 `box-shadow`；Layer blur 寫 `filter`；Background blur / Glass 寫 `backdrop-filter` / `-webkit-backdrop-filter`；Glass 可額外寫半透明背景色。**SVG 例外**：目標是 inline SVG 時，Drop shadow 必須寫 `filter: drop-shadow(...)`，不可寫 `box-shadow`；Background blur / Glass 也不可替 SVG 加背景色或 backdrop 方框，否則 icon 會多出不該存在的方形底。
   刪除 effect 後必須立即移除預覽 CSS：先還原 target 原始 style，再重播剩餘 active state；不可留下舊的 `box-shadow`、`filter`、`backdrop-filter` 或 glass background 值。

⑧ 磨砂玻璃
   模糊強度（blur px）、白色透明度（0–1）、頂部漸層差（%）、
   懸浮陰影（px）、頂部高光邊（0–1）、內層光感（0–1）

⑨ 顏色
   背景色、文字色、邊框色（均支援 rgba）、邊框寬度（px）
   - **無填色 toggle 是標準 paint 控制**：所有 paint 類 color row（background、border、fill、stroke、pseudo badge fill/border、可透明化的 mask icon color）都必須有小方框＋斜線的無填色按鈕。SVG fill/stroke 寫 `none`，background 類寫 `background:none` 或透明，border/text 類依情境寫透明。面板重建後按鈕仍要存在，且必須支援 undo/redo/reset。
   - **Linear gradient row**：非 SVG 的顏色表面可提供精簡線性漸層控制，包含即時漸層條、漸層條上的兩個可拖曳色彩 stop，以及透明度控制。拖曳 stop 直接改變該色標百分比；色票/透明度欄位寫入同一份 state。它寫入 `background: linear-gradient(...)`，需支援 history，並在確認輸出中列為 `background-gradient`。若目標元素本來已有漸層，初始 stop 顏色必須從該元素真實 `getComputedStyle(...).backgroundImage/background` 解析，不可用與畫面不一致的通用預設色。
   - **SVG icon 例外**：當選取的是 SVG 元素（如線條 pin icon）時，Color 分類要改出 **填色 `fill` / 外框 `stroke`**（兩者分開，各自帶一個會寫入 `none` 的無色 toggle）＋ **`stroke-width`**，而非 background/text/border——SVG 的可調顏色面是 `fill`/`stroke`，不是背景/邊框。無色 toggle 要用圖示按鈕（小外框方形＋斜線），不要用寫著「None」的文字按鈕。初始值一樣用 `getComputedStyle` 讀（`fill:none` → 預設勾無色並淡化色票；`stroke` 讀回 currentColor 解析出的 rgb），面板重建時也要保留 state 裡的 `#rrggbb` 色值。套用走 `el.style.setProperty('fill'|'stroke', val, 'important')`，確認輸出用 `fill:`／`stroke:`。判斷方式：`el.tagName.toLowerCase()==='svg'`。
   - **預設 SVG Icon 控制**：選取 inline SVG 時，在 Color 上方新增獨立「SVG Icon」分類，包含：等比 icon size（同時寫 width / height）、padding、stroke thickness、scale %、flip/mirror 按鈕、rotation preset、direction preset。這些控制必須支援 history、reset/undo，並輸出到確認結果。不可在沒有實際向量 path outlining 或專案真實 source API 的情況下暴露 trace / expanded-outline 控制；trace width 不等於 CSS `stroke-width`。

字體 select 建議選項：
- `inherit`
- `system-ui`
- `-apple-system`
- `Arial`
- `Georgia`
- `Noto Sans TC`
- `PingFang TC`

#### 文字內容編輯（預設支援）

每個產生出的控制面板都必須預設支援直接改「顯示文字」本身（系統文案、按鈕字、placeholder、標題等），而不只是調樣式。對有意義文字的元素，在「① 字型」分類**頂部**放一個「文字內容」輸入框；使用者不需要另外說出特殊的文字編輯模式。

- **依元素型別讀寫對應目標**：一般元素改 `textContent`；`<input>` 改 `placeholder`（顯示給使用者的提示文字；若顯示的是值才改 value）；`<select>` 預設顯示的是**目前選取的 `<option>`**，不是第一個——所以多半要用 `mode:'optionSel'`（讀寫 `el.options[el.selectedIndex]`），而不是 `option0`（第一個 option，常常是 placeholder「請選擇…」）。先用一張小對照表標明每個可編輯 key 用哪種模式。
- **預設自動偵測＋覆寫表**：一般葉節點文字、`<input>`、`<textarea>`、`<select>` 都要自動開文字欄位。保留 `TEXT_TARGETS` 供特殊情況覆寫（`false` 關閉某個 key；`{mode:'text'|'input'|'value'|'optionSel'}` 強制指定模式），避免自動偵測抓錯來源。
- **`<select>` 要忠實呈現「選取後」狀態＋資料驅動的選項文字**：預覽的 `<select>` 不能只放 placeholder 或自創選項。若使用者在意的是「選了某項之後」的樣子，預覽就要讓一個**真實選項帶 `selected`**，文字目標用 `optionSel` 讀那個選項。更重要：選項文字若是**資料／程式組出來的**（例如主題 = `emoji + 空格 + label`，由 `getThemeParts(t).value`／`composeThemeLabelRaw` 組出，emoji 來自 `THEME_EMOJI_MAP`），預覽選項就要照同一格式重現（含 emoji），不可自己編一組沒 emoji 的假選項——這是「預覽忠於真實 code」的延伸。確認輸出要在該 selector 註明「option 文字是資料，不是字面字串；要改的是主題資料／emoji map，不是某個 literal」。
- **`<input>` 的「輸入前／輸入後」雙槽編輯（雙 slot）**：使用者常想同時看／調**輸入前的提示字（placeholder）**和**輸入後的實際輸入值（value）**兩種狀態的文字＋字體。把 `<input>` 的文字模式設成 `mode:'input'`，拆成兩個 slot：`_ph`（輸入前／placeholder）與 `_val`（輸入後／value），各出一列「文字內容」輸入框（標清「輸入前文字（提示字 placeholder）」「輸入後文字（實際輸入內容）」）。其他模式維持單槽 `_text`。`ORIG_TEXT[key]` 由字串改存物件 `{_ph,_val}`，`slotsFor(key)` 回傳對應 slot 陣列。
- **若需求包含隱藏文字，每槽要可獨立隱藏／開啟**：每列文字旁加一個眼睛 toggle，存 `_phHidden`／`_valHidden` 旗標；隱藏時把該 slot 渲染成空字串但**保留原值在 state/ORIG_TEXT**，再開啟可復原。
- **placeholder 與 value 的字體必須各自獨立（KEY：要走 `::placeholder` 規則，不能用 `el.style`）**：一個 `<input>` 的 placeholder 和輸入值**共用同一份字體**，所以「placeholder 字級 30px、輸入值字級 16px」這種需求**無法用 `el.style` 達成**——`el.style.fontSize` 會同時改到兩者。正解：placeholder 的字型屬性用 `'ph:'` 前綴存（如 `state[key]['ph:font-size']='30px'`），再用 `applyPlaceholderStyle(key)` 把所有 `ph:` 屬性收集起來，注入一個每(state,key)專屬的 `<style id="tw-ph-{state}-{key}">`，內容是 `SEL[key]+'::placeholder{…!important}'`。`applyEl` 的元素屬性迴圈要**跳過 `_` 開頭與 `ph:` 開頭**的 key，避免把 placeholder 字型誤寫到元素本身。配一個 slot 切換鈕（「輸入後字體／輸入前字體」，變數 `TXSLOT`）決定「① 字型」的數字框是改元素本身屬性（`pfx=''`）還是改 placeholder 屬性（`pfx='ph:'`）；`cssVal` 讀 `ph:` 屬性時要從 `getComputedStyle(el,'::placeholder')` 讀回初值。`restore()` 要把所有 `style[id^="tw-ph-"]` 的內容清空。
- **即時更新預覽**：打字時即時寫入預覽 DOM；只在完成事件（`change`）寫一筆 history。
- **文字欄位要綁定自己的 key，不可依賴全域目前選取**：每列文字 input 要記住自己的 key（例如 `data-key`），更新 `state[textKey]` / `applyText(textKey)`。不要只讀全域 `cur`，因為多選、toolbar 互動、或 panel rebuild 都可能在使用者編輯文字時改變 `cur`。文字列 class 是 `.tw-text-edit`，不是 `.tw-row`，所以 `bindRows()` 必須另外綁定 `.tw-text-edit input`；只掃 `.tw-row` 會造成右側欄位看起來能輸入，但預覽 DOM 完全不變。
- **undo/redo/reset 要能還原**：開場先快取每個可編輯文字的原值（`ORIG_TEXT`），`restore()` 先把文字還原成原值再套用 override，避免殘留。
- **確認輸出**：以 `- text-content: 新字串` 送回；並附 NOTE 點明這是**顯示文案、不是樣式**，要去改真實字串來源（HTML 與／或 JS 模板，例如 template literal 裡的字面字串）。按鈕／標籤的裸文字節點要照「裸文字包 `<span>`」規則處理，並提醒真實模板也要補上同樣的 span。`<input>` 雙槽則分別輸出 `- placeholder: …`（輸入前）與 `- value (typed text): …`（輸入後）；placeholder 的字型屬性（`ph:` 那組）要拆成獨立的 `[OUT_SEL::placeholder]` 區塊輸出（去掉 `ph:` 前綴），與元素本身屬性的 `[OUT_SEL]` 區塊分開，提醒真實 code 要用 `::placeholder` 規則才掛得上。
- 只對「有意義的系統文案」開放此欄位；emoji／純圖示按鈕不需要。

#### 面板預設行為

- 磨砂玻璃分類：元件若無 `backdrop-filter`，預設折疊
- 確認按鈕送出前，面板顯示即將更新的 CSS 摘要供確認
