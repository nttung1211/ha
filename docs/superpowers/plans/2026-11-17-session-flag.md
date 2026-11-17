# ha run --session Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `ha run` names each Claude session `ha/<machine>/<name>` (or `--session`), so sessions from different machines are told apart in the Claude mobile app.

**Architecture:** ha is a skill made of Markdown instructions an agent follows; there is no program code. The change edits those instructions (`SKILL.md`, `run.md`) and the user-facing docs (`man.md`, `README.md`). The machine name lives in `~/.config/ha/config.json`, asked for once when needed.

**Tech Stack:** Markdown skill files; runtime tools `hc`, `herdr-app`, `claude`.

**Spec:** `docs/superpowers/specs/2026-11-17-session-flag-design.md`

## Global Constraints

- Flag: `-s, --session <session>`; non-empty, no `'`, no newline.
- Default session: `ha/<machine>/<name>`.
- Typed command: `claude [--model M] [--effort E] -n '<session>' --remote-control '<session>'`.
- Config file: `~/.config/ha/config.json`, key `machine`; value matches `^[A-Za-z0-9_-]{1,32}$`.
- Herdr core name (suggested answer): `name` in `~/.local/state/herdr-core/default/core.json`.
- Config rule: a key is asked for only when a subcommand needs it and it is missing; the answer is merged into existing keys and saved; a file that is not valid JSON is never overwritten (say so, stop).
- Machine name needed only when agent is `claude` and `--session` is absent; asked before any pane opens.
- `--session` is not applied to `agy`, `codex`, `grok`, `shell`; if given explicitly, the report says it was ignored.
- Docs stay in English, in the existing terse style of `run.md`.

## Review Focus

- `config.json` exists with other keys but no `machine`: ask, then save **merged** (other keys kept), not overwritten.
- User passes `-s` with spaces (`-s "my review"`): typed single-quoted, so Claude gets one argument.
- User answers the machine-name question with an invalid name (`my mac`, `h/x`): refuse, say allowed characters, ask again; nothing saved, no pane opened.
- `-s` given to `agy` or `shell`: pane still opens; report says the session name was ignored.
- `--session` given: no machine-name question even when `config.json` is missing.

---

### Task 1: `--session` in SKILL.md and run.md

**Files:**
- Modify: `SKILL.md` (Ground rules list: add a config bullet after "Never create a space without asking")
- Modify: `run.md` (syntax line; flag table; step 1; step 4 table; step 6 report; new section "Machine name" after "Creating a missing space"; Examples; Mistakes table)

