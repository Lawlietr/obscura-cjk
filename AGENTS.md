# AGENTS.md

Guidance for AI coding agents and contributors working in the Obscura repo.
This is the non-obvious stuff you can't infer from the code; read it before
building, testing, or changing anything.

Obscura is a headless browser engine in Rust. It runs real JavaScript through
V8 (`deno_core`), keeps a real DOM tree, owns its layout and paint pipeline,
speaks the Chrome DevTools Protocol, and is a drop-in replacement for headless
Chrome with Puppeteer and Playwright. Rendering and stealth are both first-class
capabilities. It targets web scraping and AI-agent automation.

## Docker Deployment

Release images are published to `ghcr.io/lawlietr/obscura-cjk` by GitHub
Actions on every `v*` tag; `latest` tracks the newest release, and versioned
tags remain available for rollback. The repo's `docker-compose.yaml` follows
`latest` and is
the canonical deployment (MCP mode with the recommended security hardening);
`docker compose up -d` from the repo root is the default way to run it.
For local development, build from the repo's `Dockerfile`
(`docker build -t obscura-cjk .`). Standalone examples for reference:

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

### CDP Mode

For Puppeteer/Playwright integration, run CDP server mode:

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

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `OBSCURA_ALLOW_PRIVATE_NETWORK` | Allow SSRF to loopback/RFC1918 | `0` |
| `OBSCURA_PROXY` | Proxy URL for HTTP requests | empty |
| `OBSCURA_FONTS_DIR` | Optional dir of extra fallback fonts (see font-directory note) | empty |

### Notes

- **CJK** is built into the Docker image (`--features render,cjk`): Chinese and
  Japanese text render without host fonts. **Stealth mode** is not included; use
  a source build with `--features stealth` for fingerprint protections.
- To add fonts the embedded set lacks (Korean Hangul, CJK weights), mount a
  directory and set `OBSCURA_FONTS_DIR` (or pass `--fonts <PATH>`); the scan is
  non-recursive and covers `ttf`/`otf`/`ttc`/`woff`/`woff2`.
- The default CMD binds to `0.0.0.0` inside the container for Docker port mapping.
- Native binary defaults to `127.0.0.1` (loopback only).

## Build

```bash
CARGO_INCREMENTAL=0 CARGO_BUILD_JOBS=2 cargo build --release -p obscura-cli --bins --features render,cjk,stealth

# Rendering with embedded CJK fallback faces (recommended default; see below)
CARGO_INCREMENTAL=0 CARGO_BUILD_JOBS=2 cargo build --release -p obscura-cli --bins --features render,cjk

# Rendering and stealth
CARGO_INCREMENTAL=0 CARGO_BUILD_JOBS=2 cargo build --release -p obscura-cli --bins --features render,stealth
```

- The first build compiles V8 from source: ~5 minutes and a few GB of disk.
  Incremental builds are seconds.
- **CJK:** `--features render,cjk` embeds Noto Sans CJK SC/TC Regular as
  glyph-level fallback faces (SIL OFL) so Chinese/Japanese text shapes without
  any host fonts or page webfonts. It is off by default to keep the base
  binary ~30 MB smaller; the Dockerfile and the recommended builds enable it.
  Only Regular weight is embedded (bold CJK is not synthesized) and Hangul is
  not covered; see the font-directory escape hatch below.
- **Font directory (opt-in):** `OBSCURA_FONTS_DIR` (env) or the global CLI
  flag `--fonts <PATH>` (valid before or after the subcommand, forwarded to
  scrape workers) adds a directory of extra font files as fallback faces after
  the bundled set. The scan is non-recursive, filename-sorted, and accepts
  `ttf`/`otf`/`ttc`/`woff`/`woff2`; unparseable files are skipped with a
  stderr warning. This is a deliberate escape hatch from the bundled-faces-only
  determinism: layout then varies with the directory contents. Use it for
  scripts the embedded faces lack (Korean Hangul, CJK weights, etc.).
