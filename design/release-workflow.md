# Release 與發布機制

待辦追蹤見 `TODO.md`。本文件記錄 release 機制、版本/tag 策略、GHCR
映像發佈與相關決定理由。

## GitHub Actions workflow 概況

fork 已繼承三個 workflow（`.github/workflows/`）：

- `release.yml`：push 任何 `v*` tag 觸發。5 平台全部**原生**建置
  （x86_64 linux / aarch64 linux / aarch64 macos / x86_64 macos /
  windows），每平台 5 個 feature 變體（`render,cjk`、`render`、
  `render,stealth`、`no-render`、`no-render,stealth`），各打包 `obscura`
  + `obscura-worker`，上傳前逐變體 smoke test（V8 isolate + 截圖；cjk
  變體另截漢字頁），發布 job 只下載 artifacts、不 checkout 程式碼
  （token 隔離）。
- `docker.yml`：`v*` tag 觸發，per-platform native runner 各自 build
  單平台 image 推 GHCR（`ghcr.io/lawlietr/obscura-cjk`），最後一個 job
  組 multi-platform manifest list；只用內建 `GITHUB_TOKEN`
  （`packages: write`），無外部 secret。
- `ci.yml`：PR checks，read-only token。

額度備註：公開倉庫每月 2000 分鐘免費 Actions；5 平台 × V8 全編約每平台
10–15 分鐘，單次 release 沒有額度問題。

## 版本與 tag 策略

- 全新 fork 版本線，自 **`v0.1.0-cjk`** 起跳，後續 `vX.Y.Z-cjk[.N]`。
  workspace version 已是 `0.1.0`（非先前筆記所述的上游 1.0.103），
  二進位 `--version` 與 tag 自然對齊。tag 驅動，非 workspace version
  驅動。
- **v0.2.0-cjk 選 MINOR（非 0.1.2）的理由**：容器 nonroot 屬部署行為
  變更（掛載 storage dir 權限不符時 cookie jar 靜默不持久化），加 SSRF
  封鎖範圍擴大與 ICU locale pin，對 0.x 線達 MINOR 級；0.2.0 給使用者
  明確的上游大合併分界點。
- 首個 tag（`v0.1.0-cjk` @ `84c7223`，2026-08-24）已推送並觸發首次
  release 成功。

## GHCR 映像發佈（docker.yml，路線 A）

- 改推 GitHub Container Registry（`ghcr.io/lawlietr/obscura-cjk`），
  不用 Docker Hub、不配任何外部 secret（`GITHUB_TOKEN` +
  `packages: write` 即可）。公開倉庫的 GHCR 儲存免費額度 5GB，映像
  ~60MB 綽綽有餘。
- 實作：per-platform native runner 矩陣（amd64=`ubuntu-latest`、
  arm64=`ubuntu-24.04-arm`）各自 build 單平台 image，push 到
  `ghcr.io/lawlietr/obscura-cjk:<version>-<arch>`；最後
  `publish-manifest` job 用 `docker buildx imagetools create` 組出
  `<version>` 與 `latest` 的 multi-platform manifest list 並
  `imagetools inspect` 驗證。無 QEMU、無外部 secret（job 層
  `permissions: contents: read, packages: write`）；buildx GHA cache
  按 platform scope 分離避免並行寫入衝突。映像 path 必須全小寫
  （GHCR 規定）。
- 首發後一次性設定免做：經由 repo workflow token 發佈的 GHCR package
  自動繼承 public repo 可見性，匿名 pull manifest list 回 HTTP 200
  （`0.1.0-cjk` 與 `latest` 皆公開可讀）。
- 映像 tag 慣例（2026-08-24 定案）：說明文件與 `docker-compose.yaml`
  一律以 `latest` 為主。寫死版本號會讓每次發佈都要回頭改 README ×2 /
  Installation.md / AGENTS.md 範例，必然漂移。慣例：`latest` 跟隨最新
  release；版本 tag 保留作為回退與可重現部署用，文件以「釘選特定版本」
  範例呈現該機制（範例值會過時但語意不變）。適用：README.md /
  README_ZH.md / docs/Installation.md / AGENTS.md（含兩個 compose
  範例）/ docker-compose.yaml。
- `release.yml` 的 `render,cjk` 變體：build + stage（`dist/cjk`）+
  package（每平台多一個 `<name>-cjk.tar.gz` / `.zip`）+ smoke test
  迴圈加入該變體；cjk 變體額外對含漢字/平假名的頁面截圖（HTML 用
  numeric character references 保持 data: URL 純 ASCII），實際走內嵌
  fallback 字形的 shaping 路徑。

## v0.1.0-cjk 首發抽驗（2026-08-24，通過）

- Release run 成功（約 28 分鐘），25 個資產 = 5 平台 × 5 變體；Docker
  run 成功（約 7 分鐘），manifest list 含 linux/amd64 + linux/arm64 +
  attestation。
- 本機抽檔通過：cjk 二進位 vs 預設二進位對照渲染 CJK fixture（中日文
  段落墨量比 5.7–9.2×、nocjk 版呈固定豆腐框、Latin 行逐像素相同）、
  容器內 eval + 截圖、compose 以映像重建後 MCP endpoint 正常回應
  initialize。

## v0.2.0-cjk 上游大合併（2026-09-07）

- 合併範圍：自分叉點 `c1380190` 起 114 commits（~40 PR，+9482/−1056
  行，44 檔）；無 v8/deno ABI 變動。
- 3 個衝突全數按預先評估處理：Dockerfile（保留 `debian:12-slim`+TZ，
  加上游 #859 nonroot → `USER 65532:65532`）、README.md（整份保留
  fork 版，丟上游行銷區塊）、docs/Installation.md（吸收 nonroot 事實
  段落）。
- 重點內容：SSRF 封鎖補強（embedded-address / CGNAT / IANA
  special-purpose）、fetch body 與 fetched_urls 上限、isolate
  teardown / watchdog 健壯性、一批 CDP/Playwright 相容修復、ICU
  locale 釘死 en-US、`--allow-private-network` 穿進 stealth client
  （#831）、Docker nonroot。
- tag 前文件同步（已完成）：`docs/Use-as-a-Rust-library.md` pin 改
  `v0.2.0-cjk`；AGENTS.md（compose example 說明、nonroot 條目）；
  `docs/Run-in-production-at-scale.md`（nonroot 段落改述本 fork 的
  `debian:12-slim`+USER 65532、docker run 範例改 GHCR 映像、去上游
  `--stealth` 旗標）；`docker-compose.example.yaml` 補 storage dir
  權限註解。
