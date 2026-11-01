# CSSOM View 盒模型 getter（clientLeft / clientTop / ...）

> 狀態：已實作。本文件是設計記錄，非待辦。
> 背景：自動化 Leaflet 地圖頁時發現頁面 JS 讀取 `clientLeft` 等取得
> `undefined`；Leaflet 用它換算點擊座標時算出 NaN。真實瀏覽器無此問題。

## 實作位置

- `crates/obscura-js/js/bootstrap.js`：`get clientTop` / `get clientLeft`
  （讀 computed style 的 border width）、`get clientWidth` /
  `get clientHeight`（經 `_renderClientMetrics()` 讀
  `op_layout_geometry` 的 padding box），另有 `offsetWidth` /
  `offsetHeight` / `offsetTop` / `offsetLeft` / `offsetParent`。
- upstream 已把單元素 op 換成批次測量 op `op_layout_box_metrics`（同一次
  回 client/offset 四值），同步 upstream/main 後自動取得。

## 規範邊界（CSSOM View）

- `clientLeft` = 左 border 寬（不計 padding）；`clientTop` 同。
- inline 元素回 0。
- `display: none` / detached 元素回 0。
- non-render build 的 fallback 回 0。

## 相鄰缺口

SVG 層缺口見 `design/svg-dom-api.md`：Leaflet 地圖頁的兩個獨立斷點，
盒模型 getter 已補完，SVG factory（`createSVGRect`）仍是斷點。
