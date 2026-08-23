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
  renders without host fonts or page webfonts. On in the release archives and
  the Docker image.
- **Runtime font directory.** A global `--fonts <PATH>` flag and the
  `OBSCURA_FONTS_DIR` environment variable load extra font files at runtime,
  for scripts the embedded set does not cover (e.g. Hangul).
- **docker-compose deployment recipes** for the CDP server and the MCP HTTP
  server, in [Docker](#docker).

Everything else — rendering, screenshots, PDF export, CDP, MCP, stealth — is
upstream functionality and works unchanged.

### Why Obscura over headless Chrome?

| Metric       | Obscura      | Headless Chrome |
|--------------|--------------|------------------|
| Memory       | **30 MB**    | 200+ MB          |
| Binary size  | **70 MB**    | 300+ MB          |
| Anti-detect  | **Built-in** | None          |
| Page load    | **85 ms**    | ~500 ms          |
| Startup      | **Instant**  | ~2s              |
| Puppeteer    | **Yes**      | Yes              |
| Playwright   | **Yes**      | Yes              |


## Install

### Download

Grab the latest binary from [Releases](https://github.com/Lawlietr/obscura-cjk/releases):

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
Download the `.zip` from the releases page and extract it manually.
```

No Chrome, no Node.js, no dependencies. Release archives include both
`obscura` and `obscura-worker`; keep them in the same directory for the
parallel `scrape` command.

| Archive suffix | Rendering | Stealth transport |
|----------------|-----------|-------------------|
| none | Yes | No |
| `-stealth` | Yes | Yes |
| `-no-render` | No | No |
| `-no-render-stealth` | No | Yes |

Linux release builds target Ubuntu 22.04 so the downloaded binary remains
usable on common LTS servers with glibc 2.35+.

### Docker

The image is not on a registry yet; build it from the `Dockerfile` in this
repo:

```bash
docker build -t obscura-cjk .

docker run -d --name obscura -p 127.0.0.1:9222:9222 obscura-cjk
```

Multi-stage build on `distroless/cc`, no shell, no package manager, ~57 MB
compressed. The image is built with the `cjk` feature on, so CJK text renders
out of the box (see [CJK and custom fonts](#cjk-and-custom-fonts)).

#### docker-compose

**CDP mode** (Puppeteer/Playwright, port 9222):

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

Then connect clients at `ws://localhost:9222/devtools/browser`.

**MCP mode** (AI agents over HTTP, port 3000):

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
smaller; the release archives and Docker image are built with it on.

Requires Rust 1.75+ ([rustup.rs](https://rustup.rs)). First build takes ~5 min (V8 compiles from source, cached after).
The stealth build also compiles BoringSSL and generates bindings, so it needs
CMake, Clang, and the libclang/LLVM development libraries. On Ubuntu/Debian:

```bash
sudo apt-get install build-essential cmake clang libclang-dev llvm-dev
```

The rendering build uses rustls. The rendering-and-stealth build uses
wreq/BoringSSL and therefore needs the additional build tools above.

## Quick Start

### Fetch a page

```bash
# Get the page title
obscura fetch https://example.com --eval "document.title"

# Extract all links
obscura fetch https://example.com --dump links

# Render JavaScript and dump HTML
obscura fetch https://news.ycombinator.com --dump html

# Write dump or eval output to a file
obscura fetch https://example.com --dump text --output page.txt

# Stream the raw response body verbatim (binary-safe; bypasses the JS/DOM layer).
# Use this for images, JSON, JS, CSS, or any non-HTML resource.
obscura fetch https://picsum.photos/200/300 --dump original > photo.jpg

# List every sub-resource URL the page would fetch (NDJSON; one record per asset)
obscura fetch https://example.com --dump assets

# Fetch through an HTTP or SOCKS proxy
obscura --proxy socks5://127.0.0.1:1080 fetch https://example.com --dump text

# Wait for dynamic content
obscura fetch https://example.com --wait-until networkidle0

# Bound navigation time for slow or broken pages
obscura fetch https://example.com --timeout 10

# Capture the settled page as PNG
obscura fetch https://example.com --screenshot page.png

# The screenshot flag also has a short form
obscura fetch https://example.com -s page.png

### Testing against localhost / LAN dev servers

Obscura blocks fetches to private/internal IPs by default (SSRF protection).
To point it at a local dev server, pass `--allow-private-network` (or set
`OBSCURA_ALLOW_PRIVATE_NETWORK=1`):

```bash
obscura fetch http://127.0.0.1:3000 --allow-private-network --dump text

# Works on any subcommand, e.g. the CDP server for local Puppeteer/Playwright:
obscura serve --port 9222 --allow-private-network
```

See [docs/Environment-variables.md](docs/Environment-variables.md) for the
full allow/deny rules (DNS-resolution-time checks included).
```

## Rendering

Official release archives and the Docker image include the rendering engine.
It provides CSS layout and paint, viewport and full-page screenshots,
scroll-aware fixed and sticky geometry, activity-driven CDP screencasting, and
raster PDF export without starting Chromium.

```javascript
await page.setViewport({ width: 1440, height: 1000 });
await page.goto('https://example.com', { waitUntil: 'load' });
await page.screenshot({ path: 'page.png', fullPage: true });
await page.pdf({ path: 'page.pdf', format: 'A4', printBackground: true });
```

The current implementation covers block, inline, flex, grid, table, float,
positioning, overflow, transform, text, image, SVG, canvas, background, border,
and animation paths. It remains an evolving independent
engine: long-tail CSS, some Web APIs, media playback, compositor effects, and
platform font rasterization may differ from Chromium. The existing
[Puppeteer](docs/Use-with-Puppeteer.md),
[Playwright](docs/Use-with-Playwright.md), and
[MCP](docs/Use-the-MCP-server.md) guides cover their capture APIs and limits.

### CJK and custom fonts

**What changed**

- A new `cjk` cargo feature (on in the release archives and the Docker image).
  When enabled, two faces are embedded in the binary as glyph-level fallbacks:
  **Noto Sans CJK SC** and **Noto Sans CJK TC**, Regular weight, SIL OFL 1.1
  (~16 MB each).
- A new global `--fonts <PATH>` flag and the `OBSCURA_FONTS_DIR` environment
  variable load extra font files from a directory at runtime, for scripts the
  embedded set does not cover.

**Why we did it this way**

Obscura deliberately loads **no system fonts**. That keeps layout
bit-for-bit identical on every machine and startup instant, but it meant
Chinese and Japanese text rendered as tofu (empty boxes) — a dealbreaker for
one of Obscura's core use cases, scraping CJK-language websites. The fix
intentionally does **not** reintroduce host fonts:

- **No font system scanning.** Walking fontconfig or `~/.fonts` would make
  layout depend on the host, break cross-machine determinism, and add startup
  cost. The CJK faces ship inside the binary instead.
- **No shaping changes.** cosmic-text's `Shaping::Advanced` already falls back
  per glyph across every face in its font database. The CJK faces simply join
  that database: code points the bundled Latin faces cover keep using
  Liberation/DejaVu, and only missing glyphs fall through to the CJK faces.
  Latin layout is unchanged, byte for byte.
- **Performance stays protected.** The feature is off by default, so a
  `render`-only binary stays ~30 MB smaller. And the SVG rasterizer's extended
  font database (which also carries the CJK faces) is built lazily, only for
  pages that actually contain inline SVG text, so ordinary pages pay nothing
  for the 32 MB of font data.

**What you get**

- Chinese (Simplified and Traditional), Japanese kanji and kana, and full-width
  punctuation render with real glyphs — no host fonts and no page webfonts
  required.
- Mixed CJK/Latin text gets real layout metrics: advance widths and line
  height come from the actual glyphs, not from tofu boxes.
- Everything downstream picks it up with no extra configuration: `obscura
  fetch ... --screenshot`, CDP screenshots and screencasts, PDF export, and
  the MCP `browser_screenshot` / `browser_pdf` tools.

**Limitations and the escape hatch**

- Only the Regular weight is embedded: bold CJK renders at normal weight (no
  faux bolding).
- Hangul is not covered by the embedded faces.
- For any font the bundled set lacks, point Obscura at a directory of font
  files. The scan is non-recursive, filename-sorted, and accepts
  `ttf`/`otf`/`ttc`/`woff`/`woff2`; unparseable files are skipped with a
  warning. This is opt-in: layout then depends on the directory's contents,
  which is a deliberate escape hatch from bundled-faces-only determinism.

```bash
# Flag, before or after the subcommand
obscura --fonts /path/to/fonts fetch https://example.com -s page.png

# Or via the environment (applies to fetch, serve, scrape, and mcp)
OBSCURA_FONTS_DIR=/path/to/fonts obscura fetch https://example.com -s page.png
```

### Start the CDP server

```bash
obscura serve --port 9222

# With stealth mode (anti-detection + tracker blocking)
obscura serve --port 9222 --stealth
```

### Scrape in parallel

```bash
obscura scrape url1 url2 url3 ... \
  --concurrency 25 \
  --eval "document.querySelector('h1').textContent" \
  --format json

# Suppress scrape progress on stderr for script-friendly output
obscura scrape https://example.com --quiet --format json

# Scrape workers inherit the global proxy
obscura --proxy http://127.0.0.1:8080 scrape https://example.com https://news.ycombinator.com
```

## Puppeteer / Playwright

### Puppeteer

```bash
npm install puppeteer-core
```

```javascript
import puppeteer from 'puppeteer-core';

const browser = await puppeteer.connect({
  browserWSEndpoint: 'ws://127.0.0.1:9222/devtools/browser',
});

const page = await browser.newPage();
await page.goto('https://news.ycombinator.com');

const stories = await page.evaluate(() =>
  Array.from(document.querySelectorAll('.titleline > a'))
    .map(a => ({ title: a.textContent, url: a.href }))
);
console.log(stories);

await browser.disconnect();
```

### Playwright

```bash
npm install playwright-core
```

```javascript
import { chromium } from 'playwright-core';

const browser = await chromium.connectOverCDP({
  endpointURL: 'ws://127.0.0.1:9222',
});

const page = await browser.newContext().then(ctx => ctx.newPage());
await page.goto('https://en.wikipedia.org/wiki/Web_scraping');
console.log(await page.title());

await browser.close();
```

### Form submission & login

```javascript
await page.goto('https://quotes.toscrape.com/login');
await page.evaluate(() => {
  document.querySelector('#username').value = 'admin';
  document.querySelector('#password').value = 'admin';
  document.querySelector('form').submit();
});
// Obscura handles the POST, follows the 302 redirect, maintains cookies
```

## Benchmarks

Page load:

| Page | Obscura | Chrome |
|------|---------|--------|
| Static HTML | **51 ms** | ~500 ms |
| JS + XHR + fetch | **84 ms** | ~800 ms |
| Dynamic scripts | **78 ms** | ~700 ms |

The full benchmark suite (WPT conformance, obstacle course, real-world corpus, and vs-Chrome speed) lives in the upstream repo: https://github.com/h4ckf0r0day/obscura-benchmark

## Stealth Mode

Build with `--features render,stealth`, then enable stealth at runtime with the
global `--stealth` flag. The stealth build includes the complete rendering
engine; enabling stealth does not remove screenshot, screencast, PDF, CDP, or
MCP functionality.

### Anti-fingerprinting
- Per-session fingerprint randomization (GPU, screen, canvas, audio, battery)
- Realistic `navigator.userAgentData` (Chrome 145, high-entropy values)
- `event.isTrusted = true` for dispatched events
- Hidden internal properties (`Object.keys(window)` safe)
- Native function masking (`Function.prototype.toString()` → `[native code]`)
- `navigator.webdriver = undefined` (matches real Chrome)

### Tracker Blocking
- 3,520 domains blocked
- Blocks analytics, ads, telemetry, and fingerprinting scripts
- Prevents trackers from loading entirely
- Enabled automatically with `--stealth`

## CDP API

Obscura implements the Chrome DevTools Protocol for Puppeteer/Playwright compatibility.

| Domain | Methods |
|--------|---------|
| **Target** | createTarget, closeTarget, attachToTarget, createBrowserContext, disposeBrowserContext |
| **Page** | navigate, getFrameTree, lifecycleEvents, captureScreenshot, start/stopScreencast, printToPDF |
| **Runtime** | evaluate, callFunctionOn, getProperties, addBinding |
| **DOM** | getDocument, querySelector, querySelectorAll, getOuterHTML, resolveNode |
| **Network** | enable, setCookies, getCookies, setExtraHTTPHeaders, setUserAgentOverride |
| **Fetch** | enable, continueRequest, fulfillRequest, failRequest (live interception), takeResponseBodyAsStream |
| **IO** | read, close (stream a large response body in chunks) |
| **Storage** | getCookies, setCookies, deleteCookies |
| **Input** | dispatchMouseEvent, dispatchKeyEvent |
| **LP** | getMarkdown (DOM-to-Markdown conversion) |

To download a large resource without one giant `Network.getResponseBody` blob, call `Fetch.takeResponseBodyAsStream` then read it in chunks with `IO.read` / `IO.close`. Response bodies over the cache limit (`OBSCURA_NETWORK_BODY_BUFFER_BYTES`, default 2 MiB) are not retained, so raise that limit when you intend to stream large downloads.
## CLI Reference

### Tuning V8

Obscura embeds V8 directly. Use `--v8-flags` to pass raw flags through to V8, same syntax as Chromium's `--js-flags` and Node's command-line flags. Most common use is raising the heap cap to fix `JavaScript heap out of memory` on JS-heavy pages:

```bash
obscura --v8-flags "--max-old-space-size=4096" fetch <url>
```

### Heavy SPAs (script execution budget)

Obscura caps the page's script-execution phase so one slow or hung page cannot stall a worker. The default budget is 30s; pages that finish sooner return immediately, so the cap only affects pages that keep running. A very heavy React/Vue/Angular SPA on a slow network can need more time to boot before it fires its data requests. Raise the budget with `OBSCURA_SCRIPT_DEADLINE_MS` (milliseconds), and pair it with a matching navigation timeout in your CDP client:

```bash
OBSCURA_SCRIPT_DEADLINE_MS=60000 obscura serve --port 9222
```

Modules that enhance an already-rendered page have a separate 3s per-module budget so one non-essential module cannot hold navigation open. Raise it for legitimate long-running modules such as a Vite HMR client:

```bash
OBSCURA_MODULE_BUDGET_MS=10000 obscura serve --port 9222
```

An unmounted SPA shell already gives its app modules the full `OBSCURA_SCRIPT_DEADLINE_MS` budget. `OBSCURA_FETCH_TIMEOUT_MS` controls the module's network request, not its evaluation time. See [Environment variables](docs/Environment-variables.md) for the complete timeout model.

### `obscura serve`

Start a CDP WebSocket server.

| Flag | Default | Description |
|------|---------|-------------|
| `--port` | `9222` | WebSocket port |
| `--proxy` | — | HTTP/SOCKS5 proxy URL |
| `--stealth` | off | Enable anti-detection + tracker blocking |
| `--workers` | `1` | Number of parallel worker processes |
| `--obey-robots` | off | Respect robots.txt |

### `obscura fetch <URL>`

Fetch and render a single page.

| Flag | Default | Description |
|------|---------|-------------|
| `--dump` | `html` | Output: `html`, `text`, `links`, `markdown`, `assets` (NDJSON of every sub-resource URL the page references), or `original` (raw response body) |
| `--eval` | — | JavaScript expression to evaluate |
| `--wait-until` | `load` | Wait: `load`, `domcontentloaded`, `networkidle0` |
| `--timeout` | `30` | Maximum navigation time in seconds |
| `--wait` | adaptive, up to `5` | Post-load settling; an explicit value is a fixed delay in seconds |
| `--selector` | — | Wait for CSS selector |
| `-s`, `--screenshot` | — | Write a PNG screenshot (single URL; render-enabled build) |
| `--stealth` | off | Anti-detection mode |
| `--output` | — | Write dump or eval output to a file |
| `--quiet` | off | Suppress banner |
| `--proxy` | — | Inherited global HTTP/SOCKS5 proxy URL |
| `--fonts` | — | Inherited global directory of extra fallback fonts (see CJK and custom fonts) |

### `obscura scrape <URL...>`

Scrape multiple URLs in parallel with worker processes.

| Flag | Default | Description |
|------|---------|-------------|
| `--concurrency` | `10` | Parallel workers |
| `--eval` | — | JS expression per page |
| `--format` | `json` | Output: `json` or `text` |
| `--quiet` | off | Suppress scrape progress on stderr |
| `--proxy` | — | Inherited global HTTP/SOCKS5 proxy URL for all workers |
| `--fonts` | — | Inherited global font directory, forwarded to every worker |

## MCP (Model Context Protocol)

Obscura ships an MCP server that exposes browser automation tools to AI agents (Claude Desktop, Cursor, etc.).

### Start

**stdio** (default) — for Claude Desktop and MCP clients that launch a subprocess:

```bash
obscura mcp
```

**HTTP** — for clients that connect over the network:

```bash
obscura mcp --http --port 8080
# endpoint: http://127.0.0.1:8080/mcp
```

Optional flags (both transports):

| Flag | Description |
|------|-------------|
| `--proxy <URL>` | HTTP/SOCKS5 proxy |
| `--user-agent <UA>` | Custom User-Agent string |
| `--stealth` | Enable anti-detection mode |

### Claude Desktop config

```json
{
  "mcpServers": {
    "obscura": {
      "command": "obscura",
      "args": ["mcp"]
    }
  }
}
```

### Tools

| Tool | Description |
|------|-------------|
| `browser_navigate` | Navigate to a URL (`url`, optional `waitUntil`: `load` / `domcontentloaded` / `networkidle0`) |
| `browser_snapshot` | Return the current page URL, title, readable body text, and element references |
| `browser_screenshot` | Return the current page as an MCP PNG image (render-enabled build) |
| `browser_pdf` | Return the current page as an embedded PDF resource (render-enabled build) |
| `browser_click` | Click by current snapshot reference or CSS selector |
| `browser_fill` | Set an input value by reference or selector (triggers `input` + `change`) |
| `browser_type` | Append text to an input |
| `browser_press_key` | Dispatch a keyboard event (`key`, optional `selector`) |
| `browser_select_option` | Select an `<option>` by value or text |
| `browser_evaluate` | Evaluate a JavaScript expression and return the result |
| `browser_wait_for` | Wait for a CSS selector to appear (`selector`, optional `timeout` in seconds) |
| `browser_network_requests` | List network requests made by the current page |
| `browser_console_messages` | Return console messages logged by the page |
| `browser_close` | Close the page and reset browser state |

The MCP server exposes still-image and PDF output. Use CDP when you need the
streaming `Page.startScreencast` protocol.

## Integrations

- **[Hermes agent plugin](https://github.com/SGavrl/hermes-plugin-obscura)**: run [Hermes](https://github.com/NousResearch/hermes-agent) agent browser tasks on Obscura. The plugin spawns `obscura serve` per session (or connects to an already running server) and drives it over CDP, with optional `--stealth`.

## License

Apache 2.0. Fork of [h4ckf0r0day/obscura](https://github.com/h4ckf0r0day/obscura)
(Apache-2.0); the embedded CJK fonts are Noto Sans CJK SC/TC, released under
SIL OFL 1.1 (see
[crates/obscura-render/assets/LICENSE-OFL.txt](crates/obscura-render/assets/LICENSE-OFL.txt)).

---
