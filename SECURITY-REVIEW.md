# Security review of this fork

This fork (`nisparks/solidcraft`) is reviewed before it is built and run on a personal Windows
machine. This file records what was checked, what was found, what this fork changes, and how to
take changes from upstream (`tangency-labs/solidcraft`) without losing that review. To report a
vulnerability in SolidCraft itself, follow [SECURITY.md](SECURITY.md).

- **Reviewed baseline:** upstream `main` at `47bfaf734c0ee3db4d87b45beeccb7562ffc9dda`.
- **Secure baseline tag:** `secure-baseline-2026-11-09`, the commit that adds this file, with the changes below.
- **Upstream:** <https://github.com/tangency-labs/solidcraft>.

## Scope and method

A static review of the whole tree, without running a release build first:

- **Supply chain.**
  - Every `Cargo.lock` package checksum was compared with crates.io.
  - Every package was checked against the RustSec advisory database: no known vulnerabilities.
  - `vendor/` (winit, egui-wgpu) was diffed against the published crates: only small, commented
    patches.
  - Every `build.rs` and proc-macro was read: none reach the network or write outside `OUT_DIR`.
  - The repository has no committed executables or prebuilt libraries.
- **Data leaving the machine.** Every HTTP client, socket, `open_url`, process launch and file
  write outside the design folder was traced.
  - There is no telemetry, analytics or crash upload. Crash reports stay on disk until the user
    sends one.
  - The network features are: update checks (release builds only), EasyPrint upload, model and
    library downloads, and the optional AI chat providers. Each is user-started or off in a
    self-built binary.
- **Unsafe code.**
  - `unsafe` is denied workspace-wide.
  - The only first-party exceptions are Win32 calls in `apps/solidcraft`: window chrome and 3D
    mouse device names. The rest is in vendored winit and well-known dependencies.
- **Local attack surface.** The control channel, the MCP server, the AI-app auto-connect, design
  locks and the file importers.

## Findings

