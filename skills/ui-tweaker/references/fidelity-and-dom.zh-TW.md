# 真實還原與預覽 DOM 規則（ui-tweaker 參考文件）

> 在填寫樣板佔位點 #4/#5/#6/#7（預覽 DOM、SEL/OUT_SEL、LAYERS）**之前**必讀。這些規則決定預覽如何忠實重現實際程式碼。

#### 真實樣式還原（最高優先・禁止手抄）

這是本 skill 最重要的規則，凌駕其他所有「還原樣式」字眼。預覽與控制面板的數值**必須由瀏覽器解析真實程式碼得出**，絕不可用眼睛估、也不可把 CSS 規則手抄進 widget。原因：真實畫面常由多層 `!important` override 疊出來（base 規則 + 設計系統層 + pixel polish 層 + 最終覆寫層…），手抄任何一層都會跟實際不符。

1. **連結真實來源，不要重寫**：widget 一律用 `<link rel="stylesheet" href="真實樣式表路徑">`（或 `@import` 真實檔案）載入專案實際的 CSS 檔，並用 `<link>`／同樣 `<head>` 載入專案實際使用的字體。多個來源檔就全部連結。**禁止**把 selector 的數值複製貼進 widget 的 `<style>`。
2. **用真實 DOM 結構與 class**：預覽元素要用元件在專案裡真正的標籤、class、巢狀結構，不要自訂簡化版。若內容由 JS 動態填入（動態 id），要照抄真實 JS 產生的 markup——包含 icon、`<span>` 包法、以及格式化字串（例如日期格式 `今日 14:30` 而非自己編的 `2025/06/02 14:30`）。先去讀產生該 DOM 的 JS 函式再寫。
   - **icon 占位禁令（照抄真實 icon 元素，不得替換）**：icon 的實作方式不只一種——可能是 inline `<svg>`、也可能是 class span + CSS 背景/mask（例如 `<span class="cat-icon cat-flower">` 搭配 `--cat-src:url(data:image/png;base64,…)` 背景圖、或 `mask-image`、icon font）。**一律照抄真實的 icon 元素與 class**，讓連結的真實樣式表去渲染它。**絕對禁止**用 emoji／unicode 字元／純文字（如 🌸🍄🌱、★、↕）去「代表」或「暫代」真實 icon：那會同時改掉外觀**和可被控制面板調整的樣式表面**（真實 icon 的可調屬性是 `background`／`mask`／`filter`／`fill`，emoji 則完全不同），導致預覽失真、確認輸出也對不上真實 selector。分不清 icon 怎麼來時，回去讀產生該 DOM 的 JS／CSS 再寫。
