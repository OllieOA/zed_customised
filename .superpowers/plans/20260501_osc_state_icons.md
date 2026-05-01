# OSC-driven state icons — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace v1 OSC passthrough patch (commit `c6295926e3`) with a Claude-hooks-driven state icon system: each terminal tab renders `<state-icon> <name>`, where the icon comes from single-grapheme OSC titles emitted by Claude Code hooks and the name comes from F2 rename or cwd-process fallback.

**Architecture:** New private `state_icon: Option<String>` field on `Terminal`, updated only when `AlacTermEvent::Title` carries exactly one grapheme cluster (filters out bash `PROMPT_COMMAND` redraws). `Terminal::title()` always composes `<icon> <name>`, with `❓` as the fallback when no icon has been received. Existing `breadcrumb_text` field is left alone — it powers a separate UI surface in `terminal_view`.

**Tech Stack:** Rust, GPUI, alacritty_terminal, unicode-segmentation 1.10 (already a workspace dep).

**Spec:** [`.superpowers/specs/20260501_osc_title_patch.md`](../specs/20260501_osc_title_patch.md) (commit `a1a24792bc`).

**Branch:** `osc-title-patch` (do not push without asking; see CLAUDE.md).

**No unit tests:** The terminal crate has no existing tests for `Terminal::title`, and per CLAUDE.md "no tests beyond what verifies the patch behavior" plus the spec's "do not invent a test harness for one function," verification is via the smoketest scenarios in Task 6, not Rust tests.

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

## Task 6: Smoketest scenarios S1–S9

These are the verification gate. The patch is **not** complete until all nine scenarios behave as specified. S5 is load-bearing (proves single-grapheme filtering); the rest cover the priority and composition rules.

This task involves user interaction — a Zed window, F2 renames, manual printf into the embedded terminal. As an agent you cannot perform these actions yourself. Surface the scenario, ask the user to run it, and record the result.

**Files:**
- Modify: `.tmp/smoketests/osc-title.md` (record v2 results)

- [ ] **Step 6.1: Launch the patched binary**

