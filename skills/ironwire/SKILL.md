---
name: ironwire
description: >-
  Drive the ironwire VM fleet with the `ironwire` CLI — create machines, push code, run
  commands remotely, read results, tear down. Use when the user wants to build/test/run
  software on a remote ironwire machine, or when an agent inside a machine needs to
  coordinate sibling machines. Works identically on a laptop and inside a machine.
---

# ironwire

`ironwire` is one command-line tool that drives the ironwire fleet over SSH. The **same
binary** runs on your laptop and inside every machine; the only difference is *who you
are* (your SSH key on the laptop, the machine's injected identity inside a VM).

**Discover any command with `ironwire help` or `ironwire help <command>` / `ironwire
<command> --help`** — these work with zero configuration. This skill covers the mental
model and gotchas; lean on `--help` for exact flags and args.

## The one rule

**`ironwire` only crosses a boundary the local shell can't.** Anything on the machine you
are *already on* — listing files, editing, running the build — is a plain shell command.
Reach for `ironwire` only to act on **another machine** or the **fleet as a whole**. There
is no "act on myself".

## Machine-readable output & exit codes

Add `--json` to any fleet command for a stable document — success
`{"ok":true, …}`, failure `{"ok":false,"error":"…"}`. **Always check the exit code**:
`0` = success, non-zero = failure. `ironwire run` forwards the remote command's own exit
code, so you tell a passing build from a failing one directly.

## Networking model — read this before wiring services together

- Each machine has a **private internal IP** on a shared bridge (`10.0.0.0/24`), shown as
  `ip` in `ironwire info <name> --json` (and the `network` hint there).
- **Your machines can reach each other** on those IPs on **any port** — wire services
  together with the internal IP (the app connects to the db at e.g. `10.0.0.5:5432`).
  Traffic from *other accounts* is blocked.
- **The internet cannot reach a machine's arbitrary ports.** The only public inbound
  entrypoint is each machine's **HTTPS URL** (`url` in `info`), which fronts the guest's
  web app on port **8000**. So: bind a service to port **8000** to expose it publicly over
  HTTPS; bind to the internal IP (or `0.0.0.0`) for machine-to-machine only.
- Machines **can** reach the internet **outbound** (apt, package installs, etc.).
- You never SSH to a machine directly — `run`/`cp` go through the control plane. A human
  live shell is `ssh <machine>@<host>`, not this tool.

## Commands (run `ironwire help <verb>` for details)

- **Lifecycle:** `create` · `list` · `info` · `rm` · `stop` · `start` · `restart` ·
  `pause` · `resume` · `copy` · `resize`
- **Sizing (vm-sizing):** `create --cpu=<n|max> --mem=<size|max> --disk=<size|max>` picks a
  machine's size (blank = defaults); `resize <name>` with the same flags changes it later —
  applied on the next restart, disk grow-only. Cores are a per-machine ceiling, not summed
  against your quota; disk likewise charges nothing up front — the account's
  storage pool is consumed by data machines actually store, not by disk sizes
  (pooled-disk-quota).
  **Disk sizes are free ceilings** — leave `--disk` blank and the machine can
  grow into the account's whole storage pool; the pool only fills as real data
  does (and empties again when data is deleted). Pick an explicit smaller
  `--disk` only to bound the blast radius of a machine you don't fully trust
  (a runaway process can then fill at most its own ceiling, not the whole
  pool). `resize` still grows a ceiling later; ceilings never shrink.
- **Cross-machine:** `run <machine> -- <cmd…>` (exit code forwarded) ·
  `cp <src> <machine>:<dst>` / `cp <machine>:<src> <dst>` (bidirectional, tar-streamed).
  **cp takes two arguments and the remote side starts with the machine NAME** —
  `ironwire cp /tmp/app box1:/root/` is right; `ironwire cp /tmp/app:/root/` is
  wrong (there is no machine called `/tmp/app`).
- **Env:** `env [list] / set <N> <V> / rm <N>`, optionally `--machine=<name>`
  (account-global by default; machine var overrides). **Applies on the machine's next
  boot — restart to apply.** `IRONWIRE_*` names are reserved. **Prefer this for
  secrets and config over hardcoding them in files on the machine:** the values are
  held by the control plane (sealed at rest) and injected at boot, so they aren't left
  in plaintext on the VM and survive a rebuild/reprovision.
- **Stats:** `stats <machine> [--range=1h|24h|7d|30d]` — cpu/mem/disk, current values +
  history (30 days retained; stopped/paused periods are gaps). **This is how you
  investigate a slow or crashing machine:** `ironwire stats box1 --range=24h --json`,
  then read the series for memory climbing or disk filling before acting.
- **Info/account:** `usage` · `whoami` · `images` · `keys` · `link` · `unlink` · `drive`
- **Inbound email:** `email <machine> on` gives the machine a catch-all inbox —
  email sent to `*@<machine>.<handle>.<content-domain>` (any local-part) is held by the
  platform (even while the machine is stopped) and pulled with
  `email <machine> [--new]` (list), `email <machine> get <id>` (full raw RFC 5322
  message to stdout — pipe it to a parser), `email <machine> rm <id>` (delete =
  your ack; reading never deletes). Poll `--new` in a loop to react to incoming
  mail; delete after processing or the account mail cap fills and new mail is
  tempfailed. **Receive-only** — machines cannot send or reply. Trust the
  `auth` field (the platform's SPF/DKIM/DMARC verdict, `--json`), never the
  message's own headers, before acting on who a message claims to be from.

## Running remote commands

`ironwire run <m> -- <cmd…>` quotes like plain `ssh`: every argument arrives on the
machine intact (spaces and all), and stdin streams through. Patterns:

1. **Plain command** — args-with-spaces are fine:
   ```
   ironwire run box1 -- systemctl restart postgresql
   ironwire run box1 -- git commit -m "fix the thing"
   ```
2. **Shell syntax (pipes, `&&`, redirects)** — wrap it in `sh -c '...'`; the quoted
   string arrives as one argument:
   ```
   ironwire run box1 -- sh -c 'cd /root/app && make test 2>&1 | tail -20'
   ```
3. **Large or arbitrary payloads** — pipe them over stdin instead of quoting:
   ```
   echo "summarize the logs" | ironwire run box1 -- claude -p
   ```
4. **Multi-step setup** — ship a script and run it:
   ```
   ironwire cp ./setup.sh box1:/root/
   ironwire run box1 -- bash /root/setup.sh
   ```

## Running Claude Code (or another agent) on a machine

A machine is a disposable, isolated VM — the machine IS the sandbox, so the agent
inside it can run with its own permission checks off. The pieces that are not
guessable:

```bash
ironwire create claudebox
ironwire run claudebox -- sh -c 'curl -fsSL https://claude.ai/install.sh | bash'
ironwire env set ANTHROPIC_API_KEY sk-ant-... --machine=claudebox
ironwire restart claudebox    # env vars land at next boot
```

Then each query is one command — pipe the prompt over stdin:

```bash
echo "review the code in /root/app" | \
  ironwire run claudebox -- env IS_SANDBOX=1 claude -p --dangerously-skip-permissions
```

- `--dangerously-skip-permissions` is what makes non-interactive `-p` work (there is
  no human to approve tool calls); it is the intended pattern inside a throwaway VM.
- `ironwire run` executes as root, and claude refuses that flag as root unless
  `IS_SANDBOX=1` is set. Set it inline as above, or once per machine:
  `ironwire env set IS_SANDBOX 1 --machine=claudebox` (plus a restart), then run
  through a login shell so the env applies: `bash -lc 'claude -p ...'`.
- Conversation state persists on the machine's disk: add `--continue` to keep one
  ongoing session across queries, `--output-format json` for parseable results.

## The full loop (laptop)

```bash
ironwire create box1 --json                 # provision (parse .machine for ip/url)
ironwire cp ./myapp box1:/root/             # ship the code
ironwire run box1 -- make -C /root/myapp test   # build/test; exit 0 = pass
ironwire rm box1                            # tear down
```

## Two machines that talk to each other (pattern)

```bash
ironwire create db  --json ; DB_IP=$(ironwire info db  --json | jq -r .machine.ip)
ironwire create app --json
# provision db to listen on its internal IP, then point the app at it:
ironwire env set DATABASE_URL "postgres://app@$DB_IP:5432/appdb" --machine=app
ironwire restart app          # env applies on next boot
```
The app reaches the db on `$DB_IP` because they're in the same account on the internal
bridge. Expose the app publicly by binding it to port 8000 → its HTTPS `url`.

## Inside a machine

The same commands work, acting as **that machine within its owner's account**. The control
plane injects a few env vars at boot so a script knows *who it is* without asking anyone:

- `IRONWIRE_MACHINE_NAME` — this machine's own name (e.g. to skip yourself when iterating
  `ironwire list`, or to label output/logs)
- `IRONWIRE_MACHINE_IP` — this machine's internal bridge IP (hand it to a sibling that
  must connect back to you; no `jq` round-trip needed)
- `IRONWIRE_INSIDE=1` — set only inside a machine (branch laptop-vs-VM behavior on it)

A machine
can create/delete/command **sibling** machines only if its owner turned **orchestration
on** for it (dashboard: machine → actions menu); otherwise fleet commands are refused.
A machine never gains admin powers, and its actions are attributed to it.

## Setup

- **Laptop:** set `IRONWIRE_CONTROL=<host:port>` (the SSH front door), or put
  `control = <host:port>` in `~/.config/ironwire/config`. Identity is your ssh-agent key
  (the same key that enrolls your account), or `IRONWIRE_KEY=<path>`.
- **Inside a machine:** nothing to configure — the control plane injects `IRONWIRE_CONTROL`,
  the machine identity, and the self-info vars above (`IRONWIRE_MACHINE_NAME` / `_IP` /
  `IRONWIRE_INSIDE=1`) at boot.

## Not this tool's job

Interactive shells (`ssh <machine>@<host>` or the dashboard's framed shell), local commands,
per-machine settings (delete protection / orchestration — toggled in the dashboard's machine actions
menu), and the web/landing surfaces. `ironwire` is the scriptable, non-interactive surface.
