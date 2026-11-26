# 包材計算工具（packaging-calc）

客人給的出貨明細 Excel → 依客戶規則算出每單用哪種包材 → 彙總成「工序計費要輸入的數量」。主要用途是**計費**（同事照數字手動輸入 ERP 工序計費），其次是大專案事先評估包材。

- 檔案：`packaging-calc.html`（單檔），`index.html` 只做轉址
- 版本字串：`PKG_VERSION`
- 測試模式：網址加 `?demo=1` 或 `#demo`（本機 `npx serve` 會吃掉 query，用 `#demo`），假倉庫／客戶／工序，規則存 localStorage
- 本機預覽：`.claude/launch.json` 的 `packaging-calc`（port 5180）
- `測試/` 放客人真實明細，已在 `.gitignore`，**不能推上 GitHub**（含收件人個資）
- 舊版 `..\暮雨包材計算工具.html`（v1.0，只讀第一個分頁、規則存 localStorage）保留不動

## 資料

- Firebase `ezfun-warehouse-billing`，Google 登入；權限讀 `hub_settings/users`（只讀），`blockedSystems` 含 `packaging` 就擋
- 倉庫：`warehouses`；客戶：`customers`（`wh`＝倉庫文件 ID）；包材選項＝`customers/{cid}/processes` 中 `category === '耗材'` 的工序（沒分類時用名稱含 紙箱／箱／袋 推測）
- 規則：`pkg_calc_rules/{cid}`，每次儲存另寫一份到 `pkg_calc_rules/{cid}/history`
  - `channels[{name, keywords[], maxPkg}]`、`rules[{ch, min, max, pkg}]`、`combos[{ch, keywords, pkg}]`、`points[{sku, kw, pt}]`、`extras[{ch, pkg, proc}]`、`identify[]`、`updatedAt/updatedBy/updatedByName`
- 規則用 Excel 維護（分頁：通路設定、件數規則、組合規則、商品點數、附加工序、識別關鍵字、說明）：上傳 → 顯示差異 → 確認後**整份取代**；包材名稱對不到 ERP 工序就擋下來

## 計算邏輯

1. 自動找表頭含「單號」「數量」的分頁（前 15 列內找表頭），欄位依表頭文字對應並顯示在畫面上；樞紐表分頁自動略過，可手動取消勾選
2. 同單號跨分頁合併
3. 通路：先比「物流方式」欄，再比分頁名稱（通路名稱本身也算關鍵字）；都對不到＝未分類（紅色，套共用規則）
4. 組合規則（品名全部含關鍵字）優先，否則件數規則；該通路有專屬件數規則就只用專屬的，否則用通路空白的共用規則
5. 商品點數表有資料時：件數＝Σ(點數×數量) 無條件進位；沒對到的商品以 1 點計並提醒
6. 通路 `maxPkg`：依件數規則的順序比大小，超過就標「需拆單或改宅配」（不自動拆）
7. 附加工序（v2.1）：通路相符（空白＝全部）且包材相符（空白＝不論）的每一列都套用，每單加 1，例如超取＋紙箱M → 寄件處理[超商](小)(<60)；跟著最終包材（含手動修改）變動，彙總依 ERP 工序順序列出
8. 每單可手動改包材（`S.overrides`，不存檔），會標示「已手動修改」
9. 下載：原檔每個計算的分頁最後加「包材」「附加工序」兩欄＋「包材彙總」分頁（讀檔要加 `cellNF:true`，否則日期變序號）

## 未來項目（已討論、暫不做）

- **直接寫入 ERP `records`**（wh、cid、date、items＝各包材數量×工序單價），並記錄來源檔名＋單號清單，防止同一份檔案重複計費。會寫進 Melvin 負責的 collection，要先跟他協調
- 大專案的包材預估模式
- 整合進 ezfun-hub 後，修改規則改用 Hub 的角色權限（目前能進該倉庫的人都能改，有修改紀錄）
