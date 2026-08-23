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
- [x] **決定 `docker.yml` 的去留。** 已定**路線 A**：改推 GitHub
      Container Registry（`ghcr.io/Lawlietr/obscura-cjk`），不用 Docker
      Hub、不配任何外部 secret（GH Actions 的 `GITHUB_TOKEN` 加上
      `packages:write` permission 即可）。公開倉庫的 GHCR 儲存免費額度
      5GB，映像 ~60MB 綽綽有餘。
- [ ] **`docker.yml` 改造為 GHCR 發佈。** 改動清單：
      1. base image 改成 `ghcr.io/Lawlietr/obscura-cjk`，push 目標改 GHCR；
      2. V8 在 buildx 交叉編譯（QEMU 模擬）下太慢，拆成 per-platform
         native runner 各自 build 單平台 image 並 push（tag 加 platform
         suffix，runner 矩陣可參考 `release.yml`），最後一個 job 用
         `buildx imagetools create` 組多平台 manifest list；
      3. job 加 `permissions: contents: read, packages: write`。
      全部編譯在 GitHub Actions 上完成，本機不需要 build；本機最多 pull
      驗證。
- [ ] GHCR 映像上線後同步文件：`docker-compose.yaml` 與 README / README_ZH
      的 Docker 節保留「本地 `build: .`」為主線，GHCR pull 列為可選
      （`image: ghcr.io/Lawlietr/obscura-cjk:vX.Y.Z-cjk.N`）。
- [ ] 打第一個 tag push 觸發首次 release；驗證 artifacts 的 CJK 變體
      截圖正常。
- 額度備註：公開倉庫每月 2000 分鐘免費 Actions；5 平台 × V8 全編
  約每平台 10–15 分鐘，单次 release 沒有額度問題。

## Dependabot（依賴更新）

現況：`deny.toml` + `ci.yml` 的 `cargo-deny-action` 已有被動式漏洞/授權/
bans 檢查（有 5 個明確 ignore 的 RUSTSEC），但「只擋不升」：transitive
crate 出新版本時沒有任何機制自動提 PR。加官方 Dependabot 補主動式更新，
與 cargo-deny 互補不衝突。公開倉庫免費。

- [ ] **新增 `.github/dependabot.yml`。** 建議設定：
  - `package-ecosystem: "cargo"`, `directory: "/"`, `schedule: weekly`（較安靜；
    本倉庫 V8 全編一個 PR 的 CI 成本不低，daily 會比較吵）。
  - Security updates 開，`open-limit` 調小（如 5）。
  - 例行 version updates 可先只對 lockfile 更新開（`lockfile: true`），並用
    `groups` 合併，避免 468 個套件一次開一堆 PR。
  - `v8` / `deno_core` 這類重依賴建議先 exclude 或獨立 group，升版涉及
    V8 ABI/編譯，要人工看 diff 再合。
- [ ] Repo Settings 確認 Dependabot GitHub App 權限（public repo 預設已啟用，
  無需額外開關；security alerts 預設開）。
- [ ] 觀察首批 PR：確認 `[patch.crates-io]` 的 vendored `taffy` / `cosmic-text`
  行為符合預期（本體不會被更新，屬正常 warning；其 transitive 依賴仍會進
  lockfile 更新）。
- [ ] 後續維護：升級後視需要同步清理 `deny.toml` 的 ignore 清單
  （RUSTSEC ID 綁定 transitive 版本，cargo-deny CI 會提示）。

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

- [x] **README.md 精簡（614 → 243 行）。** 結構：Header / What this fork
      adds / Why / Install（Download、Docker、docker-compose、source 全保留）
      / Documentation 索引 / License。中間各節移至 docs：CJK 節新增
      `docs/CJK-and-custom-fonts.md`；CDP domain table 移入
      `docs/Architecture-overview.md`（新節「CDP surface」）；Integrations
      移入 `docs/README.md`；`--fonts` / `OBSCURA_FONTS_DIR` 補進
      `docs/CLI-reference.md` 與 `docs/Environment-variables.md`；Quick Start /
      localhost SSRF / scrape / stealth 各節確認已有 docs 對應（Your-first-fetch、
      Extract-data、CLI-reference、Environment-variables、Configure-stealth-and-proxies）。
- [x] **README_ZH.md 重寫（237 行，與新版英文 README 標題 1:1）。**
      砍掉上游 trendshift 徽章、Docker Hub 映像、AUR/NixOS 舊節；
      icon 改本倉庫相對路徑；所有安裝/文件連結指本倉庫與 docs/。
      docs/ 保持英文不翻譯（中文版只鏈到英文 docs）。
- [ ] `docker-compose.yaml` 與文件已一致（`obscura-cjk` + `build: .`）；
      GHCR 映像上線後再同步（見上方待辦）。

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
