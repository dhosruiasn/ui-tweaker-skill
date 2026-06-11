# 確認與套用流程（ui-tweaker 參考文件）

> 收到 `[UI Tweaker 確認]`／`[UI Tweaker Confirm]` 訊息時（Step 3/4）必讀：解析數值、設計系統影響範圍、結構化更新、commit/push 邊界。

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

