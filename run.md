# ha run

Open a new pane holding a shell, then type the agent command into it, exactly as a person opens a pane and types `claude`. When the agent exits, the shell stays.

```
ha run <agent> <name> [-i|--in [space/]tab] [-a|--at dir] [-m|--model M] [-e|--effort E] [task…]
```

| Part | Meaning |
|---|---|
| `<agent>` | Required. `claude`, `agy`, `codex`, `grok` or `shell` |
| `<name>` | Required. The new terminal's hc name, used later to address it. 1–32 chars of `a-z 0-9 - _` |
| `-i`, `--in` | Where the pane goes (table below). Default: your own tab |
| `-a`, `--at` | Working folder. Default: the target space's `folder`. A relative path resolves against that folder |
| `-m`, `--model` | Passed to the agent as its model flag |
| `-e`, `--effort` | Passed to the agent as its effort flag |
| `task…` | Everything after the flags. Sent to the agent verbatim once it is ready. Add nothing to it |

`--in` resolution (names compared trimmed, your own space checked first):

| Written | Means |
|---|---|
| (absent) | Your own tab |
| `-i planner` | Tab `planner` in your space |
| `-i antrail/planner` | Tab `planner` in space `antrail` |
| `-i antrail/` | A new tab in space `antrail`, named `<name>` |
| a tab name that does not exist (space found) | A new tab with that name in that space |

Two tabs with the same name in the target space, or a space name matching two spaces: ask which one.

If your own terminal is not in `herdr-app ls` (a hidden background worker), you have no space or tab: `--in` is required, and a bare tab name is searched in every space. Exactly one match: use it. Several or none: ask.

## Steps

1. **Parse and validate.** `<name>` must match `^[a-z0-9_-]{1,32}$`. Run `hc ls --all --json`: if a **live** terminal already has that name, stop and ask for another. If only finished terminals have it, continue, but address the new terminal by its address (`tN`) afterwards and mention the clash in the report.
2. **Resolve ids** from `herdr-app ls`: your space and tab (see SKILL.md), then the target space id `sp…` and, unless a new tab is needed, the target tab id `tb…`. Resolve the folder from `--at` and check it exists.
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
   | `claude` | `claude [--model M] [--effort E]` |
   | `agy` | `agy [--model M] [--effort E]` |
   | `codex` | `codex --no-daemon …`. hc requires `--no-daemon`. Check `codex --help` for the model and effort flags |
   | `grok` | `grok …`. Check `grok --help` for the model and effort flags |

   ```sh
   hc send <address> "<command>"
   hc wait <address> --timeout 90
   hc get <address>        # expect status.app set and state idle
   ```

   If `wait` returns because a message arrived or it timed out, check `hc get` and wait again. If the agent is `blocked` (trust prompt, login, permission), `hc read <address>` and tell the user what it shows. Do not answer the prompt for them.

5. **Send the task**, if any, verbatim: `hc send <address> "<task>"`. Confirm the reply shows `"started": true`; otherwise `hc read` before deciding anything. Never resend blindly.

6. **Report** in one short block: name, address, `space / tab`, folder, the command typed, and whether a task was sent.

## Examples

```
/ha run claude reviewer                              Claude pane "reviewer" in your tab
/ha run claude reviewer -m opus -e high review src   with model, effort and a first task
/ha run agy tester -i executor -a ./web run e2e      Antigravity in tab executor, folder <space>/web
/ha run shell ops -i journal/                        new tab "ops" in space journal, shell only
```

## Mistakes to avoid

| Mistake | What happens | Do instead |
|---|---|---|
| Space or tab **name** in a label | Herdr creates a stray space or tab with that id | Resolve ids with `herdr-app ls` |
| `--argv '["claude"]'` or `hc run … claude` | The agent is the pane's program; when it exits the pane dies | Open a shell, then `hc send` the command |
| Leaving out `--no-parent` | The pane is your child: it dies when you exit and you get exit notes | Always `--no-parent` |
| `hc send <name>` when a finished terminal shares the name | Can resolve to the stale terminal | Use the address from step 3 |
| `herdr-app do pane.split` without `pane` | Splits whatever pane the person has focused | Use `hc run` with labels as above |
