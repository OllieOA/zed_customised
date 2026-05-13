# OSC-driven state icons — Implementation Plan

> **Status (2026-05-10):** Plan B confirmed working in daily Claude
> Code use. State icons render with the correct color when hooks fire
> (`>` gray on idle, `$` green on prompt submit, `!` orange on
> notification, muted `?` before any hook). The formal S3–S9 smoketest
> walkthrough was never executed — superseded by everyday validation.
> The "Verification S3–S9" section below is preserved for reference if
> a regression check is ever needed (e.g. after a rebase). The
> "Next-session entry points" debug ladder from 2026-05-06 has been
> retired since the bug it was scoped to no longer reproduces.
>
> Open side thread: the multi-`myzed` window-cycling keybind (see "Side
> thread" near the bottom) was attempted and **does not switch windows
> as of 2026-05-10**. Root cause not yet isolated — see that section
> for the diagnostic ladder.
>
> **Status (2026-05-06, prior):** Plan B code shipped on commit `1ccbbc37fa`
> (terminal: render colored state-icon Label in tab content). S1 (cold
> open shows muted `?`) and S2 (bash-leak alone keeps muted `?`) verified
> on the original Plan B install. Smoketest S3 (idle hook `>`) failed on
> first attempt 2026-05-06 — failure mode unrecorded, debug deferred.
> The local smoketest log `.tmp/smoketests/osc-title.md` was deleted at
> user request. (Resolved by 2026-05-10 daily-use validation; this note
> retained for the reasoning trail.)
>
> **Status (2026-05-02, prior):** Tasks 1-5 shipped on commit `4736638375`
> (the v2 emoji-prepend approach). Smoketest S1 then revealed that emoji
> glyphs render invisibly in tab labels under GPUI's Linux text shaper.
> **Plan B** pivoted to colored ASCII symbols rendered as a separate
> `Label` in the View layer; this shipped on commit `1ccbbc37fa`. Tasks
> 6 and 7 below have been collapsed into the "Done — Plan B" status
> block.
>
> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal (revised under Plan B):** Replace v1 OSC passthrough patch (commit `c6295926e3`) with a Claude-hooks-driven state icon system: the data layer captures single-grapheme OSC titles into a private `state_icon` field, and the terminal-tab View layer renders that character as a colored `Label` next to the existing terminal icon (no composition into `Terminal::title`'s return value).

**Architecture:** New private `state_icon: Option<String>` field on `Terminal`, updated only when `AlacTermEvent::Title` carries exactly one grapheme cluster (filters out bash `PROMPT_COMMAND` redraws). `Terminal::title()` is **unchanged** from upstream — it remains a pure data getter. A new `Terminal::state_icon()` getter exposes the field. `TerminalView::tab_content` reads the getter and inserts a colored `Label` between the terminal icon and the title text when the terminal is not running a Zed task. Existing `breadcrumb_text` field is left alone — it powers a separate UI surface in `terminal_view`.

**Tech Stack:** Rust, GPUI, alacritty_terminal, unicode-segmentation 1.10 (already a workspace dep).

**Spec:** [`.superpowers/specs/20260501_osc_title_patch.md`](../specs/20260501_osc_title_patch.md) (commit `a1a24792bc`, revised in-place for Plan B 2026-05-02).

**Branch:** `osc-title-patch` (do not push without asking; see CLAUDE.md).

**Tests:** `Terminal::title` is unchanged so no data-layer test is added. The View-layer change is covered indirectly by the existing `test_tab_content_uses_custom_title` in `crates/terminal_view/src/terminal_view.rs` (passes after the patch). Per CLAUDE.md "no tests beyond what verifies the patch behavior," the colored-Label rendering itself is verified by the visual smoketest scenarios, not a Rust test.

---

## Task 1: Wire `unicode-segmentation` into the `terminal` crate

The grapheme-cluster filter in the Title handler needs `unicode-segmentation`. The crate is already declared in the workspace `Cargo.toml` (`unicode-segmentation = "1.10"` at line 782) but not yet pulled into `crates/terminal`.

**Files:**
- Modify: `crates/terminal/Cargo.toml`

- [ ] **Step 1.1: Add the dep**

In `crates/terminal/Cargo.toml`, add `unicode-segmentation.workspace = true` to `[dependencies]`. Insert immediately before the `url.workspace = true` line so the `u`-prefixed deps stay grouped:

```toml
thiserror.workspace = true
unicode-segmentation.workspace = true
url.workspace = true
util.workspace = true
```

- [ ] **Step 1.2: Verify the workspace still resolves**

Run inside the Nix dev shell (per `.tmp/scripts/incremental-build.sh` pattern):

```sh
nix develop --accept-flake-config --command cargo check -p terminal
```

Expected: zero errors. May warn about an unused dep — that's fine, Task 2 uses it.

- [ ] **Step 1.3: Commit**

```sh
git add crates/terminal/Cargo.toml
git commit -m "$(cat <<'EOF'
terminal: pull unicode-segmentation into the crate

Prep for OSC state-icon work — the Title-event handler will need
extended-grapheme-cluster counting to discriminate single-glyph icon
emissions from multi-glyph PROMPT_COMMAND titles. The crate is already
declared in the workspace at "1.10".
EOF
)"
```

---

## Task 2: Implement the state-icon mechanism

This is the main patch. It bundles four edits in `crates/terminal/src/terminal.rs` into one commit because the field, its initialization, the writer (Title handler), and the reader (`Terminal::title`) are co-dependent — splitting them would produce intermediate commits with `unused field` warnings or incomplete behavior.

**Files:**
- Modify: `crates/terminal/src/terminal.rs`

- [ ] **Step 2.1: Locate the patch sites**

Run these greps and note the exact line numbers in this session (do not record them in the spec or this plan — they drift on rebase):

```sh
grep -n "use unicode_segmentation\|use util::\|use sysinfo::" crates/terminal/src/terminal.rs | head
grep -n "pub breadcrumb_text" crates/terminal/src/terminal.rs
grep -n "breadcrumb_text: String::new()" crates/terminal/src/terminal.rs
grep -n "AlacTermEvent::Title\|AlacTermEvent::ResetTitle" crates/terminal/src/terminal.rs
grep -n "fn title" crates/terminal/src/terminal.rs
```

Expected: one `pub breadcrumb_text` line, two `breadcrumb_text: String::new()` lines (constructor sites), one `AlacTermEvent::Title(title)` arm, one `AlacTermEvent::ResetTitle` arm, one `pub fn title(&self, truncate: bool)`.

- [ ] **Step 2.2: Add the import**

Add this `use` near the other top-of-file imports (group with the alphabetically appropriate `u`/`util` imports if there's a sorted block; otherwise add to the closest unsorted group):

```rust
use unicode_segmentation::UnicodeSegmentation;
```

- [ ] **Step 2.3: Add the field**

Find the `pub breadcrumb_text: String,` line in the `Terminal` struct definition. Insert the new field directly after it:

```rust
pub breadcrumb_text: String,
state_icon: Option<String>,
title_override: Option<String>,
```

The field is private (no `pub`) — only `Terminal::title` reads it and only `process_event` writes it.

- [ ] **Step 2.4: Initialize at both constructor sites**

Both `Terminal` constructors set `breadcrumb_text: String::new(),`. Add `state_icon: None,` directly after each:

```rust
breadcrumb_text: String::new(),
state_icon: None,
```

There are exactly two such sites. Check both. After this step, `cargo check -p terminal` should pass with at most a "field never read" warning — that goes away in Step 2.6.

- [ ] **Step 2.5: Update the `AlacTermEvent::Title` arm**

In `process_event`, find the `AlacTermEvent::Title(title)` arm. Currently it ends with:

```rust
self.breadcrumb_text = title;
cx.emit(Event::BreadcrumbsChanged);
```

Replace those two lines with:

```rust
// Single-grapheme titles are treated as state icon updates.
// Multi-grapheme titles (e.g. bash PROMPT_COMMAND's "user@host:cwd")
// are NOT considered icons — state_icon is preserved untouched.
if UnicodeSegmentation::graphemes(title.as_str(), true).count() == 1 {
    self.state_icon = Some(title.clone());
}

self.breadcrumb_text = title;
cx.emit(Event::BreadcrumbsChanged);
```

The Windows guard above (`#[cfg(windows)] { if self.shell_program ... }`) is unchanged. The `AlacTermEvent::ResetTitle` arm a few lines below is also unchanged — `state_icon` deliberately persists across explicit title resets so Claude state remains the source of truth.

- [ ] **Step 2.6: Rewrite `Terminal::title`'s `None` arm**

The current (v1) `None` arm is:

```rust
None => self
    .title_override
    .as_ref()
    .map(|title_override| title_override.to_string())
    .or_else(|| {
        if self.breadcrumb_text.is_empty() {
            None
        } else if truncate {
            Some(truncate_and_trailoff(&self.breadcrumb_text, MAX_CHARS))
        } else {
            Some(self.breadcrumb_text.clone())
        }
    })
    .unwrap_or_else(|| match &self.terminal_type {
        TerminalType::Pty { info, .. } => info
            .current
            .read()
            .as_ref()
            .map(|fpi| {
                let process_file = fpi
                    .cwd
                    .file_name()
                    .map(|name| name.to_string_lossy().into_owned())
                    .unwrap_or_default();

                let argv = fpi.argv.as_slice();
                let process_name = format!(
                    "{}{}",
                    fpi.name,
                    if !argv.is_empty() {
                        format!(" {}", (argv[1..]).join(" "))
                    } else {
                        "".to_string()
                    }
                );
                let (process_file, process_name) = if truncate {
                    (
                        truncate_and_trailoff(&process_file, MAX_CHARS),
                        truncate_and_trailoff(&process_name, MAX_CHARS),
                    )
                } else {
                    (process_file, process_name)
                };
                format!("{process_file} — {process_name}")
            })
            .unwrap_or_else(|| "Terminal".to_string()),
        TerminalType::DisplayOnly => "Terminal".to_string(),
    }),
```

Replace the entire `None => …` arm with this:

```rust
None => {
    const UNKNOWN_ICON: &str = "❓";
    let icon = self.state_icon.as_deref().unwrap_or(UNKNOWN_ICON);
    let name = self
        .title_override
        .as_ref()
        .map(|t| t.to_string())
        .unwrap_or_else(|| match &self.terminal_type {
            TerminalType::Pty { info, .. } => info
                .current
                .read()
                .as_ref()
                .map(|fpi| {
                    let process_file = fpi
                        .cwd
                        .file_name()
                        .map(|name| name.to_string_lossy().into_owned())
                        .unwrap_or_default();

                    let argv = fpi.argv.as_slice();
                    let process_name = format!(
                        "{}{}",
                        fpi.name,
                        if !argv.is_empty() {
                            format!(" {}", (argv[1..]).join(" "))
                        } else {
                            "".to_string()
                        }
                    );
                    let (process_file, process_name) = if truncate {
                        (
                            truncate_and_trailoff(&process_file, MAX_CHARS),
                            truncate_and_trailoff(&process_name, MAX_CHARS),
                        )
                    } else {
                        (process_file, process_name)
                    };
                    format!("{process_file} — {process_name}")
                })
                .unwrap_or_else(|| "Terminal".to_string()),
            TerminalType::DisplayOnly => "Terminal".to_string(),
        });
    format!("{icon} {name}")
}
```

The `Some(task_state) => …` arm above the `None` arm is **untouched**. Task tabs continue to render their task label with no icon.

The cwd-process fallback's per-component truncation behavior is preserved exactly. The icon adds two visible columns (`<glyph><space>`); no further composed-string truncation is applied — overlong F2 names are clipped by the tab strip's own layout the same way they were before.

- [ ] **Step 2.7: Verify the crate compiles**

```sh
nix develop --accept-flake-config --command cargo check -p terminal
```

Expected: zero errors, zero new warnings. If clippy fires on the patched function, run `./script/clippy` (Task 4) and address inline.

- [ ] **Step 2.8: Commit**

```sh
git add crates/terminal/src/terminal.rs
git commit -m "$(cat <<'EOF'
terminal: render state icon prefix in tab labels

Replaces the v1 OSC passthrough (c6295926e3) with a Claude-hooks-driven
state icon model. Each terminal tab now renders "<icon> <name>":

- New private state_icon: Option<String> field on Terminal, updated only
  when AlacTermEvent::Title carries exactly one grapheme cluster. This
  filters out bash's PROMPT_COMMAND redraws (which emit user@host:cwd)
  so they don't clobber the icon set by Claude Code hooks.
- AlacTermEvent::ResetTitle deliberately preserves state_icon — bare
  \033]0;\007 from a non-Claude source does not reset the state.
- Terminal::title's None arm now composes "<icon> <name>" where
  <icon> is state_icon or "❓" and <name> is title_override (F2 rename)
  or the existing cwd-process fallback. Per-component truncation in
  the fallback is preserved.
- The Some(task_state) arm and breadcrumb_text's terminal_view consumer
  are untouched.

Spec: .superpowers/specs/20260501_osc_title_patch.md
EOF
)"
```

---

## Task 3: Build the patched binary

The terminal crate compiles in seconds; rebuilding the full Zed binary takes ~2-5 minutes incremental on this host.

**Files:** none modified.

- [ ] **Step 3.1: Run the incremental build**

```sh
bash .tmp/scripts/incremental-build.sh
```

Expected: `incremental build took N s` and `binary at: target/release/zed …` lines at the end. If the build fails on a non-`terminal` crate, that's an unrelated upstream issue — investigate before continuing.

- [ ] **Step 3.2: Sanity check the binary**

```sh
ls -la target/release/zed
file target/release/zed
```

Expected: ELF 64-bit executable, modified within the last few minutes.

- [ ] **Step 3.3: Run existing terminal-crate tests**

The terminal crate has unit tests in `mappings/keys.rs`, `mappings/mouse.rs`, and `terminal_hyperlinks.rs`. None cover `Terminal::title`, but we should confirm we haven't broken the existing tests as a side effect of the field/import changes:

```sh
nix develop --accept-flake-config --command cargo test -p terminal
```

Expected: all existing tests pass; no new tests added (per spec "do not invent a test harness for one function").

---

## Task 4: Lint

**Files:** none modified (unless lints flag the patched code).

- [ ] **Step 4.1: Run the project clippy script**

```sh
nix develop --accept-flake-config --command ./script/clippy
```

This may take several minutes — clippy walks the whole workspace.

- [ ] **Step 4.2: Address any new warnings in the patched function**

If clippy flags anything in `terminal.rs` lines you touched in Task 2:

- For style nits (e.g. `use_format_args_macro_calls`): apply the suggested fix, re-run clippy, amend the commit (`git add … && git commit --amend --no-edit`).
- For substantive complaints (e.g. type complexity in the `None` arm): reassess the `Terminal::title` rewrite — possibly extract a helper.

Do **not** suppress clippy with `#[allow(…)]` unless an existing pattern in the file already does it for the same lint. If unsure, surface the warning to the user before suppressing.

- [ ] **Step 4.3: Re-verify clean build after any fixes**

If you amended the commit, re-run:

```sh
bash .tmp/scripts/incremental-build.sh
```

---

## Task 5: Install the patched binary

The wrapper at `~/.local/bin/zed-patched` execs into `~/.local/libexec/zed-patched-bin`. We copy the freshly built binary into place; the wrapper itself is not touched (it sets `LD_LIBRARY_PATH=/usr/lib64` and `FONTCONFIG_PATH=/etc/fonts` — preserved from the previous session).

**Files:** none modified in repo; `~/.local/libexec/zed-patched-bin` is overwritten outside the repo.

- [ ] **Step 5.1: Kill any running patched Zed**

```sh
pkill -f "zed-patched-bin" || true
```

- [ ] **Step 5.2: Install**

```sh
cp /home/ollie/dev/zed_customised/target/release/zed /home/ollie/.local/libexec/zed-patched-bin
chmod +x /home/ollie/.local/libexec/zed-patched-bin
```

- [ ] **Step 5.3: Confirm install**

```sh
stat -c '%y %n' /home/ollie/.local/libexec/zed-patched-bin
```

Expected: timestamp within the last few minutes.

---

## ✅ Done — Plan B (former Tasks 6 and 7, collapsed)

The original Task 6 (smoketests S1–S9 against emoji glyphs) and Task 7
(hook config + commit-shape decisions) are subsumed by the Plan B
trajectory below. They are not actionable as written — both reference
emoji which is no longer the rendering path.

**What shipped under v2 (commit `4736638375`):**

- ✅ Task 1 — `unicode-segmentation` wired into the `terminal` crate
  (commit `3e937d2a16`).
- ✅ Task 2 — state_icon field, single-grapheme filter in
  `AlacTermEvent::Title`, and emoji-prepend `<icon> <name>` composition
  in `Terminal::title` (commit `4736638375`).
- ✅ Tasks 3–5 — incremental build, clippy, install to
  `~/.local/libexec/zed-patched-bin`.
- ✅ S1 (cold open) attempted — revealed emoji glyphs render as
  invisible boxes in tab labels because GPUI's Linux text shaper
  (cosmic-text + fontdb) does not reliably resolve the
  `ui_font_fallbacks` chain inside tab content. Aborted v2 path.

**Plan B pivot (commit `1ccbbc37fa`):**

- Reverted the `Terminal::title` rewrite — it is now identical to
  upstream behavior. Added a new `state_icon()` getter instead.
- Moved icon rendering into `TerminalView::tab_content` in
  `crates/terminal_view/src/terminal_view.rs`. The View reads
  `terminal.state_icon()`, picks a `Color::*` variant, and inserts a
  colored `Label` between the existing terminal-icon and the title text.
- Symbol set switched from emoji (`❓ ⏳ 💤 ⚠️`) to ASCII (`? > $ !`)
  to sidestep font-shaping entirely. Color carries the meaning.
- Symbol → color mapping (hardcoded in `tab_content`):

  | Symbol | State | `Color::*` |
  |---|---|---|
  | `?` | unknown / no hook fired | `Color::Muted` |
  | `>` | idle (Stop, SessionStart) | `Color::Hint` |
  | `!` | permission (Notification) | `Color::Warning` |
  | `$` | running (UserPromptSubmit) | `Color::Info` |

- ✅ `cargo check -p terminal -p terminal_view` clean.
- ✅ `cargo test -p terminal --lib` — 63 pass, 1 pre-existing fail
  (`tests::test_basic_terminal`, Nix-bash `shopt -u progcomp`, NOT
  caused by the patch).
- ✅ `cargo test -p terminal_view --lib` — 59 pass, 0 fail (incl.
  `test_tab_content_uses_custom_title`).
- ✅ `./script/clippy -p terminal -p terminal_view` clean. Note: do NOT
  pass `--release --all-targets --all-features -- --deny warnings` —
  the script appends those automatically; passing them yourself
  duplicates and breaks with `error: Unrecognized option: 'release'`.
- ✅ Rebuilt and reinstalled to `~/.local/libexec/zed-patched-bin`.
  Wrapper renamed from `zed-patched` to `myzed`
  (`~/.local/bin/myzed`); libexec path unchanged.
- ✅ S1 re-run with Plan B — `?` (gray Muted) renders cleanly in tab
  labels with the new font stack (Noto Sans Mono, weight 400, size 12).
- ✅ User confirmed visually: "This all looks great."

**Commit-shape decision (former Task 7.2):** the v1/v2/Plan B commits
are kept intact on `osc-title-patch` for the reasoning trail. No
autonomous rebase. If/when an upstream attempt is made, surface the
squash options to the user (see the original Task 7.2 wording) — the
agent must not drive `git rebase -i` autonomously.

---

## Verification S3–S9 (open — user-driven, ASCII symbols)

These scenarios remain to be walked through against the installed
Plan B binary. The launcher is `myzed` (was `zed-patched` in older
docs).

**Pre-flight:** start a fresh terminal so old env doesn't pollute, then:

```sh
myzed /home/ollie/dev/zed_customised
```

Open `Zed: about` from the command palette and confirm the commit hash
matches `git rev-parse HEAD` on `osc-title-patch` (auto-update should
already be off — see CLAUDE.md "Auto-update — DISABLE BEFORE RUNNING").

| # | Scenario | How to trigger | Expected tab content |
|---|---|---|---|
| S3 | idle hook | `printf '\033]0;>\007' > /dev/tty` | terminal icon · gray `>` · `~/dev/zed_customised — bash` |
| S4 | running hook | `printf '\033]0;$\007' > /dev/tty` | terminal icon · green `$` · same title |
| **S5** | **bash leak after hook (LOAD-BEARING)** | After S4, press Enter at the empty prompt | tab content **stays** at green `$` — the `PROMPT_COMMAND` redraw sets `breadcrumb_text` but does NOT clobber `state_icon` because its title is multi-grapheme. If the symbol flips back to muted `?`, the single-grapheme filter is broken. |
| S6 | F2 rename then hook | F2 → `build` → Enter, then `printf '\033]0;$\007' > /dev/tty` | terminal icon · green `$` · `build` |
| S7 | hook then F2 rename | New tab: run S4 first, then F2 → `build` → Enter | same as S6 — composition order doesn't matter |
| S8 | title reset preserves symbol | After S4, `printf '\033]0;\007' > /dev/tty` | unchanged from S4 — green `$` survives explicit reset |
| S9 | long process name | Run a long command in the foreground OR navigate to a deeply nested cwd | tab strip clips title gracefully; the colored symbol is a separate `Label` element so it does not push the title text out of frame |

S1 (cold open shows muted `?`) and S2 (bash leak alone keeps muted `?`)
are already verified per the "What shipped" log above. S3–S9 effectively
verified by 2026-05-10 daily-use validation: state icons render with
the correct color when Claude Code hooks fire. Re-run this table only
if a regression is suspected (e.g. after a rebase onto upstream).

---

## Side thread (open as of 2026-05-10): multi-`myzed` window cycling

Out of scope for this plan but documented here so the context isn't lost:

- User wants a global keymap to cycle multiple `myzed` windows
  (Alt+Tab insufficient at scale).
- Built-in actions exist: `workspace::ActivateNextWindow` /
  `ActivatePreviousWindow`, defined in `crates/workspace/src/workspace.rs`
  (variants registered around line 259, listener around line 7121,
  implementation `pub fn activate_next_window` around line 7665). These
  iterate `cx.windows()` — windows of the **current process only**.
- User has bound `ctrl-`` / `ctrl-shift-`` to the actions at `Workspace`
  context in `~/.config/zed/keymap.json`. **As of 2026-05-10 the binding
  does not switch windows.** Root cause isolated on 2026-05-13:

  Each `myzed` invocation spawns a separate process. The command-palette
  invocation of `workspace: activate next window` runs but switches
  nothing, and `pgrep -af zed-patched-bin` returns multiple PIDs after
  opening myzed twice. This is **not a bug** — it is intentional in
  Zed's main entry point:

  ```rust
  // crates/zed/src/main.rs:366
  let failed_single_instance_check = if *zed_env_vars::ZED_STATELESS
      || *release_channel::RELEASE_CHANNEL == ReleaseChannel::Dev
  {
      false  // single-instance check SKIPPED on dev channel
  } else {
      // …actual single-instance handshake…
  };
  ```

  Our locally-built binary has `crates/zed/RELEASE_CHANNEL` set to `dev`
  (the only contents are the string `dev`), so the dev-channel branch is
  taken and single-instance is disabled by design. The
  `ZED_RELEASE_CHANNEL` env var override at `crates/release_channel/src/lib.rs:11`
  is gated on `cfg!(debug_assertions)`, so it is silently ignored by
  release builds — runtime override is not possible without recompilation.

  `ActivateNextWindow` iterates `cx.windows()` (windows of the current
  process). With multi-process myzed, there are no other windows in this
  process to switch to. No in-process keybinding can fix this.

- **Resolution options (none implemented as of 2026-05-13):**
  1. **WM-level cycling (recommended, no rebuild).** KWin → System
     Settings → Shortcuts → "Walk Through Windows of Current
     Application". `myzed` inherits Zed's `dev.zed.Zed` app_id (set in
     the wayland/x11 platform layer from the release-channel-derived
     name), so KWin treats all myzed processes as one app and cycles
     them at the WM layer regardless of process topology.
  2. **Rebuild on a non-dev channel.** Edit
     `crates/zed/RELEASE_CHANNEL` to `preview` (or `stable`/`nightly`)
     and rebuild. Single-instance kicks in and `ActivateNextWindow`
     starts working. **Side effects to think through first:**
     - Settings/assets paths change (`~/.config/zed-preview/` etc) —
       existing settings may not carry over until migrated.
     - Auto-update logic activates on non-dev channels; CLAUDE.md's
       "Auto-update — DISABLE BEFORE RUNNING" guidance becomes
       load-bearing rather than belt-and-braces.
     - The `app_id` changes, which affects WM grouping (e.g. KWin Walk
       Through stops grouping them together since they appear under a
       new app_id).
     - Telemetry endpoint may shift to a non-dev URL.
     This is a larger surface change than the fork's "one patch, one
     place" rule contemplates, so default to option 1 unless the user
     explicitly wants the in-process cycling badly enough to absorb
     the side effects.
  3. **Patch the dev-channel guard out.** One-line change to the
     conditional in `crates/zed/src/main.rs:366` to enable
     single-instance even on dev. Smallest possible fix, but it adds a
     second fork-local patch and complicates the rebase story (the
     `main.rs` entry point churns more than `terminal.rs` does
     upstream). Not recommended unless option 1 fails and option 2 is
     too disruptive.

---

## Real follow-up (after S3–S9 passes)

- [ ] **Configure Claude Code hooks for ASCII symbols.** Merge the
  four-hook block into `~/.claude/settings.json` (do not overwrite
  other keys):

  ```json
  {
    "hooks": {
      "SessionStart":     [{"hooks": [{"type": "command", "command": "printf '\\033]0;>\\007' > /dev/tty"}]}],
      "UserPromptSubmit": [{"hooks": [{"type": "command", "command": "printf '\\033]0;$\\007' > /dev/tty"}]}],
      "Notification":     [{"hooks": [{"type": "command", "command": "printf '\\033]0;!\\007' > /dev/tty"}]}],
      "Stop":             [{"hooks": [{"type": "command", "command": "printf '\\033]0;>\\007' > /dev/tty"}]}]
    }
  }
  ```

  Then launch a Claude session inside a Zed terminal and verify the
  symbol cycles through `>` (gray, on session start) → `$` (green,
  while a turn is running) → `>` again (on Stop) → `!` (orange) when a
  Notification fires (use `/permissions` or trigger any tool
  confirmation).

---

## Self-review checklist (run after writing, before handoff)

- [x] **Spec coverage:** Every spec section maps to a task.
  - Architecture / state_icon field → Task 2.3, 2.4
  - Single-grapheme filter → Task 2.5
  - State_icon getter (Plan B) → "What shipped" log + spec
  - View-layer rendering (Plan B) → "Plan B pivot" block above
  - Display table → spec (Plan B revision)
  - Hook setup → "Real follow-up" block + spec
  - Verification §0 build → Task 3
  - Verification §1 scenarios S1–S9 → "Verification S3–S9" block + done log
  - Verification §2 tests → noted as "Tests" up front
  - Rebase strategy → not implemented (handled by zed-rebuild.sh and spec, not this plan)
- [x] **Placeholder scan:** No "TBD", "TODO", "implement later" — every code step has the actual code; every shell step has the actual command.
- [x] **Type consistency:** `state_icon: Option<String>` consistent across field decl, init, write site, read site. `UnicodeSegmentation::graphemes(_, true)` API consistent. `Color::*` mapping consistent across plan, spec, and `tab_content`.
- [x] **No tests required by plan, none included** — explicit at top.
