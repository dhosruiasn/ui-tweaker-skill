<!--
  繁體中文版說明文件 — 對應英文版 SKILL.md（同一份行為的中文翻譯）。
  此檔「不含」YAML frontmatter，因此不會被註冊成第二個 skill；
  唯一被當作 skill 載入的是 SKILL.md。修改行為時兩份要同步（見 CONTRIBUTING.md）。
-->

# UI Tweaker（繁體中文）

讓使用者在 AI coding assistant 中直接用滑桿微調 UI，確認後複製結構化數值，讓 AI 精準套用到程式碼，不需要來回用文字描述數值。

核心原則：所有調整都以可回填的結構化數值為主；不要只用「感覺」描述差異。

## Token 效率鐵則（最先讀）

面板樣板約 177KB。為控制 token 消耗，以下規則優先於本 skill 與參考文件中任何其他寫法：

1. **禁止把整份樣板 Read 進 context**。用 Bash `cp` 複製；禁止用 Write 重新輸出樣板內容。
2. **佔位點用 Edit 精準替換**：先 Grep `⟦PROJECT-SPECIFIC` 取得標記行號，只 Read 那幾個小區段，逐點用 Edit 替換。標記以外的內容必須 byte-for-byte 不動。
3. **迭代只用 Edit、禁止整份重寫**：修正或重新同步已產生的 `ui-tweaker-*.html` 時，只編輯特定區段，絕不整份 Write。
4. **驗證用 Grep／指定區段 Read**，不要把產生的面板整份讀回來。
5. **參考文件按需載入**：只在對應階段才讀 `references/` 下的檔案（見下方指引），不要一開始全部讀完。

## 分發與更新

此 repo 同時包成 Claude Code plugin marketplace（`.claude-plugin/`）與實際 skill 資料夾（`skills/ui-tweaker/`），可從 GitHub-backed marketplace 安裝與更新。其他 AI 工具可把 `skills/ui-tweaker/` 視為可攜式指令資料夾，用 `git pull`／submodule／symlink 同步；除非宿主 AI 真的支援即時推送更新，否則不要描述成 live push updates。

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
4. **確認「目標狀態」**：若元件有多種狀態（載入中／偵測中／完成／空狀態／展開 vs 收合／hover…），先確認使用者要調的是哪一個畫面狀態，並**完整還原那個狀態的所有欄位**，不可只做簡化子集。狀態判斷依據是產生該 DOM 的真實 JS（例如 `status === 'detecting'` 分支），照抄那一支的完整 markup。截圖只是輔助驗證、非必要前提。

---

### Step 2：同時產生「預覽畫面」＋「控制面板」

#### 一律從固定介面樣板開始（最高優先・禁止重畫 UI）

本 skill 附帶一份固定介面樣板 `template/panel-template.html`（相對於本 SKILL.md 所在資料夾）。**每次開控制板都要複製這份樣板當基底**，不可從零手寫 UI、也不可改動樣板的版面/CSS/builder，確保任何人、任何專案打開的介面都「byte-for-byte 一致」。

流程：
1. **用 Bash `cp` 把樣板複製**到使用者專案可由瀏覽器存取的位置（例如專案根目錄，檔名如 `ui-tweaker-<元件>.html`）。禁止整份 Read、禁止用 Write 重新輸出（見 Token 效率鐵則）。
2. 在複製出的檔案 Grep `⟦PROJECT-SPECIFIC`，**只**用指定區段 Read + Edit 替換這些標記點（其餘所有區塊原封不動）：
   - **#1** `<title>`
   - **#2** 字體 `<link>`（換成該專案實際字體）
   - **#3** 樣式表 `<link href>`（連結該專案真實 CSS，禁止手抄數值進 `<style>`）
   - **#4** `#card-stage` 內預覽 DOM（貼真實元件 markup，每個原子元素加 `data-pick`）
   - **#5** `SEL` map（pick key → widget 內查詢用 selector）；另有選用的 **#5b** `VISUAL_RECTS`、**#5c** `EDIT_TARGETS`、**#5d** `TEXT_TARGETS`、**#5e** `NON_PICKABLE`
   - **#6** `OUT_SEL` map（pick key → 對外輸出 CSS selector 名稱）
   - **#7** `LAYERS` 樹（依真實 DOM 拆到原子層級）
   - **#8** 確認輸出的「元件：」「來源：」字串
   - **#9 元件舞台 reset**：把元件根節點在 `#card-stage` 內從隱藏/定位 reset 成可見/靜態的那條規則（換成你的真實根節點 selector；保留 `--tw-card-transform` 變數；production 的 `z-index` 也要一併 reset）。只有當真實元件在 production 是隱藏/絕對定位時才需要動它。
