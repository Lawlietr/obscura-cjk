# React hydration 靜默未完成 — 問題回報(2026-09-22,來自 `translate` 專案)

> **高優先級**。影響所有「點擊 → 斷言狀態改變」類功能驗證:目前只能截圖 +
> `browser_evaluate` 只讀探測,交互驗證全數改走真 Chromium(playwright-core)規避。

## 症狀(全部已在 Obscura 瀏覽器實測)

- 頁面載入正常、SSR HTML 完整渲染、**零 console/page 錯誤**
- React 永遠不 hydrate:所有 DOM 元素沒有 `__reactFiber*` key(等 30 秒以上、
  `browser_evaluate` 反覆檢查);React 事件處理器因此全部失效
- `browser_click` 點任何按鈕(MUI IconButton、tab、Select)全部無效;
  `browser_evaluate` 直接 DOM `element.click()` 也無效
- **誤導訊號**:SSR 渲染的文字讓頁面看起來「活著」;部分純 DOM 行為(如
  textarea 的 SSR 文字)正常,但任何綁 React handler 的交互都死了
- 同一頁面 + 同一 URL 在真 Chromium(Playwright)完全正常:hydration 成功、
  全數交互檢查通過(translate 專案 45 項檢查全綠)

## 重現環境

- 目標 app:Next.js 16.3.4(Turbopack,**dev 模式**)+ React 19.2.8 + MUI 9
- Obscura 端:Docker 映像 + MCP(`browser_navigate` / `browser_click` /
  `browser_evaluate`)
- Next 16 頁面結構是**雙 React root**(layout shell + content,皆掛在
  `document`)——這是 Next 架構,不是原因(真 Chromium 照常處理)
- ⚠️ **只測過 dev 模式**;production build 尚未在 Obscura 驗證——需先區分
  dev 特有(HMR client)還是普遍 hydration 問題

## 懷疑方向(未驗證,供定位參考)

1. **WebSocket / HMR 處理。** Next 16 dev 的 hydration 實證上被 HMR websocket
   卡住:translate 專案的 HTTPS→HTTP 代理**不轉發 `upgrade` 事件**時,頁面出現
   **一模一樣的死狀**(不 hydrate、零錯誤);代理補上 ws 轉發後立即恢復。
   若 Obscura 的 websocket 支援不完整(handshake、Origin 處理、frame pumping),
   Next dev client 會卡死並連帶拖死 hydration。
2. **Origin / Referer headers。** Next 16 dev 會 403 cross-origin 的
   `/_next/*` dev 資源請求(Origin host 不在允許清單)。若 Obscura 的
   fetch/XHR/websocket 省略或送錯 Origin/Referer,dev client 資源靜默 403——
   頁面「載入正常」但永遠不 hydrate。
3. V8 事件循環 / hydration 排程——嫌疑較低(已等 30 秒以上)。

## 建議的最小重現與定位

- 靜態單頁(React 19 + 一個 onClick setState 按鈕,不用 Next):純 React 頁在
  Obscura 能否 hydrate?→ 排除/確認是否框架相關
- Next 16 dev 頁 + production build 各測一輪(dev 特有 vs 普遍)
- 與真 Chromium(Playwright)跑同一頁面,diff:
  - `/_next/*` dev 資源與 chunk 請求的 headers(Origin/Referer/Sec-Fetch-*)
  - websocket 連線生命週期(`/_next/webpack-hmr` 或等價路徑:握手、升級、心跳)
- CDP 層面:確認 `Network.requestWillBeSent` 中 dev 資源的 status 是否 403

## 影響範圍

- 任何依賴 React(或同類 hydration 框架)互動的頁面,在 Obscura 中都是死
  SSR shell——scraping/讀取 SSR 內容不受影響,但「點擊 → 斷言」類 agent
  自動化全數失效
- 下游專案(translate)現行規避:功能驗證走 cached ms-playwright Chromium +
  playwright-core;Obscura 只做截圖與只讀 DOM 探測
