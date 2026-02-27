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

If they choose option 1 or 2, append the following block to the chosen CLAUDE.md:

```markdown
# Memory System

You have a persistent memory directory at `~/.claude/projects/<PROJECT_HASH>/memory/`. Its contents persist across conversations.

As you work, consult your memory files to build on previous experience.

## At Conversation Start
1. Read `working.md` for recent session context
2. Load relevant semantic files based on the topic at hand
3. Check for applicable procedures or episodes

## During Conversation
- When learning something new, note it for consolidation
- When corrected, update semantic memory immediately
- When referencing episodes, bump their access count

## At Conversation End
Run `/memory-consolidate` or manually:
1. Add today's work to `working.md` with salience scores and tags
2. Prune entries older than 7 days
3. Consolidate expiring entries (4-5 -> episodes, 3 -> semantic, 1-2 -> forget)
4. Update MEMORY.md index

## Memory Commands
- `/memory-init` - Bootstrap the memory system (you already did this)
- `/memory-consolidate` - Run end-of-session consolidation protocol
- `/memory-status` - Show memory system statistics
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
2. Run /memory-consolidate at the end of sessions to save what you learned
3. Run /memory-status anytime to check memory utilization
```
