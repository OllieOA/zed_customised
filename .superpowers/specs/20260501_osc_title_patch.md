# Spec: OSC-driven state icons for Zed terminal tab labels

Date: 2026-05-01 (originated)
Revised: 2026-05-01 (v2 — icon-prepend design)
Repo: fork of `zed-industries/zed`
Status: spec — revising; supersedes the priority-ladder design in commit
`c6295926e3`. Implementation plan to follow via writing-plans skill.

## Problem

Zed's terminal tab labels show a process-derived string like
`~/dev/zed — bash` and ignore any title the running shell emits. We want
each terminal tab to advertise — at a glance — what state the embedded
Claude Code session is in:

- **Running** (Claude is processing a prompt)
- **Idle** (Claude has stopped and is awaiting input)
- **Permission prompt** (Claude is waiting on a `[y/N]`-style confirmation)
- **Unknown** (no Claude state has been emitted to this terminal yet)

Plus we want each tab to be **nameable** — typing F2 → "build server" gives
the tab a stable identity independent of the state icon.

The mechanism is **Claude Code hooks emit OSC 0/2 escape sequences with a
single-grapheme glyph**, and Zed renders `<glyph> <name>` in the tab label.
Zed does no state detection itself; the shell-side hook config is the
source of truth. This keeps the tab label a passive renderer of an event
stream the user controls and keeps the patch small.

## Architecture

```
Claude Code event (UserPromptSubmit, Stop, Notification, …)
  │
  ▼
hook script: printf '\033]0;<glyph>\007' > /dev/tty
  │
  ▼
alacritty parses OSC → AlacTermEvent::Title(<glyph>)
  │
  ▼
Terminal::process_event:
  - sets self.breadcrumb_text  (existing behavior, drives the breadcrumb pill)
  - if title is exactly one grapheme cluster, sets self.state_icon  (NEW)
  │
  ▼
Terminal::title() composes "<state_icon-or-❓> <title_override-or-cwd_process>"
  │
  ▼
tab strip renders the composed label
```

### State / name decoupling

- **State** lives in `state_icon: Option<String>` (new field).
  - `None` initially.
  - Updated **only** when an OSC title arrives that is a single grapheme
    cluster — this is what filters out bash's `PROMPT_COMMAND` redraws
    that emit `user@host:cwd` and would otherwise clobber the icon.
  - Persists across `AlacTermEvent::ResetTitle` (Claude hooks remain
    authoritative; bare `\033]0;\007` does not reset state).
- **Name** lives in `title_override` (existing field, F2 rename) with
  cwd-process fallback (existing behavior).

### Why "single grapheme" as the discriminator

Bash's default `PROMPT_COMMAND` on most distros emits an OSC 0 every
prompt redraw (e.g. `\033]0;ollie@nobara:~\007`). Without a filter,
Claude's `▶` icon would last milliseconds before being overwritten back
to `ollie@nobara:~`. Single-grapheme detection makes the patch indifferent
to the user's shell prompt config — only intentional icon emissions
update the state.

`unicode-segmentation::UnicodeSegmentation::graphemes(true)` is the
correct API: it counts extended grapheme clusters, so `⚠️` (U+26A0 +
U+FE0F) registers as one grapheme even though it's two codepoints. Same
for skin-tone-modified emoji and other ZWJ sequences.

## Patch shape

### File scope

Changes confined to two files:

- `crates/terminal/src/terminal.rs` — struct field, constructor sites,
  `process_event` Title/ResetTitle arms, and `Terminal::title`.
- `crates/terminal/Cargo.toml` — one-line addition:
  `unicode-segmentation.workspace = true`. The crate is already declared
  in the workspace `Cargo.toml` (`unicode-segmentation = "1.10"`), so no
  version bump or new dep on the workspace.

The CLAUDE.md "one patch, one place" rule is honored — both files belong
to the same crate, and the second is just wiring an existing workspace
dep into a sibling crate that didn't previously need it.

### Struct field (alongside `breadcrumb_text`)

Locate via `grep -n "pub breadcrumb_text" crates/terminal/src/terminal.rs`.

```rust
pub breadcrumb_text: String,
state_icon: Option<String>,    // NEW
title_override: Option<String>,
```

Initialize as `None` in both `Terminal::new` constructor sites. Locate via
`grep -n "breadcrumb_text: String::new()" crates/terminal/src/terminal.rs`.

