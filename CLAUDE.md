# Fork-specific instructions (READ FIRST)

This clone is a **fork-of-one** of `zed-industries/zed`. Its sole purpose is a
~2-line patch that makes terminal tab labels reflect OSC 0/1/2 escape
sequences emitted by the running shell.

> **In upstream Zed, `CLAUDE.md` is a symlink to `.rules`.** This fork
> promotes it to a real file so fork-local rules can live here without
> rewriting the upstream-tracked `.rules` (which would conflict on every
> rebase). For Zed's canonical agent guidance — Rust style, GPUI
> conventions, PR hygiene, build commands — **read `.rules`** in this same
> directory. It is upstream-pristine and authoritative; nothing in it is
> duplicated below.

## Scope of this fork

- One patch, one place. Do not refactor adjacent code, rename fields, add
  tests beyond what verifies the patch behavior, or introduce new modules.
- If you find yourself touching anything outside
  `crates/terminal/src/terminal.rs` (or its sibling tests), stop and
  reconfirm with the user.
- The full problem statement, patch shape, and verification plan live in
  `.superpowers/specs/20260501_osc_title_patch.md`. Read it before patching.

## Patch site

- File: `crates/terminal/src/terminal.rs`
- Function: `Terminal::title(&self, truncate: bool) -> String`
- Field already populated by `AlacTermEvent::Title` handling:
  `Terminal::breadcrumb_text`
- Priority ladder, after the patch, must be:
  1. `task` label (when running a Zed task) — unchanged
  2. `title_override` (manual F2 rename) — unchanged
  3. **`breadcrumb_text` if non-empty (NEW — OSC 0/1/2 from the shell)**
  4. PTY-derived `cwd — process` fallback — unchanged
- Do not record line numbers in this file or in the spec; they drift on
  rebase. Use `grep -n "fn title" crates/terminal/src/terminal.rs` to
  relocate the function each session.

## Build commands (Fedora 43 / Nobara)

The Rust toolchain is **not** installed yet. Bootstrap once via
`rustup` per `docs/src/development/linux.md` before building.

- Debug build / run:        `cargo run`
- Release build:            `cargo build --release`  (~10–18 min cold first build)
- Workspace tests:          `cargo test --workspace`
- Lint (project standard):  `./script/clippy`  (use this, **not** `cargo clippy`)
- Install local build:      `./script/install-linux`

System dependencies are listed in `script/linux`. On this host the
following are missing and must be installed before a release build will
link: `clang`, `cmake`, `g++`, `alsa-lib-devel`, `wayland-devel`,
`libxkbcommon-x11-devel`, `libva-devel`, `sqlite-devel`, `musl-gcc`. The
optional faster linker `mold` is also absent. **Do not invoke `sudo` from
an agent session** — surface the install command to the user and let them
run it.

## Fork conventions

- **Scratch goes in `/.tmp/`** (excluded via `.git/info/exclude`, **not**
  the tracked `.gitignore`). Build logs, captured terminal output,
  screenshots, throwaway notes — all live under `.tmp/`. The tracked
  `.gitignore` is upstream's; never edit it for fork-local concerns.
- **No `&&` compounds in shell.** Multi-step shell work goes into a script
  under `.tmp/scripts/<name>.sh` and is invoked as one command. Single
  commands at the prompt are fine; chained ones are not.
- **Navigation: prefer `grep -n` / ripgrep over reading whole files.** Land
  on exact line numbers, then `Read` a tight window around them. The
  codebase is large; broad reads burn context.
- **Plans live in `.superpowers/plans/`, specs in `.superpowers/specs/`.**
  Both are tracked and commit alongside the patch so rationale travels
  with the code.
- **Never push to `origin` without an explicit ask.** This is a fork; the
  remote is on GitHub and pushes are visible. Ask before every push.

## Auto-update — DISABLE BEFORE RUNNING

Zed auto-updates by default. A successful update **silently replaces our
patched binary with the upstream release**, undoing the work without any
visible signal. Before running the patched build for any meaningful
length of time:

1. Open Zed settings (`zed: open settings`).
2. Set `"auto_update": false`.
3. Optionally set the env var
   `ZED_UPDATE_EXPLANATION="Patched fork — do not auto-update"` so manual
   update attempts surface a clear message instead of silently failing.

After launch, run `zed: about` and confirm the commit hash matches the
one you patched. If it has shifted, an update slipped through.

## Rebase strategy (one-line)

The fork rebases onto `upstream/main`. The patch is intentionally tiny
(one function body) so conflicts should be rare, but the function may
move within the file. Full rebase playbook is in
`.superpowers/specs/20260501_osc_title_patch.md` under
"Rebuild-on-upstream-rebase strategy". Always rebuild and visual-test
after rebasing — never trust the diff alone.
