# TODO

Obscura CJK fork (`Lawlietr/obscura-cjk`) 待辦追蹤。
本文件只記錄未完成的條目；已完成的工作以 git log 為準。

## 待辦

### 高優先級｜ECDSA P-256 已撤回，待 p256 0.14 遷移後帶回

- [ ] 2026-11-01：upstream PR #1308 以 cherry-pick 進入 fork（`c7f7be9`）之後，
      `v0.4.0-cjk` 的 release workflow 被 `cargo deny check advisories` 擋下：
      **RUSTSEC-2026-0387**（`crypto-bigint` 0.5.5，原本設計成 constant-time 的
      運算改為依秘密值分支：減法、零位元計數、除法、Montgomery 乘法、
      `UintRef` 比較、`BoxedMontyForm::invert`）。鏈路為 `p256` →
      `elliptic-curve` 0.13.8 → `crypto-bigint` 0.5.5，由 `obscura-js` 的
      WebCrypto P-256 路徑直接可達。advisory 只修補 `>= 0.7.6`，
      0.5 線沒有可移植的修補版。
- [ ] 撤回 commit：`a2159e9`（`git revert c7f7be9`）。撤回後 `Cargo.lock` 內的
      `p256` / `elliptic-curve` / `crypto-bigint` 全部消失（grep = 0）；
      `bootstrap.js` 回到只剩兩處既有的 `"ECDSA"` 正規化字串，與 `v0.3.0-cjk`
      完全相同。
- [ ] 帶回的做法（已查證）：改用 `p256 = "0.14"`。查 crates.io index 結果：
      `elliptic-curve 0.14.3` 的依賴清單已不含 `crypto-bigint`（改使用 `bigint`），
      連帶需要 `ecdsa-core ^0.17`、`primeorder ^0.14`、`sha2 ^0.11`、
      `serdect ^0.4`。要改 `ops.rs` 的 EC 輔助層（`SecretKey::from_bytes` 與
      `from_slice`、`PrivateKeyInfo`、`AssociatedOid`、pkcs8 的讀寫、`ecdsa` 的
      `PrehashSigner` 與 `PrehashVerifier`）以及 `webcrypto_tests.rs`。
- [ ] upstream #1308 目前仍是 OPEN（最後更新 2026-10-25），仍使用 0.13 線；
      upstream main 不含 `p256`，所以 upstream 自己的 audit 碰不到這條。若未來
      同步時 upstream 帶入未修補的 `p256`，同一條 advisory 會在我們這邊再度失敗，
      屆時優先處理。
- [ ] 驗證提醒：本機未安裝 `cargo-deny`，改完依賴後只能靠 CI 的 audit job，
      或先 `cargo install cargo-deny` 再本地預驗。

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

### 追蹤｜上游 CJK 現況（每次 upstream sync 必查，優先級高）

> 規則：上游一旦出現 CJK 相關變化（新 `cjk` feature、內嵌 CJK 字型、
> Han fallback 邏輯、`--font-dir` 範圍擴大、PDF 字型嵌入、
> `vendor/cosmic-text` fallback 清單改動），**要先回報並優先採用上游做法**，
> 不要兩邊各實作一份。

- [ ] 每次同步 upstream 後重跑下方檢查，並在有任何變化時明確回報
      （變更是什麼、影響哪個變體、要不要把 fork patch 改成上游實作）。
- [ ] 盯 upstream issue #606（screenshot 顯示不出中文，OPEN）與
      PR #991（table/select DOM APIs + CJK font fallback，OPEN）。
- [ ] 若上游接受我們的 CJK fallback 形式（opt-in feature + OFL face），
      就改用上游版並刪掉 fork 的 `inline.rs` 注入路徑與 assets。

基線證據（2026-11-01，fork HEAD 當時 = upstream 8534b40 同步後）：

