# Automation & Hooks — The Session Lifecycle

SessionStart loads context, SessionEnd captures it. This is the heartbeat of GuildHouse.

Without hooks, memory is passive — files sitting in a directory. With hooks, memory becomes active: context loads automatically, transcripts get indexed, stale knowledge gets flagged. The difference between "I set up memory files" and "my AI actually learns between sessions" is usually hooks.

Claude Code supports two hook events:

- **SessionStart** — fires before the first message. Budget: ~5 seconds.
- **SessionEnd** — fires when the session closes. Budget: ~120 seconds.

---

## SessionStart: What to Load

The goal: give the AI useful context without overloading the context window.

**What to include (fast, high-value):**

- MEMORY.md index (already automatic in Claude Code)
- Staleness warnings — scan memory files for expired TTLs, show a one-line warning
- Recent session pointers — which sessions happened recently, one line each
- Health report summary — if a nightly scan found contradictions or decay, surface it

**What NOT to include:**

- Full file contents of every memory (let the AI read on demand)
- Complete search results from vector stores
- Entire git history

**Timing matters.** SessionStart hooks have a 5-second timeout. Everything must be fast:

- File reads (MEMORY.md, health report): <100ms
- Date comparisons for staleness: <200ms
- A single API call (e.g., query for open transactions): <2s
- Leave 2-3 seconds as buffer for slow disks or cold caches

### Sample SessionStart script

```bash
#!/bin/bash
# SessionStart hook — load GuildHouse context

MEMORY_DIR="$HOME/.claude/projects/$(echo "$PROJECT_PATH" | tr '/' '-')/memory"

# 1. Check for stale memories (TTL-based)
if [ -d "$MEMORY_DIR" ]; then
  TODAY=$(date +%s)
  STALE=0
  for file in "$MEMORY_DIR"/*.md; do
    [ "$file" = "$MEMORY_DIR/MEMORY.md" ] && continue
    VERIFIED=$(grep -m1 'last_verified:' "$file" 2>/dev/null | awk '{print $2}')
    TTL=$(grep -m1 'ttl_days:' "$file" 2>/dev/null | awk '{print $2}')
    if [ -n "$VERIFIED" ] && [ -n "$TTL" ]; then
      VERIFIED_TS=$(date -j -f "%Y-%m-%d" "$VERIFIED" +%s 2>/dev/null || date -d "$VERIFIED" +%s 2>/dev/null)
      if [ -n "$VERIFIED_TS" ]; then
        DAYS_OLD=$(( (TODAY - VERIFIED_TS) / 86400 ))
        if [ "$DAYS_OLD" -gt "$TTL" ]; then
          STALE=$((STALE + 1))
        fi
      fi
    fi
  done
  if [ "$STALE" -gt 0 ]; then
    echo "WARNING: $STALE memory file(s) past TTL — consider refreshing"
  fi
fi

# 2. Surface health report if recent
HEALTH="$MEMORY_DIR/knowledge-health.md"
if [ -f "$HEALTH" ]; then
  AGE=$(( ($(date +%s) - $(stat -f %m "$HEALTH" 2>/dev/null || stat -c %Y "$HEALTH" 2>/dev/null)) / 86400 ))
  if [ "$AGE" -le 2 ]; then
    head -5 "$HEALTH"
  fi
fi
```

---

## SessionEnd: The Pipeline

The goal: capture what happened without blocking the user from closing the terminal.

**The pipeline:**

1. **Extract transcript** — get the session's conversation as structured data
2. **Prepare** — convert to searchable markdown (strip tool calls, extract decisions)
3. **Index** — update the search engine's document collection
4. **Embed** — generate vector embeddings for semantic search
5. **Commit** — version control the new knowledge

**Key design decisions:**

- **Incremental processing** — skip sessions already processed, skip tiny sessions (<5KB). Don't re-process everything every time.
- **Background the expensive step** — embedding can take 30-60 seconds. Run it in the background so the session can close immediately.
- **Idempotent** — running the pipeline twice on the same transcript should be safe. No duplicates, no corruption.

### Sample SessionEnd script

```bash
#!/bin/bash
# SessionEnd hook — index session knowledge
#
# This is a minimal version that re-indexes your search collection
# after each session. For transcript preparation (converting raw
# session logs to clean markdown), you'll need to build or adapt
# a script for your setup — the format depends on your AI assistant.

# 1. Re-index: update search collection with any new/changed memory files
qmd update 2>/dev/null

# 2. Re-embed: generate vectors for new documents (background — don't block exit)
nohup qmd embed >/dev/null 2>&1 &
```

**Note on transcript preparation:** The raw session transcript is specific to your AI assistant's format (Claude Code uses JSONL, others vary). If you want to convert transcripts to searchable markdown, you'll need a preparation script tailored to your format. The important thing is that it outputs clean `.md` files into a directory that QMD indexes. The indexing and embedding steps above handle the rest.

---

## Scheduled Maintenance

Hooks handle session-level maintenance. But some tasks need to run even when you're not working:

**Nightly maintenance (recommended):**

- Re-index all search collections (catches anything the session pipeline missed)
- Re-embed (generates vectors for any un-embedded documents)
- Run knowledge health scan (if you've implemented the Gardener pattern from [AI Implementation](ai-implementation.md))

### macOS (LaunchAgent)

```xml
<!-- ~/Library/LaunchAgents/com.guildhouse.maintenance.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.guildhouse.maintenance</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>qmd update &amp;&amp; qmd embed</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key><integer>3</integer>
    <key>Minute</key><integer>0</integer>
  </dict>
</dict>
</plist>
```

### Linux (cron)

```cron
0 3 * * * /usr/local/bin/qmd update && /usr/local/bin/qmd embed
```

---

## Hook Configuration

Claude Code registers hooks in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [{
      "type": "command",
      "command": "~/.claude/hooks/session-context.sh",
      "timeout": 5000
    }],
    "SessionEnd": [{
      "type": "command",
      "command": "~/.claude/hooks/session-export.sh",
      "timeout": 120000
    }]
  }
}
```

**Timeout guidance:**

- SessionStart: 5 seconds max. Users notice anything over 3 seconds.
- SessionEnd: 120 seconds max. But background expensive work (embedding) so the interactive portion finishes in 10-15 seconds.

**Testing hooks:**

```bash
# Test SessionStart output
echo '{"event":"SessionStart"}' | ~/.claude/hooks/session-context.sh

# Test SessionEnd (provide a session file path)
~/.claude/hooks/session-export.sh /path/to/session.jsonl
```

---

## The Automation Maturity Model

| Level | What's Automated | Effort | Impact |
|-------|-----------------|--------|--------|
| 0 | Nothing — manual memory management | Zero | Memory decays between sessions |
| 1 | SessionStart loads MEMORY.md | Minimal | AI starts with context |
| 2 | SessionEnd indexes transcripts | Medium | Sessions become searchable |
| 3 | Nightly maintenance keeps index fresh | Medium | Knowledge stays current even during breaks |
| 4 | Staleness detection flags expired memories | Low | Prevents acting on outdated knowledge |
| 5 | Health scanning finds contradictions | High | Multi-system triangulation catches drift |

Most people should aim for Level 2-3. Levels 4-5 are worth it if you're running multiple knowledge backends and care about consistency.

---

## Next Steps

Once your hooks are running, you'll accumulate searchable session transcripts quickly. See [AI Implementation](ai-implementation.md) for patterns that build on this foundation — the Gardener (knowledge health scanning) and retrieval routing across multiple backends.
