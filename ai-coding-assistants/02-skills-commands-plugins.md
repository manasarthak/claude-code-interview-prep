# Skills, Commands, and Plugins

**Applies to:** Claude Code (primary), Cursor (custom commands), Copilot (extensions)  
**Difficulty:** Beginner → Intermediate → Advanced  
**Last updated:** April 12, 2026

---

## BEGINNER — What Are Skills?

### The Concept

Skills are reusable instruction files that teach your AI assistant how to perform specific tasks. Think of them as playbooks — when you say `/deploy` or `/review-pr`, the assistant loads a markdown file with detailed instructions for that workflow.

They're different from CLAUDE.md:
- **CLAUDE.md** = "always know this" (loaded every session)
- **Skills** = "know this when asked" (loaded on demand, saving tokens)

### How Skills Work in Claude Code

A skill is a folder containing a `SKILL.md` file:

```
.claude/skills/
├── deploy/
│   └── SKILL.md
├── review-pr/
│   └── SKILL.md
└── create-migration/
    └── SKILL.md
```

You invoke them as slash commands: `/deploy`, `/review-pr`, `/create-migration`.

Claude can also auto-invoke skills when it detects a relevant task — based on the skill's description.

### SKILL.md Structure

```yaml
---
name: deploy
description: Deploy the application to staging or production
---

# Deploy Workflow

1. Run tests: `pnpm test`
2. Build: `pnpm build`
3. If deploying to production, ask for confirmation
4. Run: `./scripts/deploy.sh $ARGUMENTS`
5. Verify health check: `curl https://api.example.com/health`
```

The frontmatter is optional but powerful. The body is the instructions Claude follows.

### Skill Locations

| Location | Scope | Use Case |
|----------|-------|----------|
| `.claude/skills/<name>/SKILL.md` | This project only | Project-specific workflows |
| `~/.claude/skills/<name>/SKILL.md` | All your projects | Personal utilities |
| Plugin skills | Where plugin is enabled | Third-party/community |

### Cursor Custom Commands

Cursor's equivalent lives in Settings → Commands → Add Custom Command. You define a name and instructions, then invoke with `/<name>` in chat:

- `/review` — Code review with your standards
- `/test` — Generate tests for selected code
- `/refactor` — Optimize with specific patterns
- `/docs` — Generate documentation

### Built-In Slash Commands (Claude Code)

| Command | What It Does |
|---------|-------------|
| `/help` | Show available commands |
| `/clear` | Clear conversation, start fresh |
| `/compact` | Summarize conversation to free context |
| `/resume` | Pick and resume a previous session |
| `/rename` | Rename current session |
| `/init` | Auto-generate CLAUDE.md by analyzing your codebase |
| `/memory` | Browse/edit CLAUDE.md and auto-memory files |
| `/config` | Open settings UI |
| `/mcp` | Manage MCP server connections |
| `/hooks` | Browse configured hooks |
| `/debug` | Enable debug logging |
| `/theme` | Change terminal color theme |
| `/terminal-setup` | Install keybindings (Shift+Enter for multiline) |
| `/?` | Show all available commands and skills |

---

## INTERMEDIATE — Writing Effective Skills

### Frontmatter Options

```yaml
---
name: migrate-component              # Becomes /migrate-component
description: Migrate a React component from class to functional
disable-model-invocation: true       # Only YOU can trigger, not auto
user-invocable: true                 # Appears in / menu (true by default)
allowed-tools: Bash(git *) Read Edit # Tools pre-approved for this skill
context: fork                        # Run in isolated subagent
agent: Plan                          # Which agent type for subagent
argument-hint: [component-name]      # Shown in autocomplete
model: claude-opus-4-6               # Override model for this skill
effort: high                         # Override effort level
paths:                               # Only activate for matching files
  - "src/components/**/*.tsx"
shell: bash                          # bash or powershell
---
```

**Key decisions:**
- `disable-model-invocation: true` — Use for anything with side effects (deploy, commit, delete). Prevents Claude from auto-triggering it.
- `context: fork` — Use for heavy research tasks so they don't pollute your main conversation.
- `paths` — Use to keep skills focused. A React skill doesn't need to load when you're editing SQL.

### Argument Substitution

Skills receive arguments from the user:

```yaml
---
name: create-component
argument-hint: [ComponentName]
---

