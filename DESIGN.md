# Ripple — a calm terminal git client

A personal, keyboard-driven TUI git client written in Rust.

- **Date:** 2026-06-09
- **Status:** Approved design, pre-implementation.

## Goal

A terminal UI for git that is **calm and focused** — the opposite of a dense cockpit. It covers the full git workflow (commit loop, history, branches, stash, remotes, interactive rebase, conflict resolution) for personal daily use.

## Why (motivating constraints)

Built in reaction to lazygit's UI. The specific objections that shaped this design:

- Too much on screen at once (five panels + diff + hints feels cramped).
- The numbered-panel navigation model (hopping between panels 1–5).
- Borders, hints, and chrome everywhere — visually noisy.
- The color/density aesthetic.

Keybinding **discoverability was not** a complaint — a keyboard-driven, modal tool is acceptable.

## Guiding principle

**Every region on screen must earn its place.** The objection to lazygit was *unjustified* panels, not panel count. Density is welcome where each region does a distinct job; decorative or redundant regions are rejected. Multiple screens are fine when each contributes something.

## Scope (v1 target)

Full git workflow:

- **Core commit loop:** stage/unstage (file *and* hunk), commit, push, pull.
- **Branches & history:** switch/create/delete branches, browse the log, view diffs.
- **Stash:** save / apply / pop / drop.
- **Remotes & sync:** manage remotes, fetch, upstream tracking.
- **Interactive rebase.**
- **Conflict resolution** (3-way).

**Audience:** just the author. No theming, config files, or cross-user edge cases in v1.

## Tech stack

- **Language:** Rust.
- **TUI:** ratatui.
- **Git access:** hybrid — `git2` (libgit2) for structured reads and most writes; shell out to the `git` binary for interactive rebase and operations libgit2 handles poorly. Both sit behind a single `Git` trait.
- **Distribution:** single static binary.

## Architecture — The Elm Architecture (TEA)

- **`Model`** — all app state in one place: active screen, per-screen selection/scroll, a cached repo snapshot, and in-flight async status. Single source of truth.
- **`Msg`** — every event as data: a keypress, "git result arrived," a tick.
- **`update(&mut Model, Msg)`** — the only place state changes; no blocking I/O (it may spawn async tasks). Pure-ish, so it is unit-testable without a terminal.
- **`view(&Model, Frame)`** — pure rendering of the active screen; no logic.

The UI is a pure function of `Model` — predictable and calm.

### Git access layer

A single `Git` trait abstracts both paths (git2 and shell-out). Screens call `git.status()`, `git.commit()`, etc., and never know which path ran. The trait is also the seam faked in tests.

### Async

Network operations (fetch/push/pull) run on a background task; results return as `Msg`s. The footer shows a spinner while a task is in flight. The UI never blocks.

## Screens

Most screens reuse a two-pane **master–detail** shape; two are bespoke because master–detail is the wrong shape for them.

| Screen | Layout | Job |
|--------|--------|-----|
| **Status** (home) | master–detail: files │ diff | stage/unstage (file + hunk), commit, push, pull |
| **Log** | master–detail: commits │ commit detail + diff | browse history; checkout, revert, cherry-pick, branch-from-here |
| **Branches** | master–detail: branches │ branch detail | checkout, create, delete, merge, set upstream |
| **Stash** | master–detail: stashes │ stash diff | apply, pop, drop, save |
| **Rebase** (interactive) | bespoke: editable todo list | reorder/squash/reword/drop a sequence; shells out to `git` |
| **Conflicts** | bespoke: 3-way (ours / base / theirs + result) | resolve merge conflicts with full base context |

### Persistent chrome (three always-on regions, each justified)

- **Header (1 line):** repo · branch · `↑ahead ↓behind` · in-progress state (e.g. `REBASING 2/5`).
- **Body:** the active screen.
- **Footer (1 line):** last action result / errors (red) / async spinner — the feedback channel.

No permanent keybinding cheatsheet. `?` shows a context-relevant key overlay on demand.

## Navigation

Two key namespaces, to avoid the collisions that numbered panels work around:

- **Between screens:** a `g`-leader (`gs` status, `gl` log, `gb` branches, `gz` stash) plus a `:` command prompt for anything by name (`:rebase`, `:fetch origin`).
- **Within a screen:** `j`/`k` move the list · `Tab` (or `h`/`l`) switches pane focus · `Enter` opens/acts · contextual action keys (`s` stage, `c` commit…) · `Esc` backs out · `?` help overlay.

Since this is a personal tool, keybindings are tunable.

## Data flow

key → `Msg` → `update` mutates `Model` (may spawn an async git task) → `view` re-renders. Async results return as `Msg`s. After any mutating operation, a `Refresh` re-reads the affected git state so a screen never drifts from reality. (Optional, post-v1: `notify` file-watch for external-change auto-refresh.)

## Conflict resolution (3-way)

A dedicated screen showing the merge **base** alongside **ours** and **theirs**, plus the **result** being assembled. Chosen over a lazygit-style inline 2-way pick because the base is information neither side shows alone — it earns the extra screen.

- **Data source:** during a conflict, git's index holds all three file stages (base/ours/theirs); git2 exposes them directly via conflict entries. Structured data, not parsed `<<<<<<<` markers.
- **Per conflict:** choose ours / theirs / both, or edit the result inline; advance to the next conflict.
- **Layout:** three columns when the terminal is wide; an automatic **stacked** fallback (ours over base over theirs) when narrow. One responsive layout.
- **Escape hatch:** `e` opens `$EDITOR`; external `git mergetool` remains available for anything exotic.

## Error handling

Git errors are **data, not crashes**.

- Every git op returns `Result`; a failure becomes an error `Msg` that paints the footer red. No panics.
- *Expected* errors (nothing to commit, push rejected, conflicts) → a calm footer line with the fix. *Unexpected* errors (git2 error, corrupt repo) → footer **plus** a `:messages` view that retains the full text.
- Async/network failures carry git's **stderr verbatim** to the footer, so the real reason is visible.

## Testing (pragmatic for a personal tool)

- **`update()` logic:** pure `(state, msg) → state` — unit-tested as plain function calls; this is the bulk of behavior.
- **Git module:** tested against temp real repos (`tempfile` + git2) — e.g. manufacture a conflict and assert the three stages read correctly. Real git for the git layer, not mocks.
- **Screens:** tested against a fake `Git` to isolate screen logic.
- **Rendering:** ratatui `TestBackend` snapshot tests for a couple of key screens (Status, the 3-way). Not exhaustive.
- **Skipped deliberately:** no PTY/e2e automation — daily use is the integration test.

## Build order (each phase independently usable)

1. **Spine + daily driver:** TEA loop, `Git` module skeleton, Status screen (stage/unstage file + hunk, commit, push, pull), header + footer chrome, `g`-leader + `:` navigation. *Usable daily after this.*
2. **History & branches:** Log + Branches screens, diff viewing, checkout/create/delete.
3. **Stash & remotes:** Stash screen, remote management, fetch/upstream.
4. **The hard ones (last on purpose):** interactive Rebase (shell-out) + the 3-way Conflicts screen.

## Out of scope (YAGNI)

- Theming, config files, plugin system.
- A broader external-mergetool framework beyond invoking `git mergetool`.
- PTY/e2e test harness.
- File-watch auto-refresh (optional, post-v1).
