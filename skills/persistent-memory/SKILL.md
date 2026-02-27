# Persistent Memory System

You have a persistent memory system that preserves knowledge across conversations. This system uses psychology-inspired multi-store architecture (Atkinson-Shiffrin model) with working, episodic, semantic, and procedural memory.

## Memory Root

Your memory directory is at `~/.claude/projects/{{PROJECT_HASH}}/memory/`. Its contents persist across conversations.

## At Conversation Start (Retrieval Protocol)

Execute these steps at the beginning of every conversation:

1. **Check the clock**: Run `date "+%Y-%m-%d %H:%M %Z"` to get exact current datetime. Use this for session logging in working.md.
2. **Always loaded** (via system prompt): `MEMORY.md` - the index. Use it to orient.
3. **Read immediately**: `working.md` - the 7-day active session log.
4. **Load contextually** based on what the user is working on:
   - Code patterns/architecture -> `semantic/codebase.md`
   - Business logic/features -> `semantic/domain.md`
   - Build/CLI/environment -> `semantic/tools.md`
   - Team context -> `semantic/people.md`
   - Past events -> relevant `episodes/` file
   - Validated recipes -> relevant `procedures/` file

## During Conversation

- When encountering a familiar pattern, check if a procedure or episode exists
- When learning something new, mentally tag it for consolidation (salience + tags)
- When the user corrects you, update semantic memory immediately (**reconsolidation**)
- When referencing an episode, bump its `access_count` and `last_accessed`

## At Conversation End (Consolidation Protocol)

Execute these steps - this is the "sleep" process that prevents memory loss:

1. **Add** today's work to `working.md` - use the datetime from step 1 of retrieval. Include salience scores (1-5) and tags. Format headers as: `### YYYY-MM-DD (HH:MM TZ)`
2. **Prune** working memory entries older than 7 days
3. **Consolidate** expiring entries before deletion:
   - Salience 4-5 -> Create episodic memory in `episodes/YYYY-MM-DD-<topic>.md`
   - Salience 3 -> Extract key facts into relevant `semantic/` file
   - Salience 1-2 -> Let it fade (delete without encoding)
4. **Pattern detect**: If 2+ entries share tags, check if a `semantic/` or `procedures/` entry should be created or updated
5. **Update index**: Refresh `MEMORY.md` - Hot Knowledge, Recent Episodes
6. **Decay check**: Scan `episodes/` for files >60 days old:
   - `access_count` < 3 -> Extract key learnings to semantic, delete episode
   - `access_count` >= 3 -> Survives (clearly useful, keep it)

## Salience Scoring

| Score | Criteria | Example |
|-------|----------|---------|
| 5 | User said "remember this" / Major discovery / Saved significant time | Critical workaround found |
| 4 | Solved a real problem / Revealed a non-obvious pattern | Cross-module dependency issue |
| 3 | Completed a standard task / Routine but useful | Created a PR, ran a review |
| 2 | Minor interaction / Already documented elsewhere | Read a file, answered a question |
| 1 | Trivial / One-off / Won't recur | Typo fix, simple lookup |

## Memory File Formats

### Working Memory (`working.md`)
```markdown
### YYYY-MM-DD (HH:MM TZ)
- **Brief summary of what happened**
- Key details, decisions, outcomes
- [salience: N] [tags: tag1, tag2, tag3]
```

### Episodic Memory (`episodes/YYYY-MM-DD-<topic>.md`)
```markdown
# Episode: <Title>

- **Date**: YYYY-MM-DD
- **Salience**: N/5
- **Tags**: tag1, tag2
- **Access count**: 1
- **Last accessed**: YYYY-MM-DD

## Context
What led to this event.

## What Happened
Detailed narrative.

## Key Learnings
- Bullet points of insights

## Artifacts
- Links to PRs, files, docs created
```

### Semantic Memory (`semantic/*.md`)
```markdown
## Topic Section

- Factual statement about the topic
- [confidence: high/medium/low] [source: where this was learned]
```

### Procedural Memory (`procedures/*.md`)
```markdown
# Procedure: <Name>

- **Version**: 1.0
- **Last validated**: YYYY-MM-DD
- **Trigger**: When to use this procedure

## Steps
1. First step
2. Second step
...

## Gotchas
- Things that can go wrong
```

## Reconsolidation

When accessing a memory and finding it wrong or outdated:
1. Update the entry immediately
2. Note the correction source
3. Adjust confidence level
4. If contradicts an episode, annotate the episode

## How to Save Memories

- Organize memory semantically by topic, not chronologically
- Use the Write and Edit tools to update memory files
- `MEMORY.md` is always loaded into context - lines after 200 will be truncated, so keep it concise
- Create separate topic files for detailed notes and link to them from MEMORY.md
- Update or remove memories that turn out to be wrong or outdated
- Do not write duplicate memories - check for existing entries first

## What to Save

- Stable patterns and conventions confirmed across multiple interactions
- Key architectural decisions, important file paths, and project structure
- User preferences for workflow, tools, and communication style
- Solutions to recurring problems and debugging insights

## What NOT to Save

- Session-specific context (current task details, in-progress work, temporary state)
- Information that might be incomplete - verify before writing
- Anything that duplicates existing CLAUDE.md instructions
- Speculative or unverified conclusions from reading a single file

## Explicit User Requests

- When the user asks you to remember something across sessions, save it immediately
- When the user asks to forget or stop remembering something, find and remove the relevant entries
