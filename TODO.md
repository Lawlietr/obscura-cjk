# TODO

Obscura CJK fork (`Lawlietr/obscura-cjk`) 待辦與重要事項。
工作分支：`main`。首個 release：**`v0.1.0-cjk`**（2026-08-24 發佈成功）。

## Release：GitHub Actions 二進位發布

上游的 release 機制是 GitHub Actions，fork 已繼承三個 workflow
（`.github/workflows/`）：

- `release.yml`：push 任何 `v*` tag 觸發。5 平台全部**原生**建置
  （x86_64 linux / aarch64 linux / aarch64 macos / x86_64 macos / windows），
  每平台 5 個 feature 變體（`render,cjk`、`render`、`render,stealth`、
  `no-render`、`no-render,stealth`），各打包 `obscura` + `obscura-worker`，
  上傳前逐變體 smoke test（V8 isolate + 截圖；cjk 變體另截漢字頁），
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
- [x] **首發後一次性設定。** 實測免做：經由 repo workflow token 發佈的
      GHCR package 自動繼承 public repo 可見性，匿名 pull manifest list
      回 HTTP 200（`0.1.0-cjk` 與 `latest` 皆公開可讀）。
- [x] **GHCR 映像上線後同步文件（方案 B 收尾，2026-08-24 完成）。**
      README.md / README_ZH.md Docker 節改為「GHCR pull 為部署主線 +
      本地 build 保留為本地開發流程」，compose 範例指向
      `ghcr.io/lawlietr/obscura-cjk:0.1.0-cjk`；docs/Installation.md 同步；
      AGENTS.md 四處更新（Docker Deployment 節、MCP/CDP 兩個 compose
      範例、fork 政策節）；docs/Use-as-a-Rust-library.md 的 pin 從
      `rev` 改回 `tag = "v0.1.0-cjk"`。
- [x] **映像 tag 慣例定案：說明文件與 docker-compose.yaml 一律以
      `latest` 為主（2026-08-24）。** 寫死版本號會讓每次發佈都要回頭改
      README ×2 / Installation.md / AGENTS.md 範例，必然漂移。慣例：
      `latest` 跟隨最新 release；版本 tag 保留作為回退與可重現部署用，
      文件以「釘選特定版本」範例呈現該機制（範例值會過時但語意不變）。
      適用：README.md / README_ZH.md / docs/Installation.md /
      AGENTS.md（含兩個 compose 範例）/ docker-compose.yaml。
- [x] **README 移除上游 Chrome 對比表（2026-08-24，EN/ZH 同步）。**
      理由：數據為上游行銷數字、無出處無方法；「Anti-detect: Built-in」
      對本 fork 錯誤（stealth 是 build-time feature，Docker 映像不含）；
      fork 身份下該賣點屬上游。intro 的 V8/CDP/drop-in 描述已涵蓋核心
      資訊，AGENTS.md 的效能基準紀律條目不受影響。
- [x] **打第一個 tag（`v0.1.0-cjk`）push 觸發首次 release（2026-08-24）。**
      Release run 成功（約 28 分鐘），25 個資產 = 5 平台 × 5 變體；
      Docker run 成功（約 7 分鐘），manifest list 含 linux/amd64 +
      linux/arm64 + attestation。本機抽檔通過：cjk 二進位 vs 預設二進位
      對照渲染 CJK fixture（中日文段落墨量比 5.7–9.2×、nocjk 版呈固定
      豆腐框、Latin 行逐像素相同）、容器內 eval + 截圖、compose 以映像
      重建後 MCP endpoint 正常回應 initialize。
- 額度備註：公開倉庫每月 2000 分鐘免費 Actions；5 平台 × V8 全編
  約每平台 10–15 分鐘，单次 release 沒有額度問題。

## Dependabot（依賴更新）

現況：`deny.toml` + `ci.yml` 的 `cargo-deny-action` 已有被動式漏洞/授權/
bans 檢查（有 5 個明確 ignore 的 RUSTSEC），但「只擋不升」：transitive
crate 出新版本時沒有任何機制自動提 PR。加官方 Dependabot 補主動式更新，
與 cargo-deny 互補不衝突。公開倉庫免費。

- [x] **新增 `.github/dependabot.yml`（2026-08-24 已寫入，push 後生效）。**
      實作：兩個 cargo entry（同 directory，週一 schedule）——例行 version
      updates 用 `lockfile: true`（只改 Cargo.lock、不動 Cargo.toml）並全部
      併入 `routine` group（一個週報 PR，CI 每週約一次 V8 全編）；
      `exclude-patterns` 排除 `v8` / `deno-core` / `deno-*`（ABI 敏感，升版
      留人工），`lockfile: true` 下 major bump 也只重寫 lockfile，故不設
      update-type 過濾、直接併入同一 group。Security entry 另開，
      `security-advisories: enabled`（V8/deno 家族的 RUSTSEC 也會進 PR），
      併入 `security` group，`open-pull-requests-limit: 5`。另加
      github-actions ecosystem（低頻、CI 便宜）。
