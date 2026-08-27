# resurrect

**Browse, revive and manage your Claude Code sessions — across every project, in one table.**

Claude Code already saves every conversation you have with it. What it doesn't give you is a way to *see* them all: the built-in `/resume` picker only lists sessions for the directory you're standing in, and after a reboot nothing tells you what you had open. `resurrect` fills that gap with a single dependency-free script.

```
 resurrect — loaded sessions · 26 sessions
 Your loaded sessions. Any Claude session seen running is collected in here automatically (opening
 this table collects too). The list lives on disk and survives restarts — once loaded, a session stays
 until you unload it (ctrl-u), which only hides the row. Conversations themselves are never lost.
   AGE  LAST USED     DIRECTORY                    NAME                          ID
 ⚡  4m  27 Aug 15:15  ~/leap/svc-python            suite-driven-live-test-run    6041ed93
    7m  27 Aug 15:12  ~/leap/svc-python            PR 1203                       25c33842
   29m  27 Aug 14:50  ~/leap/svc-python            Proactive sourcing feature    d5b00e36
    2h  27 Aug 13:04  ~/dev/tax-return             Consolidate bank accounts     222d8212
 enter resume · type to filter · ctrl-n new session here · ctrl-u unload/reload · tab show unloaded · esc quit
 ⚡ running now (enter refused — attach to the live one) · 🗄 unloaded
```

## How it works

Claude Code appends every message of every session to a transcript file:

```
~/.claude/projects/<munged-working-dir>/<session-id>.jsonl
```

Nothing is ever lost when a session's process dies — crash, quit, or reboot. `claude --resume <session-id>` rebuilds the conversation exactly, context and all. `resurrect` is a thin layer over that fact:

- The default table is **your loaded list**: every session that has ever been *collected while running*, sorted by last use — your working set, not a dump of everything on disk. The list itself is persistent: it lives in a small JSON file, survives reboots, and a session stays in it indefinitely until you unload it. Selecting a row `cd`s to the session's own directory and exec's `claude --resume <id>` in your current terminal.
- **collect** scans your running `claude` processes and upserts them into the list. It also runs silently every time you open the loaded table, so new sessions you start anywhere just appear on the next launch — and the list only ever updates, never rewrites, so nothing you collected can be silently dropped.
- **all** is the discovery view: every transcript on disk within a window, across all directories — for finding older conversations. Resume one and it's running, so it joins the loaded list at the next collect.
- **print** emits `cd … && claude --resume …` lines for the loaded sessions you haven't unloaded — the paste-into-terminal-tabs bulk restore after a reboot. Rows that are already running are commented out so you don't double-attach.

Everything is read-only over Claude's transcripts. The tool never deletes or modifies one.

## Install

```sh
mkdir -p ~/.local/bin
curl -o ~/.local/bin/resurrect https://raw.githubusercontent.com/davidtedmanjones/claude-resurrect/main/resurrect
chmod +x ~/.local/bin/resurrect
```

(Ensure `~/.local/bin` is on your `PATH`.)

Requirements: macOS or Linux (Windows exits with a clear message), Python 3.8+ (system python is fine), the `claude` CLI on your PATH, a procps-compatible `ps` (busybox's isn't). No tmux, no daemon, no background processes, no third-party packages.

## Usage

### The loaded list (the default)

```sh
resurrect              # your loaded sessions, newest first
resurrect all          # discovery: everything on disk (last 30 days)
resurrect all --days 90          # wider discovery window (only `all` takes --days)
resurrect | grep -i letterhead   # non-interactive: prints the table for piping
```

Keys inside the table:

| Key | Action |
|---|---|
| any text | filter rows live (matches directory, name, id, date) |
| `↑` / `↓` | move |
| `Enter` | resume the selected session, in its own directory, in this terminal |
| `Ctrl-N` | start a **new** session in the directory you ran `resurrect` from — prompts for a name that then appears in the NAME column |
| `Ctrl-U` | **unload** the selected row (hide it from the table) — or reload it if it's already unloaded |
| `Tab` | show/hide unloaded rows (they render dimmed with 🗄) |
| `Ctrl-Y` | toggle "yolo" — launch claude with `--dangerously-skip-permissions` by default (persists to the config file; the title bar shows `⚡yolo` while active) |
| `Esc` / `Ctrl-C` | quit |

