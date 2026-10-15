---
name: waku-sync-release
description: 'Sync the Waku fork to the latest upstream release and package a signed DMG. Use when the user says "check updates / check new updates / pack dmg / build / package / sync upstream / rebuild" or asks to ship a new Waku release. Repo is the wenck/waku fork, upstream is egoist/waku, custom branch is feature/font-size-settings. Preserve all custom features (disabled checkpoints, 28px max font size, Cmd+/- zoom, custom Claude models, Amp no-archive, Pi failure reporting), use the runtime_shaders feature to build without a full Xcode install, and sign with --adhoc.'
---

# Waku fork sync + release DMG

Sync the `wenck/waku` fork's `feature/font-size-settings` branch to upstream `egoist/waku`, preserving every custom feature, then package an ad-hoc signed macOS DMG.

## Key facts (verify every run, don't rely on memory)

| Item | Value |
|---|---|
| origin (fork) | `https://github.com/wenck/waku.git` |
| upstream | `https://github.com/egoist/waku.git` |
| working branch | `feature/font-size-settings` |
| upstream branch | `upstream/main` |
| cargo on PATH | `PATH="/opt/homebrew/opt/rustup/bin:$PATH"` (without this you get `cargo not found`) |

## Machine constraint (this is a hard requirement)

This Mac has **Xcode Command Line Tools only — no full Xcode, and the user will NOT install it**. The `metal` shader compiler is therefore missing:

```bash
xcrun -f metal
# → xcrun: error: unable to find utility "metal", not a developer tool or in PATH  (exit 72)
```

`swiftc`/`swift` exist (`/Library/Developer/CommandLineTools/usr/bin/swiftc`), but `metal` does not. So the build needs the `runtime_shaders` feature or `gpui_macos`'s build script fails at `build.rs:136` (`xcrun -sdk macosx metal ...`). There is no other `xcrun metal` call in the entire dependency graph — this one feature is the complete, sufficient fix.

**This is no longer a manual step.** `gpui_macos` is declared with `runtime_shaders` in `Cargo.toml` as a committed custom feature (commit 10 below). Just build; do not add it by hand, and do not revert it afterwards — reverting it invalidates the `gpui_macos` build-script cache, which makes the *next* plain `cargo check` fail with exit 101 at the shader step.

## Custom features (must survive every rebase — 10 commits total)

`git log --oneline upstream/main..HEAD` should always show these 10 (order may vary):

1. **Disable Git checkpoints** (`Disable pre/post-turn Git checkpoints locally` + `Bypass checkpoint RPCs to stop send/queue stalls`)
   - `crates/waku-core/src/checkpoint.rs`: `capture_turn_start` / `capture_turn` return immediately, never run `git add -A`.
   - `src/app/runtime.rs`: `capture_latest_turn_checkpoint_for` is a no-op; `prepare_submission` skips the checkpoint RPC.
2. **Allow custom Claude model IDs** (`Allow custom Claude model IDs` + `Fix Claude custom model call sites`)
   - `crates/waku-core/src/model_catalog.rs` and `crates/waku-protocol/src/model_catalog.rs`: `claude_reasoning_model` marked `pub`.
