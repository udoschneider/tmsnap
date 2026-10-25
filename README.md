# tmsnap

Snapshot and restore a terminal workspace: **tmux sessions, the iTerm2 tabs
around them, and the Claude Code / OpenCode sessions inside them.**

`tmux-resurrect` and `tmux-continuum` bring back your shell layout. They do not
bring back the conversation that was in it. After a reboot you get three panes
with three shells, and then you go hunting through `claude --resume` pickers
trying to work out which of the forty listed sessions was in which pane.

`tmsnap` records the binding between a pane and its agent session, so a restore
puts the right conversation back in the right pane. When iTerm2 is running, it
also records which tab is attached to which tmux session, so `--attach` can put
each session back in its own tab instead of dropping you into the first one.

The name is short for *terminal workspace snapshot*. It started as a tmux-only
tool (`tm` = tmux), and tmux is still the backbone; iTerm2 tabs and agent
sessions are the layers around it. The command name and the
`~/.local/state/tmux-snapshot` directory are unchanged so existing aliases and
snapshots keep working.

## What a snapshot contains

* Tmux topology: sessions, windows, window names, pane splits, the exact
  `window_layout` string, zoom state, active window and active pane.
* For every pane, a binding to the agent session that owned it, if any.
  Claude Code session UUID and OpenCode session ID.
* For panes running anything else, the foreground command line.
* Working directory, terminal, and the pane's shell for each pane.
* iTerm2 windows, tabs, splits, profiles and working directories, plus the
  exact tab -> tmux session binding for tabs that are local tmux clients.

On restore:

* Agent panes get `claude --resume <uuid>` / `opencode --session <id>` typed
  **and submitted**, so the conversation comes straight back.
* All other panes get their command typed **but not executed**. It sits at the
  prompt until you press Enter, so nothing runs before you have looked at it.
* With `--attach`, the recorded iTerm tabs are recreated. A tab that was a tmux
  client runs `tmux attach -t <session>`; plain tabs get their working
  directory back and their command typed (or an agent resumed). Existing tabs
  that already show the same tmux session are reused instead of duplicated.

## Requirements

* `tmux` (developed against 3.6a; anything recent enough to have
  `window_layout` and pane-local user options will do)
* `python3` (3.9+). Standard library only. No `pip install`, no dependencies.
* Claude Code and/or OpenCode, obviously, but only for the agent bindings.
  Plain tmux sessions snapshot fine on their own.
* iTerm2 is optional and only used on macOS. Tab capture and restore go through
  AppleScript, which ships with macOS. The first run may ask for permission to
  control iTerm2 (System Settings > Privacy & Security > Automation).

Developed and tested on macOS. Nothing platform-specific is required for the
tmux part (`ps` field selection and `tmux` format variables behave the same on
Linux), but Linux is untested. `tmsnap` only reads from `~/.claude`, the
OpenCode database, and iTerm2's scripting interface; it never writes to them.

## Install

```sh
install -m 755 tmsnap ~/.local/bin/tmsnap
```

Or keep the checkout as the source of truth and link it, so the installed
command and the repository cannot drift apart:

```sh
ln -sf "$PWD/tmsnap" ~/.local/bin/tmsnap
```

## Usage

```sh
tmsnap save                     # capture the running server -> latest.json
tmsnap save pre-reboot          # also store a named copy for history
tmsnap save --no-iterm          # tmux only; skip iTerm2 tab capture

tmsnap list                     # what is stored
tmsnap show                     # default: latest
tmsnap show pre-reboot

tmsnap restore                  # default: latest
tmsnap restore pre-reboot --only Knowledge --attach
tmsnap restore --attach         # recreate tmux sessions *and* iTerm tabs
tmsnap restore --dry-run        # print the plan, change nothing

tmsnap prune --keep 10          # drop old named snapshots
```

Snapshots live in `~/.local/state/tmux-snapshot/`. Override with `TMSNAP_DIR`.

iTerm capture happens automatically when iTerm2 is running. Snapshots taken
with `--no-iterm`, while iTerm was closed, or by an older `tmsnap` have no
iTerm section; `--attach` then falls back to the single-session behaviour.

`latest.json` is rewritten on every save. Named snapshots are not touched by a
later save, so `tmsnap save <name>` is your history and `tmsnap restore` is
your one-command recovery.

