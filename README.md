# sidecar-claude-memory

A psychology-inspired persistent memory system for Claude Code. Gives Claude working, episodic, semantic, and procedural memory that persists across conversations.

## What It Does

Claude Code loses all context between conversations. This plugin gives it a structured memory system based on the Atkinson-Shiffrin model from cognitive psychology:

```
Conversation Input
       |
  [ Working Memory ]     <-- 7-day rolling session log
       |
  [ Consolidation ]      <-- "sleep" process at end of session
       |
   ┌───┴───┐
   |       |
[ Episodic ]  [ Semantic ]    <-- Long-term storage
   |               |
(events)    (facts, patterns)
   |
[ Procedural ]               <-- Validated recipes
```

- **Working Memory**: Active scratchpad. Logs what happened each session with salience scores and tags. 7-day rolling window.
- **Episodic Memory**: Significant events preserved with full context. Decays after 60 days unless frequently accessed.
- **Semantic Memory**: Factual knowledge organized by domain (codebase patterns, business logic, tools, people, user preferences). Permanent.
- **Procedural Memory**: Validated step-by-step recipes for recurring tasks. Permanent.

## Installation

### Option 1: Clone and use as plugin directory

```bash
git clone https://github.com/sidecarhealth/sidecar-claude-memory.git ~/Code/sidecar-claude-memory
```

Then add to your Claude Code settings (`~/.claude/settings.json`):

```json
{
  "plugins": ["~/Code/sidecar-claude-memory"]
}
```

### Option 2: Use directly with --plugin-dir

```bash
claude --plugin-dir ~/Code/sidecar-claude-memory
```

## Quick Start

1. Install the plugin (see above)
2. Open Claude Code in your project
3. Run `/memory-init`
4. Answer the setup questions (your name, project description)
5. Start working normally

That's it. Claude will read your memory at the start of each conversation and the Stop hook will remind you to consolidate at the end.

## Commands

### `/memory-init`

Bootstrap the memory system for the current project. Creates the directory structure, starter files, and optionally injects memory instructions into your CLAUDE.md.

Creates:
```
~/.claude/projects/<project-hash>/memory/
├── MEMORY.md           # Index (loaded every conversation)
├── working.md          # 7-day session log
├── episodes/           # Significant events
├── semantic/
│   ├── codebase.md     # Code patterns & architecture
│   ├── domain.md       # Business logic & features
│   ├── tools.md        # CLI, build, CI/CD knowledge
│   ├── people.md       # Team context
│   └── user.md         # Your preferences
└── procedures/         # Validated step-by-step recipes
```

### `/memory-consolidate`

Run the end-of-session consolidation protocol. This is the "sleep" process:

1. Logs today's work to working memory with salience scores
2. Prunes entries older than 7 days
3. Promotes high-salience entries to episodic/semantic memory
4. Detects patterns across tagged entries
5. Updates the MEMORY.md index
6. Runs decay checks on old episodes

### `/memory-status`

Show memory system statistics: entry counts, utilization, and health warnings.

```
Memory System Status
====================
Index (MEMORY.md):     87/200 lines (43% utilized)
Working Memory:        5 entries
Episodic Memory:       3 episodes
Semantic Memory:       5 files
Procedural Memory:     1 procedure
Health:                All clear. No warnings.
```

## How It Works

### Retrieval (conversation start)

Claude reads `MEMORY.md` (always in context via auto-memory) and `working.md` (recent sessions). Based on what you're working on, it loads relevant semantic files - codebase patterns for code work, domain knowledge for feature work, etc.

### During Work

As Claude encounters new information, it mentally tags entries for consolidation. When you correct Claude, it updates semantic memory immediately (reconsolidation). When it references an episode, it bumps the access count.

### Consolidation (conversation end)

The Stop hook reminds you to run `/memory-consolidate`. This scores each piece of work by salience (1-5), promotes important entries to long-term storage, and prunes stale data.

### Decay

Episodes older than 60 days get checked: if accessed fewer than 3 times, key learnings are extracted to semantic memory and the episode is deleted. Frequently-accessed episodes survive indefinitely.

## Salience Scoring

| Score | When to Use | Example |
|-------|-------------|---------|
| 5 | User said "remember this" / Major breakthrough | Found critical workaround |
| 4 | Solved a real problem / Non-obvious pattern | Traced cross-module dependency |
| 3 | Completed standard task / Routine but useful | Created a PR, ran tests |
| 2 | Minor / Already documented elsewhere | Read a file, quick answer |
| 1 | Trivial / One-off | Typo fix, simple lookup |

## Configuration

The plugin creates files in Claude Code's standard project memory directory (`~/.claude/projects/<hash>/memory/`). No additional configuration is required.

The Stop hook (consolidation reminder) can be disabled by removing or editing `hooks/hooks.json`.

## FAQ

**Q: Will this slow down Claude?**
A: No. MEMORY.md is loaded via Claude Code's auto-memory feature (already part of the system prompt). Other files are only read when contextually relevant.

**Q: Can I edit memory files manually?**
A: Yes. They're plain markdown files. Edit them however you want.

**Q: What if I switch machines?**
A: Memory lives in `~/.claude/projects/`. You'd need to sync that directory across machines (symlink to cloud storage, git repo, etc.).

**Q: Does this work with multiple projects?**
A: Yes. Each project gets its own memory directory based on its path hash. Run `/memory-init` in each project.

**Q: What about the MEMORY.md 200-line limit?**
A: MEMORY.md is loaded into the system prompt, which truncates after 200 lines. Keep it as an index/summary. Detailed knowledge goes in semantic files and episodes.

## License

MIT
