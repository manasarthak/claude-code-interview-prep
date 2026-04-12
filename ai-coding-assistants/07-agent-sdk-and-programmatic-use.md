# Agent SDK & Programmatic AI Usage

**Applies to:** Claude Agent SDK, OpenAI Agents SDK, LangChain, custom agents  
**Difficulty:** Intermediate → Advanced  
**Last updated:** April 12, 2026

---

## What Is the Agent SDK?

The Claude Agent SDK lets you use Claude Code as a library — same tools, agent loop, and context management, but controlled from your own code. Think of it as Claude Code without the terminal UI.

This is how you build production automation: CI/CD bots, code review pipelines, monitoring agents, custom dev tools.

### When to Use SDK vs CLI

| Use CLI (Interactive) | Use SDK (Programmatic) |
|-----------------------|------------------------|
| Day-to-day development | CI/CD pipelines |
| Exploring code | Automated code reviews |
| One-off tasks | Scheduled monitoring |
| Learning and experimenting | Custom applications |
| Pair programming | Multi-agent systems |

Many teams use both — CLI for development, SDK for production automation.

---

## Getting Started

### Installation

```bash
# TypeScript
npm install @anthropic-ai/claude-agent-sdk

# Python
pip install claude-agent-sdk
```

### Authentication

Set `ANTHROPIC_API_KEY` environment variable. Or configure third-party providers (AWS Bedrock, Google Vertex AI).

### Basic Example (Python)

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Find and fix the bug in auth.py",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash"]
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

### Basic Example (TypeScript)

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and fix the bug in auth.py",
  options: { allowedTools: ["Read", "Edit", "Bash"] }
})) {
  if ("result" in message) console.log(message.result);
}
```

---

## Key Capabilities

### Built-In Tools

Same tools available as in CLI: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, Agent (subagents), AskUserQuestion.

### Sessions (Persistent Context)

```python
# First query — start a session
session_id = None
async for message in query(
    prompt="Read the auth module and understand it",
    options=ClaudeAgentOptions(allowed_tools=["Read", "Glob"])
):
    if isinstance(message, SystemMessage) and message.subtype == "init":
        session_id = message.data["session_id"]

# Second query — resume with full context
async for message in query(
    prompt="Now find all callers of that module",
    options=ClaudeAgentOptions(resume=session_id)
):
    if isinstance(message, ResultMessage):
        print(message.result)
```

### Hooks (Programmatic)

```python
async def audit_edits(input_data, tool_use_id, context):
    file_path = input_data.get("tool_input", {}).get("file_path")
    with open("audit.log", "a") as f:
        f.write(f"Edited: {file_path}\n")
    return {}

async for message in query(
    prompt="Refactor utils.py",
    options=ClaudeAgentOptions(
        hooks={
            "PostToolUse": [
                HookMatcher(matcher="Edit|Write", hooks=[audit_edits])
            ]
        }
    )
):
    pass
```

### Subagents

```python
async for message in query(
    prompt="Use the reviewer agent to check code quality",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Glob", "Grep", "Agent"],
        agents={
            "code-reviewer": AgentDefinition(
                description="Code review expert",
                prompt="Review code for bugs, security issues, and best practices",
                tools=["Read", "Glob", "Grep"],
            )
        },
    )
):
    pass
```

### MCP Servers (Programmatic)

```python
async for message in query(
    prompt="Check the latest GitHub issues",
    options=ClaudeAgentOptions(
        mcp_servers={
            "github": {
                "command": "npx",
                "args": ["-y", "@anthropic-ai/mcp-github"]
            }
        }
    ),
):
    pass
```

---

## Production Patterns

### Automated Code Review Bot

```python
async def review_pr(pr_number: int):
    async for message in query(
        prompt=f"""
        Review PR #{pr_number}:
        1. Read the changed files
        2. Check for bugs, security issues, performance problems
        3. Verify tests exist for new code
        4. Output a structured review
        """,
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep", "Bash"],
            model="claude-sonnet-4-6"  # Cost-effective for reviews
        )
    ):
        if isinstance(message, ResultMessage):
            post_review_comment(pr_number, message.result)
```

### Scheduled Monitoring Agent

```python
async def daily_check():
    async for message in query(
        prompt="""
        Run daily health checks:
        1. Check test suite passes
        2. Look for new TODO/FIXME/HACK comments
        3. Check for dependency vulnerabilities (npm audit)
        4. Summarize findings
        """,
        options=ClaudeAgentOptions(
            allowed_tools=["Bash", "Grep", "Glob"]
        )
    ):
        if isinstance(message, ResultMessage):
            send_slack_notification(message.result)
```

### vs Anthropic Client SDK

| Feature | Agent SDK | Client SDK |
|---------|-----------|------------|
| Tool loop | Handled automatically | You implement it |
| Built-in tools | Read, Write, Bash, etc. | None — you build them |
| Context management | Automatic | Manual |
| Sessions | Built-in persistence | You manage state |
| Complexity | Lower (for agentic tasks) | Higher (more control) |
| Use case | Autonomous agents | Custom chat apps |

---

## Resources & Links
_(Add as you go)_