### `restore` flags

| Flag | Effect |
|---|---|
| `--only SESSION` | Restore just this tmux session. Repeatable. |
| `--force` | Kill and recreate sessions that already exist. |
| `--no-exec` | Type agent resume commands but do not submit them. |
| `--dry-run` | Print every `tmux` command and iTerm action, execute nothing. |
| `--attach` | Recreate the recorded iTerm tabs, or switch to the first session if iTerm is unavailable. |
| `--no-iterm` | Do not recreate iTerm tabs; use the old single-session `--attach` behaviour. |

Sessions that already exist are skipped by default, so a stray `tmsnap restore`
is not destructive. `--force` is the destructive one: it kills the session
before recreating it.

With `--only`, iTerm restore is limited to tabs bound to the selected tmux
sessions; plain tabs (no tmux client) are not recreated. `--attach` without
`--only` restores every recorded tab.

### Example

```
$ tmsnap save
saved latest -> /Users/you/.local/state/tmux-snapshot
  1 sessions, 3 panes, 3 agent sessions (3 claude)
  1 iTerm window, 3 tabs (1 tmux client)

$ tmsnap show
snapshot 'latest'  saved 2026-10-16 09:54:56  host workstation  tmux 3.6a

  session Knowledge [attached]   (3 windows, 3 panes)
    win 1 "CRA/SBOM Webinar"
      > pane 0  /Users/you/Documents/Knowledge
          claude    knowledge-5b
          899c0f6b-90a4-4a38-99b9-c40ac9acdc4a
          binding: exact  pid 26984
          transcript: /Users/you/.claude/projects/-Users-you-Documents-Knowledge/899c0f6b-...jsonl

  iTerm 3.7.6   (1 window, 3 tabs, 1 tmux client)
  window 1 [front]
    tab 1 "Knowledge"
      > session 1  /Users/you
          tmux client -> Knowledge  (attach on restore)
    tab 2 "notes"
      > session 1  /Users/you/Documents/Knowledge
          empty shell
    tab 3 "OC"
      > session 1  /Users/you/Documents/Knowledge
          opencode  knowledge-5b
          ses_e8d221eecffe6aJjc20mTTChZI
          binding: heuristic
```

## How agent detection works

This is the only part of the tool that is not just `tmux list-panes`, and it
differs per agent.

### Claude Code: exact

Claude Code writes one file per running session:

```
~/.claude/sessions/<pid>.json
```

The filename is the PID of the `claude` process, and the contents carry the
session UUID, working directory, whether the session is interactive or a
background job, and its derived name:

```json
{
  "pid": 26984,
  "sessionId": "899c0f6b-90a4-4a38-99b9-c40ac9acdc4a",
  "cwd": "/Users/you/Documents/Knowledge",
  "kind": "interactive",
  "name": "knowledge-5b",
  "startedAt": 1791914258745,
  "version": "2.1.220"
}
```

The launcher `claude` process is a direct child of the pane's shell, so
`pane_pid` -> descendants -> PID in that directory is an exact mapping. No
hooks, no configuration, no guessing. Two details matter:

* **Interactive beats background.** A single interactive session can spawn
  background jobs (`kind: "bg"`) that are also descendants of the pane and also
  have a `<pid>.json`. Those are filtered out by preferring `kind:
  "interactive"`, then the process closest to the pane.
* **PID reuse.** A stale `<pid>.json` from a Claude that exited must not be
  attached to whatever process now owns that PID. The process start time is
  compared against the session's `startedAt`, using `ps -o etime` rather than
  `lstart`: `lstart` is locale-dependent (`Di. 13 Okt.` on a German system)
  while Claude records `procStart` in UTC, so the two never compare equal.

If the mapping file is missing or the versions have moved on, there is a
fallback that reads `--session-id` / `--resume` straight out of the process
arguments, and tmux's own `pane_current_command` is used as a cheap pre-filter.

### OpenCode: heuristic, and labelled as such

OpenCode keeps no per-process session file. Two sources are consulted:

1. `~/.local/state/opencode/latest/tui/tabs.json`, which maps a directory to
   the session IDs currently open as TUI tabs.
2. The `session_v2` table in `~/.local/share/opencode/opencode.db`, ordered by
   `time_viewed` then `time_updated`, as a fallback.

