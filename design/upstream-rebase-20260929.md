# Upstream Rebase: re-baseline on upstream/main (deno_core 0.350 → 0.412)

> 本文檔是完整實施計劃，讓接手的 agent 不需任何事前準備即可執行。
> 日期：2026-09-29。狀態：已執行完成（2026-09-29）。
> 實際執行結果：4 個 CJK commit 衝突 + 3 個 fork-refs 衝突全數按计划解決；
> `inline.rs` 的 fallback 注入點因 upstream 重構移入 `base_font_database`
>（比计划预估的更簡潔）。驗證：1885 passed / 5 skipped / 0 failed，
> 障礙課程 32/33。結果在 `rebase-upstream-20260929` 分支，PR #1 待合併。

## 背景

Fork (`Lawlietr/obscura-cjk`) 與 upstream (`h4ckf0r0day/obscura`) 在
2026-09-19 產生 deno_core 版本分歧：

| | Fork main | Upstream main |
|---|-----------|---------------|
| deno_core | 0.350 | 0.412 |
| deno_error | 0.6 | 0.7 |
| 領先 commits | 40（fork 特有） | 80（upstream 新） |
| 分叉點 | `4b70288`（upstream PR #989, 9/14） | |

Upstream `df8b058 "upgrade: update deno_core and V8"`（9/19）把
deno_core 從 0.350 升到 0.412，改了 8 個檔案（+547/−310 行）。
Fork 最近一次 upstream merge（`694c8e1`, 9/17）在此之前，所以
之前的三次 upstream 合併沒有 deno_core 衝突。

`df8b058` 改動：
- `crates/obscura-js/Cargo.toml` — deno_core 0.350→0.412, deno_error 0.6→0.7
- `crates/obscura-js/js/bootstrap.js` — `__obscuraCore` closure bridge 取代 `Deno.core.ops` 直接存取
- `crates/obscura-js/src/runtime.rs` — 333 行變更（`v8::PinScope` 取代 `v8::HandleScope`, OpState API 等）
- `crates/obscura-js/src/ops.rs` — 20 行（scope type）
- `crates/obscura-js/src/frame.rs` — 12 行
- `crates/obscura-js/src/module_loader.rs` — 19 行
- `Cargo.lock` — 449 行
- `deny.toml` — 5 行

其後 78 個 upstream commits 都基於 0.412 API 撰寫。

## 現況

### 分支狀態（2026-09-29 確認）

- `origin/main` = fork main（deno_core 0.350）
- `upstream/main` = upstream main（deno_core 0.412）
- `merge-wave2-security-render` = 已損壞（無法編譯，不應使用）
- 本地有 10+ 個 `fix/*` 分支（個別 PR 分支，保留）

### Fork 40 個特有 commits 分類

**程式碼 commits（需 cherry-pick）：**

1. `d0712cb` — Add CJK font support via cjk feature and runtime font directory
   - 15 files, +516/−28
   - 關鍵檔案：`inline.rs`（+224 行）、`paint.rs`（+54/−28）、
     4 個 `Cargo.toml`、`main.rs`（+23）、Dockerfile、
     2 個 OTF 字型檔（各 ~16MB）、`cjk-fallback.html` fixture
   - 改動內容：`cjk` feature flag、Noto Sans CJK SC/TC 嵌入、
     `OBSCURA_FONTS_DIR`/`--fonts` runtime 字型目錄、
     SVG lazy font database、`extra_fallback_fonts` 注入
     `base_font_database()`

2. `c9af5de` — Repoint all operational references to the fork repo
   - 15 files, +250/−182
   - 關鍵檔案：`Cargo.toml`（git dependency URL）、
     `crates/obscura/Cargo.toml`、`.github/CODEOWNERS`、
     `SECURITY.md`、`AGENTS.md`、`README.md`、`docs/*`
   - 改動內容：所有引用從 upstream 改為 fork

**其餘 38 個 commits 全是 docs/CI/Docker/TODO 變更**，
不需要 cherry-pick，直接在 Step 4 從 `origin/main` 複製。

### 之前 merge-wave2 嘗試失敗原因

`merge-wave2-security-render` 分支嘗試將 upstream 80 commits
合併進 fork main（deno_core 0.350），觸發：
- `NetworkEvent` 缺少 `intercepted` 欄位
- `schedule_screencast_frame` → `queue_screencast_frame`
- `emit_same_document_navigation` 不存在於 fork
- `non_geometric_computed_style` 缺失
- deno_core API 全面不兼容（`PinScope` vs `HandleScope`）

結論：此分支不應使用，應從 `upstream/main` 重新建立。

## 策略

**方案 B：基於 upstream/main 重建分支**

