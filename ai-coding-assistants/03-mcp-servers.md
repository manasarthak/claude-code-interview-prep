# MCP Servers — Connecting AI to External Tools

**Applies to:** Claude Code (native), Cursor (via extensions), VS Code (via extensions)  
**Difficulty:** Beginner → Intermediate → Advanced  
**Last updated:** April 24, 2026
**Related:** [Project Context Files](./01-project-context-files.md) · [Skills, Commands, Plugins](./02-skills-commands-plugins.md) · [Token Optimization](./04-token-optimization-and-context.md) · [Hooks & Security](./05-hooks-security-automation.md) · [Agent SDK](./07-agent-sdk-and-programmatic-use.md) · [Prompting — ReAct](../ai/prompting/01-prompting-fundamentals.md)

---

## BEGINNER — What Is MCP?

### The Concept

MCP (Model Context Protocol) is an open standard that lets AI assistants connect to external services — databases, Slack, GitHub, browsers, file systems, APIs. Think of it as USB for AI: a universal plug that lets any AI tool talk to any service.

Without MCP: "Query my database" → AI can't, it has no access.  
With MCP: "Query my database" → AI calls the Postgres MCP server → gets results → answers your question.

### How It Works

```
You: "What's the latest error in our logs?"
    ↓
Claude Code → calls MCP tool: query_logs("error", limit=5)
    ↓
MCP Server (running locally) → connects to your log service
    ↓
Returns results to Claude → formats answer for you
```

MCP servers run as local processes. They expose "tools" (functions the AI can call), "resources" (data the AI can read), and "prompts" (reusable templates).

### Configuration (Claude Code)

Add MCP servers in `.claude/mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-github"]
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-postgres"],
      "env": {
        "DATABASE_URL": "postgres://user:pass@localhost/mydb"
      }
    }
  }
}
```

Or use the CLI: `claude mcp add github npx @anthropic-ai/mcp-github`

### Configuration Locations

| File | Scope |
|------|-------|
| `~/.claude/mcp.json` | All projects (personal) |
| `.claude/mcp.json` | This project (committed to git) |
| `.claude/mcp.local.json` | This project, gitignored (for secrets) |

Priority: local > project > user. Same server name at multiple levels → highest priority wins.

### Common Useful MCP Servers

**Development:**
- GitHub — Search repos, manage PRs/issues, read code
- GitLab — Similar to GitHub
- Linear / Jira / Asana — Issue tracking
- Slack — Read/send messages

**Data:**
- Postgres — Query databases directly
- MongoDB — Document database queries
- Google Sheets — Read/write spreadsheet data
- Airtable — Table data access

**Web & Automation:**
- Playwright — Browser automation (fill forms, scrape, test)
- Puppeteer — Headless browser control
- Filesystem — Enhanced file operations

---

## INTERMEDIATE — Managing MCP Effectively

### Token Cost Awareness

MCP servers add to your context cost:
- Tool **names and descriptions** load at session start (low cost)
- Full **JSON schemas** load on demand when Claude needs a tool (moderate cost)
- Tool **results** add to conversation (can be large)

Run `/mcp` in Claude Code to see token costs per server. Disconnect servers you're not actively using.

**Key insight:** CLI tools (jq, curl, git) are more context-efficient than MCP servers because they don't load tool schemas. If you can do it with a bash command, that's often cheaper token-wise.

### When to Use MCP vs CLI

| Use MCP When | Use CLI When |
|-------------|-------------|
| Service needs authentication/state | Simple commands work |
| Complex API with many endpoints | One-off data fetching |
| You want structured tool calls | Raw output is fine |
| Multiple team members need same setup | It's just you |

### Environment Variables & Secrets

Never put secrets in committed config files. Use `.claude/mcp.local.json` (gitignored):

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-postgres"],
      "env": {
        "DATABASE_URL": "${POSTGRES_URL}"
      }
    }
  }
}
```

Or reference environment variables that are set in your shell profile.

### Debugging MCP Issues

MCP connections can fail silently. Signs something's wrong:
- Claude says it can't find a tool you know exists
- Tool calls return errors about missing servers

**Fixes:**
- Run `/mcp` to check connection status
- Restart the session (`/clear`)
- Check server logs (usually stderr from the command)
- Verify the npm package is installed and up to date

---

## ADVANCED — Building Custom MCP Servers

### When to Build Your Own

- Your internal service has no existing MCP server
- You want to expose a specific subset of an API
- You need custom authentication flows
- You want to combine multiple data sources into one interface

### Basic Structure (TypeScript)

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";

const server = new Server({
  name: "my-custom-server",
  version: "1.0.0"
}, {
  capabilities: { tools: {} }
});

server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "get_user",
    description: "Look up a user by email",
    inputSchema: {
      type: "object",
      properties: {
        email: { type: "string", description: "User's email" }
      },
      required: ["email"]
    }
  }]
}));

server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "get_user") {
    const { email } = request.params.arguments;
    const user = await fetchUser(email); // Your logic
    return { content: [{ type: "text", text: JSON.stringify(user) }] };
  }
});
```

### Tool Schema Design Tips

- Keep descriptions specific and keyword-rich (the AI uses them to decide when to call)
- Fewer tools with clear purposes > many overlapping tools
- Return structured data (JSON) so the AI can parse and reason over it
- Include examples in descriptions for ambiguous tools

### Reducing Token Load from MCP

If your server exposes many tools, Claude Code's tool search loads only relevant ones on demand. But you can help:

1. Use specific, distinct tool names (not `query1`, `query2`, `query3`)
2. Keep descriptions concise but descriptive
3. Split large servers into focused smaller ones if possible
4. Disconnect servers you're not using in the current session

---

## Resources & Links
_(Add as you go)_
