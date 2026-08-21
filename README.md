# claude-code-openai-server

Use **Claude Code** from any OpenAI-compatible client.

A small HTTP server that makes the `claude` CLI look like an OpenAI API (the way
LM Studio or Ollama do). Point any OpenAI client — the OpenAI Python SDK,
Open WebUI, etc. — at it and you get Claude Code, using your existing
`claude login` (no API key, no per-token billing).

Under the hood it drives `claude` as a persistent subprocess over its
`stream-json` protocol, one process per conversation.

## Project idea

The core idea is simple: **treat the `claude` CLI as a model backend.** The
`claude` CLI already knows how to authenticate (via your existing `claude login`),
reason, and run tools — but it only speaks its own `stream-json` protocol. This
server wraps that protocol in the OpenAI `chat/completions` contract, so any
client that already talks OpenAI (the Python SDK, Open WebUI, LM Studio-style
frontends, etc.) can use Claude Code as if it were a local model — no API key,
no per-token billing.

Each conversation owns one long-lived `claude` subprocess. When the client sends
a request, the server folds the conversation into the subprocess; when Claude
calls one of the client's own tools, the server parks the conversation and hands
the `tool_calls` back to the client, which runs the tool and sends the result
back — the same subprocess then resumes. The whole multi-step tool loop is one
persistent conversation.

## What's different from the original