3. **初始值一律 `getComputedStyle` 讀**：所有控制項的初始數值，必須在「樣式表確定載入完成後」（`window.load` 或輪詢 sheet 就緒）用 `getComputedStyle(真實元素).getPropertyValue(prop)` 讀取真實渲染值，禁止填入基礎規則或估計值。
4. **套用調整要壓過 `!important`**：預覽即時套用一律用 `el.style.setProperty(prop, val, 'important')`，否則會被來源樣式表的 `!important` 蓋掉，導致拖了滑桿畫面卻不動。
5. **處理 JS 預設隱藏的子元素**：有些元素預設 `display:none`，靠 JS 在特定狀態才顯示（例如選單鈕）。預覽要強制還原成「真實出現時」的狀態（如 inline `style="display:flex"`），否則會漏畫元素。
6. **多值屬性要偵測並保護**：多層 `box-shadow`、漸層 `background` 等無法用單層滑桿無損還原的屬性，`getComputedStyle` 也讀不回單一值。偵測到多層/漸層時，該控制項要鎖定或顯示警告，避免單層滑桿把原本的多層值覆寫掉。
7. **確認輸出只含改過的屬性**：送回 `[UI Tweaker 確認]` 時，只輸出使用者實際調整過的屬性；沒動到的一律不進輸出，確保不會誤改使用者原本就滿意的地方。
8. **完成後實機驗證**：用瀏覽器讀 `getComputedStyle` 或截圖，確認預覽解析值與 app 實際值一致（例如挑 2–3 個關鍵屬性比對數字），再交給使用者。
9. **SEL selector 必須限定在舞台範圍——禁止裸 `[data-pick="key"]`**：填寫 `⟦PROJECT-SPECIFIC #5⟧` SEL map 時，不可用裸 `[data-pick="key"]` 屬性選取器——樣板左側圖層樹的 `.tw-lay-row` 也帶 `data-pick` 屬性，且在 DOM 順序上早於 `#card-stage`，`querySelector` 會先抓到圖層列而非預覽元素，導致面板讀錯初始值。正確寫法是把 selector 限定在舞台範圍：`#card-stage [data-pick="key"]`。
10. **控制版快照過期時必須重新同步來源**：已生成的 `ui-tweaker-*.html` 只是生成當下的專案快照。若使用者說真實程式碼已更新、但控制版仍顯示舊畫面，AI 必須立刻重讀真實 JS/HTML/CSS，並同步更新 preview DOM、`SEL`、`OUT_SEL`、`LAYERS`、`EDIT_TARGETS`、pseudo maps、visual-rect maps，以及專案專屬 stage reset。要主動搜尋殘留的舊 selector、已廢棄 hidden node、舊 pseudo layer 名稱、過期 CSS 變數或 fallback。Stage reset CSS 只能負責讓元件在舞台中可見、改成 stage-relative；若不得不替視覺值放 fallback，fallback 必須指向真實 stylesheet 目前的 source-of-truth 變數，不可沿用舊設計 token。交回控制版前，必須用真實瀏覽器 `getComputedStyle` 對照目前來源值重新驗證。

#### 自動解析 JS 動態樣板（Zero-Jargon Rule）

- 若目標元件是寫在 JS 檔案中的動態字串樣板或組件（如 template literals），AI 必須主動在預覽面板中，將其還原／mock 成瀏覽器最終渲染出的真實 HTML DOM 結構。
- 遇到與變數綁定的 inline style 或動態生成的文字節點，需自動解析並實體化到預覽 DOM 上，絕對不可當作純字串擷取。
- AI 需主動辨識並將帶有視覺屬性（如名稱輸入框、地點列、圖示、標籤、徽章、按鈕）的節點原子化，拆分成可獨立點擊的最小子元素，並掛載對應的點擊事件或 key。
- **嚴格禁止**：AI 不得要求使用者使用工程術語（如「還原 DOM」、「mock」、「原子層級」）來下指令。AI 需自行在背景完成上述所有結構轉換，只呈現可直覺點擊微調的預覽畫面給使用者。
#### 元件原子化拆解（Layers 要拆到拆不了為止）

`LAYERS`（樣板 `⟦PROJECT-SPECIFIC #7⟧`）要**依真實 code 把每個元件拆到原子層級**，不可只列大區塊：

