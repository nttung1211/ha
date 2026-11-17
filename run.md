# ha run

Open a new pane holding a shell, then type the agent command into it, exactly as a person opens a pane and types `claude`. When the agent exits, the shell stays.

```
ha run <agent> <name> [-i|--in [space/]tab] [-a|--at dir] [-m|--model M] [-e|--effort E] [-s|--session S] [task…]
```

| Part | Meaning |
|---|---|
| `<agent>` | Required. `claude`, `agy`, `codex`, `grok` or `shell` |
| `<name>` | Required. The new terminal's hc name, used later to address it. 1–32 chars of `a-z 0-9 - _` |
| `-i`, `--in` | Where the pane goes (table below). Default: your own tab |
| `-a`, `--at` | Working folder. Default: the target space's `folder`. A relative path resolves against that folder. For a space being created, see "Creating a missing space" |
| `-m`, `--model` | Passed to the agent as its model flag |
| `-e`, `--effort` | Passed to the agent as its effort flag |
| `-s`, `--session` | Claude session name, shown in the Claude mobile app, `/resume` and the terminal title. Default `ha/<machine>/<name>` (see "Machine name"). Non-empty, no `'` and no newline. Only `claude` uses it |
| `task…` | Everything after the flags. Sent to the agent verbatim once it is ready. Add nothing to it |

`--in` resolution (names compared trimmed, your own space checked first):

| Written | Means |
|---|---|
| (absent) | Your own tab |
| `-i planner` | Tab `planner` in your space |
| `-i antrail/planner` | Tab `planner` in space `antrail` |
| `-i antrail/` | A new tab in space `antrail`, named `<name>` |
| a tab name that does not exist (space found) | A new tab with that name in that space |
| `-i newspace/tab` or `-i newspace/`, space not found | Ask to create space `newspace` (see "Creating a missing space"). Its first tab is `tab`, or `<name>` for `newspace/` |

Two tabs with the same name in the target space, or a space name matching two spaces: ask which one.

If your own terminal is not in `herdr-app ls` (a hidden background worker), you have no space or tab: `--in` is required, and a bare tab name is searched in every space. Exactly one match: use it. Several or none: ask.

## Steps

1. **Parse and validate.** `<name>` must match `^[a-z0-9_-]{1,32}$`. Run `hc ls --all --json`: if a **live** terminal already has that name, stop and ask for another. If only finished terminals have it, continue, but address the new terminal by its address (`tN`) afterwards and mention the clash in the report.
   Check `--session` if given (non-empty, no `'`, no newline). If `<agent>` is `claude` and there is no `--session`, resolve the machine name now, before anything is created ("Machine name" below), and use `ha/<machine>/<name>`.
2. **Resolve ids** from `herdr-app ls`: your space and tab (see SKILL.md), then the target space id `sp…` and, unless a new tab is needed, the target tab id `tb…`. Resolve the folder from `--at` and check it exists. If the target space does not exist, follow "Creating a missing space" below, then continue at step 3 with the tab it gives you.
3. **Open the pane** (no parent, so it lives on like a pane the person opened):

   ```sh
   hc run --bg --no-parent --name <name> --cwd <folder> --label ha=1 \
     --label herdr.space=<sp…> --label herdr.tab=<tb…>
   ```

   `ha=1` marks it as an ha terminal, so `ha send` can find it.

   For a new tab, leave out `--label herdr.tab`: Herdr makes a tab in that space. Then find the tab holding the new address in `herdr-app ls` and rename it: `herdr-app do tab.rename '{"id":"<tb…>","name":"<tab name>"}'`.
   Read `address` from the JSON reply and use it from here on.

4. **Type the agent command** unless `<agent>` is `shell`:

   | Agent | Command typed |
   |---|---|
   | `claude` | `claude [--model M] [--effort E] -n '<session>' --remote-control '<session>'` |
   | `agy` | `agy [--model M] [--effort E]` |
   | `codex` | `codex --no-daemon …`. hc requires `--no-daemon`. Check `codex --help` for the model and effort flags |
   | `grok` | `grok …`. Check `grok --help` for the model and effort flags |

   The session name is always single-quoted. Only `claude` gets it; other agents get no session flag.

   ```sh
   hc send <address> "<command>"
   hc wait <address> --timeout 90
   hc get <address>        # expect status.app set and state idle
   ```

   If `wait` returns because a message arrived or it timed out, check `hc get` and wait again. If the agent is `blocked` (trust prompt, login, permission), `hc read <address>` and tell the user what it shows. Do not answer the prompt for them.