This repository is a fork of
[`schmarta/claude-code-openai-server`](https://github.com/schmarta/claude-code-openai-server)
by **Lucas Marta** (`lucas.marta0799@gmail.com`). The core architecture (OpenAI
surface, in-process MCP bridge, persistent per-conversation subprocess, bare
model mode, warm pool) is the original's. This fork diverges in two directions:

**Added in this fork**

- **`.env` auto-loading.** `Settings` now reads a `.env` file directly
  (`env_file=".env"` in [`app/config.py`](app/config.py)), and
  [`app/main.py`](app/main.py) calls `load_dotenv(override=False)` at import so
  that *non-`CCI_`* variables in `.env` (e.g. `CLAUDE_CODE_OAUTH_TOKEN`) are
  exported into the environment and inherited by the spawned `claude`
  subprocess. Real environment variables always win (`override=False`). This is
  what makes the systemd `EnvironmentFile=.env` setup work end-to-end.
- **IDE noise ignored.** `.gitignore` now excludes `.idea/`, `.junie/`, and
  `*.iml`.

**Upstream features now included**

This fork was based on an earlier snapshot of the upstream project (up to PR #7,
the uv migration). Upstream `main` has since been merged in, bringing:

- **Native image passthrough** — base64 `image_url` parts become native Claude
  image blocks (on the live turn and in folded history).
- **Cross-turn session reuse** (`CCI_CROSS_TURN_REUSE`) — a completed
  conversation is parked and re-adopted by a matching later turn, cutting
  per-turn history re-billing.
- **Rebuild-on-expiry** (`CCI_REBUILD_ON_EXPIRY`) — an expired tool continuation
  is rebuilt from the request's full history instead of returning HTTP 409.
- **Unguessable conversation ids + `/mcp` loopback-peer gate** — defense in
  depth for the MCP mount.
- **Fail-loudly model resolution** — an unknown `model` id returns 400 instead
  of silently falling back to the default.
- **Alias-based default model** — `CCI_DEFAULT_MODEL` defaults to `opus` (an
  alias the CLI resolves to the current model) instead of a dated concrete id.

## What it does

- **OpenAI surface:** `GET /v1/models` and `POST /v1/chat/completions`
  (streaming + non-streaming).
- **OpenAI function calling:** your client's `tools` are passed through to Claude
  via an in-process MCP bridge. Claude calls them, the server returns a normal
  `tool_calls` response, your client runs the tool and sends the result back, and
  the same Claude subprocess resumes — the whole multi-step tool loop is one
  conversation kept alive across continuations.
- **Claude's own built-in tools** (Read/Edit/Bash/…) run internally the whole
  time (unless bare mode strips them — see below).
- **Images:** base64 `image_url` parts become native Claude image blocks — on the
  live turn and in folded history alike, so a picture stays visible for the
  follow-up turns that ask about it.
- **Bare model mode** (default on): presents Claude as a plain model fronted by
  your client — replaces the system prompt, drops Claude Code's dynamic context,
  and exposes only the tools your client sends.
- **Markdown table flattening** (default on): rewrites pipe tables to fenced
  ASCII so they render in clients that don't support Markdown tables (Discord).
- **Ollama / llama.cpp compatibility shims** so capability-probing frontends
  (Open WebUI, etc.) accept the server.

## Requirements

- The `claude` CLI on your `PATH`, already logged in (`claude login`).
- [uv](https://docs.astral.sh/uv/) — `brew install uv`, or
  `curl -LsSf https://astral.sh/uv/install.sh | sh`.
- Python 3.11+ — uv downloads one if you don't have it.

## Install

```bash
uv sync                # runtime deps, into ./.venv
uv sync --extra dev    # + pytest & friends, for the test suite
```

Nothing else to activate: every command below is `uv run …`, which syncs first
and runs inside that venv.

## Run

**Quick start**

```bash
git clone <this-repo> && cd claude-code-openai-server
uv sync                                   # install runtime deps into ./.venv
cp .env.example .env                      # then edit .env (TTLs, model, workdir…)
uv run uvicorn app.main:app --host 127.0.0.1 --port 8787
```

The server binds `127.0.0.1:8787` by default and is ready immediately — no
database, no migration. A `.env` file in the working directory is loaded
automatically (see Configuration), so the command above picks up your config
with no extra flags.

Or via the installed console script, which pins the fast event loop
(uvloop + httptools) explicitly:

```bash
uv run claude-code-interface
```

Configuration is entirely through `CCI_*` environment variables (or a `.env`
file, which is auto-loaded). See the full list below.

## Configuration

Copy `.env.example` to `.env` and edit, or export the vars directly. Every field
is overridable via `CCI_<FIELD>`.

```bash
# claude-code-interface configuration (env prefix CCI_). Copy to .env or export.

# ── HTTP server ──────────────────────────────────────────────────────────────
CCI_HOST=127.0.0.1
CCI_PORT=8787

# ── Auth ─────────────────────────────────────────────────────────────────────
# Optional bearer token gating every /v1 request. Safe to leave UNSET only on a
# loopback bind. The server REFUSES to start on a non-loopback host unless this
# is set — it drives Claude with bypassPermissions, so an open bind with no key
# is remote code execution for anyone who can reach the port.
# CCI_API_KEY=choose-a-long-random-secret

# ── Claude CLI ───────────────────────────────────────────────────────────────
CCI_CLAUDE_BIN=claude
# An alias (opus/sonnet/haiku/fable/…) is preferred over a dated concrete id: the
# CLI resolves it to the current model, so it never goes stale. An unrecognized,
# non-`claude*` model on a request is now a 400 (no silent downgrade).
CCI_DEFAULT_MODEL=opus
# CCI_DEFAULT_EFFORT=high
CCI_PERMISSION_MODE=bypassPermissions
CCI_ENABLE_TOOL_SEARCH=false

# ── Bare model mode ──────────────────────────────────────────────────────────
# true (default): strip Claude Code's identity + native tools so it behaves as a
# plain model fronted by your client — the request's system message REPLACES
# claude's default prompt, dynamic context (env/git/identity) is dropped, and
# only the MCP tools your client passes survive. false = legacy append + full
# native tool set.
CCI_BARE_MODEL_MODE=true
# CCI_BARE_MODEL_SYSTEM_PROMPT=You are a helpful AI assistant.

# ── Output formatting ────────────────────────────────────────────────────────
# Rewrite Markdown pipe tables to fenced monospace ASCII so they render in
# clients that don't support Markdown tables (e.g. Discord).
CCI_FLATTEN_MARKDOWN_TABLES=true

# ── Workspace (Claude's --add-dir) ───────────────────────────────────────────
# Per-request `workdir` overrides must resolve under one of
# CCI_ALLOWED_WORKDIR_ROOTS (a JSON list); empty => only the default is allowed.
CCI_DEFAULT_WORKDIR=~/cci-workspace
# CCI_ALLOWED_WORKDIR_ROOTS=["/home/user/Projects"]

# ── MCP bridge ───────────────────────────────────────────────────────────────
# Path the in-process MCP server (your client's functions) mounts at; the spawned
# claude dials it back over loopback. Rarely needs changing.
# CCI_MCP_PATH_PREFIX=/mcp

# ── Lifecycle (seconds) ──────────────────────────────────────────────────────
CCI_REQUEST_TIMEOUT_S=600
CCI_SUSPENDED_TTL_S=300
CCI_IDLE_SESSION_TTL_S=900
CCI_GC_INTERVAL_S=30

# ── Warm subprocess pool ─────────────────────────────────────────────────────
# Pre-spawned, idle `claude` procs kept ready to adopt on a fresh turn, removing
# cold-start latency. 0 = disabled (default; ships dark). 1–2 is optimal for a
# single user; larger pools waste RAM (~200 MB per idle proc) and can spike tail
# latency through refill contention. See "Warm pool" below.
CCI_WARM_POOL_SIZE=0

# ── Logging ──────────────────────────────────────────────────────────────────
CCI_LOG_LEVEL=INFO
# Emit per-turn latency metrics (spawn_ms / ttft_ms / total_ms / tok_per_s) to
# the "cci.timing" logger at INFO. Off by default so normal operation pays
# nothing; scripts/bench.py turns it on for the throwaway benchmark instance.
CCI_TIMING_LOG=false
```

### Config reference

| Var | Default | Meaning |
|-----|---------|---------|
| `CCI_HOST` | `127.0.0.1` | Bind host. Non-loopback requires `CCI_API_KEY` (see Security). |
| `CCI_PORT` | `8787` | Bind port. Also used to build the per-conversation MCP callback URL. |
| `CCI_API_KEY` | _(unset)_ | Bearer token required on every `/v1` request when set. Mandatory for a non-loopback bind. |
| `CCI_CLAUDE_BIN` | `claude` | Path to the `claude` CLI. |
| `CCI_DEFAULT_MODEL` | `opus` | Model used when the request names none. An unrecognized, non-`claude*` model on a request is a 400 (no silent downgrade). |
| `CCI_DEFAULT_EFFORT` | _(unset)_ | Reasoning effort passed to `--effort` (e.g. `high`). |
| `CCI_PERMISSION_MODE` | `bypassPermissions` | Claude CLI permission mode. |
| `CCI_ENABLE_TOOL_SEARCH` | `false` | When false, tool schemas are injected directly (always visible) instead of behind tool-search. |
| `CCI_BARE_MODEL_MODE` | `true` | Strip Claude Code identity + native tools; behave as a plain model. |
| `CCI_BARE_MODEL_SYSTEM_PROMPT` | `You are a helpful AI assistant.` | Fallback system prompt in bare mode when a request sends none. |
| `CCI_FLATTEN_MARKDOWN_TABLES` | `true` | Rewrite pipe tables to fenced ASCII. |
| `CCI_DEFAULT_WORKDIR` | `~/cci-workspace` | Working dir granted to Claude via `--add-dir`. |
| `CCI_ALLOWED_WORKDIR_ROOTS` | `[]` | JSON list of extra roots a per-request `workdir` may resolve under. |
| `CCI_MCP_PATH_PREFIX` | `/mcp` | Mount path for the in-process MCP bridge. |
| `CCI_CROSS_TURN_REUSE` | `false` | Park a completed conversation and re-adopt it by a matching later turn (sends only the new message, cutting per-turn history re-billing). |
| `CCI_REQUEST_TIMEOUT_S` | `600` | Per-turn upstream timeout. |
| `CCI_SUSPENDED_TTL_S` | `300` | How long a tool-suspended conversation may wait for its results before GC. |
| `CCI_IDLE_SESSION_TTL_S` | `900` | Idle conversation eviction age. |
| `CCI_GC_INTERVAL_S` | `30` | GC sweep interval. |
| `CCI_REBUILD_ON_EXPIRY` | `true` | Rebuild a fresh session from the request's history when tool results arrive for an already-reaped conversation, instead of answering 409. |
| `CCI_WARM_POOL_SIZE` | `0` | Pre-spawned idle `claude` procs (cold-start removal). 0 = off. |
| `CCI_LOG_LEVEL` | `INFO` | Log level. |
| `CCI_TIMING_LOG` | `false` | Emit per-turn latency metrics to the `cci.timing` logger. |

## Security

The server drives the Claude CLI with `permission_mode=bypassPermissions` by
default, which means **anyone who can reach the port can run arbitrary code as
your user.** Two interlocks guard this:

1. **Non-loopback bind requires an API key.** `create_app()` refuses to start
   when `CCI_HOST` is not a loopback address and `CCI_API_KEY` is unset:

   ```
   RuntimeError: refusing to start: host='0.0.0.0' is not loopback and no
   api_key is set — set CCI_API_KEY to require a bearer token, or bind to 127.0.0.1
   ```

2. **Bearer auth on `/v1`.** When `CCI_API_KEY` is set, every `/v1` request must
   carry `Authorization: Bearer <key>` (constant-time compared). The MCP mount is
   intentionally exempt — it is reached only by the local Claude subprocess over
   loopback and carries no token.

**Recommended:** keep the bind on `127.0.0.1`. If you must expose it (containers,
remote clients), set a long random `CCI_API_KEY` and put it behind TLS. Binding
`0.0.0.0` / a public interface with no key will refuse to boot — by design.

## Troubleshooting: 409 errors & TTLs

If a client gets **HTTP 409** (`code: tool_result_expired`) in the middle of a
tool loop, the conversation was garbage-collected while a tool call was still in
flight. This happens when a single blocking tool call (a subagent run, a long
kanban/automation step, browser automation, etc.) takes longer than the
suspended-conversation TTL, so the server reaps the parked conversation before
the client returns the tool result.

Two lifecycle knobs control this (both in seconds, both overridable via `.env` —
no code change needed):

| Var | Default | Meaning |
|-----|---------|---------|
| `CCI_SUSPENDED_TTL_S` | `300` | How long a tool-suspended conversation may wait for its results before GC reaps it. **This is the one that causes 409s when too low.** |
| `CCI_IDLE_SESSION_TTL_S` | `900` | How long an idle (non-suspended) conversation is kept alive before eviction. |

**Recommended fix:** bump both to `3600` (1 hour). That comfortably covers even
the longest single blocking tool call (e.g. browser automation's 30-minute max),
so a slow in-flight tool no longer outlives its conversation:

```bash
# in .env
CCI_SUSPENDED_TTL_S=3600
CCI_IDLE_SESSION_TTL_S=3600
```

Then **restart the service** to pick up the new values. If you run under a
systemd user unit (see [Deploying as a service](#deploying-as-a-service-systemd-user-unit)),
the unit reads config from `.env` via `EnvironmentFile` and has `Restart=always`,
so restart it cleanly rather than killing the process:

```bash
systemctl --user restart cci-server.service
```

(Use your actual unit name if it differs from `cci-server.service`.) A plain
`kill` works too, but `Restart=always` will respawn the process from the old
environment unless you restart the unit.

> **Note:** `CCI_REBUILD_ON_EXPIRY` (default on) rebuilds an expired continuation
> from the request's full history instead of returning 409, so with it enabled
> the TTLs above are a secondary safety net rather than the primary lever. Set
> `CCI_REBUILD_ON_EXPIRY=false` to restore the legacy 409 behaviour.

## Endpoints

**OpenAI**
- `POST /v1/chat/completions` — chat (streaming + non-streaming, tool calling)
- `GET /v1/models` — model list

**Health / info**
- `GET /healthz` — `{"status":"ok",…}` (a JSON 404 here means the server is up but the route moved)
- `GET /` — service info

**Ollama / llama.cpp compatibility** (for capability-probing frontends)
- `GET /api/tags`, `POST /api/show`, `GET /api/version`, `GET /version`
- `GET /v1/props`, `GET /props`, `GET /api/v1/models`

**Internal**
- `/mcp/<conv_id>` — in-process MCP bridge, dialed only by the spawned Claude over loopback

## Use it

Any OpenAI client, `base_url = http://127.0.0.1:8787/v1`, any `api_key` when none
is configured:

```bash
curl http://127.0.0.1:8787/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"opus","messages":[{"role":"user","content":"hello"}]}'
```

With `CCI_API_KEY` set, add the bearer header:

```bash
curl http://127.0.0.1:8787/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $CCI_API_KEY" \
  -d '{"model":"opus","messages":[{"role":"user","content":"hello"}]}'
```

From the OpenAI Python SDK:

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8787/v1", api_key="any-string-accepted")
print(client.chat.completions.create(
    model="opus", messages=[{"role": "user", "content": "hello"}]
).choices[0].message.content)
```

Models: `opus`, `sonnet`, `haiku`, `fable`, `opusplan` (or any `claude-*` id).
Unknown ids fall back to `CCI_DEFAULT_MODEL`; the CLI resolves an alias like
`opus` to its current concrete version.

## Warm pool

A fresh turn normally pays the full cold start: fork the `claude` Node process,
let it boot, handshake the MCP bridge, then produce a first token (~2.5 s TTFT on
modest hardware). `CCI_WARM_POOL_SIZE=N` keeps `N` pre-spawned idle procs ready
to adopt, lifting that cost off the request's critical path.

- The pool holds **one signature at a time** (model + effort + workdir + system
  prompt + tool set). A request whose signature matches a pooled proc adopts it;
  anything else cold-spawns. The pool re-targets to live traffic.
- Each adopted turn kicks a **background refill**. Under back-to-back load that
  refill (a fresh Node boot) contends for CPU with the turn it's serving and can
  spike tail latency; with normal human-gapped traffic the refill finishes in the
  idle gap and you get the win cleanly.
- Each idle proc costs ~200 MB RAM. **1–2 is optimal for a single user.** Larger
  pools waste memory and amplify refill thrash when signatures interleave.

Disabled by default (`0`), so it ships dark.

## Deploying as a service (systemd user unit)

Example `~/.config/systemd/user/cci-server.service`:

```ini
[Unit]
Description=Claude Code OpenAI Server (CCI)
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/youruser/claude-code-openai-server
EnvironmentFile=/home/youruser/claude-code-openai-server/.env
Environment=PATH=/home/youruser/.local/bin:/usr/local/bin:/usr/bin:/bin
Environment=HOME=/home/youruser
ExecStart=/home/youruser/claude-code-openai-server/.venv/bin/python -m uvicorn app.main:app --host 127.0.0.1 --port 8787
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now cci-server.service
loginctl enable-linger "$USER"   # so it keeps running after you log out (headless)
```

Notes:
- The unit reads config from `.env` via `EnvironmentFile`.
- `ExecStart` points straight at the venv `uv sync` built — no `uv run` on the
  boot path, so a service start never waits on a dependency resolve.
- It launches `python -m uvicorn` directly (not the `claude-code-interface`
  console script), and uvicorn already auto-selects uvloop when installed. To pin
  the fast loop deterministically, add `--loop uvloop --http httptools` to
  `ExecStart` or switch it to the console script.
- There is no `--reload`: editing files on disk does not affect the running
  process until you restart the unit.

## Test

```bash
uv run pytest -q                        # unit tests (no CLI needed)
uv run tests/scripts/e2e_autonomous.py  # live: text, needs a running server
uv run tests/scripts/e2e_tool.py        # live: full tool loop
```

### Benchmark

`scripts/bench.py` launches its **own** throwaway uvicorn instance (never touches
a running server), fires representative autonomous / tool / continuation turns
with `CCI_TIMING_LOG=1`, and writes p50/p95 of spawn / TTFT / total / throughput
to JSON:

```bash
uv run scripts/bench.py --port 8799 --iters 3 --out bench.json
# compare warm pool vs cold:
CCI_WARM_POOL_SIZE=2 uv run scripts/bench.py --port 8799 --iters 6
```

It sets `CCI_PORT` (not just `--port`) so the per-conversation MCP callback URL
the spawned `claude` dials self-matches the throwaway port.

## How it works

```
OpenAI client ──HTTP /v1──▶ cci-server ──stream-json (stdin/stdout)──▶ claude CLI
       ▲                         │                                         │
       └──── tool_calls ─────────┤            mcp__client__* tool call     │
       │                         │◀──────── in-process MCP bridge ◀────────┘
       └──── tool result ───────▶┘            (/mcp/<conv_id>, loopback)
```

- **Autonomous turn** (no `tools`): the conversation is folded into one user turn
  for a fresh subprocess; streamed text becomes SSE or a single JSON completion.
- **Tool turn:** a conversation owns one subprocess + an MCP bridge. When Claude
  calls a tool, the subprocess blocks inside the MCP call; the server returns a
  `tool_calls` response and parks the conversation `SUSPENDED`. The next request
  (carrying the tool results) resolves the pending futures and the same
  subprocess resumes. Matching is by `tool_call_id`.
- A background GC reaps suspended-too-long and idle conversations.
- **Expired continuation:** if the tool results come back after their
  conversation was reaped (a slow tool, a long pause, a server restart), the
  server rebuilds a fresh session from the history the request already carries
  and answers the turn normally, rather than failing it with a 409 the client
  cannot retry. `CCI_REBUILD_ON_EXPIRY=false` restores the 409.
