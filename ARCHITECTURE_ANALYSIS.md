# OpenCode: Architectural Deep Dive — Performance & Agentic Workflow Engineering

> An analysis of the clever engineering patterns that make OpenCode fast, extensible, and capable of orchestrating complex agentic workflows.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [The Agentic Loop — How the Brain Works](#2-the-agentic-loop--how-the-brain-works)
3. [Performance Engineering Catalog](#3-performance-engineering-catalog)
4. [Event-Driven Reactive Architecture](#4-event-driven-reactive-architecture)
5. [Extensibility Patterns](#5-extensibility-patterns)
6. [Key Architectural Decisions & Trade-offs](#6-key-architectural-decisions--trade-offs)

---

## 1. System Overview

OpenCode is a **TypeScript/Bun monorepo** that implements an agentic coding assistant with a CLI (Ink/Solid.js TUI), a web app (Solid.js SPA), and a desktop app (Tauri). The core is a Hono HTTP server that orchestrates LLM calls, tool execution, permission gating, and session persistence.

### Package Dependency Graph

```
┌──────────────┐     ┌────────────┐     ┌───────────┐
│   desktop    │────▶│    app     │────▶│    ui     │
│   (Tauri)    │     │ (Solid.js) │     │(components)│
└──────────────┘     └─────┬──────┘     └───────────┘
                           │
                     ┌─────▼──────┐     ┌───────────┐
                     │   sdk/js   │────▶│   util    │
                     │ (TS client)│     │ (shared)  │
                     └─────┬──────┘     └───────────┘
                           │
                     ┌─────▼──────┐     ┌───────────┐
                     │  opencode  │────▶│  plugin   │
                     │  (core)    │     │ (hooks)   │
                     └────────────┘     └───────────┘
```

### Lifecycle: Process Start → Agent Loop

```
CLI Entry (yargs)
  → Global Middleware (log init, DB migration)
    → Instance.provide({ directory })
      → Project.fromDirectory() + bootstrap
        → Hono Server (HTTP + WebSocket)
          → SessionPrompt.loop() — the agent brain
```

The key insight: **everything is instance-scoped**. Each working directory gets its own isolated context (bus, state, file watchers, MCP clients). This is achieved via `AsyncLocalStorage` — no context threading through function signatures, just `Instance.directory` anywhere in the call stack.

**Reference:** `packages/opencode/src/project/instance.ts:22-114`, `packages/opencode/src/util/context.ts:1-26`

---

## 2. The Agentic Loop — How the Brain Works

### 2.1 The Core Loop (`SessionPrompt.loop()`)

**File:** `packages/opencode/src/session/prompt.ts:274-724`

This is the heartbeat. It's a **step-based iterative loop** — not just "call LLM, return response," but a stateful machine that handles tool dispatch, permission gating, context compaction, retries, and multi-agent delegation:

```
┌─────────────────────────────────────────────────────────────┐
│                   SessionPrompt.loop()                       │
└────────────────────────┬────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   Load History     Resolve Model     Check Exit
   from SQLite      & Agent Config    Conditions
        │                │                │
        └────────────────┼────────────────┘
                         ▼
              ┌─────────────────────┐
              │  Tool Resolution    │
              │  + Permission Gate  │
              │  + Schema Transform │
              │  + MCP Integration  │
              └──────────┬──────────┘
                         ▼
              ┌─────────────────────┐
              │  LLM.stream()       │
              │  (Vercel AI SDK)    │
              └──────────┬──────────┘
                         ▼
              ┌─────────────────────┐
              │  Stream Processing  │
              │  ├─ Text deltas     │
              │  ├─ Reasoning       │
              │  ├─ Tool calls      │
              │  ├─ Snapshots/diffs │
              │  └─ Error/retry     │
              └──────────┬──────────┘
                         ▼
              ┌─────────────────────┐
              │  Loop Decision:     │
              │  continue | stop |  │
              │  compact            │
              └─────────────────────┘
```

**Clever detail — Doom Loop Detection** (`packages/opencode/src/session/processor.ts:152-176`): If the LLM calls the same tool with the same input 3 times in a row, the system detects this as a doom loop and asks for user permission before continuing. This prevents infinite cycles where the model gets stuck.

### 2.2 Streaming-First Response Processing

**File:** `packages/opencode/src/session/processor.ts:45-420`

Responses stream token-by-token via the Vercel AI SDK's `streamText()`. The processor is an **async generator** that yields control on every event:

- **Text deltas** → persisted incrementally via `Session.updatePartDelta()` (only the new characters, not the full text)
- **Tool calls** → dispatched with permission checking, results stored as `PartTable` rows
- **Reasoning tokens** → accumulated and persisted separately
- **Step boundaries** → filesystem snapshots taken for undo/revert capability

**Why this matters:** Incremental delta persistence means the UI can show tokens in real-time without re-reading entire messages from the database. The persistence layer only writes the diff.

**Reference:** `packages/opencode/src/session/processor.ts:287-337`

### 2.3 Context Compaction — Memory Management for Long Sessions

**File:** `packages/opencode/src/session/compaction.ts:32-99`

When the conversation history approaches the model's context window, the system doesn't just truncate — it **intelligently prunes tool call outputs**:

```
tokens_used = input + output + cache_read + cache_write
reserved = min(20_000, maxOutputTokens)
usable = model.limit.input - reserved

if tokens_used >= usable:
  → Prune oldest tool outputs (protect last 2 turns)
  → Minimum 20k tokens pruned per pass
  → Some tools (e.g., "skill") are protected from pruning
  → Mark pruned parts with time.compacted timestamp
```

This is smarter than naive truncation because tool outputs are often the largest tokens consumers but least important for ongoing context (the model already incorporated their information into its reasoning).

**Reference:** `packages/opencode/src/session/compaction.ts:58-99`

### 2.4 Multi-Agent Composition via Subagent Sessions

**File:** `packages/opencode/src/tool/task.ts`

The `task` tool enables the primary agent to spawn child agents:

```
Primary Agent → calls "task" tool →
  1. Creates child session (parentID links to parent)
  2. Runs SessionPrompt.prompt() recursively with:
     - Different agent config (e.g., "explore" for fast read-only)
     - Restricted permissions (no todowrite in subtasks)
     - Different model (if agent has custom model)
  3. Returns result to parent session
  4. Parent continues its loop
```

Built-in agents with different permission profiles:

| Agent | Mode | Capabilities |
|-------|------|-------------|
| `build` | primary | Full access, all tools |
| `plan` | primary | Read-only, edit restricted to `.opencode/plans/` |
| `general` | subagent | Multi-step worker, no todowrite |
| `explore` | subagent | Fast read-only: grep, glob, read only |
| `compaction` | hidden | Summarizes history for context management |

**Reference:** `packages/opencode/src/agent/agent.ts:77-203`

### 2.5 Smart Retry with Server-Guided Backoff

**File:** `packages/opencode/src/session/retry.ts:1-101`

Not just exponential backoff — the retry system **respects server-provided rate limit headers**:

1. Check `Retry-After-Ms` header (millisecond precision)
2. Fall back to `Retry-After` header (seconds or HTTP-date)
3. Fall back to exponential backoff: `2000ms * 2^(attempt-1)`, capped at 30s

Error classification determines retryability:
- Rate limits, overloaded → retryable
- Context overflow → triggers compaction, not retry
- Auth failures → not retryable

**Reference:** `packages/opencode/src/session/retry.ts:28-59`

---

## 3. Performance Engineering Catalog

### 3.1 Lazy Evaluation Everywhere

**Files:** `packages/util/src/lazy.ts`, `packages/opencode/src/util/lazy.ts`

A `lazy()` wrapper defers expensive initialization until first access. The enhanced version adds error handling (don't cache failed initializations) and `reset()` for re-initialization:

```typescript
export function lazy<T>(fn: () => T) {
  let value: T | undefined
  let loaded = false
  const result = (): T => {
    if (loaded) return value as T
    try {
      value = fn()
      loaded = true
      return value as T
    } catch (e) {
      throw e // Don't mark as loaded on failure
    }
  }
  result.reset = () => { loaded = false; value = undefined }
  return result
}
```

**Used for:**
- SQLite database connection (`packages/opencode/src/storage/db.ts:72`)
- WASM bash parser (`packages/opencode/src/bash.ts:33`)
- File watcher native bindings (`packages/opencode/src/file/watcher.ts:35`)
- Tool registry state

### 3.2 SQLite Performance Tuning

**File:** `packages/opencode/src/storage/db.ts:72-98`

The database isn't just "use SQLite" — it's carefully tuned:

```sql
PRAGMA journal_mode = WAL          -- Concurrent reads while writing
PRAGMA synchronous = NORMAL        -- Skip full fsync (safe with WAL)
PRAGMA busy_timeout = 5000         -- 5s retry on lock contention
PRAGMA cache_size = -64000         -- 64MB in-memory cache
PRAGMA foreign_keys = ON
PRAGMA wal_checkpoint(PASSIVE)     -- Non-blocking checkpoints
```

**Why WAL matters here:** The agent loop writes tool results and message parts while the UI reads session state. WAL mode means readers never block writers — critical for real-time streaming UX.

**Transaction pattern with deferred effects:**
```typescript
Database.use((db) => {
  db.insert(MessageTable).run()
  // Side-effects queued, only fire AFTER commit succeeds
  Database.effect(() => Bus.publish(MessageV2.Event.Updated, ...))
})
```

This guarantees events aren't published for uncommitted data.

### 3.3 Multi-Layer Caching Architecture

The caching isn't one-size-fits-all — it's **purpose-built at each layer**:

#### a) LRU Cache with TTL (`packages/app/src/utils/scoped-cache.ts`)
- Max entries + TTL expiration
- Custom dispose callbacks for cleanup
- Automatic sweep on access

#### b) Dual-Constraint File Content LRU (`packages/app/src/context/file/content-cache.ts`)
- **Two eviction triggers**: entry count (40 max) AND byte size (20MB max)
- Pinned entries (currently viewed) protected from eviction
- Tracks byte accounting separately from entry count

#### c) 3-Level Persistence Cache (`packages/app/src/utils/persist.ts`)
- **Level 1:** In-memory Map (500 entries, 8MB)
- **Level 2:** localStorage
- **Level 3:** Intelligent eviction — drops **largest** items first when quota exceeded

#### d) View Session Cache (`packages/app/src/context/file/view-cache.ts`)
- Scoped per-session, max 20 entries, 500 files per session
- LRU eviction within session scope

### 3.4 Batch Tool Execution

**File:** `packages/opencode/src/tool/batch.ts:1-181`

The `batch` tool enables parallel tool execution with a hard cap of **25 concurrent calls**:

```typescript
const results = await Promise.all(
  toolCalls.slice(0, 25).map(call => executeCall(call))
)
```

Each call gets:
- Its own `partID` for independent progress tracking
- Individual timing (`callStartTime` / `Date.now()`)
- Isolated error handling (one failure doesn't abort others)

Overflow calls (>25) are discarded with error messages.

### 3.5 Worker Pool for CPU-Intensive Diff Rendering

**File:** `packages/ui/src/pierre/worker.ts:1-53`

Diff rendering uses **Web Workers** with a pool of 2 (not 8 — tuned for memory vs parallelism, and Safari has slow worker boot):

```typescript
const pool = new WorkerPoolManager({
  workerFactory,
  poolSize: 2,  // Deliberately small: memory > parallelism
}, { theme: "OpenCode", lineDiffType, preferredHighlighter: "shiki-wasm" })
```

**Reference counting for virtualizers** (`packages/ui/src/pierre/virtualizer.ts:66-100`):
- WeakMap-based caching keyed by DOM element
- `acquire()` / `release()` pattern — cleanup only when refcount hits 0
- Prevents duplicate virtualizer instances for the same container

### 3.6 Streaming File I/O with Bounded Buffers

**File:** `packages/opencode/src/tool/read.ts:1-80`

File reads are bounded to prevent memory blowup:
- `DEFAULT_READ_LIMIT`: 2000 lines
- `MAX_LINE_LENGTH`: 2000 characters (truncated with suffix)
- `MAX_BYTES`: 50KB
- Uses `createReadStream` + `createInterface` — never loads full file into memory

### 3.7 Debounced State Persistence

**File:** `packages/app/src/context/layout-scroll.ts:1-119`

Scroll position persistence uses debouncing (200ms) with dirty-set tracking:

```typescript
function setScroll(sessionKey, tab, pos) {
  if (prev?.x === pos.x && prev?.y === pos.y) return // No-op shortcut
  setCache(sessionKey, tab, { x: pos.x, y: pos.y })
  dirty.add(sessionKey)
  schedule(sessionKey) // Reset 200ms debounce timer
}
```

Only dirty sessions are flushed. `flushAll()` on unmount ensures nothing is lost.

### 3.8 Reactive State Batching

**File:** `packages/app/src/context/global-sync/child-store.ts:59-77`

Multiple state updates are batched into single re-renders using Solid.js `batch()`:

```typescript
batch(() => {
  setLayout("left", layoutLeft)
  setLayout("right", layoutRight)
  setLayout("preview", preview)
}) // One re-render, not three
```

**Child store memory management** uses reference counting with eviction:
- `pin(directory)` / `unpin(directory)` track active references
- `runEviction()` cleans up idle stores past TTL
- Protected stores (pinned) survive eviction

### 3.9 Sliding Window Rate Limiting

**File:** `packages/console/app/src/routes/zen/util/rateLimiter.ts:1-83`

Server-side rate limiting uses a **3-hour sliding window** (not fixed buckets):

```typescript
const intervals = [
  buildYYYYMMDDHH(now),           // Current hour
  buildYYYYMMDDHH(now - 3_600_000),  // -1 hour
  buildYYYYMMDDHH(now - 7_200_000),  // -2 hours
]
```

The `getRetryAfterHour()` function calculates precise retry-after by simulating which interval will drop out of the window first — giving clients the exact second they can retry, not a guess.

---

## 4. Event-Driven Reactive Architecture

### 4.1 Type-Safe Event Bus

**Files:** `packages/opencode/src/bus/bus-event.ts`, `packages/opencode/src/bus/index.ts`

Events are defined with Zod schemas and a global registry:

```typescript
BusEvent.define("file.watcher.updated", z.object({
  file: z.string(),
  event: z.enum(["add", "change", "unlink"])
}))
```

The registry auto-generates a discriminated union for type-safe handling. Subscriptions return unsubscribe functions and are scoped to the instance.

### 4.2 Three-Layer Event Propagation

```
Layer 1: Backend Bus
  Bus.publish(event) → instance-scoped subscribers
        │
        ▼
Layer 2: GlobalBus (Node EventEmitter)
  Relays to /global/event SSE endpoint
        │
        ▼
Layer 3: Client Event Stream
  Web App / TUI consumes SSE stream
  └─ Event coalescing (16ms batch window)
  └─ Delta deduplication by (sessionID, messageID, partID)
  └─ Heartbeat timeout (15s) → auto-reconnect (250ms delay)
```

### 4.3 Event Coalescing for UI Performance

**File:** `packages/app/src/context/global-sdk.tsx:51-79`

The web app doesn't process every event immediately — it **coalesces** them:

- De-duplicates events by composite key (directory + session + message + part)
- Batches within a 16ms flush window (one animation frame)
- Marks stale deltas when a full `part.updated` event supersedes them
- Yields every 8ms for browser event loop responsiveness

This prevents UI jank when the LLM is streaming tokens rapidly.

### 4.4 Permission as Async Event Gate

**File:** `packages/opencode/src/permission/next.ts:131-160`

Permission requests create a **blocking async gate** using the event bus:

```
Tool requests permission
  → PermissionNext.ask() publishes "permission.asked"
  → Waits for "permission.replied" event
  → User sees prompt in UI
  → User approves/denies
  → "permission.replied" unblocks the tool
  → Tool execution continues or throws DeniedError
```

This turns an inherently async UI interaction into a synchronous-looking flow within the agent loop.

### 4.5 File Watching → VCS → UI Pipeline

```
Parcel Watcher (inotify/fs-events/windows)
  → Bus.publish("file.watcher.updated", {file, event})
    → VCS subscriber detects .git/HEAD change
      → git rev-parse --abbrev-ref HEAD
        → Bus.publish("vcs.branch.updated", {branch})
          → GlobalBus → SSE → Web App state update
```

The file watcher is **lazy-loaded** with platform-specific native bindings and a subscription timeout guard.

**Reference:** `packages/opencode/src/file/watcher.ts:35-128`

---

## 5. Extensibility Patterns

### 5.1 Tool Registry — Multi-Source Discovery

**File:** `packages/opencode/src/tool/registry.ts:34-173`

Tools come from 4 sources, composed at runtime:
1. **Built-in** — hardcoded in registry.ts
2. **Config directory** — `.opencode/tool[s]/*.{js,ts}` scanned via glob
3. **Plugins** — npm packages or `file://` paths
4. **MCP servers** — Model Context Protocol tool providers

Tools are lazily initialized — `tool.init()` only runs when `tools()` is called. Conditional tools (websearch, codesearch) check provider availability. Model-specific tools (GPT gets `apply_patch`, others get `edit`/`write`).

### 5.2 Provider Strategy Factory

**File:** `packages/opencode/src/provider/provider.ts:87-591`

20+ LLM providers (Anthropic, OpenAI, Azure, Google, Bedrock, etc.) are supported via a **strategy factory** pattern:

```typescript
const BUNDLED_PROVIDERS = {
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  // ...
}
```

Each provider has a `CustomLoader` that handles:
- Auto-detection (should we enable without explicit config?)
- Credential resolution (API keys, OAuth, AWS IAM chains)
- Model-specific options (headers, regions, API versions)

### 5.3 Plugin Hooks — Mutation Pattern

**File:** `packages/plugin/src/index.ts:148-234`

Plugins extend behavior via typed hooks:

```typescript
"chat.params"?: (input, output) => Promise<void>     // Modify LLM params
"tool.execute.before"?: (input, output) => Promise<void>  // Pre-tool hook
"tool.execute.after"?: (input, output) => Promise<void>   // Post-tool hook
"experimental.chat.system.transform"?: ...             // Modify system prompts
```

The mutation pattern (`output` is mutable) is more flexible than return-based middleware — plugins can selectively modify fields without reconstructing entire objects.

### 5.4 7-Layer Configuration Precedence

**File:** `packages/opencode/src/config/config.ts:38-80`

```
1. Remote .well-known/opencode  (org defaults)
2. Global ~/.config/opencode/   (user defaults)
3. OPENCODE_CONFIG env var      (custom path)
4. Project opencode.json{,c}    (project config)
5. .opencode/ directories       (agents, plugins, skills)
6. OPENCODE_CONFIG_CONTENT      (inline content)
7. Managed config               (enterprise override — highest priority)
```

Arrays (plugins, instructions) **concatenate** instead of replacing. Enterprise managed configs always win.

### 5.5 AsyncLocalStorage for Dependency Injection

**File:** `packages/opencode/src/util/context.ts:1-26`

Just 26 lines, but it enables the entire instance-scoped architecture:

```typescript
const context = Context.create<InstanceContext>("instance")

// Set context for an async scope
context.provide(instanceData, async () => {
  // Anywhere in this async tree:
  Instance.directory  // Just works, no parameter passing
})
```

Multiple concurrent instances (different projects) each have their own context. No globals, no parameter threading.

---

## 6. Key Architectural Decisions & Trade-offs

### Decision 1: Serial Tool Execution (not parallel)

Tools execute one-at-a-time within a turn. The `batch` tool exists as an explicit opt-in for parallelism (capped at 25). This is deliberate:
- **Simpler error handling** — no partial failure states
- **Deterministic ordering** — tool results are always in predictable order
- **Permission model simplicity** — one permission prompt at a time

### Decision 2: Database-Backed Everything (SQLite with WAL)

Sessions, messages, parts, permissions — all in SQLite. Not files, not an ORM abstraction. Benefits:
- **ACID guarantees** for the agent loop (no corrupt state on crash)
- **WAL mode** for concurrent read/write (UI reads while agent writes)
- **Transaction + effect pattern** ensures events fire only after commit

### Decision 3: Event Bus over Direct Calls

Components communicate via typed pub/sub events, not direct function calls. This enables:
- **Decoupled UI/backend** — TUI, web app, and desktop all consume the same SSE stream
- **Multi-instance support** — GlobalBus relays across project instances
- **Testability** — events can be intercepted and verified

### Decision 4: Incremental Delta Persistence

Text streaming doesn't rewrite the full message on every token. Instead, `updatePartDelta()` appends only the new characters. Benefits:
- **Less database write pressure** during streaming
- **More efficient event propagation** (smaller payloads)
- **UI can append without re-rendering full message**

### Decision 5: Compaction over Truncation

When context overflows, the system prunes tool outputs (largest, oldest first) rather than truncating conversation history. The LLM's reasoning about tool results is preserved even when the raw output is removed.

---

*Analysis generated by parallel research agents examining 5 architectural dimensions across the codebase.*
