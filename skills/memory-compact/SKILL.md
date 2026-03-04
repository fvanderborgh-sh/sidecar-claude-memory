# /memory-compact - Monthly Memory Deep-Clean

Run a deep compaction pass over all memory stores. This reduces bloat, removes stale references, clusters old episodes, and rebuilds the index from actual filesystem state.

**Recommended cadence:** Monthly, or when memory feels sluggish/cluttered.

## Steps

### Step 1: Locate Memory Root

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
PROJECT_HASH=$(ls ~/.claude/projects/ | grep "$(echo "$PROJECT_ROOT" | sed 's|/|-|g' | sed 's|^-||')" | head -1)
MEMORY_ROOT="$HOME/.claude/projects/$PROJECT_HASH/memory"
```

Verify `$MEMORY_ROOT/MEMORY.md` exists. If not, tell the user to run `/memory-init` first and stop.

### Step 2: Snapshot Current State

Capture before-counts for the compaction report:

```bash
# Semantic files — total lines across all files
SEMANTIC_LINES_BEFORE=$(cat "$MEMORY_ROOT"/semantic/*.md 2>/dev/null | wc -l | tr -d ' ')
SEMANTIC_FILES_BEFORE=$(ls "$MEMORY_ROOT"/semantic/*.md 2>/dev/null | wc -l | tr -d ' ')

# Episode files — count individual episodes
EPISODE_COUNT_BEFORE=$(ls "$MEMORY_ROOT"/episodes/*.md 2>/dev/null | wc -l | tr -d ' ')

# MEMORY.md line count
INDEX_LINES_BEFORE=$(wc -l < "$MEMORY_ROOT/MEMORY.md" | tr -d ' ')

# Working memory entries
WORKING_ENTRIES_BEFORE=$(grep -c "^### " "$MEMORY_ROOT/working.md" 2>/dev/null || echo 0)
```

### Step 3: Deduplicate Semantic Files

For each file in `$MEMORY_ROOT/semantic/*.md`:

1. **Remove duplicate facts**: Read all entries. If two entries convey the same information (even with different wording), keep the one with higher confidence or more recent source date. Delete the other.

2. **Resolve contradictions**: If two entries contradict each other, keep the newer one. Leave an HTML comment noting the resolution:
   ```markdown
   <!-- Resolved YYYY-MM-DD: removed older entry stating "X" in favor of newer "Y" -->
   ```

3. **Condense related entries**: If 3+ entries cover the same sub-topic, group them under a `##` heading and merge into a concise summary. Preserve all unique facts — condense the prose, not the information.

Be conservative: when in doubt about whether two entries are duplicates, keep both.

### Step 4: Prune Stale Codebase Facts

**This step applies only to `$MEMORY_ROOT/semantic/codebase.md`.**

For each fact that references a specific file path, function name, or class name:

1. **Cross-reference against the actual codebase:**
   ```bash
   # Check if a referenced file still exists
   ls "$PROJECT_ROOT/<referenced-path>" 2>/dev/null

   # Check if a referenced function/class still exists
   grep -r "<function-or-class-name>" "$PROJECT_ROOT/src" 2>/dev/null | head -3
   ```

2. **Artifact confirmed gone** → delete the fact entirely.

3. **Uncertain** (e.g., file exists but function may have been renamed) → annotate instead of deleting:
   ```markdown
   - Original fact here [confidence: low] [stale-check: YYYY-MM-DD, uncertain]
   ```

4. **Artifact confirmed present** → leave unchanged.

Be conservative: when in doubt, keep the fact and annotate rather than delete.

### Step 5: Cluster Old Episodes

List all files in `$MEMORY_ROOT/episodes/` (excluding the `clusters/` subdirectory). For each episode older than 30 days:

1. **Group by shared tags**: Episodes that share 2+ tags belong in the same cluster.

2. **Create cluster files**:
   ```bash
   mkdir -p "$MEMORY_ROOT/episodes/clusters"
   ```

   Write to `$MEMORY_ROOT/episodes/clusters/<theme-slug>.md`:
   ```markdown
   # Episode Cluster: <Theme Title>

   - **Tags**: tag1, tag2, tag3
   - **Episodes absorbed**: N
   - **Date range**: YYYY-MM-DD to YYYY-MM-DD
   - **Combined access count**: N
   - **Last compacted**: YYYY-MM-DD

   ## Summary
   High-level summary of what this cluster of episodes covers.

   ## Key Learnings
   - Merged insight 1 (from episode A and B)
   - Merged insight 2 (from episode C)
   - Insight 3

   ## Notable Events
   - YYYY-MM-DD: One-line summary of episode 1
   - YYYY-MM-DD: One-line summary of episode 2
   - YYYY-MM-DD: One-line summary of episode 3

   ## Artifacts
   - Combined list of PRs, files, documents from all absorbed episodes
   ```

3. **Delete absorbed individual episode files** after successfully creating the cluster.

4. **Thematically unique episodes** (no tag overlap with others) older than 30 days with `access_count < 3`:
   - Extract key learnings to the relevant `semantic/` file
   - Delete the episode

5. **Thematically unique episodes** with `access_count >= 3`:
   - Keep as-is (clearly still referenced)

If a cluster file for a theme already exists, merge new episodes into it: append to Notable Events, update the Summary and Key Learnings if the new episodes add information, increment the combined access count, and update the date range.

### Step 6: Rebuild MEMORY.md

Rebuild `$MEMORY_ROOT/MEMORY.md` from the actual filesystem state:

1. **Read the current `MEMORY.md`** to preserve:
   - Any user-written or project-specific context at the top
   - Valid Hot Knowledge entries that are still relevant
   - The overall structure and section headers

2. **Scan the filesystem** to rebuild dynamic sections:
   ```bash
   # Active episodes (individual + clusters)
   ls "$MEMORY_ROOT/episodes/"*.md 2>/dev/null
   ls "$MEMORY_ROOT/episodes/clusters/"*.md 2>/dev/null

   # Semantic files
   ls "$MEMORY_ROOT/semantic/"*.md 2>/dev/null

   # Procedures
   ls "$MEMORY_ROOT/procedures/"*.md 2>/dev/null

   # Working memory latest
   tail -20 "$MEMORY_ROOT/working.md" 2>/dev/null
   ```

3. **Rewrite MEMORY.md** with:
   - Quick Context (current focus, recent work — derived from working.md)
   - Active Episodes (from filesystem scan)
   - Episode Clusters (from filesystem scan)
   - Hot Knowledge (preserved valid entries + any new high-frequency facts)
   - Semantic files index (list what exists)
   - Procedures index (list what exists)

4. **Enforce the 200-line limit.** If the rebuilt file exceeds 200 lines, trim by:
   - Removing older/lower-priority episodes from the listing
   - Condensing Hot Knowledge entries
   - Keeping only the most recent 5 working memory references

### Step 7: Output Compaction Report

```
Memory compaction complete.

Before → After:
  Semantic files: N files, M lines → N files, M lines (removed X duplicate/stale entries)
  Episodes: N individual → M individual + K clusters (absorbed X into clusters, decayed Y to semantic)
  MEMORY.md: N lines → M lines (limit: 200)
  Working memory: N entries (unchanged by compaction)

Actions taken:
  - Deduplicated: X entries across N semantic files
  - Stale references removed: X (from codebase.md)
  - Stale references annotated: X (uncertain, kept with low confidence)
  - Episodes clustered: X episodes → Y clusters
  - Episodes decayed to semantic: X
  - MEMORY.md rebuilt from filesystem state

Next recommended compact: YYYY-MM-DD (30 days from now)
```
