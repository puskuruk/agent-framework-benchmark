# Running the benchmark

How to run this fork (branch `geajs`) end to end, including on an Apple Silicon Mac. The
README covers the design; this file is only about getting a run done.

## 1. Requirements

- Podman 4.3+ and Node 20+ (the containers use the Node version in `template/.nvmrc`).
- About 15 GB of free disk.
- Claude credentials (section 3).
- Linux works as described in the README. macOS works through `podman machine` (section 2).

## 2. Podman setup

**Linux:** install Podman from your package manager. Rootless is recommended.

**macOS (Apple Silicon):**

```sh
brew install podman
podman machine init --cpus 6 --memory 10240 --disk-size 60
podman machine start
```

- Each container gets `--cpus=4 --memory=8g` by default, so the VM needs more than that.
- Volumes are mounted from your home directory. Keep the repo and `runs/` under `~`;
  paths like `/tmp` aren't visible inside the VM.
- After a restart, run `podman machine start` again.
- On ARM, agent-browser uses Playwright's Chromium, because Chrome for Testing has no Linux
  ARM64 build. `agent/run-info.txt` records `arch` and `agent_browser_chromium` for each
  run.

## 3. Credentials

The harness reads **one** of these environment variables and passes it to the agent
containers. It's never written to the run directory.

| Variable | Where it comes from | Billing |
| --- | --- | --- |
| `CLAUDE_CODE_OAUTH_TOKEN` | `claude setup-token` | Uses your Claude subscription (Pro/Max) and its 5-hour and 7-day limits |
| `ANTHROPIC_API_KEY` | console.anthropic.com → API keys | Billed per token. Set a spending limit |

The agent can read the credential from inside its container, so use one that's only for
this.

### Subscription token

```sh
claude setup-token          # opens a browser login, then prints sk-ant-oat01-...
```

Save it to a private file. **Don't** paste it into chats, issues or the repo.

```sh
printf 'export CLAUDE_CODE_OAUTH_TOKEN=%s\n' 'sk-ant-oat01-...' > ~/.bench-token
chmod 600 ~/.bench-token
source ~/.bench-token       # in every new terminal before running
```

When you're done, revoke it in claude.ai → Settings → Claude Code.

### API key

```sh
export ANTHROPIC_API_KEY=sk-ant-api03-...
```

## 4. Quick trial (1 agent run)

```sh
git checkout geajs
source ~/.bench-token
caffeinate -i node bench/run.mjs --frameworks=gea --iterations=1 --effort=high   # drop caffeinate on Linux
```

This takes about 20–40 minutes: building images, the agent, the tests, then Lighthouse.
Check `runs/<id>/report.md`. The `Tests` column should say `1/1`.

## 5. Full run

Use the same settings as the blog: `--effort=high` (the default is `medium`), 5 iterations,
and the default model and Claude Code version.

```sh
source ~/.bench-token
caffeinate -i nohup node bench/run.mjs \
  --frameworks=gea,gea-no-skill,solidstart,sveltekit,vanilla \
  --iterations=5 --effort=high > runs/full.log 2>&1 &
tail -f runs/full.log
```

- `gea` installs the official `gea-framework` skill. `gea-no-skill` installs nothing. Both
  use the label `Geajs`, so the only difference between them is the skill.
- Reference frameworks are in the same run because TBT and LCP depend on the host CPU.
  Compare them only within one machine. Cost, tokens, time and lines of code are comparable
  with the blog.
- One agent run takes about 15–40 minutes, so 25 runs take about 7–10 hours plus about an
  hour of measurement.

### Usage limits (subscription token)

A run this long hits the 5-hour limit. A rejected agent exits within seconds and its tests
fail, which looks like this in the log:

```
Agent done:  gea-no-skill-5 (container exit 0)
Tests done:  gea-no-skill-5 (fail, exit 1)
```

with `"status":"rejected"` in `runs/<id>/<iteration>/agent/claude-stderr.log`. Either use an
API key, or run in batches and resume:

```sh
# delete the rejected iterations, then after the limit resets:
node bench/run.mjs --frameworks=<fw> --iterations=<missing> --effort=high \
  --append-to=runs/<id> --skip-image-build --no-measure
# when every framework has 5 iterations:
node bench/run.mjs --measure-only=runs/<id> --skip-image-build
```

`--append-to` fills in the missing iteration numbers. `--no-measure` skips Lighthouse until
the end, so you measure everything once.

### Automatic retries

The script below does those batches for you. It runs one framework at a time, deletes
iterations that were rate-limited, waits for the reset and tries again. The wait checks the
wall clock, because on macOS `sleep` doesn't count time while the Mac is asleep. Save it as
`runs/drive.sh` (`runs/` is gitignored) and start it with
`caffeinate -i nohup runs/drive.sh > runs/drive.log 2>&1 &`.

```bash
#!/usr/bin/env bash
set -uo pipefail
cd "$(dirname "$0")/.."
source ~/.bench-token
FRAMEWORKS=(gea gea-no-skill solidstart sveltekit vanilla)
N=5
RUN=runs/full
log() { echo "[drive $(date -u +%H:%M:%S)] $*"; }
for fw in "${FRAMEWORKS[@]}"; do
  while :; do
    need=$((N - $(ls -d $RUN/$fw-* 2>/dev/null | wc -l)))
    [ "$need" -le 0 ] && break
    if [ ! -d $RUN ]; then
      node bench/run.mjs --frameworks=$fw --iterations=$need --effort=high --no-measure --out=runs
      mv "$(ls -td runs/2*/ | head -1)" $RUN
    else
      node bench/run.mjs --frameworks=$fw --iterations=$need --effort=high --no-measure \
        --append-to=$RUN --skip-image-build
    fi
    reset=0
    for d in $RUN/$fw-*; do
      if grep -q '"status":"rejected"' "$d/agent/claude-stderr.log" "$d/agent/transcript.jsonl" 2>/dev/null; then
        r=$(grep -ho '"resetsAt":[0-9]*' "$d"/agent/*.log "$d"/agent/transcript.jsonl | head -1 | cut -d: -f2)
        log "rate limited: $d; removing"; rm -rf "$d"
        [ -n "$r" ] && [ "$r" -gt "$reset" ] && reset=$r
      fi
    done
    if [ "$reset" -gt 0 ]; then
      log "waiting until $(date -r $((reset + 120)) 2>/dev/null || date -d @$((reset + 120)))"
      while [ "$(date +%s)" -lt $((reset + 120)) ]; do sleep 60; done
    fi
  done
  log "$fw done"
done
node bench/run.mjs --measure-only=$RUN --skip-image-build
log "finished"
```

## 6. Results

Each run writes to `runs/<id>/`:

| Path | Contents |
| --- | --- |
| `report.md` | Tables: tests, tokens, cost, time, Lighthouse (home page), JS size, lines of code, versions |
| `summary.json` | The same data, machine-readable |
| `<fw>-<n>/agent/` | `transcript.jsonl`, `run-info.txt`, `tooling-versions.json`, `final-test.log`, `repo.tar.gz` |
| `<fw>-<n>/measure/` | `results.json` (all 6 URLs) and Lighthouse reports |

To rebuild only the report: `node bench/run.mjs --report-only=runs/<id>`.

To check whether the agent used the Gea skill:

```sh
jq -c 'select(.type=="assistant") | .message.content[]? | select(.type=="tool_use" and .name=="Skill") | .input' \
  runs/<id>/gea-1/agent/transcript.jsonl
```

## 7. Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `podman not found` / connection refused | Podman isn't installed, or on macOS the VM is stopped: `podman machine start` |
| `No Claude credentials` | `source ~/.bench-token` in this terminal |
| Every agent run finishes in seconds and its tests fail | Usage limit or a bad token. Check `claude-stderr.log` (see section 5) |
| `agent-browser install` fails with "no Linux ARM64 builds" | You're on an older commit. Use the `geajs` branch |
| Run makes no progress for hours | The Mac slept. Use `caffeinate -i`, keep it plugged in with the lid open |
| `Measure failed: ... skipped: final tests failed` | Expected for failed apps. `--measure-failed` measures them anyway |
| Mount errors, or files missing in containers (macOS) | Keep the repo and `runs/` under your home directory |