5. **Send the task**, if any, verbatim: `hc send <address> "<task>"`. Confirm the reply shows `"started": true`; otherwise `hc read` before deciding anything. Never resend blindly.

6. **Report** in one short block: name, address, `space / tab`, folder, the command typed, the session name (claude), whether a task was sent, and whether the space (and its folder) was created. If `--session` was given to another agent, say it was ignored.

## Machine name

The Claude mobile app lists sessions from every machine, and the names the phone gives machines cannot be read on the computer, so ha asks for this machine's name once and keeps it in `~/.config/ha/config.json` (see SKILL.md).

1. Read `machine` from `~/.config/ha/config.json`. Present: use it.
2. Missing (no file, or no key): read `name` from `~/.local/state/herdr-core/default/core.json` and ask once: `Machine name for session names? (Herdr calls this machine "<core name>")`. The core name is the suggested answer.
3. The answer must match `^[A-Za-z0-9_-]{1,32}$`. Otherwise save nothing, say which characters are allowed and ask again.
4. Save it: `mkdir -p ~/.config/ha`, then write the file with `machine` set and every other key kept. Continue with `run`.

## Creating a missing space

Only when `--in` names a space (`space/tab` or `space/`) that `herdr-app ls` does not show. A bare tab name never creates a space: `-i test` is always a tab; write `-i test/` for a space.

1. **Check the name.** A new space name must match `^[A-Za-z0-9_-]+$` (letters, digits, `-`, `_`). Otherwise create nothing: say which characters are allowed and ask for another name.
2. **Work out the folder.** "Your working directory" is your own current directory (`pwd`), not your space's folder.
   - no `--at`: `<your working directory>/<space>`. Started in `~/dev/bwa`, `-i test/` gives `~/dev/bwa/test`.
   - `--at` absolute: that folder, as given (no `/<space>` added).
   - `--at` relative: resolved against your working directory (no `/<space>` added).

   Check whether that folder exists.
3. **Ask once, create nothing yet.** Say the space does not exist, list the spaces that do, and ask one question covering both: create space `<space>` with folder `<folder>`, and, if the folder is missing, say it will be created too. Example: `Space "test" does not exist (spaces: journal, antrail). Create it with folder ~/dev/bwa/test? The folder does not exist yet; it will be created.` If the user says no, stop. If they give another folder or name, check it again from step 1.
4. **Create the folder** if it was missing: `mkdir -p <folder>`. Every terminal opened in the new space starts here by default.
5. **Create the space**, naming its first tab (`tab`, or `<name>` for `space/`), so no rename is needed:

   ```sh
   herdr-app do space.new '{"name":"<space>","folder":"<folder>","tab":"<tab>","focus":false}'
   ```

   Herdr starts a plain shell in that tab. Read its `terminal` (and `pane`) from the reply, then find that terminal in `herdr-app ls`: its enclosing space and tab give the new `sp…` and `tb…`.
6. **Open the pane** as in step 3, with both `herdr.space=<sp…>` and `herdr.tab=<tb…>`, and `--cwd <folder>`.
7. **Close the starter shell** only after the new pane appears in `herdr-app ls`: `herdr-app do pane.close '{"pane":"<starter pane>"}'`. Closing it first would empty the tab.

## Examples

```
/ha run claude reviewer                              Claude pane "reviewer" in your tab, session ha/<machine>/reviewer
/ha run claude reviewer -m opus -e high review src   with model, effort and a first task
/ha run claude reviewer -s "api review"              session named "api review"
/ha run agy tester -i executor -a ./web run e2e      Antigravity in tab executor, folder <space>/web
/ha run shell ops -i journal/                        new tab "ops" in space journal, shell only
```

## Mistakes to avoid

| Mistake | What happens | Do instead |
|---|---|---|
| Space or tab **name** in a label | Herdr creates a stray space or tab with that id | Resolve ids with `herdr-app ls` |
| Creating a missing space without asking | The user gets a space they did not want | Ask, then `space.new` only after a yes |
| `--argv '["claude"]'` or `hc run … claude` | The agent is the pane's program; when it exits the pane dies | Open a shell, then `hc send` the command |
| Leaving out `--no-parent` | The pane is your child: it dies when you exit and you get exit notes | Always `--no-parent` |
| `hc send <name>` when a finished terminal shares the name | Can resolve to the stale terminal | Use the address from step 3 |
| Typing the session name unquoted | A name with spaces splits into several arguments | Always single-quote it |
| `herdr-app do pane.split` without `pane` | Splits whatever pane the person has focused | Use `hc run` with labels as above |
