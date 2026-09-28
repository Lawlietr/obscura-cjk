# SVG DOM API 補齊（shim 層）

待辦追蹤見 `TODO.md`「待辦」節。背景：自動化 Leaflet 地圖頁時發現
`bootstrap.js` 的 SVG shim 是簡化版 binding，真實瀏覽器無此問題，
目前 test page 以 polyfill 規避。已評估：API 皆已定案多年，難度低、
維護成本近零。2026-09-16 正式問題回報（`obscura-cjk:merged-local` +
Leaflet 1.9.4 實測）。

## 現況盤點（唯讀分析結論）

非 build flag 裁切，純 shim 層缺口（與 `stealth` / `cjk` 等 feature
無關）：

- SVG 類別階層（`bootstrap.js` ~L11691）全是空 class，只暴露
  SVGElement / SVGGraphicsElement / SVGGeometryElement /
  SVGPathElement / SVGSVGElement。
- `_elementClassFor` / `_elementClassForKnownName`（~L6443）只映射
  `path` → `SVGPathElement`、`svg` → `SVGSVGElement`，其餘 shape
  一律回退 `SVGElement`。
- 全缺（頁面實測均 `undefined`）：
  - `SVGSVGElement.createSVGRect` / `createSVGPoint` /
    `createSVGNumber` / `createSVGAngle` / `createSVGMatrix` /
    `createSVGTransform`（SVG 1.1 factory，Chromium 主線仍保留）
  - `SVGAnimatedTransformList`（`transform.baseVal`；現有
    `SVGAnimatedString` 只服務 class/href 反射）
  - `getCTM` / `getScreenCTM`
  - `getTotalLength` / `getPointAtLength`
  - 各 shape 專屬 class（`SVGCircleElement` 等；class 不存在，
    `instanceof` 必假）
  - SVG2 屬性反射（`c.r` 等，只能走 `getAttribute`）
- 另：`getBBox` 存在但為零值 stub（`bootstrap.js` ~L13147，回
  `{x:0,y:0,width:0,height:0}`），拿它量測會得假資料。
- 存在且正常：`createElementNS`、innerHTML namespace 解析、
  set/getAttribute、渲染管線（SVG 正常 paint）、canvas 2D。

## 影響

- Leaflet 1.9.4 功能偵測：
  `Browser.svg = !(!document.createElementNS ||
  !document.createElementNS(NS, "svg").createSVGRect)` →
  `createSVGRect` undefined → 偵測失敗 → `_createSvgRenderer` 回傳
  null → 第一個向量層 `addTo(map)` 時 `stamp(null)` 拋 TypeError
  （"Cannot use 'in' operator to search for '_leaflet_id' in null"）。
  全部 circleMarker / polygon / polyline 停用，`.leaflet-overlay-pane`
  為空 div，且 Leaflet 無 canvas fallback 自動切換（需手動
  `preferCanvas: true`）。
- 即使偵測繞過，缺 `transform.baseVal` / CTM / `getTotalLength` 也會讓
  依賴 SVG 動畫或量測的程式（D3、地圖風圈繪製等）出錯。
- 截圖層面完全無感（SVG 畫得出，只是 API 偵測不到），極難從截圖
  發現。
- 可快速定位的探针：
  `typeof document.createElementNS("http://www.w3.org/2000/svg",
  "svg").createSVGRect` 回傳 `"undefined"` 即中招。

## 修法（分層，皆純 JS shim，不需 layout 資料）

### 最小（解鎖 Leaflet 偵測，估計 0.5 天含測試）

- `SVGSVGElement.prototype` 補 6 個 factory 方法：
  - `createSVGRect()` → `SVGRect` 物件：`{x, y, width, height}` +
    各欄位 `x.baseVal`（`SVGNumber` 包裝）+ `getX()` / `setWidth()`
    等。
  - 同類：`createSVGPoint` / `createSVGNumber` / `createSVGAngle` /
    `createSVGMatrix` / `createSVGTransform`（SVG 1.1 定案的 value
    物件語意即可，供 feature detection 與簡單用量）。
- 驗收：Leaflet 頁 `L.Browser.svg === true`、SVG 向量層正常渲染。

### 完整（後續排程）

- 各 shape 專屬 element class（`SVGCircleElement` / `SVGRectElement` /
  `SVGLineElement` / `SVGPolygonElement` / `SVGPolylineElement` /
  `SVGTextElement` 等）+ `_elementClassFor` 映射，讓 `instanceof` 與
  `constructor.name` 正確。
- `SVGAnimatedTransformList`（`transform.baseVal` / `animVal`，
  讀寫 `transform` attribute）。
- `getTotalLength` / `getPointAtLength`（path 量測）。
- SVG2 屬性反射（`c.r`、`rect.x` 等）。
- `getCTM` / `getScreenCTM`：需要 layout 資料（frame transform 鏈），
  成本高，單獨排程。
- `getBBox` stub 改為讀真實幾何（或明確回 `DOMException`），消除假
  零值。
- 文件（README 或 docs）標明 SVG DOM shim 能力範圍，避免使用者誤以為
  與 Chrome 同等。

## 注意事項

- 補完後跑 `svg` filter 的 nextest（`cargo nextest run ... -p
  obscura-render --no-fail-fast` 的 svg 篩選）確認未延遲首次
  prepare：SVG fallback 字型 lazy loading gotcha（
  `svg_font_database_with_fallbacks` 獨立 OnceLock，只在頁面含 inline
  SVG text 時建置）同類風險，首次 prepare 上多出的同步成本會延遲
  early paint 與 IntersectionObserver callback。
- 測試：最小項補 Leaflet 偵測探针（`createSVGRect` typeof 檢查 +
  `L.Browser.svg` 真值）進 test page 或 nextest；完整項補 shape
  `instanceof` 與 animated transform 讀寫測試。
- 已知相鄰缺口（另見 `design/cssom-view-box-geometry.md`）：
  `clientLeft` 等盒模型 getter 缺失會讓 Leaflet 點擊座標換算出 NaN，
  兩者都補完後 Leaflet 地圖頁才能完整可用。