從 `upstream/main`（已有 deno_core 0.412 + 80 commits）建立新分支，
cherry-pick fork 的 2 個程式碼 commits，再複製 fork 特有文件。
避免重做 deno_core 遷移（upstream 已完成）。

## 磁碟空間

**開始前確認 `df -h /` 可用空間 >= 14 GB。**

`target/` 冷建構需要 ~8 GB（V8 編譯 + release artifacts）。
如果空間不足，先 `cargo clean` 或 `rm -rf target/`。

## Step-by-step 計劃

### Phase 0: Pre-checks

```bash
cd /root/opencode-stuffs/obscura-cjk

# 1. 確認磁碟空間
df -h /
# 需要 >= 14 GB 可用

# 2. 確認 remotes
git remote -v
# 預期:
#   origin    git@github.com:Lawlietr/obscura-cjk.git
#   upstream  git@github.com:h4ckf0r0day/obscura.git

# 3. 抓取最新 upstream
git fetch upstream

# 4. 確認 upstream/main 的 deno_core 版本
git show upstream/main:crates/obscura-js/Cargo.toml | grep deno_core
# 預期: deno_core = "0.412"

# 5. 確認分叉點仍正確
git merge-base upstream/main origin/main
# 預期: 4b7028830222175ca812a5caef04f9d802e64c23 或更新

# 6. 確認 d0712cb 和 c9af5de 存在
git log --oneline -1 d0712cb
git log --oneline -1 c9af5de
```

### Phase 1: 建立新分支

```bash
# 從 upstream/main 建立
git checkout -b rebase-upstream-20260929 upstream/main

# 確認 deno_core 版本
grep deno_core crates/obscura-js/Cargo.toml
# 預期: 0.412
```

### Phase 2: Cherry-pick CJK commit

```bash
git cherry-pick d0712cb
```

**預期衝突：**

1. **`crates/obscura-render/src/inline.rs`** — 最可能的衝突點。
   CJK commit 加入 `extra_fallback_fonts` 注入 `base_font_database()`
   和 `svg_font_database_with_fallbacks()`。Upstream 80 commits 可能
   修改了 font database 架構。

   **解決策略：** 保留 CJK 的 fallback 注入邏輯，適應 upstream 的新
   font database API。具體做法：
   - 查看 upstream 版本的 `base_font_database()` 簽名和回傳型別
   - 將 CJK 的 `extra_fallback_fonts` 注入點對齊到新 API
   - 確保 `cjk` feature gate 完整
   - SVG lazy path（`svg_font_database_with_fallbacks`）獨立 OnceLock
     邏輯保留

2. **`crates/obscura-render/src/paint.rs`** — 可能衝突。
   CJK 改了 `decode_font_bytes` 和 SVG fallback 函式。

   **解決策略：** 保留 CJK 字型解碼邏輯，對齊 upstream 的 paint.rs
   變更。

3. **`crates/obscura-render/Cargo.toml`** — 低風險衝突。
   `cjk` feature 定義可能與 upstream 新增 features 衝突。

   **解決策略：** 合併兩邊的 features，保留 `cjk = ["obscura-render?/cjk"]`。

4. **`crates/obscura-js/Cargo.toml`** — 可能衝突。
   Upstream 改了 deno_core 版本行。

   **解決策略：** 保留 upstream 的 `deno_core = "0.412"` 和
   `deno_error = "0.7"`，加入 CJK feature 的 `obscura-render?/cjk`
   依賴。

5. **`Dockerfile`** — 低風險衝突。
   Upstream 可能改了 build flags。

   **解決策略：** 保留 `--features render,cjk`。

6. **`crates/obscura-render/assets/*`**（OTF 字型檔）— 無衝突預期
   （純新增）。

解決衝突後：

```bash
git add -A
git cherry-pick --continue
# 如果 commit message 需要調整：
# git cherry-pick --edit
```

### Phase 3: Cherry-pick fork references commit

```bash
git cherry-pick c9af5de
```

**預期衝突：**

1. **`Cargo.toml`**（workspace root）— 可能衝突。
   Upstream 可能改了 workspace members 或 profile 設定。

   **解決策略：** 保留 upstream 的 workspace 結構，將
   git dependency URL 改回 `Lawlietr/obscura-cjk`。

2. **`crates/obscura/Cargo.toml`** — 類似。

3. **`AGENTS.md`** — 會衝突（兩邊都改了）。
   **解決策略：** 採用 fork 版本（Phase 4 會完整複製）。
   快速解法：`git checkout origin/main -- AGENTS.md`

4. **`README.md`** — 會衝突。
   **解決策略：** 採用 fork 版本（Phase 4 會完整複製）。

5. **`.github/CODEOWNERS`**, **`SECURITY.md`**, **`docs/*`** —
   可能衝突，採用 fork 版本。

**快速解法（減少衝突痛苦）：**

如果 cherry-pick `c9af5de` 衝突太多，改用：

