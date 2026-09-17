# TODO

Obscura CJK fork (`Lawlietr/obscura-cjk`) 待辦與完成進度追蹤。
工作分支：`main`。本文件只記錄條目、優先級與進度；實施細節放
`design/` 目錄，各條目附鏈接。

## 待辦

### 上游合併 #2（2026-09-17，`694c8e1`，71 commits since `727cc46`）

- [x] 合併後回歸（nextest）：`render,cjk` 九 crate 全量 **1691 passed /
      4 skipped / 0 failed**（render 601/1、js+net 611/0、cdp+dom+cli+
      browser+mcp 479/3），2026-09-17 重跑。
- [ ] 合併後回歸（剩餘關卡）：release build（`render,cjk`）+ 障礙課程 32/33。
      障礙課程自 2026-09-17 合併後尚未重跑。
- [x] **CJK 截檢（2026-09-17）。** `target/release/obscura`（`render,cjk`
      建置，139MB）跑 `cjk-fallback.html`：text dump 繁/簡/日/混排字元全數
      正確還原；截圖 55KB 無豆腐框，字距與行高跟實字形 advance。`extra_fallback_fonts`
      注入新 `base_font_database` 快取架構兩路徑皆確認：HTML 路徑
      `base_font_database()`（inline.rs:462 注入 `BASE_FONT_DATABASE`）、SVG
      路徑 `svg_font_database_with_fallbacks()`（paint.rs:10364）。
- [x] 文件同步：兩套字型目錄機制並存已寫清楚（fork `--fonts`/
      `OBSCURA_FONTS_DIR`：非遞迴、支援 woff/woff2；上游 `--font-dir`：
      serve 層可重複、遞迴、純 sfnt、須在首次 render 前設定）。
      2026-09-17 更新：AGENTS.md、docs/CJK-and-custom-fonts.md、
      docs/Environment-variables.md；docs/CLI-reference.md 原本已兩套並存。

### 中-高優先級（2026-09-16 問題回報，Leaflet 1.9.4 + `obscura-cjk:merged-local` 實測）

- [ ] SVG DOM API 補齊（factory 方法 + animated-value + shape class +
      SVG2 反射），分「最小 / 完整」兩層。
      細節：[design/svg-dom-api.md](design/svg-dom-api.md)
- [ ] MCP `browser_console_messages` 接線（目前死欄位，永遠回
      "No console messages."；未捕獲例外同樣捕不到）。
      細節：[design/mcp-console-capture.md](design/mcp-console-capture.md)
- [ ] CSSOM View 盒模型 getter（`clientLeft` / `clientTop` 等，
      Leaflet 點擊座標換算出 NaN）。
      細節：[design/cssom-view-box-geometry.md](design/cssom-view-box-geometry.md)

### Release：v0.2.1-cjk

- [x] push main + 打 tag `v0.2.1-cjk`（2026-09-17 早上觸發 release，
      已上線：https://github.com/Lawlietr/obscura-cjk/releases/tag/v0.2.1-cjk）。
- [x] 本機 `docker-compose.yaml` 切回 `ghcr.io/lawlietr/obscura-cjk:latest`
      （2026-09-17）。pull + `docker compose up -d` 重建容器；MCP smoke：
      initialize 回 `serverInfo 0.2.1-cjk`、navigate example.com、
      evaluate 含 CJK 字串、screenshot PNG、close 全過。
- [ ] 抽驗 cjk-stealth 變體 smoke test（#831 動了 wreq，本機無 cmake
      無法本地建置驗證）。
- 細節：[design/release-workflow.md](design/release-workflow.md)

### Dependabot

- [ ] Repo Settings 確認 Dependabot GitHub App 權限（public repo 預設
      已啟用，無需額外開關；security alerts 預設開）。
- [ ] 觀察首批 PR：確認 `[patch.crates-io]` 的 vendored `taffy` /
      `cosmic-text` 行為符合預期（本體不會被更新，屬正常 warning；其
      transitive 依賴仍會進 lockfile 更新）。