| # | Severity | Area | Finding | In this fork |
|---|----------|------|---------|--------------|
| 1 | 🟡 MEDIUM | `crates/ui-egui/src/ai_connect.rs` | At every start, SolidCraft added itself as an MCP server to the config files of every AI app it found (Claude, Cursor, VS Code, Copilot CLI…), without asking. | **Fixed:** off by default |
| 2 | 🟡 MEDIUM | `easyprint_ui.rs`, `prefs.rs` | An agent or script could upload a model to Printables by passing `consent: true` itself, or by setting `easyprint_consent` through `ui.prefs`. The app's own question was skipped. | **Fixed** |
| 3 | 🟡 MEDIUM | Whole repository | A young, fast-moving project written mostly by agents: hundreds of commits a day, and a large amount of new code between reviews. | Process below |
| 4 | ⚪ LOW | `crates/io/src/design_lock.rs:103,110` | The `.~<design>.lock` file next to an open design records the computer name and user name. It can leak if the design folder is shared or synced. | Documented |
| 5 | ⚪ LOW | `crates/ui-egui/src/update/install.rs:33` | `PUBLIC_KEY` is `None`, so updates have no signature check beyond HTTPS. | Documented (self-built binaries don't update) |
| 6 | ⚪ LOW | `crates/control` | The control token can do anything the user can: run any command, change preferences, click through dialogs. | Documented |
| 7 | ⚪ LOW | `apps/solidcraft/Cargo.toml` | The comment listing the files allowed `unsafe` had drifted from the code. | **Fixed** |
| 8 | ⚪ LOW | `cadmpeg` importers | Young parsers of untrusted files (STEP, 3MF and others). | Documented |
| 9 | ⚪ LOW | `crates/engine/src/easyprint.rs` | The check that the EasyPrint answer opens a `printables.com` address read the host loosely: `https://evil.example\.printables.com` passed. | **Fixed** |

There were no critical or high findings.

## What this fork changes

1. **AI apps are no longer connected automatically** (#1).
   - The preference `connect_ai_apps` (Settings > Assistant > **Add SolidCraft to AI apps**) now
     defaults to off.
   - Saved preferences that already have it on keep it on.
   - Help > Connect AI assistants… still does it by hand.
2. **Only the user can agree to EasyPrint uploads** (#2).
   - The session's upload refuses unless `Prefs::easyprint_consent` is set.
   - Only the Upload button in the app's Open with EasyPrint question sets it.
   - `ui.prefs`, `prefs::set` and settings profiles can withdraw it, not give it.
   - A caller's `consent: true` is still required too.
3. **The EasyPrint answer must be a plain `https://` address on `printables.com`** (#9).
   - A backslash, whitespace or any control character refuses it.
   - So does a host with anything but ASCII letters, digits, `.` and `-`.
4. **The `unsafe` allow-list comment matches the code** (#7).

Changes a default: yes, `connect_ai_apps` is now off. Turn it on in Settings > Assistant to get the
upstream behaviour back.

## Remaining risks and how to run it safely

- **The control channel** (#6).
  - It listens on loopback only, needs a token, and is on by default.
  - Turn off Settings > Assistant > **Let AI agents connect** (`agent_control`) when no agent
    should drive the app.
  - A program that can read the token runs as you anyway (see [SECURITY.md](SECURITY.md)).
  - A control client can still click through the EasyPrint question with `ui.click`/`ui.confirm`.
    That is visible on screen, and is the reason this fork turns off finding #1 rather than relying
    on prompts.
- **Updates** (#5).
  - A binary built from source (`cargo build`, `cargo run`) never checks for updates: that needs
    `SOLIDCRAFT_VERSION` set at build time and a release build (`crates/ui-egui/src/update.rs:138`).
  - Keep building from this fork rather than installing upstream releases.
- **Design locks** (#4). Don't share design folders with people who shouldn't see your computer and
  user names. The lock file is removed when the design is closed, but a crash can leave it behind.
- **Importers** (#8). Open STEP, 3MF and other files from people you trust.
  - Parsing errors are `Result`s.
  - `unsafe` is denied.
  - A malformed file can still be slow, or use a lot of memory.
- **The AI chat providers** send the conversation, and what the agent reads from the design, to the
  provider you choose. Leave them unconfigured to keep everything local.

## Taking changes from upstream

Keep `main` of this fork at a reviewed state. Take upstream changes on a branch, review the parts
that run at build time or reach the network, then merge.

```powershell
git remote add upstream https://github.com/tangency-labs/solidcraft.git   # once
git fetch upstream
$base = git merge-base HEAD upstream/main
git switch -c upstream-sync-$(Get-Date -Format yyyy-MM-dd)

# 1. What runs while building, and what changed in dependencies.
git diff $base upstream/main --stat -- ':(glob)**/build.rs' ':(glob)**/Cargo.toml' Cargo.lock .cargo .config vendor xtask .github packaging scripts tools
git diff $base upstream/main -- Cargo.lock | Select-String '^[+-]name = '

# 2. New network, process, unsafe and file-system code.
git diff $base upstream/main -- crates apps solidcore | Select-String '^\+.*(reqwest|ureq|TcpStream|UdpSocket|open_url|Command::new|unsafe|std::env::var|home_dir|config_dir)'

# 3. Take everything, or pick commits.
git merge upstream/main           # or: git cherry-pick -x <sha> ...
```

Then:

1. Check that this fork's changes survived:
   - `connect_ai_apps: bool = false` in `crates/ui-egui/src/prefs.rs`.
   - The `easyprint_consent` guard at the top of `prepare_preferences`.
   - `agreed` in `crates/ui-egui/src/easyprint_ui.rs`.
   - The host check in `crates/engine/src/easyprint.rs`.
2. Run the tests that pin them:
   ```powershell
   cargo test --locked -p solidcraft-engine -p solidcraft-ui-egui --lib -- easyprint ai_connect settings_profiles
   ```
3. Re-run the supply-chain checks.
   - The repository's own check: `cargo xtask licences --check`.
   - With the tools installed (`cargo install --locked cargo-audit cargo-deny`), also run
     `cargo audit` and `cargo deny check`.
4. Build with `--locked`, so a dependency can't change during the build.
5. Merge into `main`, and tag the result `secure-baseline-<date>`.
6. Update the baseline lines at the top of this file.
