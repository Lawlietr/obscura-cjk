# TODO

Obscura CJK fork (`Lawlietr/obscura-cjk`) 待辦與重要事項。
分支：`cjk`（已 rebase 到 `lawlietr/main` `2810cb4`，已 force-push）。
主分支合併前，先跑完「合併前必做」的項目。

## Release：GitHub Actions 二進位發布

上游的 release 機制是 GitHub Actions，fork 已繼承三個 workflow
（`.github/workflows/`）：

- `release.yml`：push 任何 `v*` tag 觸發。5 平台全部**原生**建置
  （x86_64 linux / aarch64 linux / aarch64 macos / x86_64 macos / windows），
  每平台 4 個 feature 變體（`render`、`render,stealth`、`no-render`、
  `no-render,stealth`），各打包 `obscura` + `obscura-worker`，
  上傳前逐變體 smoke test（V8 isolate + 截圖），
  發布 job 只下載 artifacts、不 checkout 程式碼（token 隔離）。
- `docker.yml`：`v*` tag 觸發，buildx 多平台映像推 Docker Hub，
  需要 repo secrets `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`。
- `ci.yml`：PR checks，read-only token。

### 待辦

- [ ] **`release.yml` 加 `render,cjk` 變體。** 現狀只 build 上游的
      feature 組合，照現狀打 tag 出的 release 二進位**沒有內嵌 CJK 字型**，
      與 README 宣稱的 fork 特性不符。改法：build job 加一段
      `--features render,cjk` 的 build + stage + package（每平台多一個
      tarball，約 +20 行），smoke test 迴圈加入該變體。
- [ ] **首次發布前決定 tag 策略。** fork 目前沒有任何 tag
      （`git ls-remote --tags lawlietr` 為空）。version 號避免與上游撞
      （上游 `1.0.103`，Cargo.toml 已同步該版本）。建議形如
      `v1.0.104-cjk.1` 之類。`docs/Use-as-a-Rust-library.md` 的 git 依賴
      pin 已從 `tag` 改為 `rev`（等第一個 tag 出來後可再改回 tag pin）。
- [ ] **決定 `docker.yml` 的去留。** 目前會推到上游的 Docker Hub 帳密
      變數所指向的帳號，fork 沒有該帳號。政策已是「映像檔本地 build，
      不發布 registry」，選項：(a) 停用/移除 `docker.yml`，
      (b) 改推到 `Lawlietr` 自己的 Docker Hub 帳號（需在 fork 的
      Settings → Secrets 配置）。
- [ ] 打第一個 tag push 觸發首次 release；驗證 artifacts 的 CJK 變體
      截圖正常。
- 額度備註：公開倉庫每月 2000 分鐘免費 Actions；5 平台 × V8 全編
  約每平台 10–15 分鐘，单次 release 沒有額度問題。

## 合併前必做（cjk → main）

- [ ] 完整回歸：`CARGO_INCREMENTAL=0 CARGO_BUILD_JOBS=2 cargo nextest run
      --release --features render,cjk --no-fail-fast`。
      **post-rebase 尚未跑過**（rebase 前是 1399/1399，不適用現況）。
      上游 933 commits 中有 1 個動到 `paint.rs`（box-shadow clip，
      `00a8a81`，+178 行）——低風險但未驗證。
- [ ] **先查磁碟**：`df -h /` 確保 ≥15G。`target/` 目前 ~8.7G，
      nextest 會再產 ~5G 測試二進位（~134MB × ~40 個，V8-linked）。
      不足時先清 `target/release/deps` 舊 binary。
- [ ] CJK 視覺抽檢：
      `OBSCURA_BIN=./target/release/obscura fetch
      file://$PWD/render-repros/cjk/cjk-fallback.html --screenshot
      "$RUN_ROOT/cjk.png"`，確認繁/簡/日文字形正確。
- [ ] 非 cjk 路徑抽測（`--features render`）確認無回歸。
- [ ] `obscura-benchmark` 障礙課程 33/33（該倉庫只存在於上游，本機沒有
      clone；可選，但 AGENTS.md 列為正式 gate）。

## 文件同步

- [ ] **README_ZH.md 重寫。** 目前仍是舊結構（含上游 trendshift 徽章、
      Docker Hub 連結、舊 Install 連結）。英文 README 已重寫為 fork 版本
      （34 個標題、fork 歸屬、本地 Docker build、docker-compose 範例），
      中文版需完整跟進，繁體中文、技術識別字保留英文、結構 1:1 對應。
- [ ] `docker-compose.yaml` 與文件已一致（`obscura-cjk` + `build: .`）；
      若 (b) 選項生效（發布自己的 Docker Hub），需再同步。

## 本機環境（非仓库變更）

- [ ] 可選：`obscura-benchmark` 倉庫 clone 下來跑完整驗證。
- [ ] 磁碟衛生：`target/release/deps` 定期清舊 binary；重 build 前查
      `df -h /`。

## 已記錄在 AGENTS.md（無需再跟）

- fork 政策：本倉庫是獨立 fork，所有操作面引用（文件、安裝、release、
  Docker、CI、issue/security）指向本倉庫；上游僅保留於 Apache-2.0 授權
  歸屬、`obscura-benchmark`（僅存於上游）、歷史 PR 引述。
- Docker：映像檔本地 build（`obscura-cjk`，`docker-compose.yaml` 為正式
  部署方式）；本機容器已由 compose 管理。
- 不自動跑驗證：build / nextest / render capture / obstacle course
  只在用戶明確要求時執行。
- SVG fallback 字型 lazy loading 的 gotcha（`svg_font_database_with_fallbacks`
  獨立 OnceLock，只在頁面含 inline SVG text 時建置）。
- `render-repros/cjk/` 置於子目錄是故意的（`run.sh` 只 glob 頂層
  `*.html`）。

## 分支 / remote 現況

- `origin` → `h4ckf0r0day/obscura`（upstream，只讀參考）
- `lawlietr` → `Lawlietr/obscura-cjk`（自己的倉庫）
- `lawlietr/main` @ `2810cb4`（= 上游最新，933 commits ahead of 舊 base）
- `lawlietr/cjk` @ 本分支（rebased，5 commits：Docker config → CJK feature
  → AGENTS.md flags → README/README_ZH → AGENTS.md fixture note，
  + 本輪的 fork 身份重寫）
- 合併入口：https://github.com/Lawlietr/obscura-cjk/pull/new/cjk