Create a new React component called $0 in src/components/$0/.
Include: $0.tsx, $0.test.tsx, $0.stories.tsx, index.ts barrel export.
```

Invoke: `/create-component UserProfile`  
`$0` becomes `UserProfile`.

Multiple args: `$0`, `$1`, `$2`, etc. Or use `$ARGUMENTS` for the full string.

### Dynamic Context with Shell Preprocessing

Pull live data into your skill before Claude sees it:

```yaml
---
name: pr-summary
---

## Current PR Context

- Branch: !`git branch --show-current`
- Changed files: !`git diff --name-only main`
- PR description: !`gh pr view --json body -q .body`

## Task

Summarize this PR for the team. Focus on what changed, why, and any risks.
```

The `` !`command` `` syntax runs the command and injects its output. Claude sees the results, not the command.

### Supporting Files

Skills can have companion files:

```
review-pr/
├── SKILL.md           # Main instructions
├── checklist.md       # Detailed review checklist
├── examples/
│   ├── good-pr.md     # Example of a well-structured PR
│   └── bad-pr.md      # Common anti-patterns
└── scripts/
    └── collect-stats.sh
```

Reference them from SKILL.md:
```markdown
Follow the review checklist in [checklist.md](checklist.md).
See [examples/](examples/) for reference.
```

### Skills vs CLAUDE.md vs Rules — Decision Matrix

| Question | → Use |
|----------|-------|
| Should this apply to every conversation? | CLAUDE.md |
| Should this only apply to certain file types? | `.claude/rules/` with `paths` |
| Is this a workflow someone invokes? | Skill with `user-invocable: true` |
| Is this reference material Claude should use when relevant? | Skill with `user-invocable: false` |
| Must this happen deterministically, 100% of the time? | Hook (not a skill) |

---

## ADVANCED — Production Skill Patterns

### The Verification Skill

A skill that runs after any major change to verify quality:

```yaml
---
name: verify
description: Verify code quality after changes
context: fork
agent: general-purpose
---

1. Run the test suite and capture output
2. Run the linter
3. Check for any TypeScript errors
4. Look for console.log statements in changed files
5. Verify no .env files were modified
6. Report a pass/fail summary
```

Running in `context: fork` keeps the verification isolated from your main session.

### Chaining Skills

One skill can invoke another:

```yaml
---
name: ship
description: Full ship workflow
disable-model-invocation: true
---

Execute these in order:
1. Use the /verify skill to check code quality
2. If verification passes, use /deploy staging
3. Run smoke tests against staging
4. If smoke tests pass, ask for production deploy confirmation
```

### Community Skill Ecosystem (2026)

As of 2026, there are 1,200+ community skills following the universal SKILL.md format. They work across Claude Code, Cursor, Gemini CLI, Codex CLI, and others. Common categories:

- **Code quality:** review, lint, refactor, simplify
- **Git workflows:** commit, PR creation, changelog generation
- **Testing:** generate tests, mutation testing, coverage analysis
- **Documentation:** README generation, API docs, architecture diagrams
- **DevOps:** deploy, rollback, infrastructure checks

### MCP + Skills = Powerful Combinations

MCP provides the *connections* (tools), Skills provide the *knowledge* (how to use them):

```
MCP: Connects to your Postgres database
Skill: Knows your data model, common queries, naming conventions

MCP: Connects to Slack
Skill: Knows which channels to post to, message formatting, notification rules

MCP: Connects to GitHub
Skill: Knows your PR template, review checklist, branch naming convention
```

### Plugins (Claude Code)

Plugins bundle skills + MCP servers + tools into installable packages:

```
my-plugin/
├── skills/
│   ├── deploy/SKILL.md
│   └── monitor/SKILL.md
├── mcp.json              # MCP server configs
├── settings.json         # Permission rules
└── plugin.json           # Plugin metadata
```

Plugins are the distribution mechanism — skills are the building blocks inside them.

---

## Resources & Links
_(Add as you go)_
