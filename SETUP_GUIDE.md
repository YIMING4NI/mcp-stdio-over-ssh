# SETUP_GUIDE — stdio-over-SSH MCP Server

> **Last updated**: 2026-04-25
> **Status**: Final.
> **Companion**: `LESSONS_LEARNED.md` (failure post-mortems referenced inline).

---

## Part 0 — Who this is for

This document is a **step-by-step setup guide**, not an API spec, not a design
doc, not a marketing post. The two readers I had in mind while writing:

1. **A new engineer joining the project from zero**, who needs to stand up an
   MCP server end to end: open a fresh laptop, install nothing presumed, get
   to a working `tools/list` in under an hour.
2. **Future-me, six months from now**, who has forgotten which flag was load
   bearing, why the wrapper has an `exec`, and which reload path applies to
   which kind of edit.

**Assumed background.** You can drive a Linux shell, you have written Python
before, and you have used `ssh` to log into a server. You have **not** set up
an MCP server before. Anything you would have learned by deploying one is
written out below — not assumed.

**Not in scope.** This guide does not cover:

- Writing an MCP server from scratch (use the Anthropic Python SDK quickstart
  instead).
- Hardening for multi-tenant production.
- Monitoring, alerting, on-call playbooks (those live with the service).
- Specific Tier 1/2/3 governance content (covered separately, see Part 6).

If a step in here doesn't work for you, skip ahead to Part 5 (Trap clinic)
before re-reading the setup steps. Most failures are one of six known traps,
and the trap clinic is the fastest path back.

---

## Part 1 — Why this shape (motivation and fit)

The constraint this guide answers:

> A remote Linux server holds the data, services, and governance state.
> The Mac runs the chat UI. We want the MCP server **on the remote**, the
> client on the Mac, and we won't operate a public HTTP endpoint just to
> bridge them.

That single constraint rules out three of the four mainstream options.
The remaining one — `command="ssh ..."` in the desktop config — is what
this document covers.

### 1.1 Why **not** a remote HTTP / SSE connector

This is the documented Anthropic pattern for "MCP server on a remote
machine." It works. It is also operationally heavy:

- A public port (or reverse proxy) on the remote.
- A TLS certificate, with renewal automation.
- A bearer token, issued and rotated.
- Whatever ACL / firewall configuration that implies.

For a single-user setup with existing SSH trust, the HTTP route adds
permanent maintenance with no matching benefit: data already flows over
SSH, and adding HTTP doubles the surface area (audit, network, secrets)
without halving the work. There's also a more philosophical reason —
**governance lives on the server**, and an HTTP port adds a transport
surface (TLS, auth) on top of the application surface (governance).
With stdio over SSH, the application is the only surface to manage.

### 1.2 Why **not** the "ssh-mcp" npm packages

A naive search for "MCP" + "SSH" returns roughly seven npm packages
(`tufantunc/ssh-mcp`, `mixelpixx/ssh-mcp`, `jasondsmith72/ssh-mcp`,
`AiondaDotCom/ssh-mcp`, `Machine-To-Machine/m2m-mcp`, `t-suganuma/ssh-mcp`,
plus a handful of forks). They sound like exactly what we want.

**They are not.** All seven share the same architecture: an MCP server
runs **on the local Mac** and exposes "ssh into a remote and run a
command" as one of its tools. The remote machine sees ad-hoc shell
sessions, not a long-lived MCP server holding governance state. Our
requirement is the inverse: the **MCP server itself** runs on the
remote, with all state and audit there, and the Mac is the thinnest
possible client.

### 1.3 Why **not** Claude Code's "SSH session" mode

Claude Code can run the whole agent — chat, tool dispatch, everything —
inside a remote SSH session. That moves not just the server but the
*whole client* to the remote; you interact through the terminal. For
some workflows that's fine. For ours, the desktop chat GUI is
non-negotiable, so collapsing into "everything in the terminal" is the
opposite trade-off from the one we want.

### 1.4 When this pattern fits

Good fit when **all** of these are true:

- A remote Linux server holds the data and services Claude needs.
- The Mac is a thin client; heavy compute or live state on it is
  wasteful or impossible.
- SSH key-trust to the remote already exists (ideally with an
  `~/.ssh/config` alias).
- You have or plan to add **server-side governance** — audit log,
  permission tier, kill switch — and don't want it scattered across
  multiple bridges.
- You can tolerate a small amount of reload discipline (Part 4).

Not a fit when:

- Multiple users / clients need shared state — use HTTP-SSE.
- The remote is behind a firewall where SSH itself is awkward (bastion
  hops, frequent reauth).
- You need write access from a mobile or web client — stdio is by
  design per-spawned-process.

---

## Part 2 — Architecture, in pictures

Four configurations for "MCP server reaches a remote machine." It helps to
see them side by side before we walk through the stdio-over-SSH one.

### 2.1 Comparison table

