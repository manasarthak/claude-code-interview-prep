# Token Optimization & Context Window Management

**Applies to:** All AI coding assistants (Claude Code, Cursor, ChatGPT, Copilot)  
**Difficulty:** Beginner → Intermediate → Advanced  
**Last updated:** April 12, 2026

---

## BEGINNER — Why Tokens and Context Matter

### What Is the Context Window?

The context window is the total amount of text the AI can "see" at once — your instructions, conversation history, file contents, tool outputs, everything. It's measured in tokens (roughly 1 token ≈ 0.75 words).

Current context sizes (2026): Claude Opus/Sonnet: 1M tokens, GPT-4: 128K, Gemini: 1M+.

Even with massive windows, more context ≠ better results. Models perform best when context is relevant and focused. Stuffing in everything degrades quality and costs more.

### What Eats Your Tokens

In a typical Claude Code session, here's where tokens go:

| Component | Loaded When | Approx Cost |
|-----------|------------|-------------|
| System prompt | Every request | ~4,200 tokens (fixed) |
| CLAUDE.md | Every request | Varies (keep under 200 lines) |
| Auto-memory | Every request | Up to ~25KB |
| Environment info (CWD, git status) | Every request | ~280 tokens |
| Skill descriptions | Session start | ~1–2KB total |
| Conversation history | Accumulates | Grows every turn |
| File reads | On demand | Full file content |
| Tool outputs | On demand | Can be very large |
| Skill content (when invoked) | On demand | Full SKILL.md |
| MCP tool schemas | On demand | Per-tool JSON schema |

**The key insight:** Things in the "every request" category compound fast. A 500-line CLAUDE.md costs thousands of tokens on every single interaction.

### The Simplest Optimizations

1. **Keep CLAUDE.md short** — Under 200 lines. Move reference material to Skills.
2. **Use `/compact` when conversations get long** — Summarizes history, frees ~20–50% context.
3. **Start new sessions for new tasks** — Don't let one conversation grow forever.
4. **Be specific in prompts** — "Fix the auth bug in src/auth/login.ts" is better than "Fix the bug" (avoids unnecessary file reads).

---

## INTERMEDIATE — Strategic Token Management

### The Prompt Quality Paradox

Research from Cursor's usage data shows that detailed first prompts (task + requirements + context + expected result) increase AI accuracy by ~50% while *decreasing* total token consumption by ~30%.

Why? A vague prompt causes the AI to explore, read wrong files, try wrong approaches, and backtrack — all costing tokens. A precise prompt gets it right the first time.

**Expensive prompt:** "Make the login work better"  
**Cheap prompt:** "In src/auth/login.ts, the JWT token refresh logic on line 47 doesn't handle expired refresh tokens. Add a try-catch that redirects to /login on 401."

### File Reference vs File Content

Every file the AI reads costs tokens equal to the file's full content. Minimize unnecessary reads:

- **Reference by path** when you can: "The config is in `src/config/db.ts`" — the AI reads only when needed
- **Use @ mentions** (Cursor) instead of pasting code — more token-efficient
- **Avoid broad searches** — "Find all bugs" makes the AI read everything. "Check the auth module" is focused.

### Model Routing (Cursor/Multi-Model Setups)

Different tasks need different models. Using the right model per task saves money without losing quality:

| Task | Model Tier | Why |
|------|-----------|-----|
| Autocomplete, simple edits | Fast/cheap (Haiku, Flash, GPT-3.5) | Speed matters more than depth |
| Standard feature development | Mid-tier (Sonnet, GPT-4o) | Good balance |
| Architecture, complex debugging | Top-tier (Opus, GPT-4) | Need deep reasoning |

Cursor lets you switch models per request. Claude Code supports model override per skill via `model:` frontmatter.

### Conversation Lifecycle Management

```
Session Start → [Small context, fast responses]
    ↓ (after 20-30 turns)
Context growing → [Slightly slower, still good]
    ↓ (after 50+ turns)
Context pressure → [AI may miss details, slower]
    ↓
/compact → [Context summarized, CLAUDE.md re-injected]
    ↓ (after 2-3 compactions)
Diminishing returns → [Start a new session]
```

**When to start fresh vs compact:**
- Switching to a completely different task? → New session
- Still working on the same thing but context is long? → `/compact`
- Getting weird or inconsistent answers? → New session (context may be confused)

### MCP Token Cost Management

MCP servers have hidden costs:
- Tool descriptions load schemas into context
- Rich tool outputs (database query results, API responses) can be very large
- Each connected server adds baseline cost

**Mitigations:**
- Disconnect servers you're not using (`/mcp` to manage)
- Prefer CLI tools (jq, curl, git) for simple operations — no schema overhead
- Check per-server costs with `/mcp`

---

## ADVANCED — Power-User Token Optimization

### Progressive Disclosure Architecture

The AGENTS.md optimization pattern (applicable to all context files): structure your instructions so core rules are tiny and details load on demand.

**Before (expensive):** One 400-line CLAUDE.md with everything.

**After (efficient):**
```
CLAUDE.md (50 lines)          ← Always loaded
.claude/rules/api.md          ← Loaded only when touching API files
.claude/rules/testing.md      ← Loaded only when touching tests
.claude/skills/deploy/SKILL.md ← Loaded only when deploying
```

This pattern reduces initial token load by up to 70%.

### Semantic Caching Patterns

If you frequently ask similar questions, the AI re-computes each time. Some tools implement caching:

- Claude Code: Auto-memory saves learnings across sessions, reducing re-explanation
- Cursor: Caches file indexes for faster reference
- Custom: Build a local knowledge base that your CLAUDE.md references

### Monitoring Your Usage

**Claude Code:** Use `/mcp` for per-server costs. Watch for compaction messages — frequent compaction means you're burning through context.

**Cursor:** Check usage dashboard in settings. Monitor "fast request" vs "slow request" consumption.

**General rule of thumb:**
- If responses are getting slow → context is large
- If the AI is "forgetting" earlier instructions → context was compacted or lost
- If you're hitting rate limits → optimize prompts and reduce unnecessary tool calls

### The 60-70% Rule

Research shows poor context management accounts for 60–70% of total AI spend. The biggest offenders:

1. **Re-reading the same files** — Reference by name instead of re-reading
2. **Ignoring context growth** — Long conversations are expensive
3. **Broad tool calls** — "Search the entire codebase for X" vs "Check src/auth/ for X"
4. **Unused MCP servers** — Each adds schema overhead
5. **Oversized context files** — CLAUDE.md that's a novel

### Cost Estimation Mental Model

Rough token math for a Claude Code session:
- Base cost per turn: ~5K tokens (system + CLAUDE.md + env)
- Average user message: ~200 tokens
- Average Claude response: ~500–2K tokens
- File read: depends on file size (1KB file ≈ 400 tokens)
- Tool call overhead: ~200–500 tokens per call

A 30-turn session with moderate file reads ≈ 200K–400K tokens total. At API rates, this could be $1–5 depending on the model. With a subscription, it's included — but context quality still matters for response quality.

---

## Resources & Links
_(Add as you go)_