```bash
git cherry-pick --abort
# 直接從 origin/main 複製 fork 特有文件
git show origin/main:.github/CODEOWNERS > .github/CODEOWNERS
git show origin/main:.github/ISSUE_TEMPLATE/config.yml > .github/ISSUE_TEMPLATE/config.yml
git show origin/main:SECURITY.md > SECURITY.md
git show origin/main:Cargo.toml > Cargo.toml  # 注意：需手動確認 deno_core 0.412
git show origin/main:crates/obscura/Cargo.toml > crates/obscura/Cargo.toml
```

**重要：** `Cargo.toml` 不能直接複製 origin/main 版本（那是 0.350），
需要手動確認 deno_core 行是 0.412。

### Phase 4: 複製 fork 特有文件

```bash
# 以下檔案全部從 origin/main 複製（fork 特有，upstream 沒有）
for f in \
  AGENTS.md \
  TODO.md \
  README.md \
  README_ZH.md \
  docker-compose.example.yaml \
  design/cssom-view-box-geometry.md \
  design/dependabot.md \
  design/hydration-bug-report.md \
  design/mcp-console-capture.md \
  design/release-workflow.md \
  design/svg-dom-api.md \
  docs/Architecture-overview.md \
  docs/Build-from-source.md \
  docs/CJK-and-custom-fonts.md \
  docs/CLI-reference.md \
  docs/Environment-variables.md \
  docs/Installation.md \
  docs/README.md \
  docs/Run-in-production-at-scale.md \
  docs/SUMMARY.md \
  docs/Use-as-a-Rust-library.md \
  docs/Use-the-MCP-server.md \
  render-repros/cjk/cjk-fallback.html \
; do
  git show origin/main:"$f" > "$f" 2>/dev/null || echo "SKIP $f"
done

# design/ 下的本文件也要加入
mkdir -p design
# (upstream-rebase-20260929.md 已在 origin/main 中，會被上面的迴圈複製)

# .github/ 下的 fork 特有檔案
for f in \
  .github/CODEOWNERS \
  .github/ISSUE_TEMPLATE/config.yml \
  .github/dependabot.yml \
  .github/workflows/release.yml \
  .github/workflows/docker.yml \
; do
  git show origin/main:"$f" > "$f" 2>/dev/null || echo "SKIP $f"
done

# Dockerfile 需要確認（兩邊都可能改過）
# 檢查 --features 是否包含 cjk
grep -n 'features' Dockerfile
# 預期包含: --features render,cjk
```

**不要複製的檔案：**
- `Cargo.lock` — 讓 cargo 自動生成（deno_core 版本不同）
- `crates/obscura-js/*` — 採用 upstream 版本（deno_core 0.412 API）
- `crates/obscura-cdp/*` — 採用 upstream 版本
- `crates/obscura-browser/*` — 採用 upstream 版本
- `crates/obscura-net/*` — 採用 upstream 版本

### Phase 5: 建置

```bash
# 先確認磁碟空間
df -h /

# 冷建構（target/ 應該不存在）
CARGO_INCREMENTAL=0 CARGO_BUILD_JOBS=2 cargo build --release \
  -p obscura-cli --bins --features render,cjk 2>&1 | tee /tmp/build.log

# 確認編譯成功
tail -5 /tmp/build.log
# 預期: Finished `release` profile

# 確認 binary 存在
ls -lh target/release/obscura
```

**如果編譯失敗：**
- 先檢查是否 deno_core 版本正確：`grep deno_core Cargo.lock | head -3`
- 檢查 Cargo.toml 的 features 是否完整
- 錯誤訊息中的 API 差異可能表示 upstream 在 df8b058 之後
  又有 API 變更，需要逐一修正

### Phase 6: 測試

```bash
# 1. 全量 nextest
cargo nextest run --release --features render,cjk --no-fail-fast 2>&1 | tail -20
# 預期: 全部 passed（可能有少量 skipped）

# 2. SVG filter（確認 CJK SVG lazy font 沒有回歸）
cargo nextest run --release --features render,cjk -p obscura-render -- svg 2>&1 | tail -10

# 3. Release build（已確認）

# 4. CJK fixture 驗證
RUN_ROOT="$(mktemp -d)"
OBSCURA_BIN=./target/release/obscura \
  ./target/release/obscura fetch \
  "file://$(pwd)/render-repros/cjk/cjk-fallback.html" \
  --screenshot "$RUN_ROOT/cjk.png"
# 確認 PNG 非空白且無豆腐框
ls -lh "$RUN_ROOT/cjk.png"

# 5. 障礙課程（需要 obscura-benchmark 倉庫）
# 如果有 clone:
# OBSCURA_BIN=./target/release/obscura \
#   python3 /path/to/obscura-benchmark/obstacle-course/run.py \
#   --runs 1 --warmup 0
# 預期: 32/33（observer-intersection 是已知問題）
```