| # | Configuration | `mcpServers.<name>.command` | Where the server process runs | Best fit |
|---|---|---|---|---|
| A | **Local stdio** | `python -m my_mcp.server` | Mac (the client machine) | Single-user, data already on the Mac. The 90% case in Anthropic docs. |
| B | **HTTP / SSE connector** | n/a — uses a URL field | Remote server, listens on a port | Multi-user, public-facing, you are willing to run TLS + tokens. |
| C | **Claude Code SSH session** | (built into Claude Code, not user config) | Remote server, *whole agent* runs there | You want the agent itself on the remote; you accept terminal-only UX. |
| D | **stdio over SSH (this guide)** | `ssh -T <host> /path/to/run_stdio.sh` | Remote server, fresh process per desktop client | Single-user, GUI desktop, governance lives on remote, no public port. |

The fourth row is the one this document explains in full. It is not in the
official docs — see Appendix A for evidence. It is, however, the natural
composition of two things that *are* documented: `command` accepts any
executable, and `ssh` is an executable.

### 2.2 The data flow, explicitly

```
+----------------------------------------------+        +------------------------------------------+
|  Mac (Claude Desktop or Claude Code)         |        |  <remote-host> (Linux server)            |
|                                              |        |                                          |
|  claude_desktop_config.json                  |        |                                          |
|    "mcpServers": {                           |        |                                          |
|      "<name>": {                             |        |                                          |
|        "command":                            |        |                                          |
|          "/Users/<user>/bin/wrapper.sh"      |        |                                          |
|      }                                       |        |                                          |
|    }                                         |        |                                          |
|        |                                     |        |                                          |
|        | (Electron fork at app start;        |        |                                          |
|        |  one process per desktop client)    |        |                                          |
|        v                                     |        |                                          |
|  wrapper.sh                                  |        |                                          |
|    exec /usr/bin/ssh -T <remote-host> \      |        |                                          |
|      ~/<mcp-dir>/scripts/run_stdio.sh        |        |                                          |
|        |                                     |        |                                          |
|        v       stdin (JSON-RPC requests)     |        | run_stdio.sh                             |
|     ssh client ==============================+========> exec python -m <mcp-pkg>.server          |
|                                              |        |   (long-running stdio server, reads      |
|                stdout (JSON-RPC responses)   |        |    stdin / writes stdout via the same    |
|     ssh client <=============================+========  ssh-allocated pipes)                     |
|                                              |        |                                          |
+----------------------------------------------+        +------------------------------------------+

Wire format end to end: line-delimited JSON-RPC over a single bidirectional pipe.
The pipe happens to traverse SSH; MCP's stdio transport is unaware.
```

A few non-obvious facts about this picture:

- **One** SSH tunnel per desktop client. The desktop app forks the wrapper
  exactly once at startup. The tunnel stays alive until the desktop app
  quits or the SSH channel breaks.
- The remote process is `python -m <mcp-pkg>.server`. MCP stdio servers
  are **long-running**: they receive request *N*, write response *N*, then
  wait for request *N+1* on the same file descriptors. They do not exit
  between requests.
- `ssh -T` disables remote pseudo-TTY allocation. **This flag is load
  bearing** — without it, SSH and the remote shell can emit banners
  ("Last login...", motd, locale warnings) on the same stream MCP uses for
  JSON-RPC, corrupting the protocol. The failure mode is silent: desktop
  never sees a valid `tools/list`, your MCP entry shows "0 tools," and
  there is no useful error log. Part 5 trap #6 covers this in full.

---

## Part 3 — Install from zero

This part assumes you have nothing set up. Read it linearly the first time;
each step's success is a precondition for the next.

Throughout, substitute these placeholders for your real values:

| Placeholder | Meaning | Example (do **not** copy literally) |
|---|---|---|
| `<user>` | Your Mac account username | `linyiming` |
| `<remote-host>` | SSH alias or hostname for the remote machine | `t68` (an `~/.ssh/config` entry) |
| `<remote-user>` | Your username on the remote machine | `yiming` |
| `<mcp-dir>` | Install path on remote | `/home/<remote-user>/amoeba_mcp_server` |
| `<mcp-pkg>` | Python package name | `amoeba_mcp` |
| `<name>` | The string you'll use to refer to this MCP in chat (`@<name>`) | `amoeba` |

### 3.1 Remote: lay down the server

#### 3.1.1 Directory and venv

SSH into the remote, create the install directory, and put your source
there. The exact transfer mechanism doesn't matter (`scp`, `git clone`,
`tar`, even `rsync` of a working tree from your Mac).

```bash
ssh <remote-host>
mkdir -p <mcp-dir>
cd <mcp-dir>
# transfer your source here (scp / git clone / tar / rsync)
```

