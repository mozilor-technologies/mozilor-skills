# GitNexus CLI Reference

All commands work via `npx` — no global install required.

Run from the project root directory.

## Commands

### analyze — Build or refresh the index

```bash
npx gitnexus analyze
```

Build or refresh the knowledge graph index. This parses all source files, builds the knowledge graph with symbols and relationships, writes it to `.gitnexus/`, and generates `CLAUDE.md` / `AGENTS.md` context files.

**When to run:**
- First time in a project
- After major code changes
- When `gitnexus://repo/{name}/context` reports the index is stale
- Manually when you want to ensure index is up to date

**What it does:**
1. Scans all source files in the repository
2. Extracts symbols (functions, classes, methods, etc.)
3. Analyzes relationships (calls, imports, extends, etc.)
4. Detects execution flows (processes)
5. Writes graph to `.gitnexus/` directory
6. Generates CLAUDE.md and AGENTS.md with stats

**Flags:**

| Flag           | Effect                                                           |
| -------------- | ---------------------------------------------------------------- |
| `--force`      | Force full re-index even if up to date                           |
| `--embeddings` | Enable embedding generation for semantic search (off by default) |

**Examples:**
```bash
# Basic analyze
npx gitnexus analyze

# Force full re-index
npx gitnexus analyze --force

# Enable semantic search
npx gitnexus analyze --embeddings
```

**Notes:**
- In Claude Code, a PostToolUse hook runs `analyze` automatically after `git commit` and `git merge`, preserving embeddings if previously generated
- Embeddings enable semantic search via `gitnexus_query` tool
- Embedding generation can be slow; omit `--embeddings` if not needed
- Set `OPENAI_API_KEY` environment variable for faster API-based embeddings

**Output:**
```
Analyzing repository...
Found 5,537 symbols
Found 14,590 relationships
Detected 300 execution flows
Generating embeddings (if --embeddings flag used)
Index written to .gitnexus/
Generated CLAUDE.md and AGENTS.md
```

### status — Check index freshness

```bash
npx gitnexus status
```

Shows whether the current repo has a GitNexus index, when it was last updated, and symbol/relationship counts.

**When to use:**
- Check if re-indexing is needed
- Verify index exists
- See basic statistics

**Output:**
```
Repository: /Users/you/project
Status: Indexed
Last updated: 2 hours ago
Symbols: 5,537
Relationships: 14,590
Processes: 300
Embeddings: 3,482 (enabled)
```

**Status values:**
- `Indexed`: Index exists and is relatively fresh
- `Stale`: Index exists but may be outdated
- `Not indexed`: No index found (run `analyze`)

### clean — Delete the index

```bash
npx gitnexus clean
```

Deletes the `.gitnexus/` directory and unregisters the repo from the global registry (`~/.gitnexus/registry.json`).

**When to use:**
- Before re-indexing if the index is corrupt
- After removing GitNexus from a project
- To free up disk space

**Flags:**

| Flag      | Effect                                            |
| --------- | ------------------------------------------------- |
| `--force` | Skip confirmation prompt                          |
| `--all`   | Clean all indexed repos, not just the current one |

**Examples:**
```bash
# Clean current repo (with confirmation)
npx gitnexus clean

# Clean without confirmation
npx gitnexus clean --force

# Clean all indexed repos
npx gitnexus clean --all
```

**Warning:**
This is destructive. You'll need to run `analyze` again to rebuild the index.

### wiki — Generate documentation from the graph

```bash
npx gitnexus wiki
```

Generates repository documentation from the knowledge graph using an LLM. Requires an API key (saved to `~/.gitnexus/config.json` on first use).

**When to use:**
- Generate architectural documentation
- Create codebase overview for new developers
- Export knowledge graph insights to markdown
- Publish documentation as GitHub Gist

**Flags:**

| Flag                | Effect                                    |
| ------------------- | ----------------------------------------- |
| `--force`           | Force full regeneration                   |
| `--model <model>`   | LLM model (default: minimax/minimax-m2.5) |
| `--base-url <url>`  | LLM API base URL                          |
| `--api-key <key>`   | LLM API key                               |
| `--concurrency <n>` | Parallel LLM calls (default: 3)           |
| `--gist`            | Publish wiki as a public GitHub Gist      |

**Examples:**
```bash
# Generate wiki with default settings
npx gitnexus wiki

# Force regeneration
npx gitnexus wiki --force

# Use specific model
npx gitnexus wiki --model openai/gpt-4

# Publish as GitHub Gist
npx gitnexus wiki --gist
```

**Output:**
Generated documentation is written to `.gitnexus/wiki/` directory with:
- Overview of architecture
- Functional areas (clusters)
- Execution flows (processes)
- Key symbols and their roles

**API Key Setup:**
On first run, you'll be prompted for an API key. It's saved to `~/.gitnexus/config.json` for future use.

### list — Show all indexed repos

```bash
npx gitnexus list
```

Lists all repositories registered in `~/.gitnexus/registry.json`. The MCP `list_repos` tool provides the same information.

**When to use:**
- See which repos are indexed
- Verify registration
- Multi-repo navigation

**Output:**
```
Indexed repositories:
1. sr-import-export-backend (/Users/you/Desktop/Store Robo/sr-import-export-backend)
2. my-other-project (/Users/you/projects/other)
```

