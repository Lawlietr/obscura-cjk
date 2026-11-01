# TODO

Obscura CJK fork (`Lawlietr/obscura-cjk`) 待辦追蹤。
本文件只記錄未完成的條目；已完成的工作以 git log 為準。

## 待辦

### 中優先級｜cjk-stealth smoke test

- [ ] 抽驗 `cjk-stealth` 變體 smoke test（#831 動了 wreq，本機無 cmake
      無法本地建置驗證）。v0.2.1-cjk 起就標記，至今未完成。

### 中優先級｜Dependabot

- [ ] Repo Settings 確認 Dependabot GitHub App 權限（public repo 預設
      已啟用，無需額外開關；security alerts 預設開）。
- [ ] 觀察首批 PR：確認 `[patch.crates-io]` 的 vendored `taffy` /
      `cosmic-text` 行為符合預期（本體不會被更新，屬正常 warning；其
      transitive 依賴仍會進 lockfile 更新）。
- [ ] 後續維護：升級後視需要同步清理 `deny.toml` 的 ignore 清單
      （RUSTSEC ID 綁定 transitive 版本，cargo-deny CI 會提示）。
- 細節：[design/dependabot.md](design/dependabot.md)

### 低優先級（可選）｜React hydration 靜默未完成（2026-09-22）

> 下游專案採兩階段策略：Obscura 負責截圖 + 只讀 DOM 探測，點擊類驗證
> 改走真 Chromium（Playwright）。WebSocket 支援非核心需求。
> upstream PR h4ckf0r0day/obscura#1080 截至 2026-09-28 仍 OPEN。
> 細節：[design/hydration-bug-report.md](design/hydration-bug-report.md)

- [ ] （可選）最小重現：純 React 19 單頁（非 Next）在 Obscura 能否 hydrate
- [ ] （可選）驗證 production build 是否同受影響
- [ ] （可選）定位：與 Playwright 對照 `/_next/*` dev 資源 headers 與 ws 生命週期
- [ ] （可選）修復 + 回歸（需實作頁面級 WebSocket；候選來源：upstream #1080）

### 低優先級｜Leaflet 功能補齊（2026-09-16）

> 建議順序 C → D → E → F 決策 → G。F、G 獨立；
> SVG 鏈有依賴 C → D → E。

- [ ] **（C）SVG factory 方法（最小層）**：`createSVGRect` 等，解鎖
      Leaflet 功能偵測。細節：[design/svg-dom-api.md](design/svg-dom-api.md)
- [ ] **（D）SVG shape class + `instanceof` 映射**。依賴 C。
      upstream PR h4ckf0r0day/obscura#1112（「define the missing concrete
      SVG element interfaces」，已 merged）會隨 upstream 同步進來；
      同步後先探測殘留缺口再決定要不要自己補。
- [ ] **（E）SVG animated transform + `getTotalLength` 等**。依賴 D。
      `getTotalLength` 已有 upstream open PR h4ckf0r0day/obscura#1204
      （native Rust geometry）可考慮 cherry-pick；`transform.baseVal`
      仍需自己做。
- [ ] **（F）`getBBox` 處理決策**：快速回 `DOMException` vs 讀真實幾何。
- [ ] **（G）`getCTM` / `getScreenCTM`**：需 layout 資料，預估 2–4 天。

### 低優先級（可選）｜本機環境

- [ ] 可選：`obscura-benchmark` 倉庫 clone 下來跑完整驗證。
- [ ] 磁碟衛生：`target/release/deps` 定期清舊 binary；重 build 前查
      `df -h /`。

## 參考

### 分支 / remote 現況

- `origin` → `Lawlietr/obscura-cjk`（唯一 remote）
- 工作分支：`main`
- 最新 tag：`v0.3.0-cjk`（2026-09-28，deno_core 0.412 基底）
- `upstream` → `h4ckf0r0day/obscura`
- fork main 已 rebase 到 upstream/main（deno_core 0.412），2026-09-28 完成。
  計劃：[design/upstream-rebase-20260929.md](design/upstream-rebase-20260929.md)

### 已完成，僅存設計記錄

- **CSSOM View 盒模型 getter**（`clientLeft` / `clientTop` /
  `clientWidth` / `clientHeight`）已上線：`bootstrap.js` 的 getter 讀
  `op_layout_geometry`（同步 upstream 後改走批次 op
  `op_layout_box_metrics`）。設計記錄：
  [design/cssom-view-box-geometry.md](design/cssom-view-box-geometry.md)
- **MCP `browser_console_messages`** 已接線：`obscura-mcp/src/lib.rs` 的
  `tool_console_messages()` 讀 `take_pending_console_messages()`，建頁時
  `set_console_messages_enabled(true)`，並有 nextest 覆蓋。
  設計記錄：[design/mcp-console-capture.md](design/mcp-console-capture.md)

### 已記錄在 AGENTS.md（無需再跟）

- fork 政策、Docker 部署、CI 慣例
- 不自動跑驗證（build / nextest / obstacle course 只在用戶要求時執行）
- SVG fallback 字型 lazy loading gotcha
- `render-repros/cjk/` 置於子目錄是故意的
