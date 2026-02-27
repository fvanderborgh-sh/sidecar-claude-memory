# /memory-init - Bootstrap Persistent Memory System

Initialize the multi-store memory system for this project. This creates the directory structure, starter files, and optionally injects memory instructions into CLAUDE.md.

## Steps

### Step 1: Detect Project Context

Run these commands to determine the project path and Claude Code project hash:

```bash
# Get project root
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
PROJECT_NAME=$(basename "$PROJECT_ROOT")

# Find Claude Code project hash
# Claude stores project data at ~/.claude/projects/<hash>/ where hash is the path with slashes replaced by dashes
PROJECT_HASH=$(ls ~/.claude/projects/ | grep "$(echo "$PROJECT_ROOT" | sed 's|/|-|g' | sed 's|^-||')" | head -1)
```

If `PROJECT_HASH` is empty, compute it:
```bash
PROJECT_HASH=$(echo "$PROJECT_ROOT" | sed 's|/|-|g' | sed 's|^-||')
```

Set the memory root:
```bash
MEMORY_ROOT="$HOME/.claude/projects/$PROJECT_HASH/memory"
```

### Step 2: Check for Existing Memory

Check if memory is already initialized:
```bash
ls "$MEMORY_ROOT/MEMORY.md" 2>/dev/null
```

If MEMORY.md already exists, warn the user and ask if they want to reinitialize (which would overwrite existing memories) or abort.

### Step 3: Gather User Info

Use AskUserQuestion to ask:
1. **Your name** (for personalizing memory files)
2. **Project description** (1-2 sentences describing what this project is)

### Step 4: Create Directory Structure

```bash
mkdir -p "$MEMORY_ROOT"/{episodes,semantic,procedures}
```

### Step 5: Populate Template Files

Read each template from `${CLAUDE_PLUGIN_ROOT}/templates/` and write them to the memory root, replacing placeholders:

| Template | Destination | Placeholders |
|----------|-------------|-------------|
| `MEMORY.md.template` | `$MEMORY_ROOT/MEMORY.md` | `{{USER_NAME}}`, `{{PROJECT_NAME}}`, `{{PROJECT_DESCRIPTION}}` |
| `working.md.template` | `$MEMORY_ROOT/working.md` | `{{PROJECT_NAME}}`, `{{INIT_DATE}}` |
| `codebase.md.template` | `$MEMORY_ROOT/semantic/codebase.md` | `{{PROJECT_NAME}}` |
| `domain.md.template` | `$MEMORY_ROOT/semantic/domain.md` | `{{PROJECT_NAME}}` |
| `tools.md.template` | `$MEMORY_ROOT/semantic/tools.md` | `{{PROJECT_NAME}}` |
| `people.md.template` | `$MEMORY_ROOT/semantic/people.md` | (none) |
| `user.md.template` | `$MEMORY_ROOT/semantic/user.md` | `{{USER_NAME}}` |

For `{{INIT_DATE}}`, use the current date/time from `date "+%Y-%m-%d (%H:%M %Z)"`.

### Step 6: Offer CLAUDE.md Integration

Check if CLAUDE.md exists at the project root or globally:
```bash
ls "$PROJECT_ROOT/CLAUDE.md" 2>/dev/null
ls "$HOME/.claude/CLAUDE.md" 2>/dev/null
```

Use AskUserQuestion to ask if they want to add the memory system instructions to their CLAUDE.md. Offer three options:
1. **Add to project CLAUDE.md** (recommended) - adds memory retrieval/consolidation instructions to the project's CLAUDE.md
2. **Add to global CLAUDE.md** - adds to `~/.claude/CLAUDE.md` (applies to all projects)
3. **Skip** - don't modify any CLAUDE.md (user will manually configure)

If they choose option 1 or 2, append the following block to the chosen CLAUDE.md (replace `<PROJECT_HASH>` with the actual hash computed in Step 1):

```markdown
# Memory System

You have a persistent memory system that preserves knowledge across conversations using psychology-inspired multi-store architecture (Atkinson-Shiffrin model).

Your memory directory is at `~/.claude/projects/<PROJECT_HASH>/memory/`. Its contents persist across conversations.

## At Conversation Start

1. **Check the clock**: Run `date "+%Y-%m-%d %H:%M %Z"` for session timestamps.
2. **Read immediately**: `working.md` - the 7-day active session log.
3. **Load contextually** based on what the user is working on:
   - Code patterns/architecture -> `semantic/codebase.md`
   - Business logic/features -> `semantic/domain.md`
   - Build/CLI/environment -> `semantic/tools.md`
   - Team context -> `semantic/people.md`
   - Past events -> relevant `episodes/` file
   - Validated recipes -> relevant `procedures/` file

## During Conversation

- When encountering a familiar pattern, check if a procedure or episode exists
- When learning something new, mentally tag it for consolidation (salience + tags)
- When the user corrects you, update semantic memory immediately (reconsolidation)
- When referencing an episode, bump its `access_count` and `last_accessed`

## At Conversation End

Run `/sidecar-claude-memory:memory-consolidate` or manually execute:
1. **Add** today's work to `working.md` with salience scores (1-5) and tags
2. **Prune** entries older than 7 days
3. **Consolidate** expiring entries:
   - Salience 4-5 -> Create episodic memory in `episodes/YYYY-MM-DD-<topic>.md`
   - Salience 3 -> Extract key facts into relevant `semantic/` file
   - Salience 1-2 -> Let it fade (delete without encoding)
4. **Pattern detect**: If 2+ entries share tags, create/update semantic or procedural entry
5. **Update index**: Refresh `MEMORY.md` - Hot Knowledge, Recent Episodes
6. **Decay check**: Episodes >60 days old with access_count < 3 -> extract to semantic, delete

## Salience Scoring

| Score | Criteria | Example |
|-------|----------|---------|
| 5 | User said "remember this" / Major discovery | Critical workaround found |
| 4 | Solved a real problem / Non-obvious pattern | Cross-module dependency |
| 3 | Completed standard task / Routine but useful | Created a PR, ran a review |
| 2 | Minor interaction / Already documented | Read a file, answered a question |
| 1 | Trivial / One-off / Won't recur | Typo fix, simple lookup |

## Memory Commands
- `/sidecar-claude-memory:memory-consolidate` - Run end-of-session consolidation protocol
- `/sidecar-claude-memory:memory-status` - Show memory system statistics
```

**Important**: Show the user exactly what will be added and get confirmation before modifying any CLAUDE.md file.

### Step 7: Confirmation Summary

Output a summary:
```
Memory system initialized for <PROJECT_NAME>.

Created:
  <MEMORY_ROOT>/
  ├── MEMORY.md           (index - loaded every conversation)
  ├── working.md          (7-day session log)
  ├── episodes/           (significant events)
  ├── semantic/
  │   ├── codebase.md     (code patterns)
  │   ├── domain.md       (business logic)
  │   ├── tools.md        (CLI/build knowledge)
  │   ├── people.md       (team context)
  │   └── user.md         (your preferences)
  └── procedures/         (validated recipes)

CLAUDE.md: <updated/skipped>

Next steps:
1. Start working normally - Claude will read your memory at conversation start
2. Run /sidecar-claude-memory:memory-consolidate at the end of sessions to save what you learned
3. Run /sidecar-claude-memory:memory-status anytime to check memory utilization
```