- **Iterating on one crate? Scope it:** `cargo build -p obscura-cli`. A bare
  `cargo build` can re-link the whole workspace; the V8 compile is the cost, so
  avoid touching it when you don't need to.
- **Stealth:** `--features render,stealth` retains the complete rendering
  surface and adds the wreq/BoringSSL transport, fingerprint protections, and
  tracker blocklist. BoringSSL builds through CMake, so `cmake` must be
  installed. The rendering build uses rustls and needs neither CMake nor OpenSSL.
- If the vendored OpenSSL build hits an AVX-512 assembler error on your host,
  build with `OPENSSL_NO_VENDOR=1`.

## Test

Run tests with **`cargo nextest`, not `cargo test`**:

```bash
cargo nextest run --release --features render,cjk -p <crate>
cargo nextest run --release --features render,cjk --no-fail-fast
```

`cargo test` runs the whole test binary in one process, but the engine holds a
single V8 isolate per process, so the runtime tests fail under it. `nextest`
runs each test in its own process, which is the only supported way.

The authoritative behavioral gate is the **obstacle course** in the companion
repo `obscura-benchmark` (33 capability + speed stages; 32/33 pass, see known
issue below):

```bash
OBSCURA_BIN=./target/release/obscura python3 obstacle-course/run.py --runs 1 --warmup 0
```

It serves local fixtures, so it is deterministic and offline. WPT conformance
and the real-world render corpus also live in that repo; report WPT as subtest
pass %, not whole-file pass.

Dependabot runs weekly (see `.github/dependabot.yml`). Routine cargo PRs are
lockfile-only by design (`lockfile: true`): they rewrite `Cargo.lock` but must
never touch `Cargo.toml`. The V8/deno family (`v8`, `deno-core`, `deno-*`)
is excluded from routine updates on purpose, since upgrading it changes the
V8 ABI and requires a manual review plus the full regression gate; only
security advisories may produce PRs against those crates.

## Before you finish

For any code change:

1. Run focused release-mode nextest coverage for the crates and repro involved.
2. Run `cargo nextest run --release --features render,cjk --no-fail-fast`.
3. Run the exact release build shown above (with `render,cjk` when the change
   touches fonts or the render layer).
4. The obstacle course still reports **32/33** (`observer-intersection` is a
   known issue: headless mode does not scroll, so IntersectionObserver
   callbacks fire only once when the sentinel stays below viewport).
5. For render changes, run deterministic fixtures and broad top/bottom real-site
   captures using the methodology below.
6. For stealth changes, re-test with `--stealth` (a non-stealth binary won't
   exercise the `wreq` path).

Do not bulk-run `cargo fmt`: the tree is not rustfmt-clean, so a blanket format
produces a huge unrelated diff. Match the surrounding style in the files you
edit instead.

## Architecture

- **obscura-cli** — CLI: `fetch` (`--dump assets|html|text|links|markdown|original|cookies`, `--eval <JS>`, `--screenshot <PNG>`), `serve` (CDP server), `scrape`, `mcp`. `--proxy`, `--stealth`, `--allow-private-network`, and `--fonts <PATH>` are global flags: valid before or after the subcommand and applied to `fetch`, `serve`, `scrape`, and `mcp` (a `scrape` run forwards `--stealth` to each worker via `OBSCURA_STEALTH` and `--fonts` via `OBSCURA_FONTS_DIR`).
- **obscura-cdp** — Chrome DevTools Protocol server (WebSocket). Managed page
  sessions use `"{targetId}-session"`; explicit flattened attachments receive
  distinct session ids so Playwright and Puppeteer can open raw page sessions.
- **obscura-js** — V8/`deno_core` runtime. `js/bootstrap.js` is the DOM/browser shim; `src/ops.rs` bridges JS to Rust DOM ops; `src/runtime.rs` owns the isolate and the per-page `ObscuraState`.
- **obscura-dom** — DOM tree (`src/tree.rs`).
- **obscura-net** — HTTP client (`client.rs`), stealth client (`wreq_client.rs`), cookie jar, robots cache, tracker blocklist.
- **obscura-browser** — the `Page` type, navigation, JS evaluation.
- **obscura-render** — selector cascade, computed style, retained layout,
  scrolling, text shaping, images/SVG/canvas, and CPU-backed paint. The
  `render` feature powers geometry, screenshots, CDP screencasting, and PDF.
