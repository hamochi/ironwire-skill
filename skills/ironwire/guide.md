# ironwire

**ironwire** ([ironwire.sh](https://ironwire.sh)) is a hosting platform built for
agents: it rents you lightweight Linux VMs called **machines**, each with a public
HTTPS URL and root access, in seconds. There is no web console and no password —
your SSH key is your account, and everything (provisioning, control, even the
dashboard) happens over SSH. Machines in one account share a private network and can
orchestrate each other; other accounts are isolated.

`ironwire` is one command-line tool that drives the ironwire fleet over SSH. The **same
binary** runs on your laptop and inside every machine; the only difference is *who you
are* (your SSH key on the laptop, the machine's injected identity inside a VM).

Full platform documentation, agent-readable, lives at
**<https://ironwire.sh/llms-full.txt>** (index: <https://ironwire.sh/llms.txt>) — fetch
it when you need details this skill doesn't cover (exact command reference, quotas,
custom domains, shared drive, email).

**Discover any command with `ironwire help` or `ironwire help <command>` / `ironwire
<command> --help`** — these work with zero configuration. This skill covers the mental
model and gotchas; lean on `--help` for exact flags and args.

## Getting an account

No account yet? One command creates one — no web signup, no password. **The key
you sign up with becomes the account**, so run this as whichever key should own it:

```sh
ironwire signup <handle> --accept-terms
```

It does not wait. It reserves the handle and prints a **payment link** and a
**QR code** of that link. Pass the link to whoever is paying — or show them the
QR, which they can scan and pay on a phone rather than the machine you are on.
The moment the payment lands the account exists and the key is in: just connect,
nothing else to run.

- `--accept-terms` is required — it accepts <https://ironwire.sh/terms> and
  <https://ironwire.sh/privacy>. There is no way to buy without it.
- `--units N` buys a larger pool; omit it for the smallest plan, resizable later.
- Running signup again before payment returns the **same** link. It never opens a
  second checkout and never charges twice.
- Reserved the wrong handle? `ironwire signup --abort` releases it so you can
  retake immediately instead of waiting out the reservation.
- `--json` returns the link, plan and expiry as fields.

You don't need the CLI installed to sign up — plain SSH does the same thing:

```sh
ssh ironwire.sh signup <handle> --accept-terms
```

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
  applied on the next restart, disk grow-only. `resize <name> --restart=<always|on-failure|never>`
  changes the crash-recovery policy in place (live). Cores are a per-machine ceiling, not summed
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
  **`<dst>` works like plain `cp`:** an existing directory receives the copy under the
  source's own name, anything else names it — `ironwire cp ./agents-e2e.yaml
  box1:/root/agents.yaml` renames on the way in. The destination's parent must exist.
- **Env:** `env [list] / set <N> <V> / rm <N>`, optionally `--machine=<name>`
  (account-global by default; machine var overrides). **Applies on the machine's next
  boot — restart to apply.** `IRONWIRE_*` names are reserved. **Prefer this for
  secrets and config over hardcoding them in files on the machine:** the values are
  held by the control plane (sealed at rest) and injected at boot, so they aren't left
  in plaintext on the VM and survive a rebuild/reprovision.
- **Egress proxies:** `proxy [list] / add <label> --target <host> --header 'N: v' /
  attach <label> <machine> / detach / set-header / rm`. **The strongest option for
  API keys — the secret never enters the VM at all:** attached machines call
  `$PROXY_<LABEL>` (e.g. `curl $PROXY_STRIPE/v1/...`) and the platform injects the
  stored header in flight. Declare SDK envs on the proxy
  (`--env 'OPENAI_BASE_URL=$PROXY_URL/v1' --env 'OPENAI_API_KEY=unused'`) so stock
  SDKs work unmodified. Attach/detach is live; the env vars land on next restart.
  From inside a machine you can `proxy list` (values redacted) but never mutate —
  proxy management is owner-only, from a laptop or the dashboard.
- **Stats:** `stats <machine> [--range=1h|24h|7d|30d]` — cpu/mem/disk, current values +
  history (30 days retained; stopped/paused periods are gaps). **This is how you
  investigate a slow or crashing machine:** `ironwire stats box1 --range=24h --json`,
  then read the series for memory climbing or disk filling before acting.
- **Info/account:** `usage` · `whoami` · `images` · `keys` · `link` · `unlink` · `drive`
- **Inbound email:** `email <machine> on` gives the machine a catch-all inbox —
  email sent to `*@<machine>-<handle>.<content-domain>` (any local-part) is held by the
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
  `ironwire env set IS_SANDBOX 1 --machine=claudebox` (plus a restart); managed
  env vars are present in every ssh session, so plain `ironwire run` sees them.
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

- **Laptop:** works out of the box — the CLI dials the platform's SSH front door
  by default. Identity is your ssh-agent key
  (the same key that enrolls your account), or `IRONWIRE_KEY=<path>`. To act as one
  specific key, pass `--identity <keyfile>` (`-i`) or export the raw private-key PEM as
  `IRONWIRE_IDENTITY` — an explicit key is used exclusively, with no agent/default
  fallback (flag wins over env).
- **Inside a machine:** nothing to configure — the control plane injects the machine
  identity and the self-info vars above (`IRONWIRE_MACHINE_NAME` / `_IP` /
  `IRONWIRE_INSIDE=1`) at boot.

## Not this tool's job

Interactive shells (`ssh <machine>@<host>` or the dashboard's framed shell), local commands,
per-machine settings (delete protection / orchestration — toggled in the dashboard's machine actions
menu), and the web/landing surfaces. `ironwire` is the scriptable, non-interactive surface.
