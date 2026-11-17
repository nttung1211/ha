# ha run --session: design

## Intent

Claude sessions from two machines (named `h` and `n` in the Herdr phone app) look alike in the Claude mobile app, because every session joins Remote Control at startup with an auto-generated, hostname-based name. `ha run` should give each Claude session a name that says which machine and which ha terminal it is: `ha/<machine>/<name>`, e.g. `ha/h/reviewer`.

## Goals

- `ha run` gets `-s, --session <session>`. Default: `ha/<machine>/<name>`.
- The name shows the same in the Claude mobile app, `/resume`, and the terminal title.
- No manual setup: ha asks for the machine name once per machine and remembers it.

## Non-goals

- Naming sessions for `agy`, `codex`, `grok` (no name flag, or not verified). `shell` runs no agent.
- Panes on remote Herdr machines (`ha run` does not target them today).
- Renaming sessions that already exist.
- Any other `config.json` key.

## Approaches

How the name reaches Claude (verified with a spike: all three show the name in the mobile app):

- A. `claude -n <s>` only: display name; reaches the mobile app only because `remoteControlAtStartup` is on.
- B. `claude --remote-control <s>` only: names the Remote Control session; terminal title stays "Claude Code".
- **C. both (chosen)**: same name everywhere, and still on mobile if `remoteControlAtStartup` is turned off.

Where the machine name comes from (the phone's `h`/`n` labels live only on the phone; Herdr on the Mac knows only `ntt-machine` = hostname):

- 1. Herdr core name (`~/.local/state/herdr-core/default/core.json` `name`): zero questions, but not `h`.
- **2. ask once, remember in `~/.config/ha/config.json` (chosen)**, JSON so later settings can join it.
- 3. env var `HA_MACHINE`, or renaming the Mac's hostname: rejected (manual setup / system-wide change).

## Design

**Flag.** `-s, --session <session>`: any non-empty text without `'` or a newline (it is typed single-quoted into a shell).

**Default.** `ha/<machine>/<name>`, `<name>` being the hc name given to `ha run`.

**Config rule (all keys, now and later).** `~/.config/ha/config.json` holds ha's per-machine settings. ha never asks for a key up front: it asks only when a subcommand needs that key and finds it missing, then saves the answer (merging into the existing keys) and continues. This rule goes in `SKILL.md` ground rules so future keys follow it.

**Machine name.** Read `machine` from `~/.config/ha/config.json`.
- Needed only when the agent is `claude` and `--session` is absent.
- Missing (no file, or no key): before opening any pane, ask once: `Machine name for session names? (Herdr calls this machine "<core name>")`. Accept `^[A-Za-z0-9_-]{1,32}$`; the core name is the suggested answer. Save it with `mkdir -p ~/.config/ha`, merging into existing keys, then continue.
- File present but not valid JSON: do not overwrite it; say so and stop.
- Writing `~/.config` may need one permission prompt inside a sandbox; reads do not.

**Command typed** (step 4 of `run.md`):
`claude [--model M] [--effort E] -n '<session>' --remote-control '<session>'`

**Other agents.** `--session` is not applied. If the user passed it explicitly, the report says it was ignored for that agent.

**Report** adds the session name (claude only).

## Files

- `run.md`: syntax line, flag table, step 1 (validate `--session`, resolve the machine name), step 4 command table, report, a "Machine name" section, examples.
- `man.md`, `README.md`: usage line and option text.
- `SKILL.md`: ground rule for `~/.config/ha/config.json` (the config rule above).

## Left to the builder

Exact wording in the docs; where the "Machine name" section sits in `run.md`.
