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
- `docker.yml`：`v*` tag 觸發，per-platform native runner 各自 build 單平台
  image 推 GHCR（`ghcr.io/lawlietr/obscura-cjk`），最後一個 job 組
  multi-platform manifest list；只用內建 `GITHUB_TOKEN`
  （`packages: write`），無外部 secret。
- `ci.yml`：PR checks，read-only token。

### 待辦

- [x] **`release.yml` 加 `render,cjk` 變體。** build + stage（`dist/cjk`）
      + package（每平台多一個 `<name>-cjk.tar.gz` / `.zip`）+ smoke test
      迴圈加入該變體；cjk 變體額外對含漢字/平假名的頁面截圖（HTML 用
      numeric character references 保持 data: URL 純 ASCII），實際走內嵌
      fallback 字形的 shaping 路徑。
- [x] **決定 tag 策略。** 全新 fork 版本線，自 **`v0.1.0-cjk`** 起跳，
      後續 `vX.Y.Z-cjk[.N]`。workspace version 已是 `0.1.0`（非先前筆記
      所述的上游 1.0.103），二進位 `--version` 與 tag 自然對齊。第一個
      tag 推出後可把 `docs/Use-as-a-Rust-library.md` 的依賴 pin 從 `rev`
      改回 `tag`。
- [x] **決定 `docker.yml` 的去留。** 已定**路線 A**：改推 GitHub
      Container Registry（`ghcr.io/Lawlietr/obscura-cjk`），不用 Docker
      Hub、不配任何外部 secret（GH Actions 的 `GITHUB_TOKEN` 加上
      `packages:write` permission 即可）。公開倉庫的 GHCR 儲存免費額度
      5GB，映像 ~60MB 綽綽有餘。
- [x] **`docker.yml` 改造為 GHCR 發佈。** 實作：per-platform native
      runner 矩陣（amd64=`ubuntu-latest`、arm64=`ubuntu-24.04-arm`）各自
      build 單平台 image，push 到
      `ghcr.io/lawlietr/obscura-cjk:<version>-<arch>`；最後 `publish-manifest`
      job 用 `docker buildx imagetools create` 組出 `<version>` 與 `latest`
      的 multi-platform manifest list 並 `imagetools inspect` 驗證。無
      QEMU、無外部 secret（job 層 `permissions: contents: read,
      packages: write`）；buildx GHA cache 按 platform scope 分離避免並行
      寫入衝突。映像 path 必須全小寫（GHCR 規定）。
- [ ] **首發後一次性設定：** 第一次 push 映像後，到 repo Packages →
      `obscura-cjk` → Package settings 把 visibility 改 public（GHCR 新
      package 預設 private，不設 public 匿名 pull 會被拒）。
- [ ] GHCR 映像上線後同步文件（方案 B：registry 相關描述等首次發布成功
      後再改）：README / README_ZH / docs/Installation.md 三處「image is
      not on a registry (yet)」改為本地 build 主線 + GHCR pull 可選
      （`ghcr.io/lawlietr/obscura-cjk:vX.Y.Z-cjk.N`）；AGENTS.md 兩處
      「not published to a registry」（Docker Deployment 節與 fork 政策節）
      同步修正；`docker-compose.yaml` 視需要加註解版 GHCR image 替代行。
- [ ] 打第一個 tag（`v0.1.0-cjk`）push 觸發首次 release；驗證 artifacts
      的 CJK 變體截圖正常、`docker pull` 多平台 manifest list 成功。
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

- [x] **Release 前置文件同步（2026-08-24，方案 B 範圍）。** README.md /
      README_ZH.md 下載變體表加「Embedded CJK / 內嵌 CJK」欄與 `-cjk` 列；
      fork 特性清單與 Build-from-source 節的「release archives 內建 CJK」
      收斂為 `-cjk` 變體專屬；README.md Docker 節、docs/Installation.md、
      docs/CJK-and-custom-fonts.md 同步；修正 Docker 映像描述錯誤（runtime
      實為 `debian:12-slim` 非 `distroless/cc`，CA 憑證取自 distroless，
      移除未驗證的 ~57 MB 數字）；docs/Installation.md 的「release archives
      皆含渲染引擎」改為精確列舉渲染變體。
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