Ask the user to launch from a fresh terminal (so old env doesn't pollute):

```sh
zed-patched /home/ollie/dev/zed_customised
```

- [ ] **Step 6.2: Confirm version**

Ask the user to open `Zed: about` from the command palette and confirm the commit hash matches `git rev-parse HEAD` on `osc-title-patch`. If it doesn't, auto-update silently slipped through — see CLAUDE.md "Auto-update — DISABLE BEFORE RUNNING."

- [ ] **Step 6.3: Run scenario S1 — cold open, no hooks**

Just open a terminal in the patched Zed (via the "terminal: new" command from the command palette, or the keyboard shortcut bound to it).

Expected tab label: `❓ <something> — bash` (e.g. `❓ zed_customised — bash`).

- [ ] **Step 6.4: Run scenario S2 — bash leak only**

In the embedded terminal, press Enter a few times.

Expected: tab label stays `❓ …` exactly as in S1. (The bash `PROMPT_COMMAND` OSC fires but is multi-grapheme so `state_icon` stays `None`.)

- [ ] **Step 6.5: Run scenario S3 — idle hook**

```sh
printf '\033]0;💤\007' > /dev/tty
```

Expected: tab label becomes `💤 <cwd> — bash`. Color emoji should render colored (system Noto Color Emoji, independent of Nerd Font).

- [ ] **Step 6.6: Run scenario S4 — running hook**

```sh
printf '\033]0;⏳\007' > /dev/tty
```

Expected: tab label becomes `⏳ <cwd> — bash`.

- [ ] **Step 6.7: Run scenario S5 — bash leak after hook (LOAD-BEARING)**

After S4, press Enter at the prompt one or more times.

Expected: tab label **stays** `⏳ <cwd> — bash`. The bash `PROMPT_COMMAND` redraw sets `breadcrumb_text` but does NOT clobber `state_icon` because its title is multi-grapheme. **If the icon flips back to `❓` here, the single-grapheme filter is broken — re-read Step 2.5.**

- [ ] **Step 6.8: Run scenario S6 — F2 rename then hook**

Right-click the tab → Rename → type `build` → Enter. Then:

```sh
printf '\033]0;⏳\007' > /dev/tty
```

Expected: `⏳ build`. Both icon and F2 name compose; neither overrides the other.

- [ ] **Step 6.9: Run scenario S7 — hook then F2 rename**

In a different tab (or after clearing): run S4 first (`printf '\033]0;⏳\007' > /dev/tty`), then F2 → `build` → Enter.

Expected: `⏳ build`. Same composition, opposite order.

- [ ] **Step 6.10: Run scenario S8 — title reset preserves icon**

After S4, run:

```sh
printf '\033]0;\007' > /dev/tty
```

Expected: tab label stays `⏳ <cwd> — bash`. The empty title resets `breadcrumb_text` but `state_icon` is deliberately preserved.

- [ ] **Step 6.11: Run scenario S9 — long process name truncation**

Run a long-named command in the foreground (e.g. `cargo run --release --package some-very-long-name`), or rely on a deeply nested cwd. Confirm the tab strip clips gracefully — the icon prefix shouldn't push the rest off-screen catastrophically.

Expected: tab label fits the tab strip. Per-component truncation in the cwd-process fallback (25 chars each) preserved from upstream.

- [ ] **Step 6.12: Update the smoketest log**

Append v2 results to `.tmp/smoketests/osc-title.md` so the file matches the implemented behavior. Note any scenario that needed re-investigation. (The `.tmp/` directory is excluded via `.git/info/exclude` and is not committed — this is a local log, not part of the patch.)

---

## Task 7: Optional final polish

These steps are **not required** for the patch to ship locally but are nice-to-have before a future upstream attempt or a fresh-machine reinstall.

- [ ] **Step 7.1: Configure Claude Code hooks (user choice)**

The spec's recommended `~/.claude/settings.json` snippet emits the four state icons via Claude hooks. This is out-of-repo configuration — the user can apply it whenever convenient. Surface the snippet (it's in the spec under "Hook setup") and ask if they want help wiring it.

- [ ] **Step 7.2: Decide commit history shape**

The branch will be:

```
HEAD  Task 2's commit  ── new state-icon implementation
      Task 1's commit  ── unicode-segmentation dep
      a1a24792bc       ── revised spec
      c6295926e3       ── v1 OSC passthrough (now superseded)
      fd42e968ca       ── fork rules + original spec
      upstream/main    ── …
```

Two reasonable end states before any push:

- **Preserve evolution:** keep all five fork commits as-is. Cleaner reasoning trail, noisier diff.
- **Squash to two commits:** spec + implementation. Cleaner diff for upstream, loses v1→v2 evolution. Use `git rebase -i fd42e968ca` to squash if you go this route — but per CLAUDE.md "do not use --no-edit with git rebase commands" and "git rebase -i requires interactive input" — the user must drive the rebase, not the agent.

Surface the choice; do not rebase autonomously.

---

## Self-review checklist (run after writing, before handoff)

- [x] **Spec coverage:** Every spec section maps to a task.
  - Architecture / state_icon field → Task 2.3, 2.4
  - Single-grapheme filter → Task 2.5
  - title() composition → Task 2.6
  - Display table → Task 6.3–6.11
  - Hook setup → Task 7.1 (out of repo, just documented)
  - Verification §0 build → Task 3
  - Verification §1 scenarios S1–S9 → Task 6.3–6.11
  - Verification §2 tests → noted as "no unit tests" up front
  - Rebase strategy → not implemented (handled by zed-rebuild.sh and spec, not this plan)
- [x] **Placeholder scan:** No "TBD", "TODO", "implement later" — every code step has the actual code; every shell step has the actual command.
- [x] **Type consistency:** `state_icon: Option<String>` consistent across field decl, init, write site, read site. `UNKNOWN_ICON: &str` consistent. `UnicodeSegmentation::graphemes(_, true)` API consistent.
- [x] **No tests required by plan, none included** — explicit at top.
