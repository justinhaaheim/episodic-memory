# CLAUDE.md - Development Guide for Episodic Memory

This file provides guidance for Claude Code when working on this project.

## Project Overview

Episodic Memory is a semantic search tool for Claude Code conversations. It indexes conversation history and enables vector-based and text-based search across past sessions.

## Build Process

**Important:** This project commits build artifacts to the repository.

```bash
# Install dependencies
npm install

# Build the project (compiles TypeScript and bundles MCP server)
npm run build

# Run tests
npm test
```

### Build Workflow

1. **Never edit files in `dist/` directly** - they are generated from source
2. After making changes to `src/`, always run `npm run build`
3. Commit both source changes AND the regenerated `dist/` files together
4. The build process:
   - `tsc` compiles TypeScript from `src/` to `dist/`
   - `esbuild` bundles `src/mcp-server.ts` into `dist/mcp-server.js`

## Project Structure

```
src/                    # TypeScript source files
├── embeddings.ts       # Vector embedding generation (local, uses @xenova/transformers)
├── db.ts               # SQLite database operations
├── search.ts           # Search functionality (vector + text)
├── mcp-server.ts       # MCP protocol server
├── sync.ts             # File synchronization from ~/.claude/projects
├── summarizer.ts       # AI-powered conversation summaries
├── parser.ts           # JSONL conversation parsing
└── *-cli.ts            # CLI command implementations

dist/                   # Compiled JavaScript (committed to repo)
cli/                    # CLI entry points
test/                   # Vitest test files
```

## Key Technical Details

### MCP Server stdout/stderr
- The MCP server uses stdout exclusively for JSON-RPC protocol messages
- All logging must use `console.error()`, never `console.log()`
- Progress callbacks from libraries must be suppressed to avoid stdout pollution

### Dependencies
- `better-sqlite3` - Native SQLite bindings (auto-fetches prebuilds)
- `sqlite-vec` - Vector similarity search extension (alpha)
- `@xenova/transformers` - Local embedding generation
- `@anthropic-ai/claude-agent-sdk` - For AI summarization

### Data Locations
- Archive: `~/.config/superpowers/episodic-memory/archive/`
- Database: `~/.config/superpowers/episodic-memory/conversations.db`
- Source: `~/.claude/projects/` (Claude Code conversation files)

## Testing

```bash
npm test              # Run all tests
npm run test:watch    # Watch mode
```

Tests require network access for embedding model download on first run.

## Common Issues

1. **MCP JSON errors** - Ensure no `console.log()` in code paths used by MCP server
2. **Install hangs** - The postinstall script was removed; if issues persist, check Node.js version compatibility
3. **Embedding model loading** - First run downloads ~25MB model from Hugging Face

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `EPISODIC_MEMORY_API_MODEL` | Model for summarization (default: haiku) |
| `EPISODIC_MEMORY_API_BASE_URL` | Custom API endpoint |
| `EPISODIC_MEMORY_API_TOKEN` | Auth token for custom endpoint |
| `EPISODIC_MEMORY_EXCLUDED_PROJECTS` | Comma-separated project names to exclude |
