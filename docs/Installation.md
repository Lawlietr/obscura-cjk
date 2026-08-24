## Linux x86_64

```bash
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-x86_64-linux.tar.gz
tar xzf obscura-x86_64-linux.tar.gz
./obscura --version
```

## Linux ARM64

```bash
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-aarch64-linux.tar.gz
tar xzf obscura-aarch64-linux.tar.gz
./obscura --version
```

Linux builds target Ubuntu 22.04 and require glibc 2.35+.

## macOS Apple Silicon

```bash
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-aarch64-macos.tar.gz
tar xzf obscura-aarch64-macos.tar.gz
./obscura --version
```

## macOS Intel

```bash
curl -LO https://github.com/Lawlietr/obscura-cjk/releases/latest/download/obscura-x86_64-macos.tar.gz
tar xzf obscura-x86_64-macos.tar.gz
./obscura --version
```

## Windows

Download the `.zip` from [Releases](https://github.com/Lawlietr/obscura-cjk/releases), extract, run `obscura.exe --version`.

## Docker

Pull the published image (linux/amd64, linux/arm64; pushed by GitHub Actions
on every `v*` tag; `latest` tracks the newest release):

```bash
docker run -d --name obscura -p 127.0.0.1:9222:9222 ghcr.io/lawlietr/obscura-cjk:latest

# Pin a specific release for reproducibility and easy rollback
docker run -d --name obscura -p 127.0.0.1:9222:9222 ghcr.io/lawlietr/obscura-cjk:0.1.0-cjk
```

The repo's `docker-compose.yaml` deploys this image. For local development,
build from this repo's `Dockerfile`:

```bash
docker build -t obscura-cjk .
docker run -d --name obscura -p 127.0.0.1:9222:9222 obscura-cjk
```

Multi-stage build on a `rust:1-slim-bookworm` builder stage; the runtime layer
is `debian:12-slim` with CA certificates taken from the distroless base image.
The `cjk` feature is on, so CJK text renders out of the box
(see [CJK and custom fonts](CJK-and-custom-fonts.md)).

The rendering release archives (`-cjk`, `-stealth`, and no suffix) and the
Docker image include the rendering engine; the `-no-render` variants omit it.
Source builders must pass `--features render`; see
[Build from source](Build-from-source.md).

## From source

See [Build from source](Build-from-source.md).

## What's in the archive

- `obscura`: CLI and CDP server.
- `obscura-worker`: helper for the parallel `scrape` command. Keep both in the same directory.

Archive suffixes identify the feature set: no suffix includes rendering,
`-cjk` adds embedded CJK fallback fonts on top of rendering, `-stealth`
includes rendering and stealth, `-no-render` includes neither, and
`-no-render-stealth` includes stealth without rendering.

## Smoke test

```bash
./obscura fetch https://example.com --eval "document.title"
./obscura fetch https://example.com --screenshot smoke.png
```

Expected output: `"Example Domain"`, followed by a nonempty PNG at `smoke.png`.

## Troubleshooting

`cannot execute binary file`: wrong arch. Check `uname -m`.

`GLIBC_2.35 not found`: distro is older than Ubuntu 22.04. Use Docker or build from source.

macOS Gatekeeper warning: `xattr -d com.apple.quarantine ./obscura`.