## After Indexing

1. **Read `gitnexus://repo/{name}/context`** to verify the index loaded
2. Use the other GitNexus workflows for your task:
   - [Exploring](exploring.md) - Understanding architecture
   - [Impact Analysis](impact-analysis.md) - Blast radius before changes
   - [Debugging](debugging.md) - Tracing bugs
   - [Refactoring](refactoring.md) - Safe code restructuring

## Common Workflows

### First-Time Setup
```bash
cd /path/to/your/project
npx gitnexus analyze --embeddings
```

### After Major Code Changes
```bash
npx gitnexus analyze
```

### Force Full Reindex
```bash
npx gitnexus clean --force
npx gitnexus analyze --embeddings
```

### Check Status Before Using
```bash
npx gitnexus status
# If stale:
npx gitnexus analyze
```

### Generate Documentation
```bash
npx gitnexus analyze --embeddings
npx gitnexus wiki
# Documentation in .gitnexus/wiki/
```

## Troubleshooting

### "Not inside a git repository"
**Cause:** Running from a directory that's not inside a git repo.

**Solution:** Run from a directory inside a git repository.

### Index is stale after re-analyzing
**Cause:** MCP server hasn't reloaded the index yet.

**Solution:** Restart Claude Code to reload the MCP server.

### Embeddings are slow
**Cause:** Local embedding generation is slow.

**Solutions:**
- Omit `--embeddings` flag (embeddings are off by default)
- Set `OPENAI_API_KEY` environment variable for faster API-based embeddings
- Use a smaller model for faster generation

### Command not found
**Cause:** `npx` not available or GitNexus not published.

**Solution:** Ensure Node.js and npm are installed. Run `npx gitnexus@latest`.

### Index is corrupt
**Cause:** Interrupted analysis, disk issues, or version mismatch.

**Solution:**
```bash
npx gitnexus clean --force
npx gitnexus analyze
```

### Wiki generation fails
**Cause:** Missing API key or invalid model.

**Solution:**
- Verify API key is set: Check `~/.gitnexus/config.json`
- Verify model is correct: Use `--model` flag with valid model name
- Check API base URL: Use `--base-url` if using custom endpoint

### Registry issues
**Cause:** Corrupted `~/.gitnexus/registry.json`.

**Solution:**
```bash
# Clean all repos
npx gitnexus clean --all --force

# Re-analyze current repo
npx gitnexus analyze
```

## Directory Structure

After running `analyze`, your project will have:

```
.gitnexus/
├── graph.db              # Neo4j database
├── embeddings/           # Vector embeddings (if --embeddings used)
├── wiki/                 # Generated documentation (if wiki run)
└── metadata.json         # Index metadata (timestamp, stats)

~/.gitnexus/
├── registry.json         # Global repo registry
└── config.json           # Global config (API keys)
```

## Environment Variables

| Variable         | Purpose                                          | Default            |
| ---------------- | ------------------------------------------------ | ------------------ |
| `OPENAI_API_KEY` | API key for faster embedding generation          | None (local only)  |
| `GITNEXUS_HOME`  | Directory for global config                      | `~/.gitnexus`      |
| `DEBUG`          | Enable debug logging (set to `gitnexus:*`)       | None               |

**Example:**
```bash
export OPENAI_API_KEY=sk-...
export DEBUG=gitnexus:*
npx gitnexus analyze --embeddings
```

## Performance Tips

### Faster Analysis
- Run on smaller codebases first to verify
- Exclude `node_modules` and other generated code (automatically excluded)
- Use `--force` sparingly (only when needed)

### Faster Embeddings
- Set `OPENAI_API_KEY` for API-based embeddings
- Use `--concurrency` flag to control parallel LLM calls
- Omit embeddings if semantic search not needed

### Disk Space
- Clean old indexes: `npx gitnexus clean`
- Embeddings take additional space (skip if not needed)
- Wiki generation creates markdown files (can be deleted)

## Integration with Claude Code

### Auto-Indexing
The PostToolUse hook automatically runs `npx gitnexus analyze` after:
- `git commit`
- `git merge`

This keeps the index fresh without manual intervention.

**Embedding preservation:**
If embeddings were previously generated, the auto-analyze preserves them.

### Manual Override
You can still run `npx gitnexus analyze` manually at any time.

### Disable Auto-Indexing
To disable the hook, remove or modify the hook configuration in Claude Code settings.

## Best Practices

### When to Reindex
- After major refactoring
- After merging large PRs
- Before starting a new feature
- When index is stale (status or context resource warns)

### When to Use --embeddings
- First-time indexing if you want semantic search
- When exploring unfamiliar codebase
- When searching by concepts ("payment processing", "authentication")

### When to Skip --embeddings
- Small codebases (< 100 files)
- Performance-sensitive environments
- Disk space limited
- Not using `gitnexus_query` tool

### When to Use --force
- Index is corrupt
- Major structural changes to codebase
- Testing index generation
- Verifying index accuracy

## Related Workflows

- [Exploring](exploring.md) - Use after indexing to understand code
- [Impact Analysis](impact-analysis.md) - Check index freshness before impact analysis
- [Debugging](debugging.md) - Ensure fresh index for accurate tracing
- [Refactoring](refactoring.md) - Reindex after major refactoring