- [ ] 後續維護：升級後視需要同步清理 `deny.toml` 的 ignore 清單
      （RUSTSEC ID 綁定 transitive 版本，cargo-deny CI 會提示）。
- 細節：[design/dependabot.md](design/dependabot.md)

### 本機環境（非仓库變更）

- [ ] 可選：`obscura-benchmark` 倉庫 clone 下來跑完整驗證。
- [ ] 磁碟衛生：`target/release/deps` 定期清舊 binary；重 build 前查
      `df -h /`。

## 完成進度

### 上游合併

- [x] **上游合併 #2（2026-09-17，`694c8e1`）。** 4 個衝突手解：
      `inline.rs` 採上游字型 cache 架構（`base_font_database`/
      `cached_web_font_database`、`WebFont.data` 改 `Arc`），fork 的
      CJK + `OBSCURA_FONTS_DIR` fallback 改注入 `base_font_database()`，
      與上游 `--font-dir`（`FONT_DIRECTORIES`）兩套並存；`paint.rs` 保留
      `decode_font_bytes` 去重 + SVG fallback 更名函式，採上游 `Arc` 型別；
      `release.yml` 保留 cjk/cjk-stealth 兩變體，採上游 `release-dist`
      profile；`README.md` 保留 fork 精簡版。其餘 30+ 檔（cdp/dom/js/net/
      mcp/render style+dom、taffy float）直接採上游 bug 修正。

### Release

- [x] **v0.2.1-cjk release（2026-09-17）。** tag 推送後 release 上線；
      本機 compose 已切回 GHCR `latest`，容器重建後 MCP smoke test
      （navigate / evaluate / CJK / screenshot / close）全數通過。
- [x] **v0.2.0-cjk 上游大合併（2026-09-07）。** 自分叉點 `c1380190`
      起 114 commits（~40 PR，+9482/−1056 行，44 檔）；3 個衝突全數
      按預先評估處理。
- [x] **合併後本地回歸（2026-09-07）。** `render,cjk` nextest
      **1647/1647**（4 skipped）、release build 6m52s、svg filter
      21/21、CJK fixture 無豆腐框、障礙課程 **32/33**（已知
      `observer-intersection`）。
- [x] **本地容器換上合併版驗證。** `obscura-cjk:merged-local`：nonroot、
      compose hardening、MCP、真實站 navigate+eval、CJK 截圖全過；
      正式容器 `obscura` 已重建。
- [x] **文件同步（v0.2.0-cjk tag 前）。** `docs/Use-as-a-Rust-library.md`
      pin、AGENTS.md、`docs/Run-in-production-at-scale.md`、
      `docker-compose.example.yaml` storage dir 權限註解。
- [x] **v0.1.0-cjk 首次 release（2026-08-24）。** Release run ~28min、
      25 資產（5 平台 × 5 變體）、Docker run ~7min、本機抽檔通過。
- [x] **release 機制落地。** `release.yml` 加 `render,cjk` 變體、tag
      策略（`v0.1.0-cjk` 起跳）、`docker.yml` 改造為 GHCR 發佈、映像
      tag 慣例（`latest` 為主）。細節：design/release-workflow.md
- [x] **README 移除上游 Chrome 對比表（2026-08-24，EN/ZH 同步）。**
      數據無出處且「Anti-detect: Built-in」對本 fork 錯誤（stealth 是
      build-time feature，Docker 映像不含）。

### 回歸驗證（下次 release 前的品質關卡）

> 首個 release 的 CI smoke test 已全數通過；完整本地回歸 2026-08-25
> 補跑。

- [x] **上游合併 #2 回歸（nextest，2026-09-17）。** `render,cjk` 九 crate
      全量 **1691 passed / 4 skipped / 0 failed**（render 601/1、js+net
      611/0、cdp+dom+cli+browser+mcp 479/3）。release build、障礙課程 32/33
      與 CJK 截檢未隨本批執行（障礙課程留待後續）。