Then create the virtual environment and install the package as **editable**:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -e ".[dev]"
```

The editable install (`-e`) is mandatory for this guide's reload story
— edits to `src/<mcp-pkg>/*.py` take effect after a process restart
with no reinstall step.

Smoke-test that the entry point launches at all:

```bash
python -m <mcp-pkg>.server
# Waits silently for stdin (the right behavior). Ctrl-C to exit.
# A traceback here means install/import is broken — fix before continuing.
```

#### 3.1.2 The `run_stdio.sh` wrapper

The desktop client's `command` will eventually invoke this script over
SSH. It activates the venv and `exec`s into Python so the SSH channel
maps directly to Python's stdin/stdout — no shell sitting in the middle:

```bash
mkdir -p <mcp-dir>/scripts
cat > <mcp-dir>/scripts/run_stdio.sh << 'EOF'
#!/usr/bin/env bash
# Run <mcp-pkg> over stdio (for Claude Desktop integration).
# Usage: ./scripts/run_stdio.sh
set -euo pipefail
cd "$(dirname "$0")/.."
source .venv/bin/activate
exec python -m <mcp-pkg>.server
EOF
chmod +x <mcp-dir>/scripts/run_stdio.sh
```

The `exec` is load-bearing: without it, the shell stays in the process
tree as a parent and signal handling has to traverse it on shutdown,
leaving zombie Python processes. With `exec`, the shell is replaced by
Python, and the SSH channel maps directly to Python's fds.

Verify on the remote that the wrapper resolves the venv and the entry
point — it should hang silently waiting for stdin:

```bash
<mcp-dir>/scripts/run_stdio.sh
# Hangs silently. Ctrl-C to exit.
```

#### 3.1.3 **Don't** add a systemd unit for this

stdio MCP servers are **per-client, on-demand** processes — the desktop
spawns one when it opens, sends EOF when it closes, and the process
exits. Putting that under systemd would either restart the process the
moment desktop EOFs it (defeats the point) or race the desktop's spawn
(two processes, one of which never gets a client).

If you also run an **HTTP** version of the server (Part 4.3), *that*
belongs under systemd. Stdio does not.

### 3.2 SSH: the bit that makes it cheap

The desktop client will spawn an SSH process for every chat session.
Make sure that's fast and silent.

#### 3.2.1 Key-based auth, no password prompt

```bash
# On the Mac, if you don't already have key auth to the remote:
ssh-copy-id <remote-user>@<remote-host>

# Then verify:
ssh <remote-host> whoami
# Should print <remote-user>, no prompt.
```

If a password prompt appears, the desktop client will hang at every
chat-session spawn. Fix this before going further.

#### 3.2.2 An `~/.ssh/config` alias with keepalive and multiplexing

You almost certainly want a host alias — it lets the wrapper say
`ssh <remote-host>` instead of `ssh -i ... -p ... <remote-user>@<ip>`.
Two extras to include: `ServerAliveInterval` (so a silently-dropped
tunnel is detected within ~3 minutes, not stuck forever), and
`ControlMaster` (so subsequent SSH invocations reuse one persistent
master and skip the 0.5–1.5s handshake).

Add to `~/.ssh/config` on the Mac:

```sshconfig
Host <remote-host>
    HostName <ip-or-dns>
    User <remote-user>
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
    ServerAliveCountMax 3
    # Connection multiplexing: keep one master, reuse from there
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 600
```

Verify the multiplexing kicks in:

```bash
ssh <remote-host> 'echo ok'
# First invocation: 0.5 - 1.5s
# Second invocation (within 600s): well under 100ms.
```

If the second call is still slow, your master isn't persisting — check
permissions on `~/.ssh/` (must be 0700) and that `ControlPath` directory
exists.

### 3.3 Mac / Windows desktop: tell the client about the server

The desktop config's location depends on platform.

| Platform | Config path |
|---|---|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` (when desktop ships there) |

#### 3.3.1 The wrapper script (Mac side)

We use a shell wrapper instead of putting `ssh ...` directly in the
config's `command`, for three reasons (explained in the script's
comments below). Create it:

```bash
mkdir -p ~/bin
cat > ~/bin/<name>-mcp-ssh.sh << 'EOF'
#!/bin/bash
# <name> MCP stdio-over-SSH wrapper for Claude Desktop.
#
# Why this wrapper exists at all (instead of putting "ssh ..." directly in
# claude_desktop_config.json's "command"):
#
# 1. Claude Desktop launches MCP "command" entries via Electron / launchd.
#    The PATH inherited there does NOT include /usr/bin in many setups, so
#    a bare "ssh" name resolution can fail. We export PATH explicitly and
#    invoke ssh by absolute path.
#
# 2. Editing a shell script is easier than editing JSON when iterating on
#    flags. Keeps claude_desktop_config.json stable.
#
# 3. -T disables remote TTY allocation. MCP stdio requires CLEAN JSON-RPC
#    on stdout. With a TTY, ssh / shell login banners (motd, "Last login:",
#    locale warnings) corrupt the stream and Desktop never sees a valid
#    tools/list. The failure is silent — your server entry shows "0 tools".
set -euo pipefail
export PATH=/usr/bin:/bin
exec /usr/bin/ssh -T <remote-host> /home/<remote-user>/<mcp-dir-tail>/scripts/run_stdio.sh
EOF
chmod +x ~/bin/<name>-mcp-ssh.sh
```

The third reason — `-T` — is the one that bites people. If you forget
it, the desktop entry silently shows "0 tools" with no useful error.
Part 5 trap 6 covers diagnosis and fix.

#### 3.3.2 Register the MCP in `claude_desktop_config.json`

Edit the config (path above for your platform). Add or merge an entry into
the `mcpServers` block:

```json
{
  "mcpServers": {
    "<name>": {
      "command": "/Users/<user>/bin/<name>-mcp-ssh.sh"
    }
  }
}
```

Notes on the config schema:

- `command` **must be an absolute path** — the desktop does not honour `~`
  expansion or shell PATH lookup here.
- No `args` array is needed — all SSH flags live inside the wrapper.
- No `env` block is needed unless you're injecting e.g. `SSH_AUTH_SOCK`
  for an agent-forwarded key.

#### 3.3.3 Restart the desktop and verify

```bash
osascript -e 'quit app "Claude"'
open -a Claude
```

(or your platform's equivalent). Then, in the desktop UI:

1. Open **Settings → Connectors** (or **Developer → MCP Servers**,
   depending on the desktop version).
2. The entry **`<name>`** should be listed and labelled **LOCAL DEV**,
   with a tool count next to it (e.g. "9 tools" once you have things
   wired).
3. If the count is "0 tools" or the entry is red, jump to Part 5 trap
   clinic.

A small surprise to expect here: the badge says **LOCAL DEV** even though
the server lives on a remote machine. That label refers to the *desktop
app's* runtime mode (a developer-defined config, not an Anthropic-managed
hosted connector). Source-of-truth is `claude_desktop_config.json`, not
the badge. See Part 5 trap 5 for the full story.

### 3.4 First end-to-end check

Open a new chat in the desktop. Ask Claude to call any tool exposed by
your server (the simplest is whatever your `read_file` equivalent is, on
a small whitelisted path).

If it works: congratulations, the four-layer chain (desktop → ssh client
→ remote shell → python interpreter → server.py) is solid.

If it doesn't, the failure mode dictates where to look. The desktop
keeps a per-MCP log file at:

| Platform | Log path |
|---|---|
| macOS | `~/Library/Logs/Claude/mcp-server-<name>.log` |
| Windows | `%APPDATA%\Claude\Logs\mcp-server-<name>.log` |

Tail it with:

```bash
tail -f "$HOME/Library/Logs/Claude/mcp-server-<name>.log"
```

The log will tell you which layer failed:

- "spawn ENOENT" → `command` path is wrong; check `claude_desktop_config.json`.
- "Permission denied (publickey)" → SSH key trust isn't set; redo §3.2.1.
- Truncated / non-JSON output → TTY corruption; check that `-T` is in the
  wrapper. (Part 5 trap 6.)
- "0 tools" silently → almost always TTY corruption (same trap), or the
  remote install failed in a way that lets the entry point start but not
  serve.

Part 5 expands each of these.

---

## Part 4 — Edit-cycle discipline (the most expensive lessons)

The setup is the easy half. Once you start *editing* the server source —
adding tools, tightening whitelists, changing schemas — you hit two pitfalls
that aren't covered anywhere in the official docs. This part is the
expensive lessons distilled.

### 4.1 The two-layer reload rule

When you edit `src/<mcp-pkg>/*.py` on the remote, **the change does not
take effect until the right reload happens**. There are two layers, and
which one you need depends on what you edited.

#### Layer 1 — Server-side: kill the running process

The MCP stdio server is forked once at desktop startup. **Editing
source on disk does nothing** by itself; the running interpreter has
the old code in memory. Kill the process and let the desktop respawn:

```bash
# On <remote-host> (or via ssh):
pkill -TERM -f "python -m <mcp-pkg>.server"
```

Desktop sees EOF (~1s), reconnects (~5s), the new process loads the
edited source. Total downtime ~6s for that client.

This works for any change inside Python source — handler logic, path
lists, log parsing, response shaping, **values** in module-level
constants. The current chat session continues working; only the
underlying process restarted.

#### Layer 2 — Client-side: schema cache

When a chat **session** starts, the desktop calls `tools/list` once and
caches the schema for that session. If your edit changed a tool's
**schema** — e.g. the `enum` of valid argument values — the cached
schema does **not** refresh. The client will reject calls with the new
value as "not in enum" even though the server accepts it.

Fix: **start a new chat session.** It re-fetches `tools/list`.

#### One-line rule of thumb

> **Logic change → pkill. Schema change → new chat.**

A more complete table for the kinds of edits you actually make:

| Edit | Layer | How to apply |
|---|---|---|
| `READ_FILE_WHITELIST_PREFIXES` (path-string list) | 1 | pkill; current session works |
| `READ_FILE_BLACKLIST_SUFFIXES` (path-string list) | 1 | pkill; current session works |
| Tool body / handler logic | 1 | pkill; current session works |
| `SQL_DB_PATHS` adding a new DB key | 1 + **2** | pkill **and** start new chat (DB enum is in schema) |
| Log whitelist for `tail_service_log` (sources an enum) | 2 | new chat (the whitelist sources the schema enum) |
| Tool description text | 2 | new chat (clients display from cached schema) |
| Adding a brand-new tool | 2 | new chat |

If your edit changes anything declared in `@tool(...)` or in the JSON-Schema
returned by `tools/list`, you need a new chat. Otherwise pkill is enough.

#### Walked-into-this (2026-04-25)

Edited `SQL_DB_PATHS` to add a new database. Ran `pkill`. Opened a new
chat. Call still failed "db_path not in enum." Fifteen minutes wasted
because I'd done only half of the two-layer reload — schema changes
require *both* (pkill so the server's `tools/list` returns the new enum,
*and* new chat so the client re-fetches it). **Always: logic or schema?
If schema, both layers.**

### 4.2 Path-string vs enum-schema, in one sentence

The same constant can need different reloads depending on how it's
wired: if the handler body checks it at call time → Layer 1; if it
*sources* a `tools/list` enum → Layer 2. Look at the tool definition —
if the constant feeds an `enum` argument, it's Layer 2.

### 4.3 Two processes on one machine: HTTP + stdio

If your MCP package has both an `http_server.py` (e.g. for a mobile client
or other HTTP consumers, on a separate port) and the stdio entry point,
you'll have **two Python processes running side by side** on the same
remote, sharing the same source modules.

```bash
ps -ef | grep -E "<mcp-pkg>" | grep -v grep
# typical:
# <remote-user>  12345 ... python -m <mcp-pkg>.http_server      <- the HTTP daemon (systemd-managed)
# <remote-user>  67890 ... python -m <mcp-pkg>.server           <- the stdio process (desktop-spawned)
```

They share the same on-disk source (editable install) but pick up new
code at different times: HTTP via `systemctl restart <mcp-pkg>_http`,
stdio via `pkill -TERM -f "python -m <mcp-pkg>.server"`.

**Restarting the HTTP service does NOT touch the stdio process.** This
catches people: you edit a tool, you `systemctl restart` because that's
the service you remember, you reopen the chat, nothing changed. The
stdio process is still the old one. Recovery is one extra `pkill`, but
the half-hour spent debugging "I restarted but nothing changed" is the
cost of forgetting this.

When in doubt, list both processes and look at start times:

```bash
ssh <remote-host> "ps -ef | grep -E '<mcp-pkg>' | grep -v grep"
```

### 4.4 The audit-log decoy

A subtler version of §4.3: with two processes (HTTP + stdio) sharing the
same source, any one chat call lands on **exactly one** of them. If
you check audit and find nothing, the call may have landed on the
*other* process and logged elsewhere. Always do `ps -ef | grep <mcp-pkg>`
**before** concluding "audit empty means no call landed" — that's the
same surface-tool trap as `systemctl is-active`, `df -h`, and
`ps | grep <service-name>`. Trap 4 in Part 5 expands the diagnostic.

---

## Part 5 — Trap clinic (the part Google can't help with)

Six traps, each with **symptom → cause → fix**. Most live debugging maps
to one of these. If you're stuck, scan the symptoms first.

### Trap 1 — "I edited the source and nothing changed."

**Symptom.** You edit `src/<mcp-pkg>/...py`, save, call a tool — old
behavior, no error.

**Cause.** The MCP stdio server is forked once at desktop startup and
runs for the whole app lifetime. Python doesn't hot-reload; the running
interpreter has the old source in memory.

**Fix.** Two-layer reload from Part 4.1:

```bash
ssh <remote-host> 'pkill -TERM -f "python -m <mcp-pkg>.server"'
# Then if the edit changed a schema (enum, description, new tool),
# also start a new chat session.
```

When in doubt, do both — opening a new chat is cheap. Doing **neither**
is what costs half-hour debug sessions.

### Trap 2 — "One bad call hung the whole server for four minutes."

**Symptom.** You call `read_file` (or any file-touching tool) with a
*directory* path (trailing slash, or one that resolves through a symlink
to a directory). The desktop sits on "calling tool" for ~4 minutes.
**Other unrelated tool calls during that window also fail.** After 4
minutes, the original returns an error and everything resumes.

**Cause.** Stdio MCP servers are **single-process, single-asyncio-loop**
— no per-request isolation. `open(directory_path, "rb")` on POSIX has
undefined behavior — typically it returns a file-like object whose
`read()` blocks. The single asyncio loop is now stuck and every queued
request is blocked behind it. (Post-mortem: `LESSONS_LEARNED.md` § L-2026-04-25b.)

**Fix.** **Validate inputs aggressively in the handler.** For any
file-touching tool:

- `os.path.isdir(real_path)` → reject with a structured error.
- `os.path.realpath(p)` → catch symlinks that resolve outside whitelist.
- `max_bytes` / `max_lines` cap on reads.
- Decode-error path-out (binary file mistakenly hit).
- Early reject anything that could put `open()`/`read()` in an undefined
  state.

Treat input validation as **the** fault-isolation mechanism, not as
defensive polish. The protocol gives you no other isolation; if you
don't reject bad inputs at the handler boundary, *the entire server*
hangs on the first malformed call.

### Trap 3 — "`pkill` killed my own SSH session and exited 255."

**Symptom.** You SSH into the remote, run `pkill -f "python -m <mcp-pkg>.server"`
to apply Layer 1 reload, and your shell prints `exit 255` and disconnects.
The desktop client is also confused (its tunnel went away during a tool
call).

**Cause.** Your interactive SSH shell is itself in the SSH process
hierarchy. `pkill -f` matches by command line; depending on your match
string, you may have killed processes in your own shell's control tree.
255 is SSH's "channel closed abnormally" exit.

**Fix.** Run `pkill` from a **fresh** SSH command that exits immediately:

```bash
ssh <remote-host> 'pkill -TERM -f "python -m <mcp-pkg>.server"'
```

That ssh subprocess has its own short-lived shell which exits as soon
as `pkill` returns, regardless of what `pkill` killed. If you're already
inside an interactive SSH session, detach the kill so it survives your
own exit:

```bash
setsid bash -c "sleep 1 && pkill -TERM -f \"python -m <mcp-pkg>.server\"" \
  </dev/null >/dev/null 2>&1 & disown
exit
```

### Trap 4 — "The audit log is empty but the desktop showed a result."

**Symptom.** A tool call returns to chat, looks correct. You go to check
the audit log on the remote — and there's no entry for that call.

**Cause.** Almost always one of:

1. **Two server processes** (HTTP daemon + stdio); the call landed on
   the *other* one. Each writes to its own audit destination
   (Part 4.3 / 4.4).
2. **Cached / hallucinated response** — the desktop sometimes returns a
   previous result from chat history if the current call fails silently.
3. **Wrong audit file** — older log path, different user, etc.

**Fix.** Diagnose in this order:

```bash
# 1. How many processes? Which ones? (Match start time to desktop spawn time)
ssh <remote-host> "ps -ef | grep -E '<mcp-pkg>' | grep -v grep"

# 2. Hit the actual audit DB. The stdio process inherits stdout from
#    desktop's spawn chain (logs land on the Mac, not on the remote);
#    audit-DB writes go wherever the server code writes them.
ssh <remote-host> 'sqlite3 <mcp-dir>/audit.db "select * from audit_log order by id desc limit 5"'
```

The single most-common version: "audit empty" → check `ps`, see HTTP
daemon got the call instead of stdio, look at HTTP logs, find the entry.
The audit *is* there; it's under a different roof.

### Trap 5 — "The desktop labels my server LOCAL DEV, but I want to confirm it's not phoning home."

**Symptom.** Your MCP entry in the desktop UI is tagged **LOCAL DEV**
(or "Custom"). You want to be sure no Anthropic-managed cloud service is
proxying the call.

**Cause.** The label refers to the *desktop app's* runtime mode, not the
server's location. **LOCAL DEV** = "the desktop is running your
locally-configured `command` from `claude_desktop_config.json`." The
desktop has no concept of "remote MCP" here; whatever your `command`
is, the badge says LOCAL DEV.

**Fix.** Source of truth is your config file, not the badge:

```bash
cat "$HOME/Library/Application Support/Claude/claude_desktop_config.json"
```

If `command` is an SSH wrapper, the call leaves your Mac via SSH to
your remote — nowhere else. For belt-and-suspenders certainty,
`tcpdump -i <iface> port 22 and host <remote-ip>` during a tool call
and confirm the only outbound flow is to your remote.

### Trap 6 — "0 tools" with no error.

**Symptom.** The MCP entry shows up in the desktop, says "0 tools," and
gives no useful error in the UI.

**Cause.** Almost always **TTY / banner corruption** of the JSON-RPC
stream. SSH is allocating a remote pseudo-TTY (because `-T` is missing
from the wrapper, or because some other ssh option flipped it on), and
the remote shell is printing its login banners (motd, `Last login: ...`,
locale warnings, anything `~/.bashrc` does on a TTY) onto stdout. The
desktop receives a mix of banner text and JSON, can't parse the
`tools/list` response, and gives up — silently — listing the server
with zero tools.

**Fix.** Confirm the cause cheaply by stripping `-T` and observing what
comes out:

```bash
# WITHOUT -T (this is what would happen if the wrapper omits it):
/usr/bin/ssh <remote-host> <mcp-dir>/scripts/run_stdio.sh </dev/null 2>&1 | head -5
```

If you see *any* non-JSON text — `Last login: ...`, motd, locale
warnings, output from `~/.bashrc` — that is what would corrupt MCP.

Two places to fix, depending on root cause:

1. **Wrapper missing `-T`.** Add it: `exec /usr/bin/ssh -T <remote-host> ...`.
2. **Remote `~/.bashrc` prints things even on non-interactive shells.**
   Wrap noisy parts in `[[ $- == *i* ]] && ...` so they only fire
   interactively.

Re-run the same command **with** `-T` and confirm silence (the server
is hanging on stdin; Ctrl-C). Silence means the stream is clean;
restart the desktop and the tool count should populate.

---

## Part 6 — Governance integration (lightweight overview)

This pattern is well-suited for adding **server-side governance** —
audit log, permission tier, kill switch. Three reasons: the server runs
on a machine you fully control (no shared tenancy); SSH is already
trusted as transport, so governance is purely "what may this tool do,"
not "is the caller real"; each desktop client gets its own MCP process,
keeping state local for fast paths with a shared audit DB for slow paths.

A typical layering (specific contents private):

- **Audit log.** Every tool call to a SQLite DB on the remote
  (timestamp, tool name, args hash, return class). For "what did Claude
  do last week" and after-the-fact red-line enforcement.
- **Permission tier.** Tools split into read/write at minimum; writes
  require a second confirmation or explicit "yes, do this" argument.
- **Kill switch.** A flag (file, Redis key, DB row) that makes the
  server refuse writes without exiting. Lets you take the system
  offline without breaking the chat surface.

Exact tier-1/2/3 contents, red-line list, and `discipline_rules.yaml`
schema are project-specific and not in this public-facing guide.

---

## Part 7 — Publication

This guide is published as a sanitized, public GitHub repository. The
companion source code, example configs, and `LESSONS_LEARNED.md` are all
in the same repo; the setup steps in Part 3 are written assuming a
reader who has just `git clone`d it.

A companion blog post — covering the same material in a more narrative
form, with the trap clinic as the anchor — is published alongside.

What the public repo includes:

- This guide (`SETUP_GUIDE.md`).
- `LESSONS_LEARNED.md` — failure post-mortems referenced inline.
- Reference config files (Appendix B, individually as files in `examples/`).
- `LICENSE`.

What the public repo does **not** include:

- The actual MCP server source for the project this guide originated
  from. The pattern is the contribution; specific server implementations
  are downstream and project-private.
- Concrete network topology, hostnames, IP ranges.
- Tier-1/2/3 governance contents (Part 6 explains why the abstract layering
  is the only public-facing part).

If you adapt this pattern, the request is only that you keep the
`LESSONS_LEARNED.md` style — append, don't delete; mark each lesson
with a date — so the next person walking into your fork has the same
benefit you got walking into this one.

---

## Appendix A — References

### Anthropic / MCP official documentation

- Model Context Protocol specification
  (whichever version is current at https://modelcontextprotocol.io/).
- Claude Desktop config schema (`claude_desktop_config.json` reference).
- MCP Python SDK (the `@tool` decorator, `tools/list`, schema generation).
- Anthropic blog posts on MCP transports (HTTP-SSE vs stdio).

### "ssh-mcp" community packages (related but **different**)

These all have an MCP server that runs on the **local Mac** and exposes
"ssh into a remote and run a thing" as one of its tools. Listed here so
you don't accidentally reach for them when you wanted the architecture
in this guide:

- `tufantunc/ssh-mcp`
- `mixelpixx/ssh-mcp`
- `jasondsmith72/ssh-mcp`
- `AiondaDotCom/ssh-mcp`
- `Machine-To-Machine/m2m-mcp`
- `t-suganuma/ssh-mcp`
- (one additional npm hit, unnamed, same architecture)

The architectural difference is large enough that "ssh-mcp" is, in retrospect,
an unfortunate shared name. None of these put the **MCP server itself** on
the remote machine.

### Related but distinct: Claude Code SSH session mode

Claude Code (the terminal agent) can run itself end-to-end inside a
remote SSH session — *whole agent* on the remote, not just the MCP
server. Different trade-off, see Part 1.3.

### Repository docs

- `LESSONS_LEARNED.md`
  - **L-2026-04-25** — two-layer reload (this guide §4.1 codifies it).
  - **L-2026-04-25b** — single-process no-isolation
    (this guide §5 trap 2).
- `SETUP.md` — repository-level setup (Phase α: HTTP server + mobile API).
- `README.md` — tool list, Phase β plans.

### Sui generis evidence

As of 2026-04-25, two rounds of web search (Anthropic docs, GitHub MCP
topic, npm registry, Reddit / HN / X traffic) found nothing that
documents **`Desktop config command="ssh ..."` directly, with the MCP
server spawning fresh on the remote per desktop session**. The
components are all documented (`command` accepts any executable; `ssh`
is an executable; MCP stdio is well-specified); the *combination*, with
its reload discipline and trap clinic, is not. The pattern is not novel
— it's the obvious composition of two documented things — but no
consolidated guide exists, and the traps have to be reconstructed from
first principles. This is that reconstruction.

---

## Appendix B — Complete config files (copy-paste ready)

All files use placeholders (`<remote-host>`, `<remote-user>`, `<mcp-pkg>`,
etc.) consistent with Part 3. Substitute your real values throughout.

### B.1 `~/bin/<name>-mcp-ssh.sh` (Mac wrapper)

```bash
#!/bin/bash
# <name> MCP stdio-over-SSH wrapper for Claude Desktop.
#
# Why this wrapper exists at all (instead of putting "ssh ..." directly in
# claude_desktop_config.json's "command"):
#
# 1. Claude Desktop launches MCP "command" entries via Electron / launchd.
#    The PATH inherited there does NOT include /usr/bin in many setups, so
#    a bare "ssh" name resolution can fail. We export PATH explicitly and
#    invoke ssh by absolute path.
#
# 2. Editing a shell script is easier than editing JSON when iterating on
#    flags. Keeps claude_desktop_config.json stable.
#
# 3. -T disables remote TTY allocation. MCP stdio requires CLEAN JSON-RPC
#    on stdout. With a TTY, ssh / shell login banners (motd, "Last login:",
#    locale warnings) corrupt the stream and Desktop never sees a valid
#    tools/list. The failure is silent — your server entry shows "0 tools".
set -euo pipefail
export PATH=/usr/bin:/bin
exec /usr/bin/ssh -T <remote-host> /home/<remote-user>/<mcp-dir-tail>/scripts/run_stdio.sh
```

### B.2 `claude_desktop_config.json` (Mac config)

Path: `~/Library/Application Support/Claude/claude_desktop_config.json` on
macOS. (See Part 3.3 for Windows / Linux paths.)

```json
{
  "mcpServers": {
    "<name>": {
      "command": "/Users/<user>/bin/<name>-mcp-ssh.sh"
    }
  }
}
```

If you have multiple MCPs, this becomes one entry alongside others:

```json
{
  "mcpServers": {
    "<name>": {
      "command": "/Users/<user>/bin/<name>-mcp-ssh.sh"
    },
    "another-one": {
      "command": "/Users/<user>/bin/another-mcp-stdio.sh"
    }
  }
}
```

### B.3 `~/.ssh/config` (Mac, full example)

```sshconfig
Host <remote-host>
    HostName <ip-or-dns>
    User <remote-user>
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
    ServerAliveCountMax 3
    # Connection multiplexing (optional, recommended for fast spawn):
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 600
```

### B.4 `<mcp-dir>/scripts/run_stdio.sh` (remote)

```bash
#!/usr/bin/env bash
# Run <mcp-pkg> over stdio (for Claude Desktop integration).
# Usage: ./scripts/run_stdio.sh
set -euo pipefail
cd "$(dirname "$0")/.."
source .venv/bin/activate
exec python -m <mcp-pkg>.server
```

### B.5 `<mcp-dir>/systemd/<mcp-pkg>_http.service` (remote, **HTTP only — for reference**)

This unit is for the HTTP server, which runs as a system service. It is
**not** for the stdio path — stdio MCP servers should not be under
systemd, see Part 3.1.3 for why. Included here for the hybrid case
(stdio for Mac + HTTP for mobile) so you can see the contrast:

```ini
[Unit]
Description=<name> MCP HTTP Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=<remote-user>
Group=<remote-user>
WorkingDirectory=/home/<remote-user>/<mcp-dir-tail>
EnvironmentFile=/home/<remote-user>/<mcp-dir-tail>/.env
ExecStart=/home/<remote-user>/<mcp-dir-tail>/scripts/run_http.sh
Restart=on-failure
RestartSec=5
StandardOutput=append:/home/<remote-user>/<mcp-dir-tail>/<mcp-pkg>_http.log
StandardError=append:/home/<remote-user>/<mcp-dir-tail>/<mcp-pkg>_http.log

[Install]
WantedBy=multi-user.target
```

To install: `sudo cp <mcp-dir>/systemd/<mcp-pkg>_http.service /etc/systemd/system/`,
then `sudo systemctl daemon-reload && sudo systemctl enable --now <mcp-pkg>_http`.

### B.6 Reload cheatsheet (paste-ready commands)

```bash
# Layer 1 — server-side (handler logic, path lists, anything in handler body):
ssh <remote-host> 'pkill -TERM -f "python -m <mcp-pkg>.server"'

# Layer 2 — client-side (schema/enum/description/new tool):
# Open a new chat session in the desktop. No command needed.

# HTTP server reload (separate process from stdio):
ssh <remote-host> 'sudo systemctl restart <mcp-pkg>_http'

# Process inventory on remote:
ssh <remote-host> "ps -ef | grep -E '<mcp-pkg>' | grep -v grep"

# Desktop-side log (Mac):
tail -f "$HOME/Library/Logs/Claude/mcp-server-<name>.log"

# HTTP server log (remote):
ssh <remote-host> 'tail -f <mcp-dir>/<mcp-pkg>_http.log'
```

---

*Last updated: 2026-04-26 (Final).*
*Authoring discipline: every concrete value (hostname, user, path) is a
placeholder in `<angle-brackets>`. Substitute yours throughout. The
matching draft 1 — closer to a `PATTERN.md`-style write-up — is
preserved alongside this file as `STDIO_OVER_SSH_PATTERN.md.bak_draft1`
for comparison.*
