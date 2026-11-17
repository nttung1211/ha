# ha send

Send a message to a terminal that `ha run` started.

```
ha send <to> [-w|--wait] [-r|--reply] <text>
```

| Part | Meaning |
|---|---|
| `<to>` | Required. The name given to `ha run` |
| `-w`, `--wait` | Wait until the recipient finishes the turn this message starts, then summarize its answer |
| `-r`, `--reply` | Ask the recipient to reply to you with `hc send` |
| `<text>` | Required. Everything after the flags, sent verbatim |

## Steps

1. **Find the recipient.** Run `hc ls --json --label ha=1` and take the terminal whose `name` equals `<to>`. Only these count: no tab names, no addresses, no terminals without the `ha=1` label. No match: stop and say so, listing the names that do exist (or that there are none). Use its `address` (`tN`) for everything below, never the name.
2. **Refuse yourself.** If the address is `$HC_TERMINAL_ID`, stop.
3. **Check its state** with `hc get <address> --json`. If `status.state` is `blocked`, do not send and never use `--force`: `hc read <address> --lines 0` and tell the user what the screen asks.
4. **Build the text.** Use `<text>` verbatim. With `-r`, append on the same line: ` (Reply with: hc send <your $HC_TERMINAL_ID> "<your reply>")`.
5. **Send.**
   - Without `-w`: `hc send <address> "<text>"`.
   - With `-w`: `hc send --wait --timeout 600 <address> "<text>"`. If it returns with `reason` `timeout` or `message`, continue with `hc wait <address> --timeout 600` until the turn ends (`done`, `idle`, `blocked`, `error` or exit). Then `hc read <address> --lines 120` and summarize the recipient's answer to this message in two to four sentences. If it ended `blocked` or `error`, say that instead.
   - A `shell` recipient has no agent status, so `-w` cannot tell when it is done: send without waiting and say so.
6. **Check delivery.** The reply must show `"started": true`. Otherwise `hc read <address> --lines 0`, report what you see, and do not resend: the text may already have arrived.
7. **Report** in one short block: recipient name, address, `space / tab` (the `where` label), whether it was delivered, whether it was busy (`working`, so it queues the message), and with `-w` the summary.

## Examples

```
/ha send reviewer review src/api.ts again
/ha send reviewer -w is the fix correct?
/ha send tester -r what failed in the last e2e run?
```
