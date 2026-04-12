# Project Context Files — CLAUDE.md, .cursorrules, and Equivalents

**Applies to:** Claude Code, Cursor, Windsurf, GitHub Copilot, Codex CLI, Gemini CLI  
**Difficulty:** Beginner → Intermediate → Advanced  
**Last updated:** April 12, 2026

---

## BEGINNER — What Are Project Context Files?

### The Problem They Solve

Every time you start a session with an AI coding assistant, it knows nothing about your project — your conventions, folder structure, preferred tools, or the things it should never do. You'd have to repeat yourself every session.

Project context files solve this. They're markdown files your AI assistant reads at session start, giving it persistent memory of your project's rules and preferences.

### The Equivalents Across Tools

| Tool | File | Location |
|------|------|----------|
| **Claude Code** | `CLAUDE.md` | Project root, `.claude/CLAUDE.md`, or `~/.claude/CLAUDE.md` |
| **Cursor** | `.cursorrules` | Project root |
| **Windsurf** | `.windsurfrules` | Project root |
| **GitHub Copilot** | `.github/copilot-instructions.md` | `.github/` folder |
| **Codex CLI** | `AGENTS.md` | Project root |
| **Aider** | `.aider.conf.yml` + conventions file | Project root |

The concept is identical across all tools — a persistent instruction file that shapes every AI response in your project.

### What Goes In It

**Always include:**
- Build & test commands ("Run `pnpm test` before committing")
- Code style rules ("Use 2-space indent, TypeScript strict mode")
- Project structure overview ("API handlers in `src/api/`, components in `src/ui/`")
- Naming conventions ("PascalCase for components, camelCase for utilities")
- "Never do X" rules ("Don't commit to main directly", "Don't modify package-lock.json")
- Key dependencies and why they were chosen

**Example CLAUDE.md:**
```markdown
# Project: InvoiceAPI

## Build & Run
- `pnpm dev` — start dev server
- `pnpm test` — run vitest
- `pnpm lint` — ESLint + Prettier check

## Code Style
- TypeScript strict mode everywhere
- Use Zod for all input validation
- Prefer `async/await` over `.then()` chains
- Error handling: throw custom errors from `src/errors/`

## Architecture
- `src/api/` — Express route handlers
- `src/services/` — Business logic (no HTTP concepts here)
- `src/db/` — Drizzle ORM schemas and queries
- `src/types/` — Shared TypeScript types

## Rules
- Never use `any` type
- Never commit .env files
- Always run tests before suggesting a commit
- Use conventional commits (feat:, fix:, chore:)
```

**Example .cursorrules:**
```markdown
You are a senior TypeScript developer working on a Next.js 14 app.

Rules:
- Use App Router, not Pages Router
- Server Components by default, 'use client' only when needed
- Use Tailwind CSS, no custom CSS files
- Prefer Server Actions over API routes for mutations
- Use Zod for form validation
- Write tests with Vitest and Testing Library

Project structure:
- app/ — routes and layouts
- components/ — reusable UI components
- lib/ — utilities and helpers
- db/ — Drizzle schema and migrations
```

---

## INTERMEDIATE — Optimizing Your Context File

### Keep It Under 200 Lines

This is the most important rule. Your context file is loaded on **every single request**, consuming tokens every time. A 500-line file means thousands of wasted tokens per session.

The fix: move reference material elsewhere.

| Content Type | Where It Should Live |
|-------------|---------------------|
| "Always do X" rules | CLAUDE.md / .cursorrules |
| API documentation | Skills (Claude) or linked docs |
| Directory-specific rules | `.claude/rules/` with path scoping |
| Personal preferences | `CLAUDE.local.md` (gitignored) |
| Complex workflows | Skills or custom commands |

### Path-Scoped Rules (Claude Code)

Instead of one massive CLAUDE.md, split rules by directory:

```
.claude/
├── CLAUDE.md              # Core rules (small)
└── rules/
    ├── api.md             # API-specific rules
    ├── testing.md         # Test conventions
    └── frontend/
        └── react.md       # React-specific rules
```

Add `paths` frontmatter so rules only load when relevant:
```yaml
---
paths:
  - "src/api/**/*.ts"
---
# API rules here — only loaded when Claude reads API files
```

This saves significant context — rules only appear when Claude is working in matching directories.

### File Imports (Claude Code)

Reference external files instead of inlining content:
```markdown
See @README.md for project overview.
@docs/api-spec.md
```

The `@path/to/file` syntax expands the file into context. Max depth: 5 hops.

### The CLAUDE.md Hierarchy

Claude Code loads multiple files, merged in priority order:

1. **Managed policy** (org-wide, can't override): `/Library/Application Support/ClaudeCode/CLAUDE.md`
2. **Project**: `./CLAUDE.md` or `./.claude/CLAUDE.md` (committed to git)
3. **User**: `~/.claude/CLAUDE.md` (personal, all projects)
4. **Local**: `./CLAUDE.local.md` (gitignored, personal per-project)

All discovered files concatenate — they don't override each other. Claude walks UP the directory tree, so monorepo subdirectories can have their own CLAUDE.md.

### Cursor-Specific Optimization

Cursor reads only one `.cursorrules` from the project root. For monorepos, combine rules into sections:

```markdown
## Global Rules
...

## Frontend (packages/web/)
...

## Backend (packages/api/)
...
```

Keep under 2,000 words. Longer rules dilute the AI's attention — focus on your top 10–15 most important conventions.

---

## ADVANCED — Power Patterns

### Surviving Compaction

When Claude Code runs `/compact`, it summarizes the conversation but re-reads CLAUDE.md from disk. Your rules survive. But nuances from conversation might be lost.

**Pattern:** Add a `SessionStart` hook that re-injects critical context after compaction:
```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "compact",
      "hooks": [{
        "type": "command",
        "command": "echo 'Current sprint: auth refactor. Use Bun not npm.'"
      }]
    }]
  }
}
```

### Excluding Files in Monorepos

Other teams' CLAUDE.md files can pollute your context:
```json
// .claude/settings.local.json
{
  "claudeMdExcludes": [
    "**/other-team/CLAUDE.md",
    "**/legacy-app/.claude/**"
  ]
}
```

### Testing Your Context File

A good context file should be testable. Try:
1. Start a fresh session
2. Ask the assistant to do something your rules should catch
3. Does it follow the convention without being reminded?
4. If not, the rule is too vague or buried

**Good rule:** "Use `pnpm`, never `npm` or `yarn`"  
**Bad rule:** "Follow standard package management practices"

Specific, actionable, short. That's what sticks.

### Cross-Tool Portability

If you use multiple AI tools, maintain one source of truth and generate tool-specific files:

```bash
# Script: generate-ai-rules.sh
cat rules/base.md > CLAUDE.md
cat rules/base.md rules/cursor-specific.md > .cursorrules
cat rules/base.md > .github/copilot-instructions.md
```

This prevents drift between tools.

---

## Resources & Links
_(Add as you go)_
