# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`tmsnap` snapshots and restores a terminal workspace: tmux topology, the iTerm2 tabs attached to it, and the Claude Code / OpenCode sessions running in each pane. The whole tool is one executable Python script, `tmsnap` (no `.py` extension), standard library only, Python 3.9+. Keeping it dependency-free is a deliberate constraint; the README's roadmap notes that iTerm's Python API would be the first non-stdlib dependency, and that the AppleScript path stays the default.

## Commands

There is no build, lint config, or test suite. Tests are listed as a roadmap item; the restore paths have only been verified by hand.

```sh
./tmsnap --version
python3 -m py_compile tmsnap          # syntax check
./tmsnap save --no-iterm scratch      # tmux-only named snapshot (also rewrites latest.json)
./tmsnap show scratch
./tmsnap restore scratch --dry-run    # prints every tmux command / iTerm action, executes nothing
```

When experimenting, set `TMSNAP_DIR` to a scratch directory so the real snapshots in `~/.local/state/tmux-snapshot/` stay untouched. Don't run a real `restore` (especially with `--force`, which kills sessions, or `--attach`, which creates iTerm tabs) against the user's live tmux server unless they ask for it. Use `--dry-run`. The installed command is usually a symlink to this checkout, so edits take effect right away.

## Architecture

The script runs top to bottom in these layers:

1. **Process helpers**: `proc_table()` builds one `ps` snapshot (pid → info, ppid → children). `walk()` follows a pane's descendants. Detectors take this table as an argument instead of calling `ps` again.
2. **Agent detectors**: `agent_for()` dispatches to:
   - `find_claude()`, which is exact: it matches a descendant PID to `~/.claude/sessions/<pid>.json`, prefers `kind: "interactive"` over `"bg"`, and guards against PID reuse by comparing `ps -o etime` against `startedAt`. It does not use `lstart`, because that output is locale-dependent. `claude_from_argv()` is the fallback.
   - `find_opencode()`, which is heuristic: it reads the TUI `tabs.json` first, then the `session_v2` table in the SQLite DB (always opened `mode=ro`).
   - `foreground_command()` handles non-agent panes using the TTY's foreground process group.
   Each binding records a `confidence`. The stored values are `exact`, `exact-unverified`, `argv`, `heuristic` and `unknown`; `print_agent()` turns them into the labels the README shows, and `cmd_save` counts the weak ones.
3. **iTerm capture** (`ITERM_LAYOUT_SCRIPT`, `iterm_layout()`, `snapshot_iterm()`): an AppleScript walk of windows/tabs/sessions. A tab is bound to a tmux session by matching its TTY against `tmux list-clients` (`tmux_clients()`). Split direction is guessed from pane sizes (`infer_split()`). Control mode (`-CC`) and hotkey windows are skipped.
4. **Snapshot I/O**: `collect()` and `build_snapshot()` produce versioned JSON (`VERSION`). `write_atomic()` writes it. `would_lose_tabs()` stops a save from overwriting a tab-bearing `latest.json` with one that has no tabs because iTerm2 was unavailable; that snapshot goes to `noiterm-<ts>.json` instead. This applies only on macOS, and an explicit `--no-iterm` bypasses it and overwrites `latest.json` as usual.
5. **Restore**: a `Runner` wraps every tmux call so `--dry-run` prints instead of executing. Panes are created with `-c <cwd>`, then the recorded `window_layout` string is replayed with `select-layout`. `wait_for_shell()` waits until the shell has drawn its first prompt before typing anything. Agent resume commands (`resume_command()`) are submitted; every other command is typed but left unexecuted. `restore_iterm()`, `iterm_plan()` and the `iterm_make_*` / `iterm_write` helpers rebuild tabs, reuse existing tabs that are already clients of the target session, and never type into a tab whose shell isn't in the foreground.
6. **CLI**: `cmd_save` / `cmd_list` / `cmd_show` / `cmd_restore` / `cmd_prune`, wired up in `main()` with argparse.

Invariants to preserve:
- Nothing may write to `~/.claude`, the OpenCode DB, or iTerm's saved state. The tool only reads them.
- In a pane or iTerm session, `agent` and `command` are mutually exclusive. An iTerm session that has `tmux` set has neither.
- Restore skips existing sessions unless `--force` is given. Only agent panes get auto-submitted input.
- The command name and `~/.local/state/tmux-snapshot` path stay as they are for compatibility, even though the scope now goes beyond tmux.

The README is the user-facing spec, covering the snapshot format, restore semantics and caveats. When behaviour changes, update it in the same change.

## Conventions

Commits use Conventional Commits with a scope where it fits (`feat(iterm): …`, `docs: …`), plus a body that explains why.