### Title event handler (in `process_event`)

Locate via `grep -n "AlacTermEvent::Title" crates/terminal/src/terminal.rs`.

```rust
AlacTermEvent::Title(title) => {
    // Existing Windows-only guard against shell-program-as-title noise.
    #[cfg(windows)] {
        if self.shell_program.as_ref().map(|e| *e == title).unwrap_or(false) {
            return;
        }
    }

    // Single-grapheme titles are treated as state icon updates.
    // Multi-grapheme titles (e.g. bash PROMPT_COMMAND's "user@host:cwd")
    // are NOT considered icons — state_icon is preserved untouched.
    if UnicodeSegmentation::graphemes(title.as_str(), true).count() == 1 {
        self.state_icon = Some(title.clone());
    }

    self.breadcrumb_text = title;
    cx.emit(Event::BreadcrumbsChanged);
}

AlacTermEvent::ResetTitle => {
    self.breadcrumb_text = String::new();
    // state_icon deliberately preserved across explicit title resets.
    cx.emit(Event::BreadcrumbsChanged);
}
```

### `Terminal::title` rewrite

Locate via `grep -n "fn title" crates/terminal/src/terminal.rs`. The
existing v1 `or_else` block (committed in `c6295926e3`) is removed; the
`None` arm becomes:

```rust
None => {
    const UNKNOWN_ICON: &str = "❓";
    let icon = self.state_icon.as_deref().unwrap_or(UNKNOWN_ICON);
    let name = self.title_override
        .as_ref()
        .map(|t| t.to_string())
        .unwrap_or_else(|| match &self.terminal_type {
            // ...existing cwd-process fallback unchanged...
        });
    format!("{icon} {name}")
}
```

Truncation behavior preserved per branch:
- `title_override`: not truncated (unchanged from existing behavior).
- cwd-process fallback: per-component `truncate_and_trailoff(_, MAX_CHARS)`
  (unchanged from existing behavior).
- The icon adds 2 chars (`<grapheme><space>`) to the visible width. We do
  not apply any further composed-string truncation; long F2 names will
  still be truncated by the tab strip's own layout pass as today.

The `Some(task_state)` arm is **untouched**. Zed-task-running tabs show
their task label with no icon prefix.

### Display table

| Tab state | Display |
|---|---|
| Zed task running | task label (no icon) |
| F2="build", running hook fired | `⏳ build` |
| F2="build", idle hook fired | `💤 build` |
| F2="build", permission hook fired | `⚠️ build` |
| F2="build", no hook fired yet | `❓ build` |
| F2 unset, idle hook fired | `💤 ~/dev/zed — bash` |
| F2 unset, no hook fired yet | `❓ ~/dev/zed — bash` |
| F2 unset, only bash PROMPT_COMMAND has emitted | `❓ ~/dev/zed — bash` (multi-grapheme leak ignored) |

## Hook setup (user-side; not in repo)

Recommended `~/.claude/settings.json` for any project where this fork is
in use:

```json
{
  "hooks": {
    "SessionStart":     [{"hooks": [{"type": "command", "command": "printf '\\033]0;💤\\007' > /dev/tty"}]}],
    "UserPromptSubmit": [{"hooks": [{"type": "command", "command": "printf '\\033]0;⏳\\007' > /dev/tty"}]}],
    "Notification":     [{"hooks": [{"type": "command", "command": "printf '\\033]0;⚠️\\007' > /dev/tty"}]}],
    "Stop":             [{"hooks": [{"type": "command", "command": "printf '\\033]0;💤\\007' > /dev/tty"}]}]
  }
}
```

Glyphs:
- ⏳ U+23F3 — running (`UserPromptSubmit`)
- 💤 U+1F4A4 — idle (`Stop`, `SessionStart`)
- ⚠️ U+26A0 + U+FE0F — permission prompt (`Notification`); the U+FE0F
  variation selector is required for color rendering on Linux.
- ❓ U+2753 — unknown (Zed-side default, no hook needed)

Hook commands write to `/dev/tty` because Claude Code hook scripts do not
have stdout connected to the controlling terminal by default.

## Verification

### 0. Build

```sh
bash .tmp/scripts/incremental-build.sh
```

Watch for new warnings in `crates/terminal`. Run `./script/clippy` if any
appear in the patched function.

### 1. Smoketest scenarios

