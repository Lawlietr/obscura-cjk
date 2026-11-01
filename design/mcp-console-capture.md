# MCP `browser_console_messages`

> 狀態：已實作。本文件是設計記錄，非待辦。

頁面內 `console.log`（含 load 期間的 JS 例外）會经由 console 捕獲路徑
進入 page 的 pending 佇列，`browser_console_messages` 工具讀取它。

## 實作位置

- `crates/obscura-mcp/src/lib.rs`：`BrowserState` 在兩個建頁點呼叫
  `page.set_console_messages_enabled(true)`；`tool_console_messages()`
  drain `page.take_pending_console_messages()` 並合併進
  `state.console_messages`。
- 事件來源：`crates/obscura-js/src/ops.rs` 的
  `pending_console_messages` / `console_messages_enabled`，由
  `op_console_msg` 排入。未捕獲例外走同一開關，所以 load 期例外也會進
  佇列，不需要另加 `window.onerror` 補捕（deno_core 的
  uncaught-exception 路徑已覆蓋；unhandled rejection 現狀由 bootstrap 內
  `preventDefault` 吞掉，要捕獲需另行評估）。
- 對照：CDP 模式（Playwright / Puppeteer）由 `Runtime.enable` 開啟同一
  捕獲，`obscura-cdp/tests/runtime_console_events.rs` 覆蓋。

## 注意事項

- 開銷：開啟開關後每個 `console.log` 多做一次 `eventArgs` JSON 序列化
  加 `op_console_msg` 的 borrows。chatty 頁面上線前需量測（performance
  是硬約束，noise floor 約正負 10%）。
- MCP 端 ring buffer 上限 1024 條，溢出丟最舊（`lib.rs`
  `tool_console_messages`）：工具文案需標明 ring-buffer 語意，避免使用者
  以為能拿全量歷史。
- drain 時機定案為 `browser_console_messages` 呼叫時（MCP 是長生命週期
  的 stateful session，避免事件在其他工具呼叫之間被無謂清空）。
- 回歸：`crates/obscura-mcp/src/lib.rs` 的 nextest 斷言涵蓋「先拿 inline
  console、再拿後續訊息」的 drain 行為；`obscura-cli/tests/mcp_client.rs`
  覆蓋 tools/list 有 `browser_console_messages`。