**Unload never deletes anything.** It adds the session id to a hide-list (`~/.claude/resurrect-hidden.json`); the transcript stays on disk and the row comes back with `Tab` → `Ctrl-U` any time. That's the only way a session leaves your view — nothing ages out.

**Live sessions.** Rows marked ⚡ have a running process that names the session id in its arguments — which includes everything `resurrect` itself launches. Pressing Enter on one exits with a pointer to the live pid instead of resuming, because resuming a session another process owns gives you a broken second copy. One honest limit: a session started by hand as plain `claude` (no `--resume`/`--session-id`) can't be tied to its process, so it appears in the list without the ⚡ guard.

Before resuming, the exact `cd … && claude --resume …` command is printed — it stays in your scrollback as a receipt of what ran and where.

### Collect & the reboot workflow

```sh
resurrect collect      # gather every running session into the list (with a receipt)
# ...reboot...
resurrect              # the list is intact; Enter each session back to life
resurrect print        # or emit resume commands to paste into terminal tabs
```

Collect pairs each running process with its session id directly from the process arguments when possible, and falls back to a most-recently-written-transcript heuristic for plain `claude` processes (headless `claude -p` and `claude mcp` processes are excluded). The receipt tells you how many of each. The heuristic's worst case is recording a recently-active transcript that wasn't actually the fresh session — a near-miss; the real one joins at the next collect once it has written its transcript.

**The list only ever updates — it never rewrites.** Each collect upserts running sessions by id (refreshing `last_seen`) and leaves everything previously recorded in place; writes are atomic, and a corrupt store file is backed up (`*.corrupt-<timestamp>`) rather than silently replaced. Because opening the loaded table also collects silently, even an *unplanned* crash or forced restart leaves your list current; the explicit `collect` command is just the deliberate version with a receipt. (`sync` and `snapshot` are accepted as aliases.)

## Configuration

How `claude` gets launched (for both resume and Ctrl-N), first match wins:

1. `RESURRECT_CLAUDE_CMD` environment variable
2. `"claude_cmd"` in `~/.claude/resurrect.json`
3. plain `claude`

For example, `~/.claude/resurrect.json` containing:

```json
{ "claude_cmd": "claude --dangerously-skip-permissions" }
```

Or per-invocation: `resurrect -y` appends `--dangerously-skip-permissions` for that run only. The default is deliberately the plain, permission-prompting `claude` — skipping permissions is a choice you make explicitly.

If you relocate Claude Code's config dir with `CLAUDE_CONFIG_DIR`, resurrect honours it for all its paths.

## Files it reads and writes

| Path | Role |
|---|---|
| `~/.claude/projects/*/*.jsonl` | Claude Code's own transcripts — **read-only** |
| `~/.claude/resurrect-snapshot.json` | the loaded list (upsert-only; note: contains session titles, which derive from your conversations) |
| `~/.claude/resurrect-hidden.json` | unloaded session ids |
| `~/.claude/resurrect-names.json` | your names for Ctrl-N sessions |
| `~/.claude/resurrect.json` | optional config |

resurrect's own files are written with `0600` permissions. Everything stays on your machine; nothing is sent anywhere.

## FAQ

**Does it need tmux?** No. It runs in any terminal — Terminal.app, iTerm, the VS Code integrated terminal. Resumed sessions open right where you invoked it. (If you host your sessions in tmux, resume from inside a tmux pane and it lands there — the tool doesn't care.)

**Why is startup a beat slow?** It reads the tail of recent transcripts to extract titles. A couple of seconds at ~30 sessions. (`--days` narrows the `all` view; the loaded view's scan window is fixed.)

**A session shows the wrong name.** Names come from your Ctrl-N alias if set, else Claude's AI-generated session title, else the first line of a recent user message (so some rows show truncated prompt text). Resume it and `/rename`, or Ctrl-N-style alias it by editing `resurrect-names.json`.

**Grepping the piped output for a full session id finds nothing.** The table shows the first 8 characters of each id; grep for those, or use `resurrect print` which emits full ids.

**How do I uninstall?** Delete the script and the four `resurrect*` files in `~/.claude`. Your transcripts are Claude's and remain untouched.

**Is my data safe to share this with colleagues?** The tool ships no data — each person sees only their own local transcripts.

## License

MIT
