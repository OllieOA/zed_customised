# Spec: OSC-driven state icons for Zed terminal tab labels

Date: 2026-05-01 (originated)
Revised: 2026-05-01 (v2 — icon-prepend in `Terminal::title`)
Revised: 2026-05-02 (Plan B — colored ASCII symbols rendered by View layer)
Repo: fork of `zed-industries/zed`
Status: implemented (commit `1ccbbc37fa`); supersedes both the v1
priority-ladder design (`c6295926e3`) and the v2 emoji-prepend design
(`4736638375`). The emoji approach failed because GPUI's Linux text
shaper (cosmic-text + fontdb) did not reliably resolve emoji fallback
glyphs in tab labels, so the icon was rendered invisibly. Plan B sidesteps
font-shaping entirely by using ASCII symbols and applying color in the
View layer via GPUI's `Color::*` enum.

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
single-grapheme ASCII symbol**, the data layer captures the symbol on a
private `state_icon` field, and the terminal-tab View layer renders it as
a colored `Label` to the left of the existing terminal icon. Zed does no
state detection itself; the shell-side hook config is the source of
truth. This keeps the tab label a passive renderer of an event stream
the user controls and keeps the patch small.

## Architecture

```
Claude Code event (UserPromptSubmit, Stop, Notification, …)
  │
  ▼
hook script: printf '\033]0;<symbol>\007' > /dev/tty
  │
  ▼
alacritty parses OSC → AlacTermEvent::Title(<symbol>)
  │
  ▼
Terminal::process_event:
  - sets self.breadcrumb_text  (existing behavior, drives the breadcrumb pill)
  - if title is exactly one grapheme cluster, sets self.state_icon  (NEW)
  │
  ▼
Terminal::title()         (UNCHANGED — pure data getter)
Terminal::state_icon()    (NEW — borrows state_icon as Option<&str>)
  │
  ▼
TerminalView::tab_content (in crates/terminal_view/src/terminal_view.rs):
  - if terminal is NOT a Zed task, reads state_icon (or "?" fallback)
  - maps the character to a GPUI Color via a small match
  - inserts a colored Label between the terminal Icon and the title text
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

Changes span three files across two crates:

- `crates/terminal/Cargo.toml` — one-line addition:
  `unicode-segmentation.workspace = true`. The crate is already declared
  in the workspace `Cargo.toml` (`unicode-segmentation = "1.10"`), so no
  version bump or new dep on the workspace.
- `crates/terminal/src/terminal.rs` — struct field, constructor sites,
  `process_event` Title arm, and a new `Terminal::state_icon` getter.
  **`Terminal::title` is unchanged from upstream.**
- `crates/terminal_view/src/terminal_view.rs` — inside `tab_content`,
  insert a colored `Label` element when `terminal.task().is_none()`.

The CLAUDE.md "one patch, one place" rule is honored in spirit — the
data-side change is one bounded edit in `terminal.rs`, and the View-side
change is a single small block in the existing `tab_content` impl. We do
NOT touch any other crate or any other function.

### Struct field (alongside `breadcrumb_text`)

Locate via `grep -n "pub breadcrumb_text" crates/terminal/src/terminal.rs`.

```rust
pub breadcrumb_text: String,
state_icon: Option<String>,    // NEW (private)
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

### Public getter (alongside `Terminal::title`)

Locate via `grep -n "fn title" crates/terminal/src/terminal.rs`. Add this
getter immediately after `Terminal::title`:

```rust
pub fn state_icon(&self) -> Option<&str> {
    self.state_icon.as_deref()
}
```

`Terminal::title` itself is **untouched** — it remains a pure data getter
that returns the F2 rename, the cwd-process fallback, or the task label
exactly as upstream does. All icon-rendering logic lives in the View.

### View-layer rendering (in `TerminalView::tab_content`)

Locate via `grep -n "fn tab_content\b" crates/terminal_view/src/terminal_view.rs`.

After the existing `let (icon, icon_color, rerun_button) = …` match
block, but before the `h_flex().gap_1()` element builder begins, insert:

```rust
let state_icon = terminal.task().is_none().then(|| {
    let c = terminal.state_icon().unwrap_or("?").to_string();
    let color = match c.as_str() {
        "!" => Color::Warning,
        "$" => Color::Info,
        ">" => Color::Hint,
        _   => Color::Muted,
    };
    (c, color)
});
```

Then in the element-builder chain, between the existing terminal-icon
child and the title-text child, insert:

```rust
.when_some(state_icon, |this, (c, color)| {
    this.child(Label::new(c).color(color))
})
```

Result: tabs with a `task` show the task label and status icon as
upstream. Tabs without a task get an extra colored `Label` to the left of
the title — `?` muted, `>` hint-colored, `$` info-colored, `!`
warning-colored.

### Display table

| Tab state | Tab content (left → right) |
|---|---|
| Zed task running | `▶` task icon · task label (no state symbol) |
| F2="build", `$` (running) hook fired | terminal icon · green `$` · `build` |
| F2="build", `>` (idle) hook fired | terminal icon · gray `>` · `build` |
| F2="build", `!` (permission) hook fired | terminal icon · orange `!` · `build` |
| F2="build", no hook fired yet | terminal icon · muted `?` · `build` |
| F2 unset, `>` (idle) hook fired | terminal icon · gray `>` · `~/dev/zed — bash` |
| F2 unset, no hook fired yet | terminal icon · muted `?` · `~/dev/zed — bash` |
| F2 unset, only bash PROMPT_COMMAND has emitted | terminal icon · muted `?` · `~/dev/zed — bash` (leak ignored) |