**Interfaces:**
- Produces: flag `-s, --session`; section heading `## Machine name` in `run.md` (Task 2's docs refer to its behavior); config path `~/.config/ha/config.json`, key `machine`.

- [ ] **Step 1: Write the failing check**

```bash
grep -q -- '-s|--session' run.md && grep -q '## Machine name' run.md \
  && grep -q "~/.config/ha/config.json" SKILL.md \
  && grep -q -- "-n '<session>' --remote-control '<session>'" run.md && echo PASS || echo FAIL
```

- [ ] **Step 2: Run it, expect `FAIL`**

- [ ] **Step 3: Edit `SKILL.md`**

Add ground rule: `~/.config/ha/config.json` holds ha's per-machine settings (JSON object). Never ask for a key up front; ask only when a subcommand needs it and it is missing, then `mkdir -p ~/.config/ha`, save merged into existing keys, continue. If the file is not valid JSON, do not overwrite it: say so and stop. Keys today: `machine` (see `run.md`).

- [ ] **Step 4: Edit `run.md`**

- Syntax line: add `[-s|--session S]` after `-e`.
- Flag table row: `-s`, `--session` | Claude session name (mobile app, `/resume`, terminal title). Default `ha/<machine>/<name>` (see "Machine name"). Non-empty, no `'` or newline. Only `claude` uses it.
- Step 1: validate `--session`; if agent is `claude` and no `--session`, resolve the machine name now (before step 2 creates anything) and build `ha/<machine>/<name>`.
- Step 4 table, claude row: `claude [--model M] [--effort E] -n '<session>' --remote-control '<session>'`; note under the table: other agents get no session flag.
- Step 6 report: add session name (claude), or "session ignored for <agent>" when `-s` was given to another agent.
- `## Machine name` section: read `machine` from config; missing → ask once `Machine name for session names? (Herdr calls this machine "<core name>")`, core name read from `~/.local/state/herdr-core/default/core.json`; accept `^[A-Za-z0-9_-]{1,32}$` else say allowed characters and ask again; save merged; one sentence on why: the Claude mobile app lists sessions from every machine, and the phone's machine labels are not readable on the computer.
- Examples: add `/ha run claude reviewer -s "api review"` and note default naming on the first example.
- Mistakes table row: typing the session unquoted → a name with spaces splits into several arguments → always single-quote it.

- [ ] **Step 5: Run the Step 1 check, expect `PASS`**

- [ ] **Step 6: End-to-end check with the edited files**

The installed skill (`~/.agents/skills/ha`) is a copy, so drive a fresh agent at the repo files. With no `~/.config/ha/config.json` present:

```bash
hc run --bg --no-parent --name e2e-host --cwd /Users/nttung1211/dev/bwa/ha --label ha=1 \
  --label herdr.space=<own sp…>      # new tab
hc send <addr> "claude"; hc wait <addr> --timeout 90
hc send <addr> "Use the ha skill from /Users/nttung1211/dev/bwa/ha (SKILL.md, run.md), not the installed copy: /ha run claude e2e-a -i e2e-host"
```

Expected, checked with `hc read <addr>` and `hc ls --all --json`:
1. It asks the machine-name question naming `ntt-machine` before any `e2e-a` terminal exists. Answer `h my` → refused; answer `h` → `~/.config/ha/config.json` is `{"machine": "h"}`.
2. `e2e-a`'s foreground argv (in `~/.local/state/herdr-core/default/terminals.json`) is `["claude","-n","ha/h/e2e-a","--remote-control","ha/h/e2e-a"]`.
3. Second request `/ha run claude e2e-b -i e2e-host -s "api review"` → no question; argv has `api review` as one argument.
4. `/ha run shell e2e-c -i e2e-host -s x` → pane opens, report says session ignored.
5. Write `{"other": 1}` to config, remove `e2e-*` and repeat (1) with a new name → saved file has both `other` and `machine`; restore `{"machine": "h"}` afterwards.

If a trust prompt blocks a pane, report it to the user rather than answering it. Clean up every `e2e-*` terminal and the tab afterwards.

- [ ] **Step 7: Commit**

```bash
git add SKILL.md run.md
git commit -m "ha run: add --session, default ha/<machine>/<name>"
```

### Task 2: User docs

**Files:**
- Modify: `man.md` (USAGE line, RUN options, EXAMPLES, NOTES)
- Modify: `README.md` (Usage block)

**Interfaces:**
- Consumes: flag `-s, --session` and `~/.config/ha/config.json` from Task 1.

- [ ] **Step 1: Write the failing check**

```bash
grep -q -- '-s, --session' man.md && grep -q -- '\[-s session\]' man.md \
  && grep -q -- '\[-s session\]' README.md && echo PASS || echo FAIL
```

- [ ] **Step 2: Run it, expect `FAIL`**

- [ ] **Step 3: Edit `man.md` and `README.md`**

Usage line in both: `/ha run  <agent> <name> [-i [space/]tab] [-a dir] [-m model] [-e effort] [-s session] [task]`. `man.md` RUN option `-s, --session`: claude session name, default `ha/<machine>/<name>`; machine name asked once, kept in `~/.config/ha/config.json`. One example with `-s`. NOTES: settings in `~/.config/ha/config.json` are asked for the first time they are needed.

- [ ] **Step 4: Run the Step 1 check, expect `PASS`**

- [ ] **Step 5: Commit**

```bash
git add man.md README.md
git commit -m "docs: document ha run --session"
```
