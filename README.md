<p align="center">
  <img src="assets/icon.png" alt="Obscura" width="80" />
</p>
<h2 align="center">Obscura (CJK fork)</h2>
<p align="center">
  <a href="https://github.com/Lawlietr/obscura-cjk/releases"><img src="https://img.shields.io/badge/Releases-1a1a1a?style=for-the-badge&logo=github&logoColor=white" alt="Releases" /></a>
</p>
<p align="center">
  A fork of <a href="https://github.com/h4ckf0r0day/obscura">Obscura</a>
  (Apache-2.0) with embedded CJK font rendering.
</p>
<p align="center">
  📄 Documentation: English (this file) · <a href="README_ZH.md">繁體中文</a>
</p>

---

This is a fork of [Obscura](https://github.com/h4ckf0r0day/obscura), a headless
browser engine written in Rust for web scraping and AI agent automation. It
runs real JavaScript via V8, speaks the Chrome DevTools Protocol, and is a
drop-in replacement for headless Chrome with Puppeteer and Playwright. The
upstream project is licensed under Apache-2.0; this fork keeps the engine
unchanged and adds CJK font rendering.

### What this fork adds

- **Embedded CJK fallback fonts.** A `cjk` cargo feature embeds Noto Sans CJK
  SC/TC (Regular, SIL OFL 1.1) in the binary, so Chinese and Japanese text
  renders without host fonts or page webfonts. On in the `-cjk` release
  archives and the Docker image.
- **Runtime font directory.** A global `--fonts <PATH>` flag and the
  `OBSCURA_FONTS_DIR` environment variable load extra font files at runtime,
  for scripts the embedded set does not cover (e.g. Hangul).
- **docker-compose deployment recipes** for the CDP server and the MCP HTTP
  server, in [Docker](#docker).

Everything else — rendering, screenshots, PDF export, CDP, MCP, stealth — is
upstream functionality and works unchanged. See
[CJK and custom fonts](docs/CJK-and-custom-fonts.md) for the design and
limitations.

## Install

### Download

Grab the latest binary from [Releases](https://github.com/Lawlietr/obscura-cjk/releases):

```bash
# Linux x86_64
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-x86_64-linux-cjk.tar.gz
tar xzf obscura-x86_64-linux-cjk.tar.gz
./obscura fetch https://example.com --eval "document.title"

# Linux ARM64 (aarch64)
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-aarch64-linux-cjk.tar.gz
tar xzf obscura-aarch64-linux-cjk.tar.gz

# macOS Apple Silicon
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-aarch64-macos-cjk.tar.gz
tar xzf obscura-aarch64-macos-cjk.tar.gz

# macOS Intel
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-x86_64-macos-cjk.tar.gz
tar xzf obscura-x86_64-macos-cjk.tar.gz

# Windows
Download the `.zip` from the releases page and extract it manually.
```

No Chrome, no Node.js, no dependencies. Release archives include both
`obscura` and `obscura-worker`; keep them in the same directory for the
parallel `scrape` command.

| Archive suffix | Rendering | Stealth transport | Embedded CJK |
|----------------|-----------|-------------------|--------------|
| `-cjk` | Yes | No | Yes |
| `-cjk-stealth` | Yes | Yes | Yes |

Linux release builds target Ubuntu 22.04 so the downloaded binary remains
usable on common LTS servers with glibc 2.35+.

### Docker

The multi-platform image (linux/amd64, linux/arm64) is published to GHCR by
GitHub Actions on every `v*` tag; `latest` tracks the newest release:

```bash
docker run -d --name obscura -p 127.0.0.1:9222:9222 ghcr.io/lawlietr/obscura-cjk:latest

# Pin a specific release for reproducibility and easy rollback
docker run -d --name obscura -p 127.0.0.1:9222:9222 ghcr.io/lawlietr/obscura-cjk:0.1.0-cjk
```

The repo's `docker-compose.yaml` deploys this image.

For local development, build it from this repo's `Dockerfile`:

```bash
docker build -t obscura-cjk .

docker run -d --name obscura -p 127.0.0.1:9222:9222 obscura-cjk
```

Multi-stage build: V8 is compiled from source on a `rust:1-slim-bookworm`
builder stage; the runtime layer is `debian:12-slim` with CA certificates
taken from the distroless base image. The image is built with the `cjk`
feature on, so CJK text renders out of the box (see
[CJK and custom fonts](docs/CJK-and-custom-fonts.md)).

#### docker-compose

**CDP mode** (Puppeteer/Playwright, port 9222):

```yaml
services:
  obscura:
    image: ghcr.io/lawlietr/obscura-cjk:latest
    container_name: obscura
    restart: unless-stopped
    ports:
      - "0.0.0.0:9222:9222"
    command: ["serve"]
```

Then connect clients at `ws://localhost:9222/devtools/browser`.

**MCP mode** (AI agents over HTTP, port 3000):

```yaml
services:
  obscura:
    image: ghcr.io/lawlietr/obscura-cjk:latest
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

The MCP endpoint is `http://localhost:3000/mcp`. To add fonts the embedded set
lacks (Hangul, CJK weights), mount a read-only directory with the font files
and set `OBSCURA_FONTS_DIR` in `environment`.

### Build from source

```bash
git clone https://github.com/Lawlietr/obscura-cjk.git
cd obscura-cjk

# Rendering with embedded CJK fallback fonts (recommended)
cargo build --release -p obscura-cli --bins --features render,cjk

# Rendering only, without CJK
cargo build --release -p obscura-cli --bins --features render

# Rendering and stealth
cargo build --release -p obscura-cli --bins --features render,stealth

# No rendering
cargo build --release -p obscura-cli --bins --no-default-features

# No rendering, with stealth
cargo build --release -p obscura-cli --bins --no-default-features --features stealth
```

The `cjk` feature embeds Noto Sans CJK SC/TC (SIL OFL) so Chinese and Japanese
render without host fonts. It is optional, to keep the base binary ~30 MB
smaller; the `-cjk` release archive and Docker image are built with it on.

Requires Rust 1.75+ ([rustup.rs](https://rustup.rs)). First build takes ~5 min (V8 compiles from source, cached after).
The stealth build also compiles BoringSSL and generates bindings, so it needs
CMake, Clang, and the libclang/LLVM development libraries. On Ubuntu/Debian:

```bash
sudo apt-get install build-essential cmake clang libclang-dev llvm-dev
```

The rendering build uses rustls. The rendering-and-stealth build uses
wreq/BoringSSL and therefore needs the additional build tools above.

## Documentation

Everything past installation lives in [docs/](docs/SUMMARY.md):

**Get started**

- [Installation](docs/Installation.md)
- [Your first fetch](docs/Your-first-fetch.md)
- [Extract data](docs/Extract-data.md)
- [Connect Puppeteer or Playwright](docs/Connect-Puppeteer-or-Playwright.md)

**Guides**

- [Build from source](docs/Build-from-source.md)
- [CJK and custom fonts](docs/CJK-and-custom-fonts.md)
- [Configure stealth and proxies](docs/Configure-stealth-and-proxies.md)
- [Markdown extraction](docs/Markdown-extraction.md)
- [Use with Puppeteer](docs/Use-with-Puppeteer.md)
- [Use with Playwright](docs/Use-with-Playwright.md)
- [Use the MCP server](docs/Use-the-MCP-server.md)
- [Watch agent sessions live](docs/Watch-agent-sessions-live.md)
- [Use as a Rust library](docs/Use-as-a-Rust-library.md)
- [Persist cookies and storage](docs/Persist-cookies-and-storage.md)
- [Intercept and modify requests](docs/Intercept-and-modify-requests.md)
- [Run in production at scale](docs/Run-in-production-at-scale.md)

**Reference**

- [CLI reference](docs/CLI-reference.md)
- [Environment variables](docs/Environment-variables.md)

**Contributing**

- [Architecture overview](docs/Architecture-overview.md)
- [Adding a CDP method or Web API](docs/Adding-a-CDP-method-or-Web-API.md)
- [Testing and debugging](docs/Testing-and-debugging.md)

The full benchmark suite (WPT conformance, obstacle course, real-world corpus,
and vs-Chrome speed) lives in the upstream repo:
https://github.com/h4ckf0r0day/obscura-benchmark

## License

Apache 2.0. Fork of [h4ckf0r0day/obscura](https://github.com/h4ckf0r0day/obscura)
(Apache-2.0); the embedded CJK fonts are Noto Sans CJK SC/TC, released under
SIL OFL 1.1 (see
[crates/obscura-render/assets/LICENSE-OFL.txt](crates/obscura-render/assets/LICENSE-OFL.txt)).

---
