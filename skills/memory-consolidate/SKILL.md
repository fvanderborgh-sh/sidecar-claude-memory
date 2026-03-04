# /memory-consolidate - End-of-Session Memory Consolidation

Run the consolidation protocol to save what was learned this session. This is the "sleep" process that prevents memory loss between conversations.

## Steps

### Step 1: Check the Clock

```bash
date "+%Y-%m-%d %H:%M %Z"
```

Use this timestamp for all entries created during consolidation.

### Step 2: Locate Memory Root

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
PROJECT_HASH=$(ls ~/.claude/projects/ | grep "$(echo "$PROJECT_ROOT" | sed 's|/|-|g' | sed 's|^-||')" | head -1)
MEMORY_ROOT="$HOME/.claude/projects/$PROJECT_HASH/memory"
```

Verify `$MEMORY_ROOT/MEMORY.md` exists. If not, tell the user to run `/memory-init` first.

### Step 3: Read Current State

Read these files:
- `$MEMORY_ROOT/working.md` - current working memory
- `$MEMORY_ROOT/MEMORY.md` - current index

### Step 4: Add Today's Session Entries

**Deduplication check — do this first:**

Read `$MEMORY_ROOT/working.md` and check for entries with today's date:
```bash
grep "^### $(date +%Y-%m-%d)" "$MEMORY_ROOT/working.md"
```

If entries for today exist, read them carefully and compare against this session's work:
- If an entry already covers the same work (same topic, overlapping details) → **skip it**, do not write a duplicate.
- If today's entries cover *different* work → you may still append entries for the uncovered work only.
- If no today entries exist → proceed normally with all entries.

Now, review the current conversation and identify significant work done. For each piece of work, add an entry to `working.md` with:

```markdown
### YYYY-MM-DD (HH:MM TZ)
- **Brief summary of what happened**
- Key details, decisions, outcomes
- [salience: N] [tags: tag1, tag2, tag3]
```

**Salience scoring guide:**
| Score | Criteria |
|-------|----------|
| 5 | User said "remember this" / Major discovery / Saved significant time |
| 4 | Solved a real problem / Revealed a non-obvious pattern |
| 3 | Completed a standard task / Routine but useful |
| 2 | Minor interaction / Already documented elsewhere |
| 1 | Trivial / One-off / Won't recur |

If this was a trivial session (just reading files, answering quick questions), note it briefly at salience 1-2 and skip the heavier consolidation steps.

### Step 5: Prune Working Memory

Check all entries in `working.md`. Remove entries older than 7 days from the current date. Before deleting, consolidate them per Step 6.

### Step 6: Consolidate Expiring Entries

For each entry being pruned from working memory:

**Salience 4-5**: Create an episodic memory file:
```bash
# Create episodes directory if needed
mkdir -p "$MEMORY_ROOT/episodes"
```
Before creating an episode file, check if it already exists:
```bash
ls "$MEMORY_ROOT/episodes/YYYY-MM-DD-<topic-slug>.md" 2>/dev/null
```
If it exists, do **not** overwrite. Instead append new learnings under an `## Addendum YYYY-MM-DD` section at the end and increment `access_count` by 1. If it does not exist, create it with the full template.

Write to `$MEMORY_ROOT/episodes/YYYY-MM-DD-<topic-slug>.md`:
```markdown
# Episode: <Descriptive Title>

- **Date**: YYYY-MM-DD
- **Salience**: N/5
- **Tags**: tag1, tag2, tag3
- **Access count**: 1
- **Last accessed**: YYYY-MM-DD

## Context
What led to this event.

## What Happened
Detailed narrative of the event.

## Key Learnings
- Insight 1
- Insight 2

## Artifacts
- Links to PRs, files, documents created
```

**Salience 3**: Extract the key facts and add them to the relevant `semantic/` file (codebase.md, domain.md, tools.md, people.md, or user.md). Include `[confidence: high/medium] [source: working memory YYYY-MM-DD]`.

**Salience 1-2**: Let it fade. Delete without encoding.

### Step 7: Pattern Detection

Scan recent working memory entries and newly created episodes. If 2+ entries share the same tags:
- Check if a `semantic/` entry already covers this pattern
- If not, create or update one
- If 3+ entries describe the same multi-step process, consider creating a `procedures/` file

### Step 8: Update MEMORY.md Index

Update the following sections in `MEMORY.md`:
- **Quick Context**: Update "Current focus" and "Recent" fields
- **Active Episodes**: Add any new episodes, remove deleted ones
- **Hot Knowledge**: Add any frequently-referenced facts discovered this session
- **Procedures**: Add any new procedures

Keep MEMORY.md under 200 lines total (it gets truncated beyond that in the system prompt).

### Step 9: Decay Check

List all files in `$MEMORY_ROOT/episodes/`. For each file:
1. Parse the date from the filename
2. If older than 60 days:
   - Read the file and check `access_count`
   - If `access_count` < 3: Extract key learnings to the relevant semantic file, then delete the episode
   - If `access_count` >= 3: Keep it (clearly still useful)

### Step 10: Summary

Output what was done:
```
Memory consolidated.

Working memory: X entries (oldest: YYYY-MM-DD, newest: YYYY-MM-DD)
  - Added: N new entries
  - Pruned: N expired entries

Consolidation:
  - Episodes created: N
  - Semantic updates: N
  - Entries forgotten: N

Index: MEMORY.md updated (N/200 lines)
Decay: N episodes checked, N archived, N kept
```
