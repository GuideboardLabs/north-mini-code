# Self-Audit

Audit and clean Hermes Agent internal state — memory stores, session database, cron jobs, disk cruft, and the session export script. Run when the agent starts re-answering messages, resurrecting shelved work, or behaving erratically due to stale context.

## What it does

Six-area audit of `~/.hermes/` internal state, reporting findings then cleaning up. Does not wipe anything — only deduplicates, de-stales, prunes cruft, and closes stale sessions.

1. **Memory stores** — deduplicate and de-stale `MEMORY.md` and `USER.md`
2. **Session database** — close sessions older than 48 hours still marked active (root cause of the re-answering bug)
3. **Cron jobs** — pause erroring jobs, flag contradictions with memory
4. **Disk cruft** — prune memory backup files, orphaned cron output dirs, trim old output to last 10 per job
5. **Session export script** — verify it closes stale sessions after export, patch if missing
6. **Verification** — confirm nothing broke after cleanup

Runs in under a minute. Prints a summary report of everything found and cleaned.

## Files

| File | What it does |
|------|-------------|
| `skills/SKILL.md` | The self-audit skill — load it in any Hermes session and follow the procedure |

## Setup

No setup required. The skill operates on base Hermes internal state at `~/.hermes/`. No env vars, no external dependencies beyond `sqlite3` (included with Hermes).

## Run once

```
# In any Hermes session, ask the agent:
"Run the self-audit skill"
```

Or follow the skill manually:

```bash
# Check for stale sessions
sqlite3 ~/.hermes/state.db "SELECT count(*) FROM sessions WHERE ended_at IS NULL AND started_at < strftime('%s', 'now', '-48 hours');"

# Check memory store sizes
wc -c ~/.hermes/memories/MEMORY.md ~/.hermes/memories/USER.md
```

## When to run

- Agent re-answering already-answered messages
- Agent autonomously resurrecting shelved or closed work
- After long periods of uptime (weekly or biweekly)
- Before archiving or migrating profiles
- When memory feels bloated or contradictory
- User asks for "cleanup", "audit", or "tidy up"

## Customization

The skill is written for base Hermes. If you run custom cron jobs that require a local model (like the [custodian](../custodian/) component), the skill will pause them when the model is unloaded rather than trying to fix them. Resume those jobs manually after loading a model.