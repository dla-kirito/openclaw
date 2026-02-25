---
title: "Memory and retrieval architecture"
summary: "Reference design for how OpenClaw writes, indexes, and recalls user memory."
read_when:
  - You are building memory and retrieval for an agent system
  - You want a practical pattern to preserve user preferences across sessions
  - You need a clear boundary between persisted memory and model weights
---

# Memory and retrieval architecture

This page documents the memory system used in OpenClaw and extracts reusable
patterns for other projects.

Related docs:

- [Memory](/concepts/memory)
- [Agent workspace](/concepts/agent-workspace)
- [Session management](/concepts/session)
- [Session management deep dive](/reference/session-management-compaction)

## Design goals

OpenClaw memory design optimizes for:

- Durable memory in plain files as the source of truth
- Low coupling between chat runtime and memory backend
- Retrieval before answer for preference and history questions
- Safe reads with bounded snippets instead of large file dumps
- Predictable behavior across compaction and session resets

## What counts as memory

OpenClaw separates memory into three layers:

1. Canonical memory files in workspace:
   - `MEMORY.md` for curated long-term memory
   - `memory/YYYY-MM-DD.md` for daily logs
2. Session transcripts:
   - `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
3. Search index:
   - Builtin SQLite index (or optional QMD backend)
   - Treat index as derived cache, not source of truth

Implication for other projects:

- Keep human-readable canonical storage
- Let index rebuild from canonical data

## Write path

OpenClaw writes user preferences and durable facts through multiple mechanisms.

### Explicit writes by the agent

- The assistant writes durable notes to workspace memory files.
- Memory retrieval tools are optimized around `MEMORY.md` and `memory/*.md`.

### Bootstrap write path

On first run, workspace bootstrap creates identity files and captures initial
preferences:

- Seeds `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`
- Creates `BOOTSTRAP.md` for first-run ritual
- Loads bootstrap files into run context

### Pre-compaction memory flush

Before compaction threshold is hit, OpenClaw can trigger a silent flush turn to
write durable memory:

- Config: `agents.defaults.compaction.memoryFlush`
- Trigger threshold:
  `totalTokens >= contextWindow - reserveTokensFloor - softThresholdTokens`
- Usually returns `NO_REPLY` to stay invisible to user
- Tracks per-session flush metadata:
  `memoryFlushAt` and `memoryFlushCompactionCount`

This pattern is useful when your system does context compaction and you want to
avoid losing near-term decisions.

### Optional plugin auto capture

`memory-lancedb` shows an extension pattern:

- Hook `agent_end` to detect capturable preference/fact text
- Embed and store long-term records with duplicate checks
- Keep this behavior pluginized and optional

## Index and sync path

Builtin retrieval backend uses `MemoryIndexManager` with these characteristics:

- Provider abstraction for embeddings (`openai`/`gemini`/`voyage`/`local`/`auto`)
- Configurable sources (`memory`, optional `sessions`)
- File watching for `MEMORY.md`, `memory.md`, `memory/`, and configured extra paths
- Optional transcript delta indexing for session memory source
- Debounced or interval sync, async safe
- Hybrid ranking (vector plus FTS) when enabled

Indexing is incremental:

- Hash file content
- Reindex only changed files/chunks
- Remove stale rows for deleted files

This gives faster steady-state sync than full rebuild per run.

## Retrieval path

Retrieval has three cooperating parts.

### 1. System prompt contract

When memory tools are available, prompt includes a mandatory recall instruction:

- Search memory before answering prior-work, decisions, dates, people,
  preferences, and todo questions
- Use targeted `memory_get` after `memory_search`

This is an architectural decision: make recall behavior part of default policy,
not only model intuition.

### 2. Tool contract

Core memory slot (`memory-core`) registers:

- `memory_search(query, maxResults?, minScore?)`
- `memory_get(path, from?, lines?)`

`memory_search` returns bounded snippets and location metadata.
`memory_get` reads only required lines to control context size.

### 3. Citation policy

Citations can be `on`, `off`, or `auto`:

- `auto`: citations in direct chat, suppressed in group/channel by default
- `off`: snippet path metadata not shown to user text

This reduces accidental leakage in shared contexts.

## Safety boundaries

Several guardrails are explicit and reusable.

### File read boundary for memory_get

`memory_get` restricts reads to:

- Workspace memory paths (`MEMORY.md`, `memory/*.md`)
- Optional configured extra paths
- Markdown files only
- No symlink traversal

### Session scope boundary

Sub-agent sessions do not inherit full bootstrap context:

- Allowed bootstrap files for sub-agent: `AGENTS.md`, `TOOLS.md`
- Excludes identity and memory files by default

### Plugin and tool gating boundary

- Memory tools exist only when memory slot plugin is active
- Tool policy can allow/deny `group:memory` by profile
- QMD backend adds optional scope rules (default direct-chat allow)

## How OpenClaw learns user preferences

OpenClaw does not fine-tune model weights from your chats.
Preference learning is stateful retrieval over persisted data:

1. Preferences are captured into files or memory plugins.
2. Memory files are indexed into searchable chunks.
3. On relevant questions, assistant is instructed to retrieve first.
4. It answers using recalled snippets instead of hidden model-state memory.
5. Pre-compaction flush reduces preference loss during long sessions.

For other projects, this pattern is easier to audit, backup, and migrate than
implicit model adaptation.

## Extension model and backend fallback

Memory uses an exclusive plugin slot:

- Default slot: `memory-core`
- Alternative slot plugin example: `memory-lancedb`

Backend fallback pattern:

- If QMD backend fails, fallback wrapper switches to builtin index
- Keep service available even when optional backend degrades

This is a production-friendly resilience pattern for memory systems.

## Reusable blueprint for other projects

You can implement the same mechanism with these components:

1. Canonical memory store:
   - Human-readable files or append-only records
2. Index manager:
   - Chunking, embedding, hybrid search, incremental sync
3. Retrieval tools:
   - `search` for candidates, `get` for narrow line reads
4. Prompt policy:
   - Enforce retrieval before answering memory-like queries
5. Compaction bridge:
   - Silent flush before summarization or truncation
6. Safety envelope:
   - Path allowlist, context scope limits, citation policy
7. Optional plugin plane:
   - Hook lifecycle for auto-capture and auto-recall

## Operational checks

For reliability, track at least:

- Index dirty state and sync progress
- Backend/provider fallback status
- Session-level memory flush timestamps
- Read error rates and path-denied events

## Source map

Key implementation references in OpenClaw:

- `src/agents/workspace.ts`
- `src/agents/bootstrap-files.ts`
- `src/agents/pi-embedded-helpers/bootstrap.ts`
- `src/agents/pi-embedded-runner/run/attempt.ts`
- `src/agents/system-prompt.ts`
- `src/agents/tools/memory-tool.ts`
- `src/memory/manager.ts`
- `src/memory/search-manager.ts`
- `src/memory/backend-config.ts`
- `src/auto-reply/reply/memory-flush.ts`
- `src/auto-reply/reply/agent-runner-memory.ts`
- `src/config/sessions/paths.ts`
- `src/config/sessions/types.ts`
- `src/plugins/slots.ts`
- `extensions/memory-core/index.ts`
- `extensions/memory-lancedb/index.ts`