| 檢查項 | 上游狀況 |
|---|---|
| `cjk` feature | 無（`obscura-render` features 只有 `default`/`paint`） |
| 內嵌 CJK face | 無；`crates/obscura-render/assets/` 只有 Liberation/DejaVu/Noto Emoji |
| cmap 覆蓋 `中/文/テ/ス` | liberation-sans、dejavu-sans、noto-color-emoji 皆 0/4 |
| 上游自己的 Han/CJK fallback 代碼 | 無；`Noto Sans CJK` 只出現在 vendored cosmic-text 的 family 對照表 |
| 官方 CJK 解法 | `--font-dir`（僅 `serve`/`mcp`，遞迴、只收 sfnt、每 process 首次渲染前一次） |
| 部署基底 | `debian:12-slim` 內 **0 個字型檔**，`load_system_fonts()` 掃不到東西 |
| 我們發布的映像 | 映像內 **0 個字型檔**，但仍繪出 CJK（glyph 來自 binary 內嵌 face） |

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
- [ ] **（E）SVG animated transform + `getPointAtLength` 等**。依賴 C。
      `getTotalLength` 已隨 upstream PR h4ckf0r0day/obscura#1204 上線
      （fork `crates/obscura-js/src/svg_geometry.rs`，`docs/SVG-geometry.md`）；
      `transform.baseVal` 與 `getPointAtLength` 仍需自己做。
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

- 2026-11-01（第二輪同步 + 揀選）：同一分支再 `git merge upstream/main`
  （upstream `8534b40`：zstd 壓縮依賴的安全修補，新增 `vendor/wreq-util`，
  **0 個衝突**），接著套用 17 個 upstream PR（共 18 個 commit，#766 是 2 個、#1204 附帶的第二個是空的 CI commit 已略過）：#1052、#1332、#1366、
  #1397、#1401、#1380、#1285、#1257、#766、#763、#1277、#1228、#1318、
  #848（該 PR 的分支含 merge commit，改用 `git merge --squash` 一次套用）、
  #1301、#1308、#1204。`[patch.crates-io]` 已由上游改成直接 path 依賴取代，
  `vendor/taffy` 與 `vendor/cosmic-text` 現在與上游完全一致（我們的 grid
  shrink-to-fit 修正與 variable-font 座標修補已被上游的 vendor 吸收）。
  驗證：nextest（`render,cjk`）2505 passed / 0 failed / 5 skipped；release
  build（`render,cjk`）乾淨（僅 1 個 upstream 自帶的 dead-code 警告）；障礙課程 33/33（median 3054.6 ms，同步前
  基準 3064.6 ms，差 0.3%，在 ±10% 噪音地板內）。
  本輪帶進的行為變化：初始 `font-family` 現在解析為 serif（#763，與
  Chromium 對齊）；`navigator.geolocation` 在沒有 `OBSCURA_GEOLOCATION` 時改回
  PERMISSION_DENIED（#1228）；`elementsFromPoint` 依 paint order 回傳（#848）。

- 2026-11-01 發布 `v0.4.0-cjk`：annotated tag（tag 物件 `561b065`）打在
  `main` 的 `2b40dcf`。第一次打的 tag 被 `release.yml` 的 `cargo deny check
  advisories` 擋下（RUSTSEC-2026-0387），因此撤回 #1308（`a2159e9`）、刪掉舊 tag
  重打。Docker 那一條是在 `publish-manifest` 建立 `:0.4.0-cjk` 與 `:latest` 之前
  就被取消，所以 `:latest` 從未指向壞 build。
  公開發布狀態：GitHub Release 10 個資產（5 個 target × `cjk` 與 `cjk-stealth`，
  117–123 MB）；GHCR `:0.4.0-cjk` 與 `:latest` 同一 digest
  `sha256:11640ff0…`；映像內 `obscura --version` 為 `0.4.0-cjk`（版本字串由 tag
  決定，`Cargo.toml` 不需改）。
  映像 smoke test（用公開發布的映像跑）：CJK fixture 截圖與本機驗證過的 build
  完全相同（55557 bytes、24234 個非白像素、239 種灰階）；
  `getTotalLength=120`；`crypto.subtle` 的 ECDSA 與 ECDH 都是
  `NotSupportedError`（與 `v0.3.0-cjk` 一致，確認撤回到乾淨狀態）。

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
