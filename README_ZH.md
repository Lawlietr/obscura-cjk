<p align="center">
  <img src="assets/icon.png" alt="Obscura" width="80" />
</p>
<h2 align="center">Obscura (CJK fork)</h2>
<p align="center">
  <a href="https://github.com/Lawlietr/obscura-cjk/releases"><img src="https://img.shields.io/badge/Releases-1a1a1a?style=for-the-badge&logo=github&logoColor=white" alt="Releases" /></a>
</p>
<p align="center">
  <a href="https://github.com/h4ckf0r0day/obscura">Obscura</a>（Apache-2.0）的 fork，
  加入內嵌 CJK 字型渲染。
</p>
<p align="center">
  📄 文件：<a href="README.md">English</a> · 繁體中文（本檔）
</p>

---

這是 [Obscura](https://github.com/h4ckf0r0day/obscura) 的 fork。Obscura 是用
Rust 寫的 headless browser engine，專攻網頁爬取與 AI agent 自動化：以 V8 執行
真實 JavaScript、講 Chrome DevTools Protocol，對 Puppeteer 與 Playwright 而言
可當 headless Chrome 直接替換。上游專案為 Apache-2.0 授權；本 fork 不更動
engine，只加入 CJK 字型渲染。

### 本 fork 增加了什麼

- **內嵌 CJK fallback 字型。** 新的 `cjk` cargo feature 把 Noto Sans CJK
  SC/TC（Regular、SIL OFL 1.1）嵌入二進位，中文與日文字形不依賴主機字型或
  頁面 webfont。`-cjk` release 打包與 Docker 映像皆已啟用。
- **執行期字型目錄。** 全域 `--fonts <PATH>` flag 與 `OBSCURA_FONTS_DIR`
  環境變數可在執行期載入額外字型檔，補內嵌字集沒覆蓋的書寫系統（例如
  Hangul 韓文）。
- **docker-compose 部署範例。** CDP server 與 MCP HTTP server 兩種模式，
  見 [Docker](#docker)。

其餘功能——渲染、截圖、PDF 匯出、CDP、MCP、stealth——全部是上游功能、原樣
可用。設計與限制詳見
[CJK 與自訂字型](docs/CJK-and-custom-fonts.md)。

### 為什麼選 Obscura 而不是 headless Chrome？

| 指標         | Obscura      | Headless Chrome |
|--------------|--------------|------------------|
| 記憶體       | **30 MB**    | 200+ MB          |
| 二進位大小   | **70 MB**    | 300+ MB          |
| 反偵測       | **內建**     | 無              |
| 頁面載入     | **85 ms**    | ~500 ms          |
| 啟動         | **即開**     | ~2s              |
| Puppeteer    | **支援**     | 支援             |
| Playwright   | **支援**     | 支援             |

## 安裝

### 下載

從 [Releases](https://github.com/Lawlietr/obscura-cjk/releases) 取得最新二進位：

```bash
# Linux x86_64
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-x86_64-linux.tar.gz
tar xzf obscura-x86_64-linux.tar.gz
./obscura fetch https://example.com --eval "document.title"

# Linux ARM64 (aarch64)
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-aarch64-linux.tar.gz
tar xzf obscura-aarch64-linux.tar.gz

# macOS Apple Silicon
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-aarch64-macos.tar.gz
tar xzf obscura-aarch64-macos.tar.gz

# macOS Intel
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-x86_64-macos.tar.gz
tar xzf obscura-x86_64-macos.tar.gz

# Windows
從 releases 頁面下載 `.zip` 並手動解壓縮。
```

不需要 Chrome、不需要 Node.js、零相依。release 打包內含 `obscura` 與
`obscura-worker` 兩個執行檔；並行的 `scrape` 命令需要它們放在同一目錄。

| 打包副檔名 | 渲染 | Stealth transport | 內嵌 CJK |
|------------|------|-------------------|----------|
| 無 | 有 | 無 | 無 |
| `-cjk` | 有 | 無 | 有 |
| `-stealth` | 有 | 有 | 無 |
| `-no-render` | 無 | 無 | 無 |
| `-no-render-stealth` | 無 | 有 | 無 |

Linux release 以 Ubuntu 22.04 為目標建置，故下載的二進位可在使用
glibc 2.35+ 的常見 LTS server 上執行。

### Docker

映像檔尚未發布到任何 registry；請用本倉庫的 `Dockerfile` 本地建置：

```bash
docker build -t obscura-cjk .

docker run -d --name obscura -p 127.0.0.1:9222:9222 obscura-cjk
```

多階段建置：builder 階段在 `rust:1-slim-bookworm` 上由原始碼編譯 V8，執行層
為 `debian:12-slim`，CA 憑證取自 distroless base image。映像檔以 `cjk`
feature 建置，CJK 文字開箱即渲染（見
[CJK 與自訂字型](docs/CJK-and-custom-fonts.md)）。

#### docker-compose

**CDP 模式**（Puppeteer/Playwright，port 9222）：

```yaml
services:
  obscura:
    image: obscura-cjk
    container_name: obscura
    restart: unless-stopped
    ports:
      - "0.0.0.0:9222:9222"
    command: ["serve"]
```

client 連線位址為 `ws://localhost:9222/devtools/browser`。

**MCP 模式**（AI agent 走 HTTP，port 3000）：

```yaml
services:
  obscura:
    image: obscura-cjk
    container_name: obscura
    restart: unless-stopped
    ports:
      - "0.0.0.0:3000:3000"
    command: ["mcp", "--http", "--port", "3000", "--host", "0.0.0.0"]
    environment:
      - OBSCURA_ALLOW_PRIVATE_NETWORK=0
      - OBSCURA_PROXY=
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    mem_limit: 256m
    cpus: 2.0
```

MCP endpoint 為 `http://localhost:3000/mcp`。若要補內嵌字集沒有的字型
（Hangul、CJK 字重），掛一個唯讀字型目錄並在 `environment` 設定
`OBSCURA_FONTS_DIR`。

### 從原始碼建置

```bash
git clone https://github.com/Lawlietr/obscura-cjk.git
cd obscura-cjk

# 渲染 + 內嵌 CJK fallback 字型（推薦）
cargo build --release -p obscura-cli --bins --features render,cjk

# 只渲染、不含 CJK
cargo build --release -p obscura-cli --bins --features render

# 渲染與 stealth
cargo build --release -p obscura-cli --bins --features render,stealth

# 無渲染
cargo build --release -p obscura-cli --bins --no-default-features

# 無渲染、含 stealth
cargo build --release -p obscura-cli --bins --no-default-features --features stealth
```

`cjk` feature 內嵌 Noto Sans CJK SC/TC（SIL OFL），讓中文與日文不依賴主機
字型即可渲染。它為選填，目的在讓基礎二進位小約 30 MB；`-cjk` release 打包
與 Docker 映像都已啟用。

需要 Rust 1.75+（[rustup.rs](https://rustup.rs)）。首次建置約 5 分鐘（V8
由原始碼編譯，之後有快取）。stealth 建置另需編譯 BoringSSL 並產生 bindings，
因此需要 CMake、Clang 與 libclang/LLVM 開發庫。Ubuntu/Debian：

```bash
sudo apt-get install build-essential cmake clang libclang-dev llvm-dev
```

渲染建置使用 rustls。渲染＋stealth 建置改用 wreq/BoringSSL，需上述額外
建置工具。

## 文件

安裝之後的一切內容都在 [docs/](docs/SUMMARY.md)：

**快速入門**

- [安裝](docs/Installation.md)
- [第一次 fetch](docs/Your-first-fetch.md)
- [抽出資料](docs/Extract-data.md)
- [連接 Puppeteer 或 Playwright](docs/Connect-Puppeteer-or-Playwright.md)

**指南**

- [從原始碼建置](docs/Build-from-source.md)
- [CJK 與自訂字型](docs/CJK-and-custom-fonts.md)
- [設定 stealth 與代理](docs/Configure-stealth-and-proxies.md)
- [Markdown 抽取](docs/Markdown-extraction.md)
- [搭配 Puppeteer](docs/Use-with-Puppeteer.md)
- [搭配 Playwright](docs/Use-with-Playwright.md)
- [使用 MCP server](docs/Use-the-MCP-server.md)
- [即時觀看 agent session](docs/Watch-agent-sessions-live.md)
- [作為 Rust library 使用](docs/Use-as-a-Rust-library.md)
- [持久化 cookies 與 storage](docs/Persist-cookies-and-storage.md)
- [攔截與修改請求](docs/Intercept-and-modify-requests.md)
- [生產環境大規模運行](docs/Run-in-production-at-scale.md)

**參考**

- [CLI 參照](docs/CLI-reference.md)
- [環境變數](docs/Environment-variables.md)

**貢獻**

- [架構總覽](docs/Architecture-overview.md)
- [新增 CDP method 或 Web API](docs/Adding-a-CDP-method-or-Web-API.md)
- [測試與除錯](docs/Testing-and-debugging.md)

完整基準測試套件（WPT 一致性、障礙課程、真實網站語料、對 Chrome 速度）在
上游倉庫：https://github.com/h4ckf0r0day/obscura-benchmark

## 授權

Apache 2.0。本 fork 源自 [h4ckf0r0day/obscura](https://github.com/h4ckf0r0day/obscura)
（Apache-2.0）；內嵌 CJK 字型為 Noto Sans CJK SC/TC，採 SIL OFL 1.1 授權
（見
[crates/obscura-render/assets/LICENSE-OFL.txt](crates/obscura-render/assets/LICENSE-OFL.txt)）。

---
