# CSSOM View 盒模型 getter（clientLeft / clientTop / ...）

待辦追蹤見 `TODO.md`「待辦」節。背景：自動化 Leaflet 地圖頁時
發現頁面 JS 讀取 `clientLeft` 等取得 `undefined`；Leaflet 用它換算
點擊座標時算出 NaN，地圖點擊定位失誤。真實瀏覽器無此問題，目前
test page 以 polyfill 規避。

## 現況

- `clientLeft` / `clientTop` / `clientRight` / `clientBottom` 未
  實作（頁面 JS 讀取 `undefined`）。
- 資料幾乎已就位：
  - `op_layout_geometry`（`obscura-js/src/ops.rs` ~L5273）已回傳
    `clientWidth` / `clientHeight`（padding box）。
  - batch op `op_resize_observer_measurements` 已回傳 computed
    style 的 border / padding 值（同一模式可照抄）。

## 修法（估計 0.5–1 天含測試）

- Rust：geometry payload 加 border width 欄位（~10 行）。
- JS：`bootstrap.js` 端 getter（~15 行）。
- 補完後 test page 的 polyfill 可移除。

## 規範邊界（CSSOM View）

- `clientLeft` = 左 border 寬（不計 padding）；`clientTop` 同。
- inline 元素回 0。
- `display: none` / detached 元素回 0。
- non-render build 的 fallback 回 0。

## 相鄰缺口

見 `design/svg-dom-api.md`：SVG 向量層停用與本項點擊座標 NaN
是 Leaflet 地圖頁的兩個獨立斷點，都補完後該類頁面才能完整可用。
