# Keyboard Shortcuts, IDE Integration, and Daily Workflow

**Applies to:** Claude Code, Cursor, VS Code + Copilot  
**Difficulty:** Beginner → Advanced  
**Last updated:** April 24, 2026
**Related:** [Project Context Files](./01-project-context-files.md) · [Token Optimization](./04-token-optimization-and-context.md) · [Hooks & Security](./05-hooks-security-automation.md)

---

## Claude Code — Terminal Shortcuts

### Essential

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Cancel current generation |
| `Ctrl+D` | Exit session |
| `Shift+Tab` / `Alt+M` | Cycle permission modes |
| `Option+P` (Mac) / `Alt+P` | Switch model |
| `Option+T` / `Alt+T` | Toggle extended thinking |
| `/` | Command/skill autocomplete |
| `!` at start | Bash mode (run command, output joins session) |
| `@` | File mention autocomplete |
| `Esc Esc` | Rewind to a message point |
| `Ctrl+R` | Reverse search command history |

### Multiline Input

| Method | Works In |
|--------|----------|
| `Shift+Enter` | iTerm2, WezTerm, Ghostty, Kitty (no setup) |
| `Option+Enter` | macOS default terminal |
| `Ctrl+J` | Any terminal |
| `\ + Enter` | Works anywhere |

Run `/terminal-setup` to install keybindings for your terminal.

### Text Editing

| Shortcut | Action |
|----------|--------|
| `Ctrl+K` | Delete to end of line |
| `Ctrl+U` | Delete to start of line |
| `Ctrl+Y` | Paste deleted text |
| `Alt+B` / `Alt+F` | Move cursor back/forward one word |
| `Ctrl+G` | Open prompt in default editor |

### Vim Mode

Enable via `/config`. Then: `Esc` for normal mode, `i`/`a`/`o` for insert, `hjkl` for navigation, `dd` delete line, `yy` copy line, `.` repeat.

---

## Cursor — Key Shortcuts

| Shortcut | Action |
|----------|--------|
| `Cmd+K` (Mac) / `Ctrl+K` | Inline edit (select code first) |
| `Cmd+I` / `Ctrl+I` | Open Composer (agent mode, multi-file) |
| `Cmd+L` / `Ctrl+L` | Open chat panel |
| `Cmd+Shift+I` | Composer in full-screen mode |
| `@filename` in chat | Reference a file without pasting |
| `@folder` | Reference entire folder |
| `@web` | Search the web for context |
| `@docs` | Reference indexed documentation |
| `Tab` | Accept autocomplete suggestion |
| `Esc` | Dismiss suggestion |

**Composer mode** (`Cmd+I`) is the power feature — it handles multi-file edits in one go. 3x more efficient than chat for changes spanning 4+ files.

---

## VS Code + Copilot Shortcuts

| Shortcut | Action |
|----------|--------|
| `Tab` | Accept inline suggestion |
| `Esc` | Dismiss suggestion |
| `Alt+]` / `Alt+[` | Next/previous suggestion |
| `Ctrl+Enter` | Open Copilot completions panel |
| `Cmd+I` / `Ctrl+I` | Open Copilot Chat inline |
| `Cmd+Shift+I` | Open Copilot Chat in sidebar |

---

## Daily Workflow Patterns

### The "Start of Day" Pattern

```
1. cd into your project
2. claude (or open Cursor)
3. AI reads CLAUDE.md, loads context
4. "What changed since yesterday?" → git log, PR reviews
5. Pick a task → work with the AI
6. /compact when context gets heavy
7. New session for new task
```

### The "Deep Work" Pattern

For a single complex task:

1. Start fresh session
2. Give detailed first prompt (task + context + constraints + expected output)
3. Let the AI work in plan mode first (review before executing)
4. Iterate in the same session
5. Use verification hooks or `/verify` skill at the end

### The "Exploration" Pattern

For learning or investigating unfamiliar code:

1. Start with a question, not a command: "How does the auth flow work?"
2. Let the AI read files and explain
3. Ask follow-ups: "What would break if I changed X?"
4. Use `context: fork` skills for deep research (doesn't pollute main session)

### The "Pair Programming" Pattern

1. You write the structure/scaffolding
2. AI fills in implementation details
3. You review and refine
4. AI writes tests
5. Both iterate until tests pass

This keeps you in control of architecture while leveraging AI for the grunt work.

---

## Maximizing Output Quality

### The Prompt Stack

Best results come from layering context:

```
CLAUDE.md / .cursorrules      → Project conventions (persistent)
    +
Skill content                 → Domain knowledge (on demand)
    +
Your prompt                   → Specific task (this turn)
    +
@ references / file reads     → Relevant code (focused)
```

Each layer narrows the AI's focus. The more specific the stack, the better the output.

### Test-Driven AI Development

One of the most effective patterns across all tools:

```
"Write tests first for [feature], then implement the code, 
then run the tests. Fix the code until all tests pass."
```

This gives the AI a concrete success criterion. Instead of generating "probably correct" code, it generates code that demonstrably works.

### When the AI Gets Stuck

Signs: repetitive suggestions, going in circles, contradicting itself.

**Fixes:**
1. Start a new session (context may be corrupted/confused)
2. Break the task into smaller pieces
3. Give it a concrete example of what you want
4. Switch to a more capable model (Opus for hard problems)
5. Do the hard part yourself, let AI handle the mechanical parts

---

## Resources & Links
_(Add as you go)_
