---
name: manyfold-cli-usage
description: Operate the Manyfold platform and delegate to peer agents on the user's behalf via the mf CLI (channels, automations, skills, files, backups, model config, usage, auth/scopes, and A2A). Run `mf help --agent` for the always-current guide.
version: 0.1.0
---
# Manyfold CLI (`mf`) — agent guide

You are an agent running in a Manyfold-managed runtime. The `mf` CLI acts
on the Manyfold platform on the user's behalf. These env vars are set:

- `MF_API_URL` — Manyfold API base (already includes the `/api` prefix)
- `MF_AGENT_ID` — your agent id
- `MF_DEPLOY_ENV` — Manyfold deployment environment (`staging` |
  `production` | `local`); may be absent on older runtimes

## Quickstart: you are already authenticated

The runtime injects your agent identity token (`MF_API_TOKEN`) and `mf`
reads it automatically — you do not log in for identity. If a command
fails because your identity lacks a scope:

```sh
mf auth ensure --scopes channels:read,channels:edit
```

Request **only the scope you are missing** (existing permissions are
KEPT — approval appends). The CLI prints a consent URL — **post exactly
that URL to the user** and ask them to approve. Never paste any token in
chat. The command exits after printing the URL; once the user approves,
just retry — the platform reads the added scope live. Details:
`mf help auth --agent`.

## Scope: your agent vs the whole account

By default every command is scoped to **your own agent** (`$MF_AGENT_ID`) and
needs **no permission** — list, read and manage your own automations, files,
channels, skills, backups, chat, terminal and runtime freely.

To reach the **whole account** — another agent's resources, or account-level
resources (all agents, model providers, account usage) — add `--account`:

```sh
mf automations list --account                       # across ALL your agents
mf automations list --account --agent-id agt_other  # a specific other agent
mf agent list --account                             # every agent on the account
```

`--account` requires a user-granted scope (e.g. `automations:read`). If your
identity lacks it, the command prints a **consent URL** — post it to the user
to approve, then retry. Targeting another agent **without** `--account` → `403`.

## Topics

- `mf help --agent` — entry guide: auth, topics, failure recovery
- `mf help auth --agent` — login, scopes, consent URL, token safety
- `mf help safety --agent` — hard rules: secrets, consent URL, scope grants
- `mf help channels --agent` — Telegram, Slack, Discord, Lark channel management
- `mf help channels create --agent` — creating a channel step by step
- `mf help automations --agent` — scheduled jobs: create, run, update, delete
- `mf help files --agent` — agent workspace files: list, read, write, mv, rm
- `mf help model-config --agent` — read or update the agent model configuration
- `mf help skills --agent` — install, discover and manage agent skills
- `mf help runtime --agent` — runtime lifecycle, control UI, dashboard
- `mf help agent --agent` — agent CRUD, storage, credentials, logs
- `mf help backups --agent` — agent snapshots: list, create, restore
- `mf help usage --agent` — token and cost statistics
- `mf help a2a --agent` — talk to any A2A server directly (no account needed)

Add `--json` to any `mf help … --agent` call for a machine-readable
envelope (`topic`, `cliVersion`, `topics`, `content`). Most data commands
also accept `--json`. `mf <command> --help` shows human-readable flags.

## Failure recovery

- "not authenticated" → the runtime should already hold `MF_API_TOKEN`;
  if it is genuinely missing, tell the user (`mf help auth --agent`)
- `401` / missing scope → only `--account` (account-wide) actions need a
  scope; your own-agent actions are free. Request just that scope (existing
  ones are KEPT): `mf auth ensure --scopes <missing scope>`, then retry
- `403` → you targeted a different agent without `--account`; act on
  `$MF_AGENT_ID`, or add `--account` for account-wide access (needs a grant)
- unknown flag or command → `mf <command> --help`

## Available grant scopes

agents:read, agents:edit, agent-runtimes:read, agent-runtimes:edit, sandboxes:read, sandboxes:edit, channels:read, channels:edit, automations:read, automations:edit, chat:read, chat:edit, a2a:read, a2a:edit, model-providers:read, model-providers:edit, model-config:read, model-config:edit, secrets:read, secrets:edit, skills:read, skills:edit, backups:read, backups:edit, terminal:read, terminal:edit, files:read, files:edit, usage:read, byo-providers:read, byo-providers:edit

## Safety (always applies)

- Never print `~/.manyfold/config.json` (or any `~/.manyfold/config.*.json` profile),
  legacy `~/.config/mf/config.json` / `~/.config/nca/config.json`, or any
  token value.
- Only share the consent URL with the user — the URL alone is safe.
- Request the minimum scopes the task needs.
- Full rules: `mf help safety --agent`

## Calling peer agents (A2A)

### Purpose

Talk to other agents over the A2A (Agent2Agent) protocol — either a
**granted peer** (another Manyfold agent the user lets you call) or any
**raw A2A server** by URL. The client speaks A2A **v0.3** over JSON-RPC.

A `<target>` is resolved automatically:

