---
name: self-audit
description: Audit and clean Hermes internal state — memory stores, session database, cron jobs, disk cruft. Run when the agent starts re-answering messages, resurrecting shelved work, or behaving erratically due to stale context.
---

# Self-Audit

## When to run

- Agent re-answering already-answered messages
- Agent autonomously resurrecting shelved/closed work
- After long periods of uptime (weekly or biweekly)
- Before archiving or migrating profiles
- When memory feels bloated or contradictory
- User asks for "cleanup", "audit", or "tidy up"

## What this does

Audits six areas of Hermes internal state, reports findings, then cleans up. Does NOT wipe anything — only deduplicates, de-stales, prunes cruft, and closes stale sessions.

## Procedure

### Step 1: Gather state (parallel reads)

Run these in parallel — they're independent:

```
# Memory stores
read_file ~/.hermes/memories/MEMORY.md
read_file ~/.hermes/memories/USER.md

# Cron jobs
cronjob action=list

# Config
read_file ~/.hermes/config.yaml

# Session DB stats
sqlite3 ~/.hermes/state.db "SELECT count(*) as total, sum(CASE WHEN ended_at IS NULL THEN 1 ELSE 0 END) as active FROM sessions;"
sqlite3 ~/.hermes/state.db "SELECT model, count(*) FROM sessions WHERE ended_at IS NULL GROUP BY model;"

# Disk cruft
ls ~/.hermes/memories/*.bak.* 2>/dev/null | wc -l
du -sh ~/.hermes/cron/output/
du -sh ~/.hermes/skills/
ls ~/.hermes/cron/output/ | wc -l

# Cron output orphans
# Compare cron output dir names against current job IDs from jobs.json
```

### Step 2: Audit memory stores

Check MEMORY.md and USER.md for:

1. **Duplicates** — same fact spread across multiple entries. Merge into one.
2. **Stale entries** — completed tasks, shipped features, resolved issues, hardcoded counts that change over time, references to projects the user has moved on from.
3. **Contradictions** — entries that say opposite things (e.g. "use X" vs "never mention X").
4. **Verbosity** — entries that take a paragraph where a sentence would do.