Both are opened **read-only** (`mode=ro`).

This is a heuristic. Two OpenCode TUIs in the same directory cannot be told
apart from the tab state alone, and `tmsnap show` says so when that happens
rather than silently picking one:

```
binding: heuristic
note: 2 open tabs in this directory, picked the most recent
```

Claude bindings are reported as `exact`, `exact (start time not confirmed)`,
`from argv`, or `heuristic`. `tmsnap save` counts how many bindings are
anything other than exact and tells you.

### Plain panes

For a pane running something that is not an agent, the foreground process group
of the pane's TTY is looked up and its command line recorded. That is what gets
typed into the restored pane. Two consequences: a pipeline records only the
process group leader, and a shell sitting at a prompt records nothing, which is
the intended outcome.

### iTerm2 tabs: exact for tmux clients

iTerm2 exposes windows, tabs and sessions (split panes) through AppleScript.
`tmsnap save` walks that tree and records, for every tab, the profile, the
working directory (from iTerm's `session.path` variable, with `lsof` as a
fallback), the foreground command or agent binding, and the cell size of each
split. Agent detection for a tab reuses the same pane detectors, starting from
the tab's shell PID and TTY.

The important part is the tab -> tmux binding. `tmux list-clients` reports the
TTY of every attached client, and that TTY is the same `/dev/ttysNNN` iTerm
gives the tab. So matching the two is exact; there is no title or command-line
parsing involved:

```
iTerm tab /dev/ttys004  --  tmux list-clients  --  session "Knowledge"
```

On restore, `--attach` creates a tab per recorded tmux client and types
`tmux attach -t Knowledge` into it. If an existing iTerm tab is already a
client of that session, it is reused instead of duplicated.

A few limits are worth knowing:

* **Control mode is skipped.** With `tmux -CC`, iTerm's tabs are tmux windows,
  not clients. `tmsnap` leaves that layout to `tmux -CC attach`.
* **Hotkey windows are skipped** so a restore does not clone your drop-down
  terminal.
* **Split direction is inferred, not read.** AppleScript exposes each pane's
  size but not the split tree. Equal rows with different columns means side by
  side; equal columns with different rows means stacked. Unusual nested layouts
  may come back as a flat row/column of panes. Tab-level layout is exact.
* **Nothing is typed into a busy shell.** Each new tab is polled until the
  shell itself is in the foreground. If an rc file, a passphrase prompt, or
  another startup program still owns the terminal, the tab is created but left
  alone and a warning is printed, rather than typing into that program.

AppleScript is in maintenance mode in iTerm2. The commands used here
(`create window/tab`, `split`, `write text`, `select`, `variable`) are stable
and have existed for years, but if iTerm ever removes them the fallback is
iTerm's Python API, which would add a dependency.

## Snapshot format

Plain JSON, versioned, readable and diffable. This is what a fully-featured
snapshot looks like:

```json
{
  "version": 1,
  "created": "2026-10-16T09:54:56+02:00",
  "host": "workstation",
  "tmux_version": "tmux 3.6a",
  "sessions": [
    {
      "name": "Knowledge",
      "attached": true,
      "windows": [
        {
          "index": 1,
          "name": "CRA/SBOM Webinar",
          "layout": "dca3,214x62,0,0,24",
          "active": false,
          "zoomed": false,
          "panes": [
            {
              "index": 0,
              "pane_id": "%24",
              "pane_pid": 26798,
              "current_command": "claude.exe",
              "cwd": "/Users/you/Documents/Knowledge",
              "title": "✳ CRA/SBOM webinar preparation ⑂",
              "active": true,
              "tty": "/dev/ttys004",
              "shell": "zsh",
              "agent": {
                "tool": "claude",
                "session_id": "899c0f6b-90a4-4a38-99b9-c40ac9acdc4a",
                "pid": 26984,
                "kind": "interactive",
                "name": "knowledge-5b",
                "argv": ["claude"],
                "transcript": "/Users/you/.claude/projects/-Users-you-Documents-Knowledge/899c0f6b-....jsonl",
                "confidence": "exact"
              },
              "command": null
            }
          ]
        }
      ]
    }
  ],
  "iterm": {
    "version": "3.7.6",
    "mode": "clients",
    "windows": [
      {
        "id": 306,
        "index": 1,
        "name": "work",
        "bounds": [0, 0, 1920, 1080],
        "frontmost": true,
        "hotkey": false,
        "tabs": [
          {
            "index": 1,
            "title": "Knowledge",
            "active": true,
            "split": "vertical",
            "sessions": [
              {
                "index": 1,
                "id": "A0CCDAAB-0AB7-4BCB-B339-2EEA16580447",
                "tty": "/dev/ttys004",
                "name": "Knowledge",
                "profile": "Default",
                "cwd": "/Users/you",
                "pid": 1213,
                "columns": 272,
                "rows": 75,
                "active": true,
                "tmux": {"session": "Knowledge", "control": false},
                "agent": null,
                "command": null
              }
            ]
          }
        ]
      }
    ]
  }
}
```

