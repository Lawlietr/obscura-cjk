# Dependabot 依賴更新策略

待辦追蹤見 `TODO.md`。

## 背景

現況：`deny.toml` + `ci.yml` 的 `cargo-deny-action` 已有被動式漏洞/授權/
bans 檢查（有 5 個明確 ignore 的 RUSTSEC），但「只擋不升」：transitive
crate 出新版本時沒有任何機制自動提 PR。加官方 Dependabot 補主動式
更新，與 cargo-deny 互補不衝突。公開倉庫免費。

## `.github/dependabot.yml` 設計（2026-08-24 已寫入，push 後生效）

- 兩個 cargo entry（同 directory，週一 schedule）：
  - 例行 version updates：`lockfile: true`（只改 Cargo.lock、不動
    Cargo.toml），全部併入 `routine` group（一個週報 PR，CI 每週約
    一次 V8 全編）。
  - `exclude-patterns` 排除 `v8` / `deno-core` / `deno-*`（ABI 敏感，
    升版留人工，需完整回歸門）；`lockfile: true` 下 major bump 也只
    重寫 lockfile，故不設 update-type 過濾、直接併入同一 group。
  - Security entry 另開：`security-advisories: enabled`（V8/deno
    家族的 RUSTSEC 也會進 PR），併入 `security` group，
    `open-pull-requests-limit: 5`。
- 另加 github-actions ecosystem（低頻、CI 便宜）。

## 後續維護要點

- 觀察首批 PR 時注意 `[patch.crates-io]` 的 vendored `taffy` /
  `cosmic-text` 行為：本體不會被更新，屬正常 warning；其 transitive
  依賴仍會進 lockfile 更新。
- 升級後視需要同步清理 `deny.toml` 的 ignore 清單（RUSTSEC ID 綁定
  transitive 版本，cargo-deny CI 會提示）。