## Hook setup (user-side; not in repo)

Recommended `~/.claude/settings.json` for any project where this fork is
in use:

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

Symbols (each is a single ASCII character — one grapheme cluster, passes
the filter):

- `$` — running (`UserPromptSubmit`); rendered green via `Color::Info`
- `>` — idle (`Stop`, `SessionStart`); rendered gray via `Color::Hint`
- `!` — permission prompt (`Notification`); rendered orange via
  `Color::Warning`
- `?` — unknown (Zed-side default, no hook needed); rendered muted via
  `Color::Muted`

ASCII was chosen over emoji because GPUI's Linux text shaping
(cosmic-text + fontdb) does not reliably resolve emoji fallback fonts
inside tab labels — a previous v2 attempt at `❓ ⏳ 💤 ⚠️` rendered as
invisible glyphs. Color is applied at render time by the View, so ASCII
is no less informative.

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

Run patched Zed (`myzed .` from a fresh terminal — see CLAUDE.md for
wrapper details). Open the embedded terminal. For each scenario, verify
the tab content matches the description.

| Scenario | How to trigger | Expected tab content |
|---|---|---|
| S1: cold open, no hooks | Just open a terminal in patched Zed | terminal icon · muted `?` · `~/dev/zed — bash` (or similar cwd-process) |
| S2: bash leak only | Press Enter at the prompt a few times (PROMPT_COMMAND fires) | unchanged from S1 (state_icon stays `None`, symbol stays muted `?`) |
| S3: idle hook | `printf '\033]0;>\007' > /dev/tty` | terminal icon · gray `>` · `~/dev/zed — bash` |
| S4: running hook | `printf '\033]0;$\007' > /dev/tty` | terminal icon · green `$` · `~/dev/zed — bash` |
| S5: bash leak after hook | After S4, press Enter (bash PROMPT_COMMAND fires) | unchanged from S4 — green `$` STICKS |
| S6: F2 rename then hook | F2 → "build" → Enter, then `printf '\033]0;$\007' > /dev/tty` | terminal icon · green `$` · `build` |
| S7: hook then F2 rename | Run S4, then F2 → "build" → Enter | same as S6 — composition order doesn't matter |
| S8: title reset | After S4, `printf '\033]0;\007' > /dev/tty` | unchanged from S4 — green `$` preserved across explicit resets |
| S9: long process name | Let process name exceed 25 chars in fallback | per-component truncation; the colored symbol is a separate Label so it does not push the title text — tab strip clips the title gracefully |

S5 is the **load-bearing test**: it proves single-grapheme filtering is
working and that bash's `PROMPT_COMMAND` does not clobber the state. The
filter logic is unchanged between v2 and Plan B, so S5 also serves as
the regression check that the View-layer move did not perturb the data
layer.

### 2. Tests

`Terminal::title` is unchanged under Plan B, so no new data-layer tests
are needed. The View-layer addition is covered indirectly by the
existing `test_tab_content_uses_custom_title` in `terminal_view` — when
that test passes after the patch, `tab_content`'s element tree still
produces the expected text.

Per CLAUDE.md "no tests beyond what verifies the patch behavior" — do
not invent a test harness for the colored-Label rendering itself; the
visual smoketest scenarios cover it.

```sh
nix develop --accept-flake-config --command cargo test -p terminal --lib
nix develop --accept-flake-config --command cargo test -p terminal_view --lib
```

## Rebase-on-upstream strategy

The fork's value is recurring. Each upstream pull may move patch sites.

1. `git fetch upstream && git rebase upstream/main`.
2. If the rebase succeeds cleanly: re-run §1 smoketest scenarios S1–S5
   before trusting the build.
3. If conflicts land in `crates/terminal/src/terminal.rs`:
   - `grep -n "fn title\|fn state_icon" crates/terminal/src/terminal.rs`
     to relocate the title getter and the new state_icon getter.
   - `grep -n "AlacTermEvent::Title" crates/terminal/src/terminal.rs` to
     relocate the event handler.
   - `grep -n "breadcrumb_text:" crates/terminal/src/terminal.rs` to
     relocate the struct field and constructor sites.
   - Inspect: did upstream add their own state-icon mechanism, OSC
     handling, or change the `Title` event shape? **Do not auto-resolve.**
     Read the upstream commit message and reassess fork necessity.
   - Otherwise re-apply the same shape: `state_icon: Option<String>`
     field + single-grapheme update in `Title` handler + `state_icon()`
     getter. **Do NOT compose the icon into `Terminal::title`'s return
     value** — that was the v2 mistake; rendering belongs in the View.
4. If conflicts land in `crates/terminal_view/src/terminal_view.rs`:
   - `grep -n "fn tab_content\b" crates/terminal_view/src/terminal_view.rs`
     to relocate the tab-content builder.
   - Re-insert the `state_icon` `let` binding before the element-builder
     chain and the `.when_some(state_icon, …)` child between the
     terminal-icon child and the title-text child.
5. Always rebuild and visual-test after rebasing — never trust the diff
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
- **Configurable color mapping via Zed settings.** The four color
  assignments (`Color::Muted`/`Hint`/`Info`/`Warning`) are hardcoded in
  `TerminalView::tab_content`. A future patch could expose them via
  `terminal.toolbar` or similar settings, but that's separate work.
  (Note: this item used to read "color comes for free via system emoji
  font" back when the patch was emoji-based — Plan B made colored
  rendering an explicit, in-scope mechanism, so the prior out-of-scope
  carve-out is obsolete.)
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
