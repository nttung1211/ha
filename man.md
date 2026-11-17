# ha: user guide

Print this file to the user verbatim.

```
ha: work with terminals and agents in the Herdr desktop app

USAGE
  /ha run  <agent> <name> [-i [space/]tab] [-a dir] [-m model] [-e effort] [task]
  /ha send <to> [-w] [-r] <text>
  /ha man

RUN   open a pane with a shell, then type the agent command into it
  <agent>            claude | agy | codex | grok | shell
  <name>             name for the new terminal (a-z 0-9 - _, up to 32); used by send
  -i, --in           where: tab (in your space) | space/tab | space/ (new tab)
                     default: your own tab; a missing tab is created;
                     a missing space: asks whether to create it, with folder
                     <current dir>/<space> (created too if missing);
                     new space names: A-Z a-z 0-9 - _ only
                     a bare name (test) is always a tab; write test/ for a space
  -a, --at           working folder; default: the space's folder; relative to it
  -m, --model        model flag passed to the agent
  -e, --effort       effort flag passed to the agent
  task               first message, sent verbatim once the agent is ready

SEND  send a message to a terminal started by ha run
  <to>               the name given to ha run (nothing else is accepted)
  -w, --wait         wait for its answer, then summarize it in a few sentences
  -r, --reply        ask it to reply to you with hc send
  <text>             the message, sent verbatim

EXAMPLES
  /ha run claude reviewer
  /ha run claude reviewer -m sonnet -e low review src/api.ts
  /ha run agy tester -i executor -a ./web run the e2e tests
  /ha run shell ops -i journal/
  /ha send reviewer -w is the fix correct?
  /ha send tester -r what failed?

NOTES
  Panes from run have no parent: they stay open after the caller exits.
  Replies arrive as <hc from="t7 name">…</hc>.
  Codex and Grok flags are unverified; the agent checks --help first.
```
