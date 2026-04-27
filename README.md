# Interview Prep — AI & Data Engineering

**Owner:** Manas
**Last updated:** April 24, 2026
**Approach:** Mentor-led, depth-over-breadth, AWS-primary cloud focus.

---

## What's In This Repo

This is a working notebook for two intersecting career tracks: **AI Engineering** and **Data Engineering**. The two converge in production systems (RAG, feature stores, evals, vector DBs, agent backends), so the prep deliberately interlinks the two.

| Folder | Purpose | Where to start |
|---|---|---|
| [`ai/`](./ai/) | LLMs, RAG, prompting fundamentals | [LLM Fundamentals](./ai/llms/01-llm-fundamentals.md) |
| [`ai-coding-assistants/`](./ai-coding-assistants/) | Claude Code, Cursor, MCP, Agent SDK | [Project Context Files](./ai-coding-assistants/01-project-context-files.md) |
| [`data-engineer/`](./data-engineer/) | DE fundamentals → senior interview topics | [DE Hub](./data-engineer/README.md) |
| [`projects/`](./projects/) | High-leverage portfolio project ideas | [Projects README](./projects/README.md) |

---

## How to Use This Repo

Every topic file follows the same format:

1. **Why This File Exists** — what's at stake in interviews.
2. **BEGINNER / INTERMEDIATE / ADVANCED** — same topic, three depths.
3. **Worked Example** — at least one practical scenario.
4. **Interview-Ready Cheat Sheet** — Q&A and quick trade-off pairs you can drop into a round.
5. **Resources & Links** — papers, official docs, and follow-ups.

Cross-references between files use plain markdown links so you can navigate concepts without losing context.

---

## The Recommended Reading Path

**If you have 1 month before interviews:**
1. [DE: SQL Deep Dive](./data-engineer/01-sql-deep-dive.md) (most-tested topic)
2. [DE: Data Modeling](./data-engineer/02-data-modeling.md)
3. [DE: ETL vs ELT](./data-engineer/04-etl-vs-elt.md)
4. [DE: Spark Fundamentals](./data-engineer/07-spark-fundamentals.md)
5. [DE: System Design](./data-engineer/15-system-design-de.md)
6. [AI: LLM Fundamentals](./ai/llms/01-llm-fundamentals.md)
7. [AI: RAG Fundamentals](./ai/rag/01-rag-fundamentals.md)

**If you have 3 months:** the full DE topic index (1–15) + the AI fundamentals + one [Tier-S project](./projects/README.md#tier-s--highest-leverage-do-at-least-one-of-these).

**If you have 6 months:** add a certification (see the [DE README's certification roadmap](./data-engineer/README.md#certifications)) and ship 2 projects publicly.

---

## The "AI × DE" Crossover

Some of the most marketable jobs in 2026 are at the intersection of these two fields:

| Role | What it needs | Files to focus on |
|---|---|---|
| ML Platform Engineer | Spark + feature stores + serving | DE 7, 8, 12, 13 + AI LLMs |
| RAG Platform Engineer | Embedding pipelines + vector DBs + LLM eval | AI RAG + DE 4, 11, 13 |
| Analytics Engineer + AI | dbt + LLM enrichment of warehouse data | DE 1, 10, 11 + AI Prompting |
| Agent / Workflow Engineer | Agent SDKs + tool orchestration | AI-Coding 03, 05, 07 + AI Prompting |

Build a project that hits two of these (e.g. [S2 — RAG with evals](./projects/README.md#s2-rag-system-over-a-real-corpus-with-evals)) and you become harder to replace.

---

## Maintenance Notes

- Each file ends with a Resources & Links section — keep adding to it as you read papers and posts.
- When a concept is added in one file, add a cross-link from any other file that mentions it.
- Date-stamp updates ("Last updated:" line) when you make material changes.
- Keep the Cheat Sheet sections punchy — they're meant to be skimmed in the 30 minutes before a round.

---

*This repo is yours to shape. The mentor's job is to keep it pointed at high-leverage topics; your job is to do the reps.*
