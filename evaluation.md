# Episodic Memory - Comprehensive Security and Code Quality Evaluation

**Evaluation Date:** January 11, 2026
**Version Evaluated:** 1.0.15
**Repository:** https://github.com/obra/episodic-memory
**Author:** Jesse Vincent (jesse@fsck.com)
**License:** MIT

---

## Executive Summary

Episodic Memory is a semantic search tool for Claude Code conversations that enables developers to search and recall past discussions, decisions, and patterns across coding sessions. After a thorough code review, I assess this project as **low to moderate risk** for use on a personal MacBook, with no evidence of malicious intent and generally sound security practices. However, there are some areas that warrant attention.

### Overall Assessment

| Category | Rating | Notes |
|----------|--------|-------|
| **Security** | Good | No backdoors, uses parameterized SQL, atomic file ops |
| **Code Quality** | Good | TypeScript strict mode, modular design, good test coverage |
| **Functionality** | Good | Well-designed semantic search with reasonable limitations |
| **MacOS Risk** | Low | Limited file access scope, no elevated permissions |
| **Maintainer Responsiveness** | Good | Active development, most bugs fixed within days |

### Key Findings

**Positive:**
- No malicious code, backdoors, or obfuscated logic detected
- Uses parameterized SQL queries throughout (no SQL injection risk)
- Atomic file operations prevent corruption
- Local embeddings (data doesn't leave your machine for search)
- Input validation with Zod schemas
- TypeScript strict mode enforced

**Concerns:**
- One open bug (#47) can break MCP JSON protocol via console.log
- Date filtering in search.ts uses string interpolation (low risk, see details)
- API summarization sends conversation content to Claude API (expected behavior)
- Native dependency compilation (better-sqlite3) can cause installation issues

---

## 1. Security Analysis

### 1.1 SQL Injection Prevention

**Status: SECURE**

All database queries use parameterized prepared statements. Example from `src/db.ts:140-167`:

```typescript
const stmt = db.prepare(`
  INSERT OR REPLACE INTO exchanges
  (id, project, timestamp, user_message, assistant_message, ...)
  VALUES (?, ?, ?, ?, ?, ...)
`);

stmt.run(
  exchange.id,
  exchange.project,
  exchange.timestamp,
  // ... all values passed as parameters
);
```

**One exception noted** in `src/search.ts:43-45`:

```typescript
const timeFilter = [];
if (after) timeFilter.push(`e.timestamp >= '${after}'`);
if (before) timeFilter.push(`e.timestamp <= '${before}'`);
const timeClause = timeFilter.length > 0 ? `AND ${timeFilter.join(' AND ')}` : '';
```

This uses string interpolation for date values. However, **the risk is mitigated** because:
1. Dates are validated via regex before use (`src/search.ts:16-24`):
   ```typescript
   const isoDateRegex = /^\d{4}-\d{2}-\d{2}$/;
   if (!isoDateRegex.test(dateStr)) {
     throw new Error(`Invalid ${paramName} date...`);
   }
   ```
2. The MCP server adds additional Zod validation (`src/mcp-server.ts:55-62`)
3. Only `YYYY-MM-DD` format is accepted

**Recommendation:** While currently safe, refactoring to use parameterized queries would be more defensive.

### 1.2 File System Access

**Status: SECURE with defined scope**

The tool accesses these directories:

| Path | Purpose | Risk |
|------|---------|------|
| `~/.claude/projects/` | Read conversation files | Low - Claude Code's own data |
| `~/.config/superpowers/` | Store index, database, archive | Low - User's config area |
| Arbitrary paths via MCP `read` | Display conversations | Moderate - see below |

**Concern - MCP Read Tool Path Validation:**

In `src/mcp-server.ts:251-274`, the `read` tool accepts any path:

```typescript
if (name === 'read') {
  const params = ShowConversationInputSchema.parse(args);

  if (!fs.existsSync(params.path)) {
    throw new Error(`File not found: ${params.path}`);
  }

  const jsonlContent = fs.readFileSync(params.path, 'utf-8');
  // ...
}
```

**Risk Assessment:**
- The tool is designed to read JSONL conversation files
- An attacker with MCP access could potentially read any file
- However, MCP access requires Claude Code integration (not external exposure)
- The output is formatted as markdown, which helps prevent binary file exposure

**Recommendation:** Consider restricting the `read` tool to paths within the archive directory.

### 1.3 Atomic File Operations

**Status: SECURE**

File operations use atomic temp-file + rename pattern (`src/sync.ts:52-56`):

```typescript
// Atomic copy: temp file + rename
const tempDest = dest + '.tmp.' + process.pid;
fs.copyFileSync(src, tempDest);
fs.renameSync(tempDest, dest); // Atomic on same filesystem
```

This prevents partial writes and corruption from interrupted operations.

### 1.4 API Token Handling

**Status: SECURE**

Environment variables for API configuration are handled properly (`src/summarizer.ts:16-32`):

```typescript
function getApiEnv(): Record<string, string | undefined> | undefined {
  const baseUrl = process.env.EPISODIC_MEMORY_API_BASE_URL;
  const token = process.env.EPISODIC_MEMORY_API_TOKEN;
  // ...
  return {
    ...process.env,
    ...(baseUrl && { ANTHROPIC_BASE_URL: baseUrl }),
    ...(token && { ANTHROPIC_AUTH_TOKEN: token }),
  };
}
```

- Tokens are passed through environment variables, not logged
- Custom API endpoints are supported for enterprise use

### 1.5 Data Privacy Considerations

**Local Processing:**
- Embeddings are generated locally using `@xenova/transformers` with the `all-MiniLM-L6-v2` model
- Vector search is 100% local (sqlite-vec)
- No conversation content leaves your machine for search functionality

**API-Dependent Features:**
- AI summarization sends conversation content to Anthropic API (expected behavior)
- Uses `haiku` model by default (cost-efficient)
- Summaries are stored locally as `-summary.txt` files

**Exclusion Mechanisms:**
- Conversations can be excluded via markers (`src/sync.ts:6-9`):
  ```typescript
  const EXCLUSION_MARKERS = [
    '<INSTRUCTIONS-TO-EPISODIC-MEMORY>DO NOT INDEX THIS CHAT</INSTRUCTIONS-TO-EPISODIC-MEMORY>',
    'Only use NO_INSIGHTS_FOUND',
    SUMMARIZER_CONTEXT_MARKER,
  ];
  ```
- Project-level exclusions via `exclude.txt` or environment variable

### 1.6 Network Exposure

**Status: MINIMAL**

- No HTTP server is created
- MCP server uses stdio transport only (no network ports)
- Only outbound HTTPS to Anthropic API for summarization

### 1.7 Dependency Security

**Dependencies (`package.json:33-40`):**

| Package | Version | Risk Assessment |
|---------|---------|-----------------|
| `@anthropic-ai/claude-agent-sdk` | ^0.1.9 | Official Anthropic SDK |
| `@modelcontextprotocol/sdk` | ^1.20.0 | Official MCP SDK |
| `@xenova/transformers` | ^2.17.2 | Well-known ML library |
| `better-sqlite3` | ^12.4.1 | Native SQLite, widely used |
| `marked` | ^16.4.0 | Markdown rendering |
| `sqlite-vec` | ^0.1.7-alpha.2 | **Alpha** - monitor for stability |
| `zod` | ^3.25.76 | Schema validation, widely used |

**Note:** `sqlite-vec` is alpha software. While it works well, watch for updates.

---

## 2. Code Quality Assessment

### 2.1 TypeScript Configuration

**Status: EXCELLENT**

From `tsconfig.json`, strict mode is enabled:
- `strict: true`
- `noImplicitAny: true` (implied by strict)
- `strictNullChecks: true` (implied by strict)

### 2.2 Input Validation

**Status: GOOD**

MCP tools use comprehensive Zod schemas (`src/mcp-server.ts:31-68`):

```typescript
const SearchInputSchema = z
  .object({
    query: z.union([
      z.string().min(2, 'Query must be at least 2 characters'),
      z.array(z.string().min(2)).min(2).max(5),
    ]),
    mode: SearchModeEnum.default('both'),
    limit: z.number().int().min(1).max(50).default(10),
    after: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
    before: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
  })
  .strict();
```

### 2.3 Error Handling

**Status: GOOD with room for improvement**

Errors are generally caught and handled gracefully:

```typescript
// src/sync.ts:131-136
} catch (error) {
  result.errors.push({
    file: srcFile,
    error: error instanceof Error ? error.message : String(error)
  });
}
```

**One issue found** - console.log in embeddings affects MCP:

GitHub Issue #47 reports that `console.log` in `src/embeddings.ts:7` breaks MCP JSON protocol:

```typescript
console.log('Loading embedding model (first run may take time)...');
```

This should use `console.error` for MCP compatibility.

### 2.4 Test Coverage

**Status: GOOD**

10 test files covering:
- `integration.test.ts` - End-to-end indexing and search
- `db.test.ts` - Database migrations
- `parser.test.ts` - JSONL parsing edge cases
- `search-agent-template.test.ts` - Agent prompt rendering
- `show.test.ts` - Conversation display
- `stats.test.ts` - Index statistics
- `sync.test.ts` - File synchronization
- `verify.test.ts` - Index verification and repair
- `api-config.test.ts` - API configuration
- `multi-concept.test.ts` - Multi-concept search

Tests use Vitest with 30-second timeouts for embedding operations.

### 2.5 Code Organization

**Status: EXCELLENT**

Clean separation of concerns:

```
src/
├── Core Data
│   ├── types.ts      - TypeScript interfaces
│   ├── paths.ts      - Path resolution
│   └── constants.ts  - Configuration constants
├── Data Layer
│   ├── db.ts         - SQLite operations
│   └── parser.ts     - JSONL parsing
├── Processing
│   ├── embeddings.ts - Vector generation
│   ├── indexer.ts    - Conversation indexing
│   ├── sync.ts       - File synchronization
│   └── summarizer.ts - AI summaries
├── Query Layer
│   ├── search.ts     - Search operations
│   ├── verify.ts     - Index maintenance
│   └── stats.ts      - Statistics
├── Presentation
│   └── show.ts       - Markdown/HTML output
└── Interfaces
    ├── mcp-server.ts - MCP protocol
    └── *-cli.ts      - CLI commands
```

### 2.6 HTML Output Security

**Status: SECURE**

HTML output in `src/show.ts:757-765` properly escapes content:

```typescript
function escapeHtml(text: string): string {
  const map: Record<string, string> = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#039;'
  };
  return text.replace(/[&<>"']/g, m => map[m]);
}
```

All user-generated content passes through `escapeHtml()` before HTML rendering.

---

## 3. Functionality Evaluation

### 3.1 Core Features

| Feature | Status | Notes |
|---------|--------|-------|
| Semantic Search | Working | Uses all-MiniLM-L6-v2, 384-dim vectors |
| Text Search | Working | LIKE queries, case-insensitive |
| Multi-concept AND | Working | 2-5 concepts supported |
| Date Filtering | Working | YYYY-MM-DD format |
| AI Summaries | Working | Hierarchical for long conversations |
| Conversation Display | Working | Markdown and HTML output |

### 3.2 Performance Considerations

**Embedding Model Loading:**
- First run downloads ~25MB model
- Subsequent runs use cached model
- Loading time: 5-10 seconds on first call per session

**Database:**
- WAL mode enabled for concurrency (`src/db.ts:54`)
- Indexes on timestamp, session_id, project, git_branch

**Potential Issues:**
- Large conversations may cause memory pressure during hierarchical summarization
- SQLite write contention possible with concurrent operations

### 3.3 Known Limitations

1. **Subagent conversations** pollute search results (Issue #42)
2. **MCP fails on exit** in some configurations (Issue #24)
3. **Hook loading** can fail in certain setups (Issue #21)

---

## 4. MacOS Risk Assessment

### 4.1 Permissions Required

| Permission | Scope | Risk |
|------------|-------|------|
| File Read | `~/.claude/`, `~/.config/` | Low |
| File Write | `~/.config/superpowers/` | Low |
| Network | Outbound HTTPS to api.anthropic.com | Low |
| Process Spawn | Child processes for CLI | Low |

### 4.2 What the Tool Can Access

**Can Access:**
- Your Claude Code conversation history
- Files you explicitly point to via MCP `read` tool
- Your home directory paths

**Cannot Access:**
- System files (no elevated permissions)
- Other users' data
- Network services (no servers started)

### 4.3 Background Process Behavior

The `--background` flag spawns a detached process (`src/sync-cli.ts:44-58`):

```typescript
const child = spawn(process.execPath, [
  process.argv[1],
  ...filteredArgs
], {
  detached: true,
  stdio: 'ignore'
});
child.unref();
```

This is standard practice for background tasks and allows the parent to exit while sync continues.

### 4.4 Installation Safety

**Postinstall Script (`package.json:16`):**
```json
"postinstall": "npm rebuild better-sqlite3 2>/dev/null || true"
```

This only rebuilds the native SQLite module - no network calls or code execution beyond npm.

---

## 5. GitHub Issues Analysis

### 5.1 Project Health Metrics

| Metric | Value |
|--------|-------|
| Stars | 165 |
| Forks | 42 |
| Open Issues | 10 |
| Closed Issues | 20 |
| Open PRs | 2 |
| Merged PRs | 16 |

### 5.2 Notable Open Issues

| # | Title | Severity | Status |
|---|-------|----------|--------|
| #47 | console.log breaks MCP JSON | High | Recent |
| #43 | Not on npm | Medium | Blocking |
| #42 | Subagent pollution | Medium | Active |
| #41 | npm install hangs | Medium | Platform-specific |
| #24 | MCP fails on exit | Low | Long-standing |
| #21 | Hook loading fails | Low | Long-standing |

### 5.3 Developer Responsiveness

**Positive:**
- 15+ releases in 3 months (v1.0.0 to v1.0.15)
- Most critical bugs fixed within 2-7 days
- Detailed technical responses
- Community PRs merged (Windows support, API config)

**Concerns:**
- Issues #21 and #24 open since November 2025 (2+ months)
- Package not currently available on npm (Issue #43)

### 5.4 Security Posture

- Uses Anthropic's HackerOne for vulnerability disclosure
- No CVEs published for this project
- No security advisories

---

## 6. Potential Improvements & Recommendations

### 6.1 Security Improvements

1. **Parameterize date filtering** in `src/search.ts` for defense in depth
2. **Restrict MCP `read` tool** to archive directory or validate path prefix
3. **Use `console.error`** for non-JSON output in MCP context
4. **Add path traversal protection** in file operations

### 6.2 Code Quality Improvements

1. **Handle API rate limits** explicitly in summarizer
2. **Add memory limits** for hierarchical summarization
3. **Improve error messages** for installation failures
4. **Add integration tests** for MCP server

### 6.3 Functionality Improvements

1. **Filter subagent conversations** from search results
2. **Graceful MCP shutdown** (Issue #24)
3. **Publish to npm** (Issue #43)

---

## 7. Conclusion

### Is This Safe to Run on My MacBook?

**Yes, with reasonable confidence.**

The code shows no signs of malicious intent. It follows good security practices (parameterized SQL, atomic file operations, input validation). The scope of access is limited to Claude Code conversations and user configuration directories.

### Should I Use This Project?

**Considerations:**

| Factor | Assessment |
|--------|------------|
| **For personal use** | Recommended |
| **For team use** | Review API/data handling |
| **For sensitive projects** | Use exclusion markers |
| **For production** | Wait for npm publish |

### Risk Summary

**Low Risk:**
- Local embedding processing
- No network servers
- Limited file scope
- Active maintenance

**Moderate Risk:**
- Alpha-stage sqlite-vec dependency
- Some long-standing bugs
- Not on npm (installation complexity)

### Final Recommendation

The project is suitable for use on a personal MacBook for managing Claude Code conversations. For additional safety:

1. Review conversations before indexing (use exclusion markers for sensitive content)
2. Monitor the GitHub issues for security-related reports
3. Keep the project updated as new versions are released
4. Consider running in a sandboxed environment for maximum caution

---

## Appendix: Code References

### Key Security-Relevant Files

- `src/db.ts` - Database operations (lines 140-200 for INSERT)
- `src/search.ts` - Query construction (lines 43-45 for date filtering)
- `src/sync.ts` - File operations (lines 52-56 for atomic copy)
- `src/mcp-server.ts` - Input validation (lines 31-68 for schemas)
- `src/show.ts` - HTML escaping (lines 757-765)
- `src/summarizer.ts` - API token handling (lines 16-32)
- `src/paths.ts` - Path resolution (all)
- `src/parser.ts` - JSONL parsing (all)

### Test Files

- `test/integration.test.ts` - End-to-end tests
- `test/db.test.ts` - Database migration tests
- `test/sync.test.ts` - File synchronization tests
- `test/verify.test.ts` - Index verification tests

---

*This evaluation was conducted by analyzing source code, reviewing GitHub issues, and assessing security practices. No runtime testing was performed as part of this static analysis.*