- 先讀「產生該 DOM 的真實 JSX/JS/HTML」列舉所有原子元素，再建樹。
- 把混排元素拆開分別選取：icon vs 文字（如徽章＝時鐘 SVG＋時間文字、地點＝圖釘 SVG＋地點文字）、按鈕 vs 其圖示（選單鈕＋三點 SVG）、座標鈕框 vs 框內文字。
- **不要假拆 CSS 生成的 pseudo/composite paint**：如果某個視覺部件其實是單一 CSS pseudo-element（`::before` / `::after`）或一張 `background-image` / PNG / mask 合成圖，就把它視為**單一原子圖層**；除非你明確建立 synthetic DOM 替代層，且也能把等價變更寫回真實 source。不要把同一個 pseudo-element 假拆成「slot / badge / icon」等子圖層，否則 X/Y/size/color 控制會被綁在一起或完全無效。這類 composite layer 只暴露真實可寫回 source selector 的屬性，例如 pseudo transform/scale、用 `background-size` 控制 icon size、background-color、border、box-shadow。若 icon 是 PNG background，不要暴露 SVG 專用的 `fill` / `stroke` 控制。PNG icon size 要寫成**單值** `background-size`，例如 `70%`，讓瀏覽器保留圖片原始比例；避免 `70% 70%` 這類雙軸值，並限制控制範圍，避免圖片被圓形／圓角容器裁切到看起來變形。
- 重複清單每一項各算獨立元件（如標籤一/標籤二/標籤三，用 `:nth-of-type(n)`），且每個標籤再拆 emoji（`.theme-emoji`）與文字（`.theme-label-text`）。
- 純分組容器（無對應可調樣式）用無 key 的節點當分組標題即可。
- **裸文字節點**（真實 code 中文字直接放在容器裡、無獨立元素，如徽章時間、座標文字）：在預覽 DOM 包成帶 class 的 `<span>` 並加 `data-pick` 才能單獨選取/套樣式；此 span 真實 app 沒有，送 `[UI Tweaker 確認]` 套用時要在輸出點明「需於真實模板/JS（產生該 DOM 的函式）補上同名 span」，否則樣式掛不上。
- **單一元素也能拆成兩個邏輯圖層（框 vs 文字）**：拆分有兩種——(a) 真實兩節點（如星號鈕＝`<button>`＋`<svg>`、徽章＝SVG＋文字）本來就是不同 DOM，各給一個 key 即可；(b) **單一元素**（最典型是一個 `<input>`，框與文字其實是同一個 DOM）也常需要在圖層樹拆成「框 Box」與「文字 Text」兩層，方便使用者分開調。關鍵：框類屬性（`width`/`height`/`padding`/`border-radius`/`background`/`border`/`box-shadow`）和文字類屬性（`font-size`/`font-weight`/`font-family`/`color`/`letter-spacing`/`line-height`）本來就是同一元素上**互不重疊的 CSS 屬性**，所以「綁在一起」只是分類呈現問題，不是技術限制。做法：用一張 `ROLES` 對照表給每個 key 標 `'box'`／`'text'`／`'full'`，`buildPanel` 依角色過濾分類——`showText=(role!=='box')` 包住「① 字型」；`showBox=(role!=='text')` 包住其餘七類（間距/尺寸/圓角/位置/陰影/磨砂玻璃/顏色）。框 key 與文字 key **共用同一個 SEL（同一元素）**但各自只寫自己那組屬性，故 `applyEl` 不會衝突。圖層樹把文字層掛成框層的子節點（如「名稱框 Name box ▸ 文字 Text」）；預覽點該元素＝選到框層（`data-pick="nameField"`），文字層從圖層樹列點選。送確認時兩層各輸出一個區塊（box 區塊帶框屬性、text 區塊帶字型屬性），並標明 selector 相同、只是屬性分組。
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
  3. 掃 preview DOM 內所有 icon 位置：每個 icon 都要是真實的 icon 元素（inline `<svg>`／class span＋CSS 背景或 mask／icon font），不得出現 emoji／unicode／純文字暫代；發現暫代 → 回真實來源照抄真正的 icon markup。
  4. 對產生的 preview fragment 做 HTML 合法性掃描：`data-pick` 一律放在開啟標籤上（絕不可出現 `</span data-pick="...">`），每個結尾標籤都要完整（不可有截斷的 `</div`），且瀏覽器實際 DOM tree 要維持 `#card-stage`、root component、以及 stage 內 fixed 元件的預期巢狀。preview HTML 不合法時，就算 CSS 正確，scoped reset 也可能完全失效。
  5. 實機用 `getComputedStyle` 抽查 2–3 個元素（尺寸／`flex`／顏色），確認與真實卡片渲染一致。
- **stage layout 必須做多版位實機驗證**：產生面板後，至少用真實瀏覽器或 headless browser 測窄版／預設／寬版三種 stage 寬度。檢查：沒有水平裁切、responsive 子元素會跟著 stage 縮放、fixed/dock 元件置中且完整、SVG icon 正常渲染、reset/undo 後回到原始視覺狀態。只跑 key consistency script 不夠。
- 心裡（或輸出）保留一份「來源對應表」：每個 pick key → 真實檔:行（例：`loc → 產生該 DOM 的檔案:行號`、`menu → 模板檔:行號`）。對不上來源的元素不准進控制板。