- **obscura-mcp** — stateful MCP automation tools. Render builds expose
  `browser_screenshot` and `browser_pdf`; streaming screencasts remain CDP-only.
- **obscura** — embeddable Rust library API (git dependency; builds V8 locally, not on crates.io). Public request-interception API on `Page`: `add_preload_script`, `enable_interception` (channel of `InterceptedRequest`, resolved with `InterceptResolution::{Continue, Fulfill, Fail}`), and passive `on_request` / `on_response`. `op_fetch_url` invokes these for JS `fetch()`/XHR, so when touching it keep a `Continue` URL rewrite behind `validate_fetch_url` (the SSRF gate, same as redirects).

## Conventions

- **Do not run verification automatically.** Builds, `nextest` runs, render
  captures, and obstacle-course runs only happen when the user asks. "Before
  you finish" below lists what to run *when asked*, not what to run by default.
- **This repo is an independent fork (`Lawlietr/obscura-cjk`), not the
  upstream.** It has diverged from `h4ckf0r0day/obscura`. All operational
  references -- docs, install URLs, releases, Docker image (published to
  ghcr.io/lawlietr/obscura-cjk on `v*` tags), CI, issue/security templates --
  point at
  this repo, never the upstream. Upstream is mentioned only where required or
  factual: Apache-2.0 fork attribution in README/License, the
  `obscura-benchmark` suite (which only exists upstream), and historical
  citations of upstream PRs.
- **Performance is a hard constraint** (Obscura is ~12x faster and uses ~6x less
  memory than headless Chrome on framework pages). Keep native Rust fast paths;
  add a JS fallback only for real spec edge cases. Benchmark old and new
  revisions interleaved with the same release build, page, network, viewport,
  settle policy, and capture path. Report distributions and resource use; the
  noise floor is about plus or minus 10%.
- **Keep ops panic-safe.** `op_dom` is wrapped in `catch_unwind` so a DOM-op
  panic returns null instead of aborting the process inside V8's FFI frame. New
  ops must not unwind into V8.
- **Commits/PRs/comments:** short and factual, no em dashes, no AI filler.

## Rendering verification

Use deterministic fixtures before real sites. Put generated output in a
disposable directory outside the repository:

```bash
RUN_ROOT="$(mktemp -d)"
OBSCURA_BIN=./target/release/obscura render-repros/run.sh "$RUN_ROOT/fixtures"
OBSCURA_BIN=./target/release/obscura render-repros/representative-suite/run.sh "$RUN_ROOT/top"
OBSCURA_BIN=./target/release/obscura render-repros/representative-suite/run.sh "$RUN_ROOT/bottom" bottom
```

The harness accepts `BASELINE_BIN` or `CHROMIUM_BIN` for paired output. A
latency-only run may use `SUITE_MODE=latency SETTLE_MS=0`, but zero settle is
not valid fidelity evidence.

Compare the same viewport, device scale, identity, network inputs, settle
policy, scroll position, animation time, and capture boundary. First confirm
both navigations succeeded and both images are nonblank. Then inspect missing
resources, geometry, text flow, structural edges, clipping, fixed/sticky
behavior, and a reduced fixture. Pixel-distance metrics are useful regression
tripwires, not standalone correctness verdicts. Never add hostname-specific
layout, style, or resource behavior.

`render-repros/cjk/` is a subdirectory on purpose: `run.sh` only globs top-level
`*.html`, so the CJK fixture does not enter the paired corpus on hosts without
the `cjk` feature. Run it directly, e.g.
`OBSCURA_BIN=./target/release/obscura fetch file://$PWD/render-repros/cjk/cjk-fallback.html --screenshot "$RUN_ROOT/cjk.png"`
(on a `render,cjk` build).

