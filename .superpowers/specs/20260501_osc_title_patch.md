# Spec: OSC 0/1/2 escape sequences set Zed terminal tab title

Date: 2026-05-01
Repo: fork of `zed-industries/zed`
Status: spec only — no code changed yet

## Problem

When a shell or TUI program emits an OSC 0/1/2 escape sequence (the standard
"set terminal title" mechanism, e.g. `printf '\033]0;mytitle\007'`), most
terminals — and Zed's tooltips — update accordingly. Zed's **terminal tab
labels** do not: they show a process-derived string like `home — bash` instead
of the title the shell asked for.

The plumbing already exists. `crates/terminal/src/terminal.rs` parses the
sequence in `Terminal::process_event` (the `AlacTermEvent::Title` arm,
~line 939) and stores it in the `Terminal::breadcrumb_text` field
(declared ~line 865). It is consumed by the breadcrumbs UI but not by the
tab label code path.

The tab label is computed by `Terminal::title(&self, truncate: bool)`
(~line 2155), which today consults exactly two sources:

1. `self.task` — Zed task label, when this terminal hosts a `cargo test`
   etc. (highest priority).
2. `self.title_override` — what the user typed when they hit F2 to rename
   the tab manually (second priority).
3. Otherwise: a PTY-derived string formatted from `cwd` and the running
   process's argv.

OSC titles never enter this ladder, so any shell-emitted title is invisible
in the tab strip.

## The patch shape

Insert `breadcrumb_text` between `title_override` and the PTY fallback:

```text
task            (unchanged)
  ↓ none
title_override  (unchanged — F2 manual rename still wins)
  ↓ none
breadcrumb_text (NEW — non-empty OSC title, optionally truncated)
  ↓ empty
PTY-derived     (unchanged fallback)
```

Concretely, in `Terminal::title`, the `None` arm of the `match &self.task`
becomes:

```rust
None => {
    if let Some(title_override) = &self.title_override {
        return title_override.to_string();
    }
    if !self.breadcrumb_text.is_empty() {
        return if truncate {
            truncate_and_trailoff(&self.breadcrumb_text, MAX_CHARS)
        } else {
            self.breadcrumb_text.clone()
        };
    }
    // ...existing PTY-derived fallback unchanged...
}
```

(Exact form may differ depending on whether the existing code uses
`unwrap_or_else` chaining — keep the change minimal and idiomatic to the
surrounding style. The semantic ordering above is what matters.)

### Why this ordering

- `title_override` is an *explicit user action* (F2 rename) and must keep
  beating an automatic shell-emitted title — otherwise the rename is
  immediately clobbered by the next prompt redraw.
- `breadcrumb_text` is *more specific* than the PTY-derived fallback: the
  shell knows what it's doing (e.g. emitting `ssh user@host` while running
  ssh) better than process-name inference does.
- The fallback is unchanged, so terminals that don't emit OSC titles look
  identical to today.

### Truncation

The existing path applies `truncate_and_trailoff(..., MAX_CHARS=25)` to the
PTY-derived strings when `truncate` is true. The new branch must do the same
to `breadcrumb_text` for visual consistency in narrow tab strips.

## Verification

### 0. Establish baseline

Before patching, build and run unmodified `main`:

```sh
cargo build --release
./target/release/zed   # or script/install-linux
```

In the embedded terminal, run:

```sh
printf '\033]0;baseline-title\007'
```

Expected: tab label is **unchanged** (still shows `cwd — bash` or similar).
This confirms the bug exists on this commit.

### 1. Build the patched binary

After applying the patch:

```sh
cargo build --release
```

Watch for: zero new warnings in `crates/terminal`. If `cargo clippy` (run via
`./script/clippy`) flags anything in the patched function, fix it.

### 2. Visual smoke test

Run the patched Zed. In the embedded terminal:

```sh
printf '\033]0;HELLO\007'
```

Expected: tab label becomes `HELLO`.

```sh
printf '\033]0;\007'   # empty OSC title resets
```

Expected: tab label falls back to PTY-derived (`cwd — bash`).

Right-click the tab → Rename → type `manual` → Enter.
Expected: label becomes `manual`. Now run the OSC printf again — label
**stays** `manual` (the override wins). This proves priority is correct.

Run a Zed task (e.g. `task: spawn` something simple). The task label must
still win over both override and OSC.

### 3. Tests

If `crates/terminal/src/terminal.rs` already has unit tests covering
`Terminal::title`, extend them with the four cases above (PTY-only,
breadcrumb-only, override-beats-breadcrumb, task-beats-everything). If it
does not, do not invent a test harness for one function — flag this in the
PR description and rely on the visual check.

```sh
cargo test -p terminal
```

## Rebuild-on-upstream-rebase strategy

The fork's value is recurring. Each upstream pull may move the patch site.

1. `git fetch upstream && git rebase upstream/main`.
2. If the rebase succeeds cleanly: re-run §2 visual smoke test before
   trusting the build.
3. If conflicts land in `crates/terminal/src/terminal.rs`:
   - `grep -n "fn title" crates/terminal/src/terminal.rs` to find the
     function's new location.
   - Inspect the priority ladder — if upstream changed it (e.g. added their
     own OSC handling, reordered cases, replaced `title_override` with a
     different mechanism), **do not auto-resolve**. Read the upstream
     commit message and reassess whether this fork is still needed.
   - Otherwise re-apply the same shape: insert the `breadcrumb_text` check
     between `title_override` and the PTY fallback.
4. Always rebuild and visual-test after rebasing — never trust the diff
   alone.

If upstream merges an equivalent change (track:
[zed PR search for "OSC title"](https://github.com/zed-industries/zed/pulls?q=OSC+title)),
the fork is retired: drop the patch, switch the local branch to track
upstream directly, and remove this spec.

## Out of scope

- OSC 7 (working-directory hint) wiring into the tab label.
- Configurable maximum length / formatting of the OSC-derived title.
- Per-shell emission of OSC titles (that's a user dotfile concern, e.g.
  bash `PROMPT_COMMAND` setting `\033]0;...\007`).
- Any change to the **breadcrumbs** UI — `breadcrumb_text` is already
  consumed there and we do not want to change that surface.