3. 替換後，每個 `LAYERS` 有 key 的節點都要在 `SEL`／`OUT_SEL`／預覽 DOM 的 `data-pick` 三處有對應條目；交付前用腳本交叉檢查 keys 一致、`node --check` 驗證 JS。
4. 樣板已內建固定 UI（Layers 樹、三段式面板、九大分類、數字框、拖曳刷數值、undo/redo、transform box、尺規、右鍵選單…）。這些**不要重做或改樣式**。

> 若樣板檔不存在或被刪，才退回「依規格手寫」——此時讀 `references/template-behaviors.zh-TW.md` 取得完整介面規格；但只要樣板在，一律以樣板為準。

widget 是一個三欄 HTML 頁面：左側 Pages／Layers 樹（Figma 式）、中間元件即時預覽（連結真實樣式表還原）、右側控制面板（九大分類折疊區塊）。預設展開與目前元件最相關的分類。

#### 填佔位點前的必讀文件

- **`references/fidelity-and-dom.zh-TW.md`** — 填 #4/#5/#6/#7 **之前**必讀。涵蓋：真實樣式還原（連結真實 CSS、真實 DOM/class、`getComputedStyle` 讀初始值、`!important` 套用、禁止 emoji 代替真實 icon）、JS 動態樣板重建、`LAYERS` 原子化拆解（icon 與文字分離、裸文字節點包 `<span>`、Box/Text 角色拆分），以及開面板前的強制核對（grep 每個 class/id 對照真實原始碼、禁止自創 inline style、HTML 有效性、多寬度瀏覽器驗證）。
- **`references/template-behaviors.zh-TW.md`** — 只在需要時讀對應段落：處理專案特例（替換 demo root key `card`、stage reset 與 z-index、`VISUAL_RECTS`/`EDIT_TARGETS`、SVG icon 來源遷移、預設 stage 寬度）、除錯樣板行為、或手刻 fallback。樣板已內建其中所有行為。
- **Zero-Jargon Rule**：不得要求使用者使用「還原 DOM」、「mock」、「原子層級」等工程術語——AI 自行在背景完成結構轉換，只呈現可直覺點擊微調的預覽畫面。

---

### Step 3：確認按鈕觸發 AI 更新

面板底部「確認，更新程式碼」按鈕透過 `sendPrompt()` 把目前所有數值以結構化格式送回 AI assistant，例如：

```
[UI Tweaker 確認]
元件：首頁資料夾
來源：/path/to/source.css

[folder-wrapper]
  - width %: 114
  - left %: -7.5

[folder-glass-panel]
  - backdrop-filter blur px: 4

請直接更新對應程式碼
```

（單一元素也可用不分 key 區塊的扁平格式。）

收到以 `[UI Tweaker 確認]`（或英文 `[UI Tweaker Confirm]`）開頭的訊息後，**讀 `references/confirm-apply.zh-TW.md`** 並照做：解析數值 → 設計系統影響範圍確認（全域／元件／僅此處，預設僅此處）→ 多欄位更新走結構化解析/dry-run/寫入流程 → 驗證（build＋computed style 檢查）→ commit/push 邊界規則。

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

- 預覽要還原使用者實際樣式，不用假資料、不准自創 DOM（開面板前通過 `references/fidelity-and-dom.zh-TW.md` 的強制核對）
- 控制面板初始數值從使用者的程式碼讀取（`getComputedStyle`），不用預設值；未定義的屬性以 CSS 預設值為初始值
- 選取提示不得污染 `box-shadow`、`filter`、磨砂玻璃或陰影預覽
- 多狀態元件要先確認目標狀態，並完整還原該狀態的所有欄位
- 預設在「① 字型」頂部對有意義文字元素提供「文字內容」輸入框；確認時以 `text-content:` / `placeholder:` / `value (typed text):` 送回並註明要改真實字串來源
- 已產生的 `ui-tweaker-*.html` 是快照：使用者說真實程式碼改了時，重讀真實原始碼並用 Edit 重新同步受影響的佔位點區段（預覽 DOM、`SEL`/`OUT_SEL`/`LAYERS`、stage reset），再用真實瀏覽器 `getComputedStyle` 重新驗證
- 收到 [UI Tweaker 確認] 訊息後，直接套用不再詢問；但若偵測到可重用的設計系統樣式且尚未指定更新範圍，需先確認作用範圍（見 `references/confirm-apply.zh-TW.md`）
