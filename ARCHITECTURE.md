# Architecture: Agent Workflow

## Codebase Overview

Claude Code is an agentic CLI where the user talks to Claude, Claude reasons about what to do, then invokes tools — and those tools can themselves spawn more Claude agents recursively. The whole system is built as a streaming pipeline of async generators in TypeScript, running on Bun, with React+Ink rendering the terminal UI.

---

## The Main Loop: Three Layers

### Layer 1 — `QueryEngine.ts`: Conversation Owner

One `QueryEngine` instance per conversation. It owns:
- `mutableMessages` — full conversation history across turns
- `totalUsage` — cumulative token/cost tracking
- `readFileState` — LRU cache of recently read files

When you send a message, `submitMessage()`:
1. Calls `processUserInput()` to parse slash commands, file attachments, etc.
2. Renders the system prompt (tools list, MCP servers, memory docs)
3. Hands off to the `query()` generator and yields its output

### Layer 2 — `query.ts`: The LLM Tool-Calling State Machine

`query()` is an async generator that runs in a loop until the model stops calling tools:

```
LOOP:
  1. Render system prompt (with compaction if context is large)
  2. Call Claude API with streaming
  3. As assistant streams, parse tool_use blocks
  4. Execute tools (concurrently if read-only, serially if write)
  5. Yield tool results back as a user message
  6. Continue if model hit token limit or needs compaction
  7. Otherwise return terminal state
```

Continuation scenarios: hitting `max_output_tokens`, context too large (triggers auto-compact), or post-sampling hooks requesting continuation.

### Layer 3 — Tool Execution

Every tool_use block goes through this per-tool flow:
1. **Validate input** — Zod schema check
2. **Check permissions** — `canUseTool()` (described below)
3. **Run pre-tool hooks** — from `settings.json`
4. **Execute** — `tool.call()`
5. **Run post-tool hooks**
6. **Return result** as a `ToolResultBlockParam`

Read-only tools (`isConcurrencySafe = true`) run in parallel batches; write tools run serially. Max concurrency is 10.

---

## The Tool System

`Tool.ts` defines the shape every tool must implement:

```typescript
type Tool<Input, Output, Progress> = {
  name: string
  inputSchema: ZodSchema          // validated before execution

  call(args, context, canUseTool, parentMessage, onProgress?)
  checkPermissions(input, context)
  validateInput(input, context)
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
}
```

`tools.ts` assembles the tool pool: built-in tools + MCP server tools, filtered by enabled status and permission mode. Feature-gated tools (`REPL_TOOL`, coordinator tools, etc.) are stripped at build time by Bun's `feature()` DCE.

---

## The Agent Workflow: Sub-Agent Spawning

The `AgentTool` is what makes this a multi-agent system. When Claude calls `AgentTool`, a full nested Claude conversation is spun up.

### Input Schema (what Claude fills in)

```typescript
{
  prompt: string          // The sub-task
  description: string     // 3-5 word label
  subagent_type?: string  // "general-purpose", "Explore", "Plan", etc.
  model?: 'sonnet' | 'opus' | 'haiku'
  run_in_background?: boolean
  isolation?: 'worktree' | 'remote'
  name?: string           // for named multi-agent teams
}
```

### Spawning Flow

```
AgentTool.call()
  │
  ├─ Determine path:
  │    ├─ Fork subagent (no subagent_type, fork gate on)
  │    ├─ Team teammate (team_name + name provided)
  │    └─ Regular subagent (subagent_type specified)
  │
  ├─ Build agent context (isolated from parent):
  │    ├─ New AbortController (kill child without killing parent)
  │    ├─ New file state cache
  │    ├─ setAppState = no-op (async agents can't mutate parent state)
  │    └─ Cloned permission rules from parent
  │
  ├─ Assemble tool pool for the agent
  ├─ Render system prompt (frozen for prompt cache sharing)
  │
  ├─ If run_in_background=true:
  │    └─ Register task, start background process → return task_id
  │
  └─ If sync:
       └─ Call runAgent() generator → yield each message to parent
```

### `runAgent()`: The Recursive Heart

`runAgent()` is just `query()` wrapped in an agent context. It's the same LLM loop, but:
- Messages are recorded to a sidechain transcript (separate from parent)
- Results are yielded up to the parent `query()` loop
- Nested agents can themselves call `AgentTool`, creating arbitrary-depth trees

This means Claude can orchestrate agents that orchestrate agents.

### Context Isolation

Async agents get a no-op `setAppState` — they cannot mutate the parent conversation's state. But `setAppStateForTasks` reaches the root store so background tasks can still register themselves.

---

## Permission System

`canUseTool()` is the security boundary called before every tool execution — including inside sub-agents.

```
canUseTool(toolName, input, context):
  1. Check blanket deny rules
  2. Run permission hooks (from settings.json)
  3. Run tool.validateInput()
  4. Run tool.checkPermissions()
  5. Run auto-classifier (Bash tool security heuristics)
  6. If still undecided + interactive mode → show user prompt
  7. Log decision for telemetry
```

Permission rules live in `ToolPermissionContext`:
- `alwaysAllowRules` / `alwaysDenyRules` / `alwaysAskRules`
- Keyed by source (`settings`, `session`, `policySettings`)
- Can be tool-level (`"Bash"`) or pattern-level (`"Bash(git *)"`)

Modes: `default` (prompt on writes), `auto` (approve everything), `plan` (read-only), `bypassPermissions` (used internally by sub-agents in some modes).

---

## Coordinator Mode

When `COORDINATOR_MODE` is on, a top-level "coordinator" agent runs a specialized system prompt and only has access to `AgentTool`, `SendMessageTool`, and `TaskStopTool`. It cannot directly call Bash or edit files — it only orchestrates worker agents that each get the full tool set. Workers report progress via `<task-notification>` XML messages. The coordinator synthesizes results.

---

## State Management

`AppState.tsx` holds session state: permission context, MCP connections, file history, etc. It's passed through the query loop as `getAppState()`/`setAppState()` callbacks. Async/background agents receive a no-op `setAppState` so they can't corrupt the parent session.

---

## Data Flow

```
User types message
        │
        ▼
QueryEngine.submitMessage()
        │
        ▼
query() loop (async generator)
        │
        ├──► Claude API (streaming)
        │         │
        │    tool_use blocks
        │         │
        │         ▼
        │   StreamingToolExecutor
        │    ├── read-only tools → parallel
        │    └── write tools → serial
        │         │
        │    for each tool:
        │    1. validate
        │    2. canUseTool() ──► permission prompt/deny
        │    3. tool.call()
        │         │
        │    If AgentTool:
        │    └──► runAgent()
        │              └──► query() (recursive)
        │                        └──► [same tree, new context]
        │         │
        │    tool results → back to API as user message
        │         │
        └── continue? ──► next iteration of loop
                │
                ▼
        Terminal state → yield final messages to UI
```

---

## Key Design Insights

- **Everything is a stream.** `query()`, `runAgent()`, tool execution — all async generators. This is why the UI streams responses in real time and why cancellation works cleanly (`.return()` on the generator).
- **Tools are the abstraction boundary.** The LLM never directly runs code — it fills in a tool input, and the runtime decides whether to execute it (after permission checks).
- **Sub-agents are recursive `query()` calls.** There's no special "agent runtime" — it's the same loop with an isolated context. This keeps the system simple while supporting arbitrary nesting.
- **Concurrency is opt-in and safe.** Only tools that declare `isConcurrencySafe = true` run in parallel, preventing accidental file races.
- **Prompt caching is baked in.** System prompts are frozen at agent-spawn time and shared across fork subagents so the Anthropic API can cache them.