### Phase 7: Push + cleanup

```bash
# 1. Commit 所有變更
git add -A
git commit -m "rebase: re-baseline on upstream/main (deno_core 0.412)

Re-baseline the fork on upstream/main to pick up the deno_core
0.350 -> 0.412 upgrade (df8b058) and 78 subsequent commits.
Cherry-pick CJK font support (d0712cb) and fork references
(c9af5de); reapply fork-specific docs and CI config.

Supersedes the broken merge-wave2-security-render branch."

# 2. Push 到新的遠端分支
git push origin rebase-upstream-20260929

# 3. 開 PR 或 force-push 到 main（由用戶決定）
# 選項 A: 開 PR（安全）
# gh pr create --base main --head rebase-upstream-20260929 \
#   --title "rebase: re-baseline on upstream/main"
# 選項 B: 直接 force-push（快速，但丟失 history）
# git push origin rebase-upstream-20260929:main --force-with-lease

# 4. 清理
# 舊的 merge-wave2 分支可選刪除
# git branch -D merge-wave2-security-render
# git push origin --delete merge-wave2-security-render
# 本地 fix/* 分支保留（個別 PR 參考）
```

## 預期衝突總表

| 檔案 | 風險 | 解決策略 |
|------|------|----------|
| `crates/obscura-render/src/inline.rs` | **高** | 對齊 CJK fallback 注入到新 font DB API |
| `crates/obscura-render/src/paint.rs` | 中 | 保留 CJK 字型解碼，對齊 upstream paint 變更 |
| `crates/obscura-render/Cargo.toml` | 低 | 合併 features |
| `crates/obscura-js/Cargo.toml` | 中 | 保留 0.412 版本 + 加 cjk feature |
| `crates/obscura-cli/Cargo.toml` | 低 | 加 cjk feature 傳遞 |
| `crates/obscura-cli/src/main.rs` | 低 | `--fonts` flag（純新增，不易衝突） |
| `crates/obscura-browser/Cargo.toml` | 低 | feature 傳遞 |
| `Dockerfile` | 低 | 保留 `--features render,cjk` |
| `Cargo.toml`（root） | 中 | workspace 結構採 upstream，URL 改 fork |
| `AGENTS.md` | 確定衝突 | 採 fork 版本 |
| `README.md` | 確定衝突 | 採 fork 版本 |
| `deny.toml` | 低 | 合併兩邊 |

## 驗證 checklist

- [ ] `grep 'deno_core' Cargo.toml` → 0.412
- [ ] `grep 'deno_core' Cargo.lock` → 0.412.0
- [ ] `cargo build --release --features render,cjk` 成功
- [ ] `cargo nextest run --release --features render,cjk --no-fail-fast` 全過
- [ ] CJK fixture 截圖無豆腐框
- [ ] SVG nextest filter 通過
- [ ] 障礙課程 32/33（如有 obscura-benchmark）
- [ ] `--fonts` flag 功能正常（可選）
- [ ] `OBSCURA_FONTS_DIR` 功能正常（可選）

## Rollback

```bash
# 如果 rebase 失敗或需要回退：
git checkout main  # 回到 fork main（deno_core 0.350）
git branch -D rebase-upstream-20260929
# origin/main 不受影響（除非已 force-push）
```

## 時間估計

| 階段 | 估計 |
|------|------|
| Phase 0: Pre-checks | 5 分鐘 |
| Phase 1: 建立分支 | 1 分鐘 |
| Phase 2: Cherry-pick CJK + 解衝突 | 1-3 小時（取決於 inline.rs 衝突程度） |
| Phase 3: Cherry-pick fork refs | 30 分鐘 - 1 小時 |
| Phase 4: 複製文件 | 10 分鐘 |
| Phase 5: 建置 | 5-10 分鐘（V8 冷編譯） |
| Phase 6: 測試 | 15-30 分鐘 |
| Phase 7: Push | 5 分鐘 |
| **合計** | **2-6 小時** |

## 注意事項

1. **不要自動跑驗證**（AGENTS.md convention）。建置和測試只在用戶
   明確要求時執行。本計劃中的建置/測試步驟是接手的 agent 在用戶
   授權後才執行的。
2. **不要 bulk-run `cargo fmt`**。
3. **磁碟空間**是硬約束。冷建構前務必 `df -h /` 確認 >= 14 GB。
4. **`bootstrap.js` 的 `__obscuraCore` bridge**（deno_core 0.412 引入）
   不要改回 `Deno.core.ops` 直接存取。這是 upstream 的架構變更。
5. **V8/deno 家族不進 dependabot**。rebase 後 dependabot 配置
   保持不變（已排除 v8/deno）。
