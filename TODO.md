# TODO

Obscura CJK fork (`Lawlietr/obscura-cjk`) 待辦追蹤。
本文件只記錄未完成的條目；已完成的工作以 git log 為準。

## 待辦

### 中優先級｜cjk-stealth smoke test

- [ ] 抽驗 `cjk-stealth` 變體 smoke test（#831 動了 wreq，本機無 cmake
      無法本地建置驗證）。v0.2.1-cjk 起就標記，至今未完成。

### 中優先級｜Dependabot

- [ ] Repo Settings 確認 Dependabot GitHub App 權限（public repo 預設
      已啟用，無需額外開關；security alerts 預設開）。
- [ ] 觀察首批 PR：確認 `[patch.crates-io]` 的 vendored `taffy` /
      `cosmic-text` 行為符合預期（本體不會被更新，屬正常 warning；其
      transitive 依賴仍會進 lockfile 更新）。
- [ ] 後續維護：升級後視需要同步清理 `deny.toml` 的 ignore 清單
      （RUSTSEC ID 綁定 transitive 版本，cargo-deny CI 會提示）。
- 細節：[design/dependabot.md](design/dependabot.md)

### 中優先級｜PDF 可選取文字（upstream open PR #1391）

> fork 目前 `Page.printToPDF` 是純 raster（`obscura-browser/src/pdf.rs`
> 只寫 JPEG XObject），輸出無法選取/搜尋文字。
> upstream PR h4ckf0r0day/obscura#1391（"feat(pdf): export selectable text
> with embedded fonts"，截至 2026-11-01 仍 OPEN，+1344/-39 x19）會用
> `subsetter` + CID CMap 把已 shaping 的文字嵌入 PDF。

- [ ] 同步 upstream 後評估 cherry-pick #1391。
- [ ] fork 端必改點：該 PR 只用指標相等白名單
      （`inline.rs` `is_bundled_print_font()`）放行內建 14 個 TrueType
      face；我們的 `CJK_SC_R` / `CJK_TC_R`（`inline.rs`，`include_bytes!`）
      不在清單 → 會整份 16-32 MB 嵌入。且那兩份是 OTF/CFF，而 PR 的嵌入
      格式是 `/CIDFontType2`（TrueType `glyf`），需決定放進白名單 subset、
      走 raster fallback，或改提供 TTF CJK face。
- [ ] 加一個 CJK 的 PDF fixture（`render-repros/cjk/`）驗證可選取文字。

### 低優先級（可選）｜React hydration 靜默未完成（2026-09-22）

> 下游專案採兩階段策略：Obscura 負責截圖 + 只讀 DOM 探測，點擊類驗證
> 改走真 Chromium（Playwright）。WebSocket 支援非核心需求。
> upstream PR h4ckf0r0day/obscura#1080 截至 2026-11-01 仍 OPEN
> （最後更新 2026-09-26，無 review）。
> 細節：[design/hydration-bug-report.md](design/hydration-bug-report.md)

- [ ] （可選）最小重現：純 React 19 單頁（非 Next）在 Obscura 能否 hydrate
- [ ] （可選）驗證 production build 是否同受影響
- [ ] （可選）定位：與 Playwright 對照 `/_next/*` dev 資源 headers 與 ws 生命週期
- [ ] （可選）修復 + 回歸（需實作頁面級 WebSocket；候選來源：upstream #1080）

### 低優先級｜Leaflet 功能補齊（2026-09-16）

> 建議順序 C → E → F 決策 → G。F、G 獨立；
> SVG 鏈有依賴 C → E。

- [ ] **（C）SVG factory 方法（最小層）**：`createSVGRect` 等，解鎖
      Leaflet 功能偵測。細節：[design/svg-dom-api.md](design/svg-dom-api.md)
- [ ] **（E）SVG animated transform + `getTotalLength` 等**。依賴 C。
      `getTotalLength` 已有 upstream open PR h4ckf0r0day/obscura#1204
      （native Rust geometry）可考慮 cherry-pick；`transform.baseVal`
      仍需自己做。