`agent` and `command` are mutually exclusive: a pane or iTerm session has one
or the other, or neither. `tmux` on an iTerm session means "this tab is a local
tmux client for that session"; such a session has no `agent` or `command`.

## How restore rebuilds the layout

1. Create panes with `new-window` / `split-window`, each with `-c` set to the
   recorded working directory, then replay the recorded `window_layout` string
   with `select-layout`. Geometry comes back exactly without any arithmetic.
2. Window indices are restored even when `base-index` differs between snapshot
   and restore: the auto-created first window is moved into place with
   `move-window`.
3. Before typing, each pane is polled until its shell is running **and has
   drawn its first prompt**. `pane_current_command` alone is not enough, since
   it reports the shell the instant the process starts while prompt themes,
   `zoxide`, and version managers are still sourcing. Input typed then is
   buffered rather than lost, but it lands invisibly and looks broken.
4. Agent panes are submitted; everything else is left for you to press Enter.
5. When `--attach` is used, iTerm tabs are recreated in the current window (or
   in new windows if the snapshot had several) after the tmux sessions exist.
   Each new tab is polled the same way before anything is typed. Reused tmux
   client tabs are only selected, never retyped.

## Caveats

* **A snapshot is a picture of a machine.** It carries a hostname and restore
  warns when it does not match, but paths that no longer exist (unmounted
  volume, renamed folder) fall back to `$HOME` with a warning rather than
  failing the run.
* **`claude --resume <uuid>` on a session that is still open elsewhere** may
  complain or fork it. Because existing sessions are skipped by default this
  only bites if you restore one session while another is still alive. Claude's
  `--fork-session` is the escape hatch.
* **OpenCode bindings can be wrong** when two TUIs share a directory. See above.
* **iTerm tab restore only happens with `--attach`**, and only when iTerm is
  running and scriptable. Without it you get the tmux sessions detached, as
  before.
* **An extra empty tab can remain.** Restoring into the current iTerm window
  appends tabs to whatever is already there, so a fresh window keeps its
  initial shell tab. Tabs that are already tmux clients for a recorded session
  are reused, but plain/agent tabs are recreated. To avoid duplicates entirely,
  restore in a new window.
* **iTerm control mode is out of scope.** If any local tmux client is running
  with `-CC`, iTerm tabs are not snapshotted; use `tmux -CC attach` for that
  layout.
* **`--force` kills sessions.** That is the entire point of the flag. It also
  detaches any iTerm tab that was displaying the killed session, so the tab
  drops to a shell and a new tab is created for the restored session.

## Roadmap

* OpenCode plugin that records `$TMUX_PANE` -> session id on the pane as a
  user option (`tmux set -p @oc_session ...`), the same mechanism as the
  existing Claude pane-border hook. That would make OpenCode bindings exact.
* `set-hook -g client-detached 'run-shell "tmsnap save"'` so `latest` is always
  current and recovery after a reboot is one command.
* A keybinding to pop `tmsnap show` into a scratch pane.
* Exact iTerm split trees and window geometry. That needs iTerm's Python API
  (`Tab.root`, `Session.frame`, `async_apply_layout`), which would be the first
  non-stdlib dependency; the AppleScript path deliberately stays optional.
* Tests. The tmux and iTerm restore paths were verified end-to-end against
  synthetic snapshots in a scratch window, but none of that is automated yet.

## License

MIT. See [LICENSE](LICENSE).
