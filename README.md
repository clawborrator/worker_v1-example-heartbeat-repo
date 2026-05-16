# heartbeat — worker_v1 example agent

A self-documenting demo of the clawborrator [worker_v1](https://github.com/clawborrator/worker_v1)
swarm pattern. A long-running parent agent spawns three specialist
children (CPU, RAM, disk) every minute, collects their JSON
replies, commits a snapshot to `data/recent.json`, and pushes
back to this repo. GitHub Pages serves `index.html`, which polls
the JSON and renders a live three-line chart.

## Live dashboard

**<https://clawborrator.github.io/worker_v1-example-heartbeat-repo/>**

## Layout

```
worker_v1-example-heartbeat-repo/
  CLAUDE.md                   parent's playbook — the loop the
                              agent follows every 60s
  specialists/
    cpu.md                    initial-prompt for the CPU child
    ram.md                    initial-prompt for the RAM child
    disk.md                   initial-prompt for the disk child
  data/
    recent.json               rolling 24h of snapshots (≤1440)
  index.html                  Chart.js dashboard, polls recent.json
```

## How it works

Once per cycle:

1. Parent spawns three sibling containers (`spawn-worker --role
   cpu` etc.) on the same Docker host via the parent's mounted
   `/var/run/docker.sock`.
2. Each child boots a Claude Code session with one of the
   `specialists/*.md` files as its initial prompt — *that's the
   entire program*. The child reads `/proc/stat` (or `/proc/meminfo`
   / `df`), formats the result as JSON, and `reply`s once.
3. Parent collects the three JSON replies, merges into a single
   snapshot with a UTC timestamp, appends to `data/recent.json`,
   caps the array at 1440 entries.
4. Parent `git add data/recent.json && git commit && git push`.
5. Parent `terminate-worker`s each child.
6. Parent `sleep 60`s and loops.

GitHub Pages serves `data/recent.json` to whichever client opens
`index.html`. The page refreshes itself once a minute.

## Deploying the agent

This repo is **what the parent reads from**. The actual `docker
compose up` happens in the sibling repo:

→ **[worker_v1-example-heartbeat-worker](https://github.com/clawborrator/worker_v1-example-heartbeat-worker)**

Operator sets `REPO_URL` there to point at this repo, plus auth +
a `REPO_PAT` with `repo` scope (so the parent's `git push` works).

## What the children actually measure

- **CPU** — `/proc/stat` deltas across a 1-second window
- **RAM** — `/proc/meminfo`'s `MemTotal` and `MemAvailable`
- **Disk** — `df -P -B 1 /` for the root filesystem

A containerized worker reads the container's view of these.
Without explicit cgroup quotas, that's effectively the host. With
quotas, it's the container's slice. Either way the demo proves
the swarm-data-plane pattern; this is not a production monitoring
system.

## Costs

- Per cycle (Haiku 4.5, default): ~3 child cold-boots + parent
  reasoning + 4 git ops ≈ **$0.01-0.02 / cycle**.
- At 1/min cadence: **~$15-25 / day**.
- Tune `sleep 60` → `sleep 600` in `CLAUDE.md` to drop to 10/min
  cadence for ~$1.50-2.50 / day.

## Git history caveat

The parent commits once per cycle. At 1/min that's 1440 commits/
day. Easy to grow into a fat git history. Options if you care:

- Bump cadence (5-min cycle = 288 commits/day).
- Periodically squash + force-push: `git reset --soft <old-sha> &&
  git commit -m "squash" && git push --force-with-lease`.

This v1 favors demo clarity over history hygiene.

## Hacking the agent

Want different metrics? Edit the matching `specialists/*.md` and
push — the parent reads them fresh every cycle, so a change lands
on the next minute without restarting the container.

Want a different cadence, more children, a Slack notifier? Edit
`CLAUDE.md`. The agent re-reads its instructions implicitly on
each turn (Claude Code's CLAUDE.md is part of context); for
durable changes that survive a context reset, just keep the file
the source of truth.

## License

MIT. Use as a template; build your own.