3. **Keep Amp threads unarchived** (`Keep Amp threads unarchived`)
4. **Surface Pi message-end failures** (`Surface Pi message-end failures`) — `crates/waku-core/src/driver/pi.rs`
5. **28px max font size** (`Raise max font size to 28px`) — `sanitized_font_size` in `crates/waku-client/src/persistence.rs` clamps to `28.0`
6. **Cmd/Ctrl +/- zoom bindings** (`Bind Ctrl/Cmd +/- to UI font size zoom`) — `IncreaseUiFontSize/DecreaseUiFontSize/ResetUiFontSize` actions + keybindings in `src/lib.rs`; `FONT_SIZES` + `neighbor_font_size` in `src/app/settings.rs`
7. **rg/fd tooling preference** (`Prefer rg and fd over grep and find in shell commands`) — a `## CLI tooling preferences` section in `AGENTS.md`, mirroring `CLAUDE.md`. Upstream also edits `AGENTS.md`, so watch for a conflict here.
8. **Runtime shaders** (`Build GPUI shaders at runtime so Command Line Tools can compile`) — `gpui_macos` declared with `runtime_shaders` under `[target.'cfg(target_os = "macos")'.dependencies]` in `Cargo.toml`, plus its `Cargo.lock` entry. This is what lets the app build without Xcode. **Do not revert it and do not re-add it by hand** — see the machine constraint above.
9. **Broader transcript line width** (`CONTENT_MAX_WIDTH = 1080.0`, `USER_MESSAGE_MAX_WIDTH = 920.0`) in `src/app.rs` and `src/app/components.rs`.
10. **Proxy support & PI_PROXY / extension discovery**:
    - Settings ➔ General ➔ Network proxy (`settings.proxy`), auto-detects `http://127.0.0.1:2080`.
    - Injects `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `GRPC_PROXY`, **`PI_PROXY`** (critical: Oh My Pi/Bun global fetch does not use HTTP_PROXY without PI_PROXY!), `ANTHROPIC_PROXY`, `OPENAI_PROXY`.
    - Oh My Pi model probe (`crates/waku-core/src/model_catalog.rs`) must NOT pass `--no-extensions`, ensuring extension-based providers (like `antigravity`) are discovered.
11. **Screen capture protection** (`NSWindowSharingType::None` / `configure_capture_protection` in `src/platform.rs` and `src/lib.rs`): window contents are blanked out in screenshots and screen recordings to protect against enterprise surveillance monitoring. Can be disabled via `WAKU_ALLOW_SCREEN_CAPTURE=1`.

## Packaging Discipline (Hard Rule — DO NOT VIOLATE)

- **NEVER trigger DMG packaging (`release.ts`) automatically without explicit user instruction ("打包" / "构建DMG").**
- Full DMG release packaging takes 1.5–2 minutes because it involves AppleScript Finder layout, APFS compression, codesign, and verification.
- For normal validation during development, **ONLY run `cargo check --all-targets`** (which finishes in 2–3 seconds), or rely on the dev watcher (`bun ./scripts/dev.ts`).
- When user explicitly asks to build release DMG:
  - Add `CARGO_BUILD_JOBS=$(sysctl -n hw.ncpu)` (e.g. 10 cores) for maximum parallel compilation.
  - If unmounting the temporary volume fails due to enterprise agents (e.g. `tbAgent` / Threatbook dissented), run `diskutil unmount force <mount_path>` to clean up.

## Full workflow

### 1. Fetch and review upstream changes

```bash
git fetch upstream
git log --oneline HEAD..upstream/main   # see the new commits and tag
git log -1 --oneline upstream/main
```

Report to the user: the new version number plus what each commit does. Wait for "go on / 来吧 / please go on as usual" before continuing.

### 2. Rebase the custom branch onto upstream

```bash
git rebase upstream/main
```

If it applies cleanly, skip to step 4; otherwise resolve conflicts as below.

### 3. Conflict resolution (nearly always happens)

The most common collision: the `claude_reasoning_model` visibility in `crates/waku-core/src/model_catalog.rs` and `crates/waku-protocol/src/model_catalog.rs`.

**Rule**: keep upstream's new content (e.g. `grok_model_reasoning_efforts`), but keep `claude_reasoning_model` as `pub`.

The conflict markers look like:

```rust
<<<<<<< HEAD
fn grok_reasoning_model(...) { ... }   // upstream new — keep
fn claude_reasoning_model(...) { ... }  // upstream is private `fn`
=======
pub fn claude_reasoning_model(...) { ... } // ours is `pub`
>>>>>>> <commit>
```

**Resolution**: keep all upstream content, change only `claude_reasoning_model` to `pub fn`, delete the conflict markers.

```bash
# after fixing the conflict
git add <changed files>
git rebase --continue
```

If the next commit also conflicts, repeat. When unsure, read the file's surrounding context and preserve both upstream's new logic and our customization.

### 4. Verify compilation

```bash
PATH="/opt/homebrew/opt/rustup/bin:$PATH" cargo check --all-targets
```

> No feature to add first: `runtime_shaders` is committed (custom feature 8). This takes ~2 minutes and will hit the 120s timeout, getting moved to the background (you'll get a task-notification). Read the output file and confirm it ends with `Finished`.

### 5. Push

```bash
git push origin feature/font-size-settings --force-with-lease
```

### 6. Package the DMG

```bash
PATH="/opt/homebrew/opt/rustup/bin:$PATH" \
WAKU_ANALYTICS_ENDPOINT="https://localhost/disabled" \
WAKU_ANALYTICS_WEBSITE_ID="disabled" \
bun run ./scripts/release.ts --skip-notarize --adhoc
```

Key points:
- `--adhoc` is required (otherwise it errors with `Set WAKU_SIGNING_IDENTITY ...`).
- The two `WAKU_ANALYTICS_*` values are dummies; omitting them errors with `Set WAKU_ANALYTICS_ENDPOINT ...`.
- This also exceeds 120s — run in background, repeatedly read the output file, and wait for `Finished release` + `swift-driver version`. The DMG lands at `dist/Waku-<version>.dmg`.
- If it stalls on `hdiutil: couldn't unmount "diskX" - resource busy`: run `hdiutil detach /dev/diskX -force`, delete leftover `dist/rw.*.dmg` files, and rerun.

### 7. Install to /Applications

The DMG is at `dist/Waku-<version>.dmg`. Mount and copy:

```bash
hdiutil attach dist/Waku-<version>.dmg -nobrowse
cp -R /Volumes/Waku/Waku.app /Applications/    # overwriting needs user confirmation (rm -rf /Applications/Waku.app is blocked)
hdiutil detach /Volumes/Waku
```

> Overwriting `/Applications/Waku.app` is an irreversible delete and will be blocked by auto mode. Either have the user drag it in Finder, or get explicit confirmation before deleting.

## Notes

- `runtime_shaders` stays in `Cargo.toml` / `Cargo.lock` permanently (custom feature 8). It is not a build-time toggle to add and revert.
- Release uses `--adhoc` local signing and skips notarization (`--skip-notarize`). The appcast signing step will fail for lack of a private key, but the DMG is already produced by then — ignore it.
- Visual verification: after UI-related changes (font size / zoom / grouping), confirm the dev watcher's freshly signed `Waku Debug.app` launched and applied them; per CLAUDE.md, don't start a second watcher manually.
