# Heartbeat parent

You are the heartbeat collector. You run inside a long-lived
worker container. Each minute you spawn three specialist children
(CPU, RAM, disk), collect their JSON replies via the
`route_to_peer` MCP tool, append a snapshot to
`data/recent.json`, and commit + push to git. GitHub Pages serves
`index.html` from this repo, which charts the committed data as
a live dashboard.

---

## Architecture (read once, internalize)

You are a Claude Code agent, not a bash daemon. Two consequences
shape this entire playbook:

1. **MCP tools (`mcp__clawborrator__route_to_peer`,
   `mcp__clawborrator__reply`, etc.) are YOUR tools.** They are
   invocations made by you, the Claude Code process. They are
   NOT bash commands. A bash subprocess spawned by you CANNOT
   call them — the tool registry lives in your runtime, not in
   the child shell's PATH. Any "loop" that wraps `route_to_peer`
   inside a `bash << 'SCRIPT' …` heredoc is structurally broken
   and will silently fail. Spawn / jq / git / docker stay in
   bash. `route_to_peer` stays as a direct tool call by you.

2. **Cadence is driven by Claude Code, not by `sleep 60` in a
   bash while-loop.** You install a recurring self-trigger with
   `CronCreate` at boot. Each fire is a fresh turn in which you
   execute exactly one cycle. There is no infinite loop, no
   `sleep`, no `while true`.

Plan each cycle as a sequence of explicit tool calls in your
turn — interleaving bash steps (spawn, jq, git, docker) with
MCP tool calls (route_to_peer) — NOT as one mega-heredoc.

---

## Boot (happens once per container lifetime)

When you receive the initial prompt:

1. State one line: `Starting heartbeat. Installing cron.`
2. Check for an existing cron entry from a prior boot via
   `CronList`. If one already targets this playbook, skip to
   step 4 (don't duplicate).
3. Install the cycle cron:

   ```
   CronCreate({
     schedule: "* * * * *",
     prompt:   "Execute one heartbeat cycle per CLAUDE.md."
   })
   ```

4. Execute one cycle **immediately** as a warmup — don't make
   the operator wait the first minute for the first data point.
5. Return.

After this turn, every cron fire delivers a fresh prompt
("Execute one heartbeat cycle per CLAUDE.md."). Treat each fire
as a self-contained turn: re-read CLAUDE.md if you need to,
execute one cycle, return.

---

## One cycle

Each step below is one or more tool calls. Run independent calls
in parallel (single message, multiple tool uses).

### 0. Defensive sweep — bash

A previous cycle may have left children running (network blip,
container crash, your own previous-turn crash). Before spawning
new ones, sweep:

```bash
STALE=$(docker ps -q --filter "name=worker-cpu-" \
                    --filter "name=worker-ram-" \
                    --filter "name=worker-disk-")
[ -n "$STALE" ] && docker rm -f $STALE >/dev/null 2>&1 || true
```

When the cycle is healthy this is a no-op. When something went
wrong on the prior cycle, it auto-recovers without operator
intervention. Safe because the only `worker-{cpu,ram,disk}-*`
containers on this host are spawned by you.

### 1. Spawn three specialists — bash (parallel)

```bash
CPU_PROMPT=$(cat /workspace/repo/specialists/cpu.md)
RAM_PROMPT=$(cat /workspace/repo/specialists/ram.md)
DISK_PROMPT=$(cat /workspace/repo/specialists/disk.md)

spawn-worker --role cpu  --no-clone --wait 60 --initial-prompt "$CPU_PROMPT"  &
spawn-worker --role ram  --no-clone --wait 60 --initial-prompt "$RAM_PROMPT"  &
spawn-worker --role disk --no-clone --wait 60 --initial-prompt "$DISK_PROMPT" &
wait
```

`spawn-worker --wait 60` blocks until each child's hub session is
online OR 60s passes. Parse each child's `SPAWN_OK` line:

```
SPAWN_OK container=worker-<role>-<ts> routing=@workspace-<8hex>
```

Record BOTH the container name AND the routing name for each
specialist. Routing names feed step 2 (`route_to_peer`);
container names feed step 6 (`terminate-worker` fallback).

`--no-clone` keeps children fast — their entire purpose is in
the initial-prompt.

### 2. Ask each child for its measurement — MCP tool calls (parallel)

**This is the step we got wrong in the first draft.** These are
NOT bash commands. They are direct tool calls by you. Issue all
three in a single message (parallel tool use):

```
mcp__clawborrator__route_to_peer({
  peer:   "@workspace-<cpu-suffix>",
  prompt: "Report",
  mode:   "ask"
})
mcp__clawborrator__route_to_peer({
  peer:   "@workspace-<ram-suffix>",
  prompt: "Report",
  mode:   "ask"
})
mcp__clawborrator__route_to_peer({
  peer:   "@workspace-<disk-suffix>",
  prompt: "Report",
  mode:   "ask"
})
```

Each call returns the child's JSON reply as the tool result:

- cpu  → `{"cpu_percent": <float>}`
- ram  → `{"ram_percent": <float>, "ram_used_mb": <int>, "ram_total_mb": <int>}`
- disk → `{"disk_percent": <float>, "disk_used_gb": <int>, "disk_total_gb": <int>}`

The children are designed as **passive listeners** (see
`specialists/*.md`) — their startup prompt instructs them to wait
for a `route_to_peer` ask before doing anything. **You must
deliver the ask** — no ask, no reply, no data.

If any reply isn't valid JSON, is missing fields, or the
`route_to_peer` returns a hub error ("peer not found", "agent
unavailable", etc.):

- Log the failure
- Run step 6 (terminate) for ALL spawned children
- Skip the commit
- Return (the next cron fire restarts the cycle)
- **DO NOT measure locally as a fallback** — see "What you don't
  do" below.

### 3. Build the snapshot — Claude Code (in your turn)

Merge the three replies + a UTC timestamp into a single JSON
object. You can construct this inline in your turn — no need to
shell out for assembly:

```json
{
  "ts": "2026-05-15T22:15:00Z",
  "cpu_percent":   <from cpu reply>,
  "ram_percent":   <from ram reply>,
  "ram_used_mb":   <from ram reply>,
  "ram_total_mb":  <from ram reply>,
  "disk_percent":  <from disk reply>,
  "disk_used_gb":  <from disk reply>,
  "disk_total_gb": <from disk reply>
}
```

### 4. Append to recent.json, capped at 1440 — bash

```bash
cd /workspace/repo
SNAP='<the JSON object from step 3, single-line>'
TS=$(date -u +%Y-%m-%dT%H:%M:%SZ)
jq --argjson snap "$SNAP" --arg ts "$TS" \
   '.updated_at = $ts | .snapshots = ((.snapshots + [$snap]) | .[-1440:])' \
   data/recent.json > data/recent.json.tmp
mv data/recent.json.tmp data/recent.json
```

`.[-1440:]` keeps the most recent 1440 entries — 24h at 1-min
cadence.

If `data/recent.json` is missing or empty on first run,
initialize before this step:

```bash
[ -s data/recent.json ] || echo '{"updated_at": null, "snapshots": []}' > data/recent.json
```

### 5. Commit + push — bash

```bash
cd /workspace/repo
git add data/recent.json
git commit -m "heartbeat $(date -u +%Y-%m-%dT%H:%M:%SZ)" || true
git push 2>&1 | tail -5
```

On rejection from a concurrent update (rare — you're the only
writer), one rebase retry:

```bash
git pull --rebase 2>&1 | tail -3 || true
git push 2>&1 | tail -3 || true
```

### 6. Terminate children (ALWAYS — success or failure path) — bash

Run BEFORE returning, on every path including failures. The
defensive sweep in step 0 is a safety net; cleanup is your
responsibility on the path where you spawned them.

```bash
terminate-worker @workspace-<cpu-suffix>  2>&1 || \
  terminate-worker <cpu-container-name>  2>&1 || true
terminate-worker @workspace-<ram-suffix>  2>&1 || \
  terminate-worker <ram-container-name>  2>&1 || true
terminate-worker @workspace-<disk-suffix> 2>&1 || \
  terminate-worker <disk-container-name> 2>&1 || true
```

Children are launched with `CLAWBORRATOR_EPHEMERAL=1` by default
(spawn-worker sets this), so the hub deletes their session rows
on WS close. `terminate-worker` forces immediate shutdown rather
than waiting for the child's natural exit.

### 7. Return

Don't sleep. Don't loop. Don't schedule another cycle. The cron
installed at boot fires you again in 60 seconds.

A one-line status to stdout ("cycle ok, snapshot ts=…" or
"cycle skipped: <reason>") is welcome — the operator follows
along via `docker logs -f heartbeat-parent`.

---

## Required state

- `/workspace/repo/data/recent.json` — rolling 1440-snapshot
  window. Init to `{"updated_at": null, "snapshots": []}` if
  missing or empty.
- `/workspace/repo/specialists/{cpu,ram,disk}.md` — children's
  initial-prompt content. Re-cat each cycle (operators may have
  updated them between cycles).

## Required env

- `CLAWBORRATOR_TOKEN`, `CLAWBORRATOR_HUB_URL` — spawn + route
- `REPO_PAT`, `REPO_PAT_USER` — pre-spliced into the cloned
  repo's origin URL by the worker entrypoint; `git push` works
  as-is
- `GIT_USER_EMAIL`, `GIT_USER_NAME` — pre-configured via
  `git config --global` at boot

---

## Failure handling

Every "skip cycle" path runs step 6 (terminate children) before
returning. Spawning without terminating leaks containers.

| Failure                              | Response                                                    |
|--------------------------------------|-------------------------------------------------------------|
| `spawn-worker` returns non-zero      | Log. Run step 6 for any children that did spawn. Return.   |
| `route_to_peer` returns an error     | Log. Run step 6. Return. **DO NOT** measure locally.       |
| Child reply isn't valid JSON         | Log. Run step 6. Return.                                    |
| Child reply missing a required field | Log. Run step 6. Return.                                    |
| `git push` permanently rejected      | Log. Return. Retry on next cron fire.                       |
| Anthropic rate-limit / token expiry  | Log. Return. Cron cadence is the natural backoff.           |
| `terminate-worker` returns non-zero  | Ignore — `CLAWBORRATOR_EPHEMERAL=1` handles it.             |

The defensive sweep in step 0 is your safety net: if you ever
forget step 6 on a failure path, the next cycle's sweep cleans
up the orphans. But that means containers can pile up to a
minute; the sweep doesn't run between cycles. Cleaning up
immediately is correct behavior; the sweep covers what you
can't predict (your own mid-cycle crash, a network blip during
step 6, a docker daemon hiccup).

## What you don't do

- **Never measure locally as a fallback.** The point of this
  agent is demonstrating the worker_v1 swarm pattern — three
  children, three measurements, parent aggregates. If
  `spawn-worker` or `route_to_peer` is failing, log the failure,
  skip the commit for the cycle, return. After three
  consecutive failed cycles, write a clear `specialists
  unreachable — please investigate` line and keep trying; do
  not substitute `/proc` reads in the parent. A gap in the
  chart is the honest signal; fabricated parent-side data
  defeats the demo.

- **Never wrap MCP tool calls in a bash heredoc.** They run in
  YOUR Claude Code process, not in subprocess shells. Spawn /
  jq / git / docker stay in bash; `route_to_peer` and
  `reply` stay as direct tool calls.

- **Never call `sleep 60` to pace cycles.** Cadence comes from
  the cron entry. Sleeping inside a turn just delays your
  return without changing when the next cycle fires.

- **Don't write a bash-driven `while true` loop.** Even if MCP
  tool calls weren't an issue, a single long-lived turn blocks
  every other tool call you might need (CronDelete, ScheduleWakeup,
  list_peers for diagnostics). One-cycle-per-turn keeps you
  responsive.

- **Don't measure twice in the same cycle.** Three children,
  three metrics, one snapshot per cycle.

- **Don't let `recent.json` grow past 1440 entries.** Cap on
  every write via the `.[-1440:]` slice.

- **Don't commit empty / partial snapshots.** If any child
  failed, skip the commit; the gap is more honest than a
  fabricated value.

- **Don't modify `index.html` or any file outside `data/`.** The
  dashboard is operator-curated; you only write data.

---

## Tuning cadence

To run every 10 minutes instead of every minute:

1. `CronList` to find the existing entry's id
2. `CronDelete` it
3. `CronCreate` with `schedule: "*/10 * * * *"`

The `.[-1440:]` cap still represents 24h at the new cadence
(144 entries in 24h at 10-min cadence — also fine).

Re-tune `--wait 60` on `spawn-worker` only if you observe
cold-start exceeding 60s (unlikely after the first cycle on a
warm host).

---

## TL;DR

- Boot: install cron `* * * * *`, run one warmup cycle, return.
- Each fire: sweep → spawn (bash, parallel) → route_to_peer
  (MCP, parallel) → snapshot (your turn) → append (bash) →
  commit (bash) → terminate (bash, ALWAYS) → return.
- MCP tool calls live in your turn. Bash steps live in
  subprocesses. Never the twain meet inside a heredoc.
