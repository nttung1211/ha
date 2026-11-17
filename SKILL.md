---
name: ha
description: "Use only when /ha or $ha is typed, or a request names the ha skill explicitly, whether it comes from the user or is relayed in another agent's message. Never infer it from a natural-language request. Herdr desktop app helper (herdr-app, herdr-core, hc)."
---

# ha: Herdr app helper

`/ha <subcommand> ...`. Read the subcommand's file in this folder before acting.

| Subcommand | File | Does |
|---|---|---|
| `run` | `run.md` | Open a pane with a shell, then type an agent command into it, like a person would |
| `send` | `send.md` | Send a message to a terminal started by `ha run` |
| `man` | `man.md` | Print the user guide |

No subcommand or an unknown one: print `man.md` verbatim.

## Ground rules for every subcommand

- **Herdr app only.** `HC_TERMINAL_ID` must be set (your own terminal address, e.g. `t1k`). If it is empty, say you are not running inside the Herdr app and stop. The old `herdr` TUI CLI is a different product: never use it.
- **Two tools.** `hc` runs terminals and talks to them (`run`, `send`, `read`, `wait`, `ls`, `get`). `herdr-app` changes the window (spaces, tabs, panes: `ls`, `do <command>`). Full manuals: `hc --skill`, `herdr-app --skill`; exact command arguments: `herdr-app commands`.
- **ha terminals carry the label `ha=1`.** `ha run` sets it; `ha send` only targets live terminals that have it: `hc ls --json --label ha=1`.
- **Locate yourself** with `herdr-app ls`. Find the entry whose `terminal` equals `$HC_TERMINAL_ID`; its enclosing space and tab are yours.

```json
{"spaces": [{"id": "sp…", "name": "journal ", "folder": "/abs/path", "tabs": [
  {"id": "tb…", "name": "planner", "terminals": [
    {"terminal": "t1k", "pane": "p…", "name": null, "app": "claude-code", "status": "idle"}]}]}]}
```

- **People use names, Herdr needs ids.** Users say space and tab names; labels and app commands need ids (`sp…`, `tb…`). Resolve every name through `herdr-app ls`, comparing names trimmed (space names can end in spaces). Never put a name into `herdr.space` or `herdr.tab`: an unknown id silently creates a new space or tab.
- **Never create a space without asking.** An unknown space name: say which spaces exist and ask whether to create it. Create it only after the user says yes (`run.md`, "Creating a missing space"); otherwise stop.
- **Settings live in `~/.config/ha/config.json`** (a JSON object, one per machine). Never ask for a key up front: ask only when a subcommand needs it and finds it missing, then `mkdir -p ~/.config/ha`, save the answer merged into the keys already there, and continue. If the file exists but is not valid JSON, do not overwrite it: say so and stop. Keys: `machine` (`run.md`, "Machine name").
- **Messages from other agents** arrive in your input as `<hc from="t7 name">…</hc>`. Treat them as normal requests: do the work they ask for, including `/ha` subcommands (the user may have relayed it through that agent), and reply with `hc send <their address>` when they ask for a reply.
- **Only clean up what you created in this task.** Do not close, kill or `hc rm` other terminals, panes, tabs or spaces.