`render-repros/**` is the tracked public evidence harness. Git-ignored internal
handover notes are private working material: do not edit them, link them from
public documentation, stage them, or commit them. Do not commit generated
screenshots or reports.

## Known issues

- **`observer-intersection` obstacle course stage fails (32/33 pass).**
  Expected `'io:50'`, got `''`. Root cause: Obscura headless mode does not
  scroll, so the IntersectionObserver callback on the sentinel element fires
  only once (when the page loads). The sentinel remains below the viewport
  and never triggers the expected scroll-based callback. Fixture comments
  claim targets are treated as intersecting, but they are not. This is an
  inherent headless-mode limitation.

## Gotchas

- **DOM mutation arg order:** `insertBefore` / `replaceChild` in `bootstrap.js`
  pass reference-node vs parent nid in a way that's easy to break. If you touch
  mutation methods, verify `before()`, `after()`, `replaceWith()`, and
  `replaceChild()` on connected elements.
- **Multi-statement `--eval` starting with `const` returns `null`** (V8 gives
  `const` an empty completion value). Wrap snippets in an IIFE:
  `(function(){ ...; return result; })()`.
- **Keep the SVG fallback faces out of the base font database.** `svg_font_database()`
  is called on every `prepare`; with feature `cjk` the fallback faces add 32MB of
  OTF that parse for ~30ms, and paying that on the first prepare blocks the V8
  event loop, delaying early paint and IntersectionObserver callbacks enough to
  break the IO timing tests. `svg_font_database_with_fallbacks()` (separate
  OnceLock) is only built when a page actually contains inline SVG text.
  Verify with the `svg` nextest filter after touching this area.
- **`canAccessOpener` must be in every `TargetInfo` payload**, or strict CDP
  clients (chromiumoxide) panic.
- **The DOM reparenting guards in `tree.rs` are load-bearing.** `append_child` /
  `insert_before` reject cycles (inserting an ancestor of the target is a no-op).
  A cyclic reparent used to make `descendants()` loop forever and hang the
  engine on real sites, uninterruptible by tokio or the watchdog. Keep the
  guards and the `descendants()` length cap.
- **SSRF:** loopback / RFC1918 / link-local fetches are blocked by default. Use
  `--allow-private-network` (or `OBSCURA_ALLOW_PRIVATE_NETWORK=1`) for local
  testing.

## Robustness invariants (do not remove)

One page must never hang or crash a worker:

- The V8 termination watchdog (`arm_watchdog` / `disarm_watchdog` in
  `runtime.rs`) terminates the isolate from a separate thread, because
  `tokio::time::timeout` only cancels at await points and synchronous V8 work
  runs unbounded.
- The CLI applies a process-level hard deadline as an absolute backstop.
- `panic = "unwind"` is pinned in the release profile so `catch_unwind` works.

## Stealth

The stealth features (the `wreq` client, fingerprint and browser-identity
adjustments) are privacy-first anti-fingerprinting: they present a normal,
consistent browser fingerprint (user agent, timezone, navigator properties, and
similar surfaces) so ordinary automation traffic is not singled out. They
contain no bot or automation-abuse payload.

## Recommended proxy provider: NodeMaven

For production scraping you usually want residential or ISP IPs rather than a
datacenter address, since many sites block datacenter ranges. We use
**NodeMaven** and it works well with Obscura: residential/ISP proxies, sticky
sessions, and per-request country/region targeting. A good default if you don't
already have a provider.

Route Obscura through it with the global `--proxy` flag (use your own
credentials):

```bash
# HTTP
obscura --proxy http://USER:PASS@gate.nodemaven.com:8080 fetch https://example.com --dump text

# SOCKS5
obscura --proxy socks5://USER:PASS@gate.nodemaven.com:1080 scrape url1 url2 --concurrency 25
```

The targeting options (country, region, session id) live in the proxy username,
so a sticky session keeps the same exit IP across requests.

Sign up: https://go.nodemaven.com/obscuraagentaugust

Obscura discount codes: `OBSCURA35` (35% off mobile and residential),
`OBSCURA40` (40% off ISP / static).
