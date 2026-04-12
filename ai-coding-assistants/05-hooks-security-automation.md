# Hooks, Security, and Deterministic Automation

**Applies to:** Claude Code (hooks are native), general security applies to all tools  
**Difficulty:** Beginner → Intermediate → Advanced  
**Last updated:** April 12, 2026

---

## BEGINNER — Why Hooks Exist

### The Problem

CLAUDE.md instructions are "advisory" — the AI tries to follow them but might not 100% of the time. If you say "always run prettier after editing," the AI might forget on the 15th edit.

Hooks are **deterministic**. They're shell commands (or LLM evaluations) that fire automatically at specific lifecycle points. If you set up a hook to run prettier after every file edit, it runs every single time. No exceptions.

### When to Use Hooks vs CLAUDE.md

| Need | Use |
|------|-----|
| "Please try to format code after edits" | CLAUDE.md (advisory) |
| "Code MUST be formatted after every edit, always" | Hook (deterministic) |
| "Prefer conventional commits" | CLAUDE.md |
| "Block any commit that doesn't follow conventional format" | Hook |
| "Be careful with .env files" | CLAUDE.md |
| "Never allow edits to .env files" | Hook + permission deny rule |

### Hook Events (When They Fire)

| Event | When | Common Use |
|-------|------|-----------|
| `SessionStart` | Session begins or resumes | Inject context, setup checks |
| `PreToolUse` | Before a tool executes | Block dangerous operations |
| `PostToolUse` | After a tool succeeds | Auto-format, auto-lint |
| `Stop` | Claude finishes responding | Verify task completion |
| `Notification` | Claude waiting for input | Desktop notification |

### Your First Hook — Auto-Format After Edits

In `.claude/settings.json`:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

Every time Claude edits or writes a file → prettier runs on it automatically.

---

## INTERMEDIATE — Security Best Practices

### Permission Modes (Claude Code)

| Mode | Behavior | Use When |
|------|----------|----------|
| Default | Prompts for each tool use | Learning, sensitive work |
| Plan mode | Shows plan first, you approve | Complex changes |
| Accept edits | Auto-approves file edits, prompts for rest | Trusted editing session |
| Auto mode | Classifier auto-decides based on rules | Experienced users |
| Bypass | Skip all prompts | CI/CD only, never interactive |

Cycle modes with `Shift+Tab`.

### Permission Rules

Control exactly what the AI can and cannot do:

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(pnpm test)"
    ],
    "deny": [
      "Edit(.env*)",
      "Bash(sudo *)",
      "Bash(rm -rf *)",
      "Bash(curl * | sh)"
    ]
  }
}
```

**Deny always wins.** If something is in both allow and deny, it's denied.

### Security Rules That Apply to ALL AI Assistants

**1. Never put secrets in context files**
- No API keys in CLAUDE.md, .cursorrules, or any committed file
- Use environment variables or `.local` files (gitignored)

**2. Review before executing**
- AI can generate malicious or destructive commands accidentally
- Always read what it's about to run, especially `rm`, `git push --force`, database mutations

**3. Be wary of prompt injection**
- If the AI reads a file or webpage containing hidden instructions, it might follow them
- Claude Code has built-in defenses, but stay vigilant with untrusted content
- Never blindly run AI-generated code that processes untrusted input

**4. Scope permissions narrowly**
- Give the AI access to what it needs, not everything
- In Cursor: limit workspace trust settings
- In Claude Code: use allow/deny rules

**5. Audit trail**
- Claude Code logs sessions (viewable with `/resume`)
- Set up hooks to log high-risk operations
- Review what the AI did, especially in auto mode

### Protecting Sensitive Files

Hook to block edits to protected files:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "./.claude/hooks/protect-files.sh"
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/protect-files.sh
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path')

PROTECTED=(".env" "package-lock.json" "secrets/" ".git/")
for pattern in "${PROTECTED[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern $pattern" >&2
    exit 2  # exit 2 = block the action
  fi
done
exit 0  # exit 0 = allow
```

---

## ADVANCED — Sophisticated Hook Patterns

### Prompt-Based Hooks (LLM Judgment)

When the decision requires judgment, not just pattern matching:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Review the changes made in this session. Are there any security issues, missing error handling, or untested code paths? Return {\"ok\": true} if clean, or {\"ok\": false, \"reason\": \"...\"} if issues found."
          }
        ]
      }
    ]
  }
}
```

This runs a separate LLM check every time Claude finishes a response. Returns `ok: false` to force Claude to address issues.

### Agent-Based Hooks (Multi-Step Verification)

For verification that needs to read files and run commands:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Run the test suite. If any tests fail, report which ones and why.",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

Agent hooks can use tools (Read, Bash, Glob) to verify work.

### Configuration Change Auditing

Log whenever config files are modified:

```json
{
  "hooks": {
    "ConfigChange": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "jq -c '{timestamp: now | todate, file: .file_path}' >> ~/claude-audit.log"
          }
        ]
      }
    ]
  }
}
```

### Pre-Commit Quality Gate

Block commits that don't meet standards:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "if": "Bash(git commit*)",
        "hooks": [
          {
            "type": "command",
            "command": "./.claude/hooks/pre-commit-check.sh"
          }
        ]
      }
    ]
  }
}
```

### Hook Precedence & Permissions

Critical understanding:
- `PreToolUse` hooks fire BEFORE permission checks
- Hook denial overrides even bypass mode
- Hooks can tighten restrictions but never loosen them past permission rules
- Deny rules in settings always win, regardless of hooks

This means hooks are the strongest enforcement mechanism available.

---

## Resources & Links
_(Add as you go)_
