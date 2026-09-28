# MCP `browser_console_messages` 接線

待辦追蹤見 `TODO.md`「待辦」節。2026-09-16 問題回報：頁面內
`console.log` 後（含 load 期間的 JS 例外）`browser_console_messages`
永遠回傳 "No console messages."，導致「圖層不渲染且無任何錯誤輸出」
的症狀（見 `design/svg-dom-api.md`）無法快速定位。

## 根因（唯讀分析結論）

不是捕鉤掛載時機問題（`console` 與 `window.onerror` 都在 bootstrap
生效、load 前已掛上），而是 MCP 端完全沒接線，三層全斷：

1. `obscura-mcp/src/lib.rs` 的 `BrowserState.console_messages`
   （L73）是死欄位：只初始化（L90）與 clear（L1197），全 repo 無
   任何寫入點；`tool_console_messages`（L1180）只讀空 vec。
2. 實際 console 路徑：`globalThis.console`（`bootstrap.js` L822）→
   `op_console_msg`（`obscura-js/src/ops.rs:2136`）。該 op 永遠打
   tracing（target `obscura::console`），但排入
   `page.pending_runtime_events` 需 `runtime_events_enabled ==
   true`；此 flag 只被 CDP `Runtime.enable` 翻開（
   `obscura-cdp/src/dispatch.rs:326`）。MCP 直接 `Page::new`
   （lib.rs L103/114）從不呼叫 `set_runtime_events_enabled`（預設
   false，`obscura-browser/src/page.rs:1169`）。bootstrap 側的
   `eventArgs` 序列化也隨 flag 跳過（`bootstrap.js` L812）。
3. 就算事件入隊，MCP 也從不 drain：`take_pending_runtime_events()`
   （`page.rs:4433`）存在但無人使用。

未捕獲例外同理：`record_uncaught_exception`
（`obscura-js/src/runtime.rs:1195`）被同一 flag 閘住。load 期例外
目前只去兩個地方：`__obscura_errors` 陣列（`bootstrap.js` L79-85，
需手動 evaluate 讀）與 server tracing warn（`runtime.rs:3046`，
"page task error, continuing the event loop"）。

對照：CDP 模式（Playwright / Puppeteer）的 console 捕獲是正常的
（`obscura-cdp/tests/runtime_console_events.rs` 覆蓋），只有 MCP
工具壞。

## 修法

1. MCP 建頁時（lib.rs 的 tab 建立點）`page.set_runtime_events_enabled(true)`
   （`page.rs:4441` 已存在，每次 re-prepare 會同步到 runtime，
   `page.rs:1868`）。
2. `tool_console_messages` drain `take_pending_runtime_events()`，
   把 `RuntimeEvent::Console`（kind + args + timestamp）與
   `RuntimeEvent::Exception`（name + description + url + line/col +
   stack）格式化進 `console_messages`（格式定案：每行
   `[level] text` 或 `[exception] Name: desc @ url:line:col`，含
   時間戳可選）。
3. 如此 load 期 console 與未捕獲例外自動進來，原 issue 建議的
   `window.onerror` / `unhandledrejection` 補捕不必做：deno_core
   的 uncaught-exception 路徑已覆蓋（unhandled rejection 現狀為
   bootstrap 內 `preventDefault` 吞掉，若要捕獲可另行評估接
   `record_uncaught_exception` 同類事件）。

## 注意事項

- 開銷：開啟 flag 後每個 `console.log` 多做一次 `eventArgs` JSON
  序列化 + `op_console_msg` 多做 borrows；chatty 頁面上線前量測
  （performance 是硬約束，noise floor ±10%）。
- 事件隊列上限 1024、溢出彈最舊（`ops.rs:2151`）：工具描述文案需
  標明 ring-buffer 語意，避免使用者以為能拿歷史全量。
- drain 時機定案為 `browser_console_messages` 呼叫時（MCP 是
  stateful 長生命週期，避免事件在其他工具呼叫間被無謂清空）。
- 測試：`obscura-cli/tests/mcp_client.rs` 已有 tools/list 覆蓋
  `browser_console_messages`，補「navigate 到含 console.log + throw
  的 fixture → `browser_console_messages` 含兩者」的端對端 case。
