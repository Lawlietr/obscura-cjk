# CJK and custom fonts

A capability of the [obscura-cjk fork](https://github.com/Lawlietr/obscura-cjk):
Chinese and Japanese text renders without host fonts or page webfonts, and any
other missing font can be supplied at runtime.

## What it is

- A `cjk` cargo feature (on in the `-cjk` release archives and the Docker
  image).
  When enabled, two faces are embedded in the binary as glyph-level fallbacks:
  **Noto Sans CJK SC** and **Noto Sans CJK TC**, Regular weight, SIL OFL 1.1
  (~16 MB each).
- A global `--fonts <PATH>` flag and the `OBSCURA_FONTS_DIR` environment
  variable load extra font files from a directory at runtime, for scripts the
  embedded set does not cover.

## Why it works this way

Obscura deliberately loads **no system fonts**. That keeps layout bit-for-bit
identical on every machine and startup instant, but it meant Chinese and
Japanese text rendered as tofu (empty boxes) — a dealbreaker for scraping
CJK-language websites. The fix intentionally does **not** reintroduce host
fonts:

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

## What you get

- Chinese (Simplified and Traditional), Japanese kanji and kana, and
  full-width punctuation render with real glyphs — no host fonts and no page
  webfonts required.
- Mixed CJK/Latin text gets real layout metrics: advance widths and line
  height come from the actual glyphs, not from tofu boxes.
- Everything downstream picks it up with no extra configuration: `obscura
  fetch ... --screenshot`, CDP screenshots and screencasts, PDF export, and
  the MCP `browser_screenshot` / `browser_pdf` tools.

## Limitations and the escape hatch

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

## Two font-directory mechanisms

There are **two** separate ways to add fonts from a directory, and they are not
interchangeable:

| | `--fonts` / `OBSCURA_FONTS_DIR` | `--font-dir` |
| --- | --- | --- |
| Origin | fork | upstream |
| Where it applies | Global flag; `fetch`, `serve`, `scrape`, `mcp` | `serve` subcommand only |
| When it is read | Per request / per worker | Once, before the first render of the process |
| Scan | Non-recursive, filename-sorted | Recursive (enters subdirectories), filename-sorted |
| Accepted files | `ttf`/`otf`/`ttc`/`woff`/`woff2` | Pure sfnt only: `ttf`/`otf`/`ttc`/`otc` (woff/woff2 skipped) |
| Symlinks | Included | Skipped |

Both add fallback faces to the font database after the bundled set and are
opt-in: layout then depends on the directory contents, breaking
bundled-faces-only determinism. Use `--fonts`/`OBSCURA_FONTS_DIR` for quick
per-request additions (including `scrape`/`mcp`). Use `--font-dir` when running
`obscura serve` and you need a whole tree of fonts loaded once for the CDP
session; pass it **before** the first page renders, or it refuses with
"Font directories must be configured before the first render".