Run patched Zed (`zed-patched .` from a fresh terminal — see CLAUDE.md
for wrapper details). Open the embedded terminal. For each scenario,
verify the tab label matches the display table above.

| Scenario | How to trigger | Expected label |
|---|---|---|
| S1: cold open, no hooks | Just open a terminal in patched Zed | `❓ ~/dev/zed — bash` (or similar cwd-process) |
| S2: bash leak only | Press Enter at the prompt a few times (PROMPT_COMMAND fires) | `❓ ~/dev/zed — bash` (state_icon stays `None`) |
| S3: idle hook | `printf '\033]0;💤\007' > /dev/tty` | `💤 ~/dev/zed — bash` |
| S4: running hook | `printf '\033]0;⏳\007' > /dev/tty` | `⏳ ~/dev/zed — bash` |
| S5: bash leak after hook | After S4, press Enter (bash PROMPT_COMMAND fires) | `⏳ ~/dev/zed — bash` (icon STICKS) |
| S6: F2 rename then hook | F2 → "build" → Enter, then `printf '\033]0;⏳\007' > /dev/tty` | `⏳ build` |
| S7: hook then F2 rename | Run S4, then F2 → "build" → Enter | `⏳ build` |
| S8: title reset | After S4, `printf '\033]0;\007' > /dev/tty` | `⏳ ~/dev/zed — bash` (state_icon preserved) |
| S9: long process name | Let process name exceed 25 chars in fallback | per-component truncation, prepended icon — fits the tab strip |

S5 is the **load-bearing test**: it proves single-grapheme filtering is
working and that bash's `PROMPT_COMMAND` does not clobber the state.

### 2. Tests

If `crates/terminal/src/terminal.rs` already has unit tests covering
`Terminal::title`, extend with cases mirroring S1, S2, S3, S6, S7, S8.
Per CLAUDE.md "no tests beyond what verifies the patch behavior" — do
not invent a test harness for one function.

```sh
cargo test -p terminal
```

## Rebase-on-upstream strategy

The fork's value is recurring. Each upstream pull may move patch sites.

1. `git fetch upstream && git rebase upstream/main`.
2. If the rebase succeeds cleanly: re-run §1 smoketest scenarios S1–S5
   before trusting the build.
3. If conflicts land in `crates/terminal/src/terminal.rs`:
   - `grep -n "fn title" crates/terminal/src/terminal.rs` to relocate.
   - `grep -n "AlacTermEvent::Title" crates/terminal/src/terminal.rs` to
     relocate the event handler.
   - `grep -n "breadcrumb_text:" crates/terminal/src/terminal.rs` to
     relocate the struct field and constructor sites.
   - Inspect: did upstream add their own state-icon mechanism, OSC
     handling, or change the `Title` event shape? **Do not auto-resolve.**
     Read the upstream commit message and reassess fork necessity.
   - Otherwise re-apply the same shape: `state_icon` field +
     single-grapheme update in `Title` handler + `Terminal::title`
     `<icon> <name>` composition.
4. Always rebuild and visual-test after rebasing — never trust the diff
   alone.

If upstream merges an equivalent feature, retire the fork: drop the
patch, switch the local branch to track upstream directly, and remove
this spec.

## Out of scope

- **Configurable icons / colors via Zed settings.** Glyphs are entirely
  user-controlled via the hook command; the unknown-icon constant is
  hardcoded.
- **State detection in Zed.** Zed does no process tracking, no output
  scanning, no inference. Shell hooks are the only state source.
- **Colored portions of the tab label.** Color comes for free via the
  system's color-emoji font. A future patch could add hardcoded color
  per icon glyph by splitting icon-from-name in `terminal_view`'s tab
  item rendering — that's a separate piece of work.
- **OSC 7 (working-directory hint).** Unrelated to this patch.
- **Per-shell emission of state icons outside Claude.** Users wanting an
  always-on shell idle indicator can add their own OSC emission in their
  shell rc; the patch logic doesn't care where the OSC originates.
- **The Zed task arm of `Terminal::title`.** Task tabs continue to
  display the task label unmodified.
- **The `breadcrumb_text` field's other consumers** (the breadcrumb pill
  in `crates/terminal_view/src/terminal_view.rs`, located via
  `grep -n "breadcrumb_text" crates/terminal_view/src/terminal_view.rs`).
  Untouched.