- [ ] Repo Settings 確認 Dependabot GitHub App 權限（public repo 預設已啟用，
  無需額外開關；security alerts 預設開）。
- [ ] 觀察首批 PR：確認 `[patch.crates-io]` 的 vendored `taffy` / `cosmic-text`
  行為符合預期（本體不會被更新，屬正常 warning；其 transitive 依賴仍會進
  lockfile 更新）。
- [ ] 後續維護：升級後視需要同步清理 `deny.toml` 的 ignore 清單
  （RUSTSEC ID 綁定 transitive 版本，cargo-deny CI 會提示）。

## 回歸驗證（原「合併前必做」；cjk 分支工作已直接進 main，以下作為下次 release 前的品質關卡）

> 註：首個 release（v0.1.0-cjk）的 CI smoke test 已全數通過，完整本地
> 回歸已於 2026-08-25 補跑。

- [x] **完整回歸（2026-08-25）。**
      `render,cjk`：**1487/1487** passed（4 skipped）。
      `render`：**1486/1486** passed（4 skipped）。
      建置 6m 19s，測試各 ~16s。上游 933 commits 中有 1 個動到
      `paint.rs`（box-shadow clip，`00a8a81`）已驗證通過。
- [x] **磁碟檢查（2026-08-25）。** 起始 14G 可用，測試後 6.5G 可用
      （78%）。清理測試二進位後 target/ 從 6.5G → 802M。
- [x] **CJK 視覺抽檢（2026-08-25）。** 繁/簡/日文字形渲染正確，
      截圖 55KB，無豆腐框。
- [x] **非 cjk 路徑抽測（2026-08-25）。** 1486/1486 passed，無回歸。
- [x] **`obscura-benchmark` 障礙課程 32/33（2026-08-25）。**
      32/33 stages passed。已知問題：`observer-intersection` 失敗
      （預期 `'io:50'`，實際 `''`）。根因：Obscura headless 模式不模擬
      scroll，IntersectionObserver 的 sentinel element 在 viewport 下方，
      callback 只觸發一次。上游 Obscura 同樣有這個問題（fixture 註解
      聲稱 treat targets as intersecting，但實際上沒有）。此為 headless
      模式的本質限制，非本 fork 問題。

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
- [x] `docker-compose.yaml` 已切換為 GHCR 映像部署
      （`image: ghcr.io/lawlietr/obscura-cjk:latest`，移除 `build: .`；
      升級時改 pin 版本號即可回退），本地 build 說明保留在註解與 README；
      本機已用新 compose 重建容器驗證。

## 本機環境（非仓库變更）

- [ ] 可選：`obscura-benchmark` 倉庫 clone 下來跑完整驗證。
- [ ] 磁碟衛生：`target/release/deps` 定期清舊 binary；重 build 前查
      `df -h /`。

## 已記錄在 AGENTS.md（無需再跟）

- fork 政策：本倉庫是獨立 fork，所有操作面引用（文件、安裝、release、
  Docker、CI、issue/security）指向本倉庫；上游僅保留於 Apache-2.0 授權
  歸屬、`obscura-benchmark`（僅存於上游）、歷史 PR 引述。
- Docker：release 映像由 GitHub Actions 發佈至
  `ghcr.io/lawlietr/obscura-cjk`（每個 `v*` tag；`latest` 跟隨最新，
  版本 tag 供回退）；`docker-compose.yaml` 跟隨 `latest` 為正式部署，
  `docker build -t obscura-cjk .` 保留為本地開發流程；本機容器已由
  compose 管理。
- 不自動跑驗證：build / nextest / render capture / obstacle course
  只在用戶明確要求時執行。
- SVG fallback 字型 lazy loading 的 gotcha（`svg_font_database_with_fallbacks`
  獨立 OnceLock，只在頁面含 inline SVG text 時建置）。
- `render-repros/cjk/` 置於子目錄是故意的（`run.sh` 只 glob 頂層
  `*.html`）。

## 分支 / remote 現況

- `origin` → `Lawlietr/obscura-cjk`（自己的倉庫，唯一 remote）
- 工作分支：`main`（追蹤 `origin/main`）
- 首個 tag：`v0.1.0-cjk` @ `84c7223`（2026-08-24 推送，觸發首次 release）
- 上游 `h4ckf0r0day/obscura` 目前未設 remote，僅歷史與 Apache-2.0 授權歸屬
  參考
