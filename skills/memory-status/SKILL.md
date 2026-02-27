# /memory-status - Memory System Statistics

Show the current state of the memory system: utilization, entry counts, and health warnings.

## Steps

### Step 1: Locate Memory Root

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
PROJECT_HASH=$(ls ~/.claude/projects/ | grep "$(echo "$PROJECT_ROOT" | sed 's|/|-|g' | sed 's|^-||')" | head -1)
MEMORY_ROOT="$HOME/.claude/projects/$PROJECT_HASH/memory"
```

Verify `$MEMORY_ROOT/MEMORY.md` exists. If not, tell the user to run `/memory-init` first.

### Step 2: Gather Statistics

Read and count:

```bash
# MEMORY.md line count (max 200 useful)
wc -l "$MEMORY_ROOT/MEMORY.md"

# Working memory entries (count ### headers)
grep -c "^### " "$MEMORY_ROOT/working.md"

# Episode count
ls "$MEMORY_ROOT/episodes/"*.md 2>/dev/null | wc -l

# Semantic file count
ls "$MEMORY_ROOT/semantic/"*.md 2>/dev/null | wc -l

# Procedure count
ls "$MEMORY_ROOT/procedures/"*.md 2>/dev/null | wc -l
```

### Step 3: Working Memory Analysis

Read `working.md` and extract:
- Oldest entry date (last entry in file)
- Newest entry date (first entry after header)
- Total entry count
- Entries expiring within 2 days (approaching 7-day limit)

### Step 4: Episode Health Check

For each file in `episodes/`:
- Parse the date from filename
- Calculate age in days
- Flag any approaching 60-day decay threshold (within 7 days)
- Note access counts if available

### Step 5: Output Report

```
Memory System Status
====================

Index (MEMORY.md):     XXX/200 lines (XX% utilized)

Working Memory:        X entries
  Oldest:              YYYY-MM-DD
  Newest:              YYYY-MM-DD
  Expiring soon:       X entries (within 2 days)

Episodic Memory:       X episodes
  Oldest:              YYYY-MM-DD
  Newest:              YYYY-MM-DD
  Approaching decay:   X episodes (within 7 days of 60-day limit)

Semantic Memory:       X files
  Files: codebase.md, domain.md, tools.md, people.md, user.md [, ...]

Procedural Memory:     X procedures

Warnings:
  [!] MEMORY.md at XX% capacity - consider pruning Hot Knowledge
  [!] X working memory entries expire tomorrow - run /memory-consolidate
  [!] X episodes approaching 60-day decay - will be checked next consolidation
```

Only show warnings that actually apply. If everything is healthy, show:
```
Health: All clear. No warnings.
```