Fix by editing the files directly with `patch` (the memory tool's substring matching can fail on large compound entries). Use `memory` tool for single-entry replacements if the entry is standalone.

**Rules:**
- Keep: infrastructure facts, user preferences, project relationships, agent config, account handles, model config, hard constraints
- Cut: completed task descriptions, shipped feature lists, "X was done" status notes, verbose lists
- Never remove: vault paths, model config, household info, communication preferences, agent hierarchy, cron job IDs
- Merge: split entries about the same topic into one compact entry

### Step 3: Clean stale sessions

Close sessions older than 48 hours that are still marked as active (ended_at IS NULL). These cause the "re-answering old messages" bug.

```python
import sqlite3, time
from pathlib import Path

db = Path.home() / '.hermes' / 'state.db'
conn = sqlite3.connect(str(db))
cutoff = time.time() - (48 * 3600)
before = conn.execute("SELECT count(*) FROM sessions WHERE ended_at IS NULL").fetchone()[0]
conn.execute("UPDATE sessions SET ended_at = started_at + 3600 WHERE ended_at IS NULL AND started_at < ?", (cutoff,))
conn.commit()
after = conn.execute("SELECT count(*) FROM sessions WHERE ended_at IS NULL").fetchone()[0]
conn.close()
print(f"Closed {before - after} stale sessions (was {before}, now {after})")
```

**Important:** The session export cron job (`hermes-session-export.py`) should also be doing this. Verify the script has the stale-session-closing block after the export loop. If it doesn't, add it (see Step 6).

### Step 4: Audit cron jobs

Check each cron job for:

1. **Erroring jobs** — `last_status: error`. Run the script manually to see the error. If the error is environmental (no local model loaded, service not running), pause the job with `cronjob action=pause` and note the reason.
2. **Redundant jobs** — two jobs doing the same thing. Consolidate or pause one.
3. **Jobs producing stale context** — agent-driven jobs (not `no_agent: true`) that inject old output into sessions. Check if their output is still relevant.
4. **Jobs that contradict memory** — e.g. jobs generating posts when memory says "never post to X".

### Step 5: Prune disk cruft

```python
import shutil, json
from pathlib import Path

# 1. Memory backup files (pure cruft)
mem = Path.home() / '.hermes' / 'memories'
for f in mem.glob('*.bak.*'):
    f.unlink()

# 2. Orphaned cron output dirs (dirs with no matching job ID)
cron_out = Path.home() / '.hermes' / 'cron' / 'output'
with open(Path.home() / '.hermes' / 'cron' / 'jobs.json') as f:
    data = json.load(f)
# Parse out current job IDs (handle dict or list structure)
current_ids = set()
jobs = data.get('jobs', []) if isinstance(data, dict) else data
if isinstance(jobs, dict):
    jobs = list(jobs.values())
for job in jobs:
    jid = job.get('id', job.get('job_id', ''))
    if jid:
        current_ids.add(jid)

for d in cron_out.iterdir() if cron_out.exists() else []:
    if d.is_dir() and d.name not in current_ids:
        is_orphan = not any(d.name.startswith(cid) or cid.startswith(d.name) for cid in current_ids)
        if is_orphan:
            shutil.rmtree(d)

# 3. Trim old cron output files (keep last 10 per job)
for d in cron_out.iterdir() if cron_out.exists() else []:
    if d.is_dir():
        files = sorted(d.glob('*.txt'), key=lambda x: x.stat().st_mtime, reverse=True)
        for f in files[10:]:
            f.unlink()
```

Do NOT delete:
- `.lock` files (they're active coordination files)
- `jobs.json` itself
- `.curator_backups/` (let the curator manage its own backups)
- `skills/.archive/` (curator manages this)

### Step 6: Verify session export script

The session export cron job runs every 60 minutes. It should be doing two things:
1. Exporting new sessions to the vault as markdown
2. Closing stale sessions older than 48 hours

Check `~/.hermes/scripts/hermes-session-export.py` for the stale-session-closing block. If missing, add after the export loop:

```python
# Close stale sessions (older than 48h, still marked as active)
stale_cutoff = datetime.now(tz=_NYTZ).timestamp() - (48 * 3600)
conn.execute("""
    UPDATE sessions SET ended_at = started_at + 3600
    WHERE ended_at IS NULL AND started_at < ?
""", (stale_cutoff,))
stale_count = conn.execute("SELECT changes()").fetchone()[0]
if stale_count:
    print(f"  Closed {stale_count} stale sessions (older than 48h)")
conn.commit()
```

### Step 7: Verify

After all cleanup, verify:

1. Session DB: `SELECT count(*) FROM sessions WHERE ended_at IS NULL` should be <5 (only truly active sessions)
2. Memory files: no duplicate entries, no stale references, no contradictions
3. Cron jobs: no erroring enabled jobs
4. Disk: no orphaned output dirs, no .bak files
5. Session export script runs without error: `python3 ~/.hermes/scripts/hermes-session-export.py --since 24h`

### Step 8: Report

Print a summary table of what was found and what was cleaned:

```
SELF-AUDIT REPORT
=================
Sessions:     X total, Y active (was Z)
Memory:       MEMORY.md X bytes (was Y), USER.md X bytes (was Y)
              - N duplicates merged
              - N stale entries removed
              - N contradictions resolved
Cron:         N jobs paused (erroring), N jobs verified
Disk:         N backup files removed, N orphaned dirs removed
              Cron output: X MB (was Y MB)
Export script: verified / fixed
```

## Pitfalls

- **Memory tool substring matching fails on large compound entries.** When the memory store has one giant entry containing many facts separated by blank lines, the `memory` tool's `replace` action can't match substrings within it. Use `patch` to edit the file directly in that case.
- **jobs.json structure varies.** It can be a dict with a `jobs` key that's either a list or a dict keyed by job ID. Parse defensively.
- **Cron jobs that need a local model will error when none is loaded.** If you run agents that connect to a local LLM (like the custodian component), all their modes will error when no model is running. Pause them, don't try to fix them — they'll work again when a model is loaded.
- **Never delete .lock files.** They're coordination files for concurrent access. Deleting them can cause corruption.
- **Don't prune session DB rows.** Just close stale sessions (set ended_at). Pruning rows can break foreign key references and session search.
- **The Memory Condensation cron job runs daily and modifies memory autonomously.** If it's not catching duplicates, it may be because it uses the memory tool which fails on compound entries. Consider running self-audit manually after the condensation job runs to verify it didn't introduce issues.