- a **peer name or agent id** (from `mf a2a status`) → resolved live from
  the platform using this agent's own login token; a short-lived bearer is
  minted per call, so you never handle a URL or token. Needs the
  `a2a:read` scope. With a **user token** (`mf login`, not an agent
  runtime), pick which of your agents to act as via the global
  `--agent-id <id>` (or `$MF_AGENT_ID`): `mf --agent-id <id> a2a status`.
- an **http(s) URL** → used as a raw A2A server. A base URL discovers the
  card at `/.well-known/agent-card.json`; a `.json` card URL is fetched; a
  `/rpc` or `/a2a` URL is used as-is. Auth via `--bearer` / `$MF_A2A_BEARER`.
  Needs no Manyfold account.

### Commands

```sh
mf a2a status                              # peers you may call + in-flight calls
mf a2a send <target> "<prompt>"            # send a message, wait for the result
mf a2a send <target> "<prompt>" --async    # submit, return a task id, don't block
mf a2a send <target> "<prompt>" --stream   # stream status + artifact chunks (SSE)
mf a2a tasks list                          # your outbound calls and their state
mf a2a tasks get <target> <taskId>         # fetch a task (add --wait to poll)
mf a2a tasks cancel <target> <taskId>      # cancel a running task
mf a2a tasks subscribe <target> <taskId>   # reconnect to a task's SSE stream
mf a2a card <url>                          # print an Agent Card
```

- `mf a2a status` lists the peers you may call (name + agent id), fetched
  live each run — a newly granted peer appears immediately, no restart. It
  also shows your in-flight outbound calls. If it lists no peers, you have
  no grants yet: ask the user to grant one from the target agent's A2A tab.
  `--json` emits `{ peers, inflight }` (no tokens).
- `mf a2a send` delegates a subtask and prints the peer's final answer to
  stdout. The peer runs in its own workspace, billed to its own owner, and
  cannot read your workspace.

### Long tasks: submit async, fetch later

A blocking `send` holds the call open until the peer finishes. For a long
task prefer `--async`: it returns a **task id** immediately, so the work
survives even if your sandbox sleeps while the peer runs.

```sh
id=$(mf a2a send <peer> "<long task>" --async)   # prints the task id to stdout
mf a2a tasks list --state working                # see what's still running
mf a2a tasks get <peer> "$id" --wait             # block until done, print result
```

The result is stored durably on the platform and fetched on demand — **do
not** redirect call output to `/tmp` to keep it; `/tmp` is wiped when the
sandbox hibernates. `tasks get`/`cancel`/`subscribe` take the same
`<target>` you sent to (peer name/id, or url for a raw server).

To continue the same conversation, pass the `context <id>` printed in the
task summary back via `--context-id <id>`; without it each call is a fresh
session with no memory of earlier calls.

### Shared options

- `--bearer <token>` — bearer auth for a raw **url** target (ignored for a
  peer, which mints its own). `"-"` reads stdin; else `$MF_A2A_BEARER` is
  the fallback. Never printed, including on error.
- `--json` — emit raw A2A `Task` / `Message` / stream-event JSON.
- `--allow-http-localhost` — permit `http://` and localhost/private targets
  (local dev only; HTTPS public hosts only by default — SSRF guard).
- `--timeout <seconds>` — client deadline for `send` and `tasks get --wait`
  (`0` disables; default 900). The hosted server enforces its own per-turn
  cap; this is the client-side backstop so the CLI never hangs forever.
- `send` also accepts `--context-id <id>`, `--task-id <id>`, `--skill <id>`,
  and `--input-file <path>` (attached as an A2A file part).

### Output

Human mode writes artifact text to stdout and status/task summaries to
stderr, so stdout stays a clean, pipeable artifact (with `--async`, stdout
is just the task id). `--json` writes raw protocol JSON. Errors print to
stderr as `cli Error: …` and exit 1; tokens are never included.

### Failure recovery

- `no granted peer matching "…"` → run `mf a2a status` to see exact names;
  ask the user to grant the peer if missing.
- `no usable A2A token` / `a2a:read` missing → run
  `mf auth ensure --scopes a2a:read`, post the consent URL to the user
  (existing permissions are kept), retry after they approve.
- `needs an agent context` (user token) → add `--agent-id <id>` for an agent
  you own, e.g. `mf --agent-id <id> a2a status`.
- `too many concurrent A2A delegations` → you have hit the in-flight cap;
  wait for one to finish (`mf a2a tasks list --state working`) and retry.
- `-32001 Task not found` on `tasks get|cancel|subscribe` → the task id is
  unknown to that target or not visible to your credential.
- `unsupported A2A protocolVersion` / `exposes no JSONRPC interface` → a raw
  server speaks something this client does not (only v0.x JSON-RPC).
- `… host … is not allowed` / `private or reserved address` → the url is
  localhost/private; pass `--allow-http-localhost` for local dev.
- `401` / `403` on a url target → provide or fix `--bearer`
  (or `$MF_A2A_BEARER`).
```