- [x] **CJK 視覺抽檢（2026-09-17）。** 繁/簡/日/混排字形正確，截圖 55KB，
      豆腐框全無；`extra_fallback_fonts` 已注入新 `base_font_database` 快取
      架構（HTML + SVG 兩路徑）。
- [x] **完整回歸（2026-08-25）。** `render,cjk` 1487/1487、`render`
      1486/1486（各 4 skipped）；建置 6m19s。
- [x] **磁碟檢查（2026-08-25）。** 測試後 6.5G 可用（78%）；清
      `target/` 後 6.5G → 802M。
- [x] **CJK 視覺抽檢（2026-08-25）。** 繁/簡/日文字形正確，截圖
      55KB，無豆腐框。
- [x] **非 cjk 路徑抽測（2026-08-25）。** 1486/1486，無回歸。
- [x] **`obscura-benchmark` 障礙課程 32/33（2026-08-25）。** 已知
      `observer-intersection` 失敗：headless 模式不模擬 scroll，
      IntersectionObserver callback 只觸發一次（本質限制，AGENTS.md
      有記錄）。

### 文件同步

- [x] **Release 前置文件同步（2026-08-24，方案 B）。** README ×2、
      docs/Installation.md、docs/CJK-and-custom-fonts.md 同步；修正
      Docker 映像 runtime 描述錯誤。
- [x] **README.md 精簡（614 → 243 行）。** 中間各節移至 docs/
      （CJK、CDP surface、Integrations、`--fonts` 等）。
- [x] **README_ZH.md 重寫（237 行，與英文 1:1）。** docs/ 保持英文。
- [x] **`docker-compose.yaml` 切換為 GHCR 映像部署。** `latest` 為主，
      本機已用新 compose 重建容器驗證。

### Dependabot

- [x] **`.github/dependabot.yml`（2026-08-24）。** 例行 lockfile-only
      更新（`routine` group）+ security 獨立 group；v8/deno 家族排除
      留人工；github-actions ecosystem。細節：design/dependabot.md

## 參考

### 已記錄在 AGENTS.md（無需再跟）

- fork 政策：本倉庫是獨立 fork，所有操作面引用（文件、安裝、release、
  Docker、CI、issue/security）指向本倉庫；上游僅保留於 Apache-2.0
  授權歸屬、`obscura-benchmark`（僅存於上游）、歷史 PR 引述。
- Docker：release 映像由 GitHub Actions 發佈至
  `ghcr.io/lawlietr/obscura-cjk`（每個 `v*` tag；`latest` 跟隨最新，
  版本 tag 供回退）；`docker-compose.yaml` 跟隨 `latest` 為正式部署，
  `docker build -t obscura-cjk .` 保留為本地開發流程；本機容器已由
  compose 管理。
- 不自動跑驗證：build / nextest / render capture / obstacle course
  只在用戶明確要求時執行。
- SVG fallback 字型 lazy loading 的 gotcha
  （`svg_font_database_with_fallbacks` 獨立 OnceLock，只在頁面含
  inline SVG text 時建置）。
- `render-repros/cjk/` 置於子目錄是故意的（`run.sh` 只 glob 頂層
  `*.html`）。

### 分支 / remote 現況

- `origin` → `Lawlietr/obscura-cjk`（自己的倉庫，唯一 remote）
- 工作分支：`main`（追蹤 `origin/main`）
- 首個 tag：`v0.1.0-cjk` @ `84c7223`（2026-08-24 推送，觸發首次
  release）
- 最新 tag：`v0.2.1-cjk`（2026-09-17 推送；本機 compose 已跟隨 GHCR
  `latest` 驗證通過）
- 上游 `h4ckf0r0day/obscura` 目前未設 remote，僅歷史與 Apache-2.0
  授權歸屬參考