- [ ] **（F）`getBBox` 處理決策**：快速回 `DOMException` vs 讀真實幾何。
- [ ] **（G）`getCTM` / `getScreenCTM`**：需 layout 資料，預估 2–4 天。

### 低優先級（可選）｜本機環境

- [ ] 磁碟衛生：`target/release/deps` 定期清舊 binary；重 build 前查
      `df -h /`。

## 參考

### 分支 / remote 現況

- `origin` → `Lawlietr/obscura-cjk`（唯一 push remote）；`upstream` →
  `h4ckf0r0day/obscura`（唯讀）
- 工作分支：`main`
- 最新 tag：`v0.3.0-cjk`（2026-09-28，deno_core 0.412 基底）
- fork main 已 rebase 到 upstream/main（deno_core 0.412），2026-09-28 完成。
  計劃：[design/upstream-rebase-20260929.md](design/upstream-rebase-20260929.md)
- 2026-11-01：`sync/upstream-20261101` 分支 `git merge upstream/main`
  （upstream `800b5ba`，+402 commits），衝突 5 檔（`AGENTS.md` 贊助連結、
  `README.md` 採 ours、`cdp/src/server.rs` ready-file OpenOptions、
  `cli/src/main.rs` worker env、`render/src/paint.rs` SVG font DB）。
  deno_core 維持 0.412。驗證：release nextest `render,cjk` 2460 passed /
  5 skipped / 0 failed；release build（`render,cjk`）乾淨；障礙課程 33/33。

### 已評估，不做

- upstream PR h4ckf0r0day/obscura#786（"recheck moving intersection
  sentinels"，OPEN）已於 2026-11-01 實測過：cherry-pick 後連 PR 自帶的
  回歸測試都過不了（期望 `[50,50]`，實得 `[40,40]`），舊版 obstacle fixture
  仍然 FAIL，而且它對 zero-area sentinel 用 "grace period" 報
  `isIntersecting: true`，與 upstream 的
  `intersection_observer_honors_root_margin_zero_area_and_no_fake_refires`
  語意相反。代價是每個 IO callback 後多一次 layout checkpoint。判定為對
  當前引擎已死的 PR，不取。
- obstacle course 的 `observer-intersection` 失敗是 benchmark 倉庫的
  fixture 問題（該倉 PR #8，2026-10-03 已修）；本地 clone 已 fast-forward
  到 `2d7bc30`。本 fork 在修好後的課程上 33/33。

### 已完成，僅存設計記錄

- **具體 SVG element interfaces / shape class 與 `instanceof` 映射**
  （原 Leaflet 項 D）已隨 2026-11-01 的 upstream 同步進來（upstream
  #1112）：`bootstrap.js` 現有 `SVGRectElement` / `SVGCircleElement` /
  `SVGPolygonElement` 等具體 class 與 namespace-aware 映射。Leaflet 偵測
  仍斷在缺 `createSVGRect`（項 C）。
- **CSSOM View 盒模型 getter**（`clientLeft` / `clientTop` /
  `clientWidth` / `clientHeight`）已上線：`bootstrap.js` 的 getter 讀
  `op_layout_geometry`（同步 upstream 後改走批次 op
  `op_layout_box_metrics`）。設計記錄：
  [design/cssom-view-box-geometry.md](design/cssom-view-box-geometry.md)
- **MCP `browser_console_messages`** 已接線：`obscura-mcp/src/lib.rs` 的
  `tool_console_messages()` 讀 `take_pending_console_messages()`，建頁時
  `set_console_messages_enabled(true)`，並有 nextest 覆蓋。
  設計記錄：[design/mcp-console-capture.md](design/mcp-console-capture.md)

### 已記錄在 AGENTS.md（無需再跟）

- fork 政策、Docker 部署、CI 慣例
- 不自動跑驗證（build / nextest / obstacle course 只在用戶要求時執行）
- SVG fallback 字型 lazy loading gotcha
- `render-repros/cjk/` 置於子目錄是故意的
