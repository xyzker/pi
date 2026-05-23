# packages/agent Architecture

Stateful agent runtime with tool execution, event streaming, session persistence, and
context compaction. Built on `@earendil-works/pi-ai`.

## Core Concepts

| Concept | Description |
|---------|-------------|
| **AgentMessage** | Union of LLM `Message` + custom message types (`bashExecution`, `custom`, `branchSummary`, `compactionSummary`). Extended via declaration merging. |
| **AgentTool** | Extended `Tool` with `execute`, `label`, optional `prepareArguments`, `executionMode`, and `onUpdate` callback for streaming partial results. |
| **AgentContext** | Runtime snapshot passed to the agent loop: system prompt + messages + tools. |
| **AgentLoopConfig** | Per-run configuration: model, reasoning level, tool hooks (`beforeToolCall`, `afterToolCall`), message queues (`getSteeringMessages`, `getFollowUpMessages`), `convertToLlm`, `transformContext`, `shouldStopAfterTurn`, `prepareNextTurn`. |
| **AgentEvent** | Lifecycle events emitted during a run: `agent_start`/`agent_end`, `turn_start`/`turn_end`, `message_start`/`message_update`/`message_end`, `tool_execution_start`/`tool_execution_update`/`tool_execution_end`. |
| **AgentHarness** | Higher-level wrapper that adds session persistence, hook system, compaction, and tree navigation on top of the Agent loop. |
| **Session** | Append-only tree of entries persisted via `SessionStorage`. Navigation tracks a leaf pointer; compaction and branch summaries create entries without mutating history. Entries include messages, compaction summaries, branch summaries, model/thinking changes, custom data, and labels. |
| **ExecutionEnv** | Filesystem + shell abstraction (`FileSystem` & `Shell` interfaces) for portable tool execution and skill loading. |

---

## Layer Architecture

```
┌─────────────────────────────────────────────┐
│               AgentHarness                   │
│  session persistence, hooks, compaction,     │
│  tree navigation, skill/template loading     │
├─────────────────────────────────────────────┤
│                 Agent                        │
│  stateful wrapper, lifecycle events,         │
│  steer/followUp queues, tool hooks           │
├─────────────────────────────────────────────┤
│              Agent Loop (low-level)          │
│  agentLoop(), agentLoopContinue()            │
│  streaming async generators                  │
├─────────────────────────────────────────────┤
│           Stream Functions                   │
│  streamSimple (direct) / streamProxy         │
│  (via packages/ai)                           │
└─────────────────────────────────────────────┘
```

---

## Key Files

### Core Agent (`src/`)

#### `src/types.ts`
The canonical data model for the core agent runtime.

- **`StreamFn`**: Type alias for the stream function — `streamSimple`, `streamProxy`, or a custom implementation.
- **`AgentTool<TParameters, TDetails>`**: Tool definition with `label`, `execute`, `prepareArguments`, `executionMode`, and `onUpdate` streaming callback.
- **`AgentToolResult<T>`**: Result with `content` (text/image blocks), `details` (structured data), and optional `terminate` hint.
- **`AgentMessage`**: Union of standard `Message` + custom types via `CustomAgentMessages` declaration merging.
- **`AgentContext`**: System prompt, messages, and tools passed into the loop.
- **`AgentLoopConfig`**: Full per-run configuration — model, reasoning, tool hooks, message queues, `convertToLlm`, `transformContext`, `shouldStopAfterTurn`, `prepareNextTurn`, `getApiKey`.
- **`AgentEvent`**: 11 event types covering agent lifecycle, turn lifecycle, message streaming, and tool execution.
- **Hook contexts**: `BeforeToolCallContext`, `AfterToolCallContext`, `ShouldStopAfterTurnContext`, `PrepareNextTurnContext`.

#### `src/agent.ts`
The `Agent` class — the primary public API for non-harness use.

- **State management**: owns `systemPrompt`, `model`, `thinkingLevel`, `tools[]`, `messages[]`, and runtime state (`isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage`).
- **Lifecycle**: `prompt(text | message)` starts a new turn; `continue()` resumes from existing context (retry).
- **Message queues**: `steer()` injects messages mid-run after the current turn; `followUp()` queues messages for after the agent would stop. Drain modes: `"one-at-a-time"` or `"all"`.
- **Event subscription**: `subscribe(listener)` emits all `AgentEvent`s; listeners are awaited in order and count toward run settlement.
- **`processEvents()`**: Reduces loop events into agent state (pushes messages, tracks pending tool calls, sets error messages), then notifies subscribers.
- **`runWithLifecycle()`**: Wraps the loop with abort-controller management, `isStreaming` flags, and failure handling (emits synthetic error events on crash).
- **Default fallbacks**: `defaultConvertToLlm` filters to standard LLM roles; `DEFAULT_MODEL` placeholder for unconfigured agent.

#### `src/agent-loop.ts`
The low-level agent loop — shared by `Agent` and `AgentHarness`.

- **`runAgentLoop(prompts, context, config, emit, signal)`**: Starts a new prompt run. Emits `agent_start`, `turn_start`, prompt messages, then enters the main loop. Returns `newMessages[]`.
- **`runAgentLoopContinue(context, config, emit, signal)`**: Continues from existing context (retry). Validates last message is not `assistant`. Emits `agent_start` and `turn_start`, then enters the loop.
- **`agentLoop()` / `agentLoopContinue()`**: Public `EventStream`-returning variants for users who want async iteration.
- **`runLoop()`**: Nested loop:
  - **Outer loop**: drains follow-up messages after the agent would stop.
  - **Inner loop**: processes tool calls and steering messages. Each iteration:
    1. Drains pending steering/follow-up messages
    2. Calls `streamAssistantResponse()` to get an LLM response
    3. Executes tool calls (sequential or parallel)
    4. Emits `turn_end`
    5. Calls `prepareNextTurn()` (allows hook to swap context/model)
    6. Calls `shouldStopAfterTurn()` (opportunity for early exit)
    7. Polls `getSteeringMessages()` for injectable messages
- **`streamAssistantResponse()`**: The LLM call boundary. Applies `transformContext` (AgentMessage[] → AgentMessage[]), then `convertToLlm` (AgentMessage[] → Message[]). Builds `Context`, resolves API key, streams deltas, returns `AssistantMessage`.
- **`executeToolCalls()`**: Dispatches to `executeToolCallsSequential` or `executeToolCallsParallel` based on global `toolExecution` config and per-tool `executionMode` overrides. Prepares each tool (find, validate, `beforeToolCall` hook), executes, finalizes (`afterToolCall` hook), emits events.

#### `src/proxy.ts`
Stream function for browser apps that route LLM calls through a backend server.

- **`streamProxy(model, context, options)`**: POSTs to `{proxyUrl}/api/stream`, reads SSE stream, reconstructs `AssistantMessageEvent` deltas from bandwidth-optimized `ProxyAssistantMessageEvent` (no `partial` field), emits via `ProxyMessageEventStream`.
- Handles abort signals, auth tokens, and error responses from the proxy.

#### `src/index.ts`
Barrel export for the core Agent, agent-loop, proxy, types, and all harness modules. Also used as the Node.js entry point when re-exported from `src/node.ts`.

---

### Harness (`src/harness/`)

#### `src/harness/types.ts`
Type definitions for the harness layer — the largest type file in the package.

- **Error types**: Typed errors for every subsystem — `SessionError`, `CompactionError`, `BranchSummaryError`, `ExecutionError`, `FileError`, `AgentHarnessError` — each with stable error codes.
- **Result type**: `Result<T, E>` discriminated union (`ok` / `err`) used pervasively instead of thrown exceptions for expected failures.
- **`Skill`**: Name, description, content, filePath, `disableModelInvocation`.
- **`PromptTemplate`**: Name, description, content (with `$1`, `$@` argument placeholders).
- **`FileSystem`** interface: 18 methods (read/write/append, list/stat, exists, mkdir/rm/tmp, etc.) all returning `Result<T, FileError>`.
- **`Shell`** interface: `exec(command, options)` returning `Result<{stdout, stderr, exitCode}, ExecutionError>`.
- **`ExecutionEnv`**: Union of `FileSystem & Shell`.
- **`SessionStorage<TMetadata>`**: Storage abstraction — `getEntries`, `getPathToRoot`, `appendEntry`, `getLeafId`, `setLeafId`, etc.
- **`SessionTreeEntry`** union: `MessageEntry`, `CompactionEntry`, `BranchSummaryEntry`, `ModelChangeEntry`, `ThinkingLevelChangeEntry`, `CustomEntry`, `CustomMessageEntry`, `LabelEntry`, `SessionInfoEntry`, `LeafEntry`.
- **`AgentHarnessOwnEvent`**: Harness-specific events — `queue_update`, `save_point`, `abort`, `settled`, `before_agent_start`, `context`, `before_provider_request`, `before_provider_payload`, `after_provider_response`, `tool_call`, `tool_result`, `session_before_compact`, `session_compact`, `session_before_tree`, `session_tree`, `model_select`, `thinking_level_select`, `resources_update`.
- **Hook contracts**: Each hook has typed event + result types (e.g., `before_agent_start` → `BeforeAgentStartResult`).

#### `src/harness/agent-harness.ts`
`AgentHarness` — the full-featured agent runtime used by the coding-agent package.

- **`prompt(text)`**: Full turn lifecycle with `before_agent_start` hook, `AgentLoopConfig` wiring, session persistence, and `agent_end` settlement.
- **`skill(name)`**: Invokes a loaded skill by formatting `<skill>` XML blocks.
- **`promptFromTemplate(name, args)`**: Invokes a prompt template with argument substitution.
- **`steer(text)` / `followUp(text)` / `nextTurn(text)`**: Queue messages for the current or subsequent run.
- **`compact()`**: Runs context compaction via `prepareCompaction` + `compact` (or hook-provided summary), persists compaction entry to session.
- **`navigateTree(targetId)`**: Switches the session leaf to a different tree node, optionally generating a branch summary.
- **`abort()`**: Aborts the current run, clears queues, waits for idle.
- **`subscribe(listener)`**: Catches all events (AgentEvent + AgentHarnessOwnEvent).
- **`on(type, handler)`**: Registers hooks for specific harness event types. Handlers can return result patches.
- **`setModel(model)` / `setThinkingLevel(level)` / `setActiveTools(names)` / `setResources(resources)`**: Mutate state with session persistence and event emission.
- **Phase management**: Tracks `"idle"`, `"turn"`, `"compaction"`, `"branch_summary"`, `"retry"` to prevent concurrent operations.
- **`createStreamFn()`**: Wraps `streamSimple` with auth resolution, `before_provider_request`/`before_provider_payload`/`after_provider_response` hooks.
- **`createLoopConfig()`**: Wires harness hooks (`context`, `tool_call`, `tool_result`) into `AgentLoopConfig`.
- **`flushPendingSessionWrites()`**: Deferred session writes (model changes, thinking level changes, custom entries) flushed at save points.

#### `src/harness/session/session.ts`
`Session` class — tree-structured transcript manager.

- **`buildContext()`**: Walks the session tree from root to leaf, handles compaction entries (splits at `firstKeptEntryId`, injects summary as synthetic message), builds final `AgentMessage[]` with model/thinking state.
- **`moveTo(entryId, summary?)`**: Changes the leaf, optionally appending a branch summary entry.
- Entry append methods: `appendMessage`, `appendCompaction`, `appendModelChange`, `appendThinkingLevelChange`, `appendCustomEntry`, `appendCustomMessageEntry`, `appendLabel`, `appendSessionName`.

#### `src/harness/session/jsonl-repo.ts` + `jsonl-storage.ts`
Filesystem-backed session persistence using JSONL.

- **`JsonlSessionRepo`**: Creates sessions as timestamped `.jsonl` files under `{sessionsRoot}/{encodedCwd}/`.
- **`JsonlSessionStorage`**: Entry-level append/read/write with JSONL parsing, leaf-id tracking, header metadata (cwd, sessionId, parentSessionPath).
- Supports `fork()` for branching sessions.

#### `src/harness/session/memory-repo.ts` + `memory-storage.ts`
In-memory session storage for testing.

- **`InMemorySessionRepo`**: Stores sessions in a `Map<string, Session>`.
- **`InMemorySessionStorage`**: Array-backed entry store for fast test setup.

#### `src/harness/compaction/compaction.ts`
Context window management via LLM summarization.

- **`prepareCompaction(entries, settings)`**: Analyzes session tree, finds the compaction cut point, identifies messages to summarize, handles split turns (partial turn prefix kept as context).
- **`findCutPoint(entries, start, end, keepRecentTokens)`**: Walks backwards from the end, accumulating estimated tokens, finds the nearest valid cut point at a user-message or turn boundary.
- **`compact(preparation, model, apiKey)`**: Calls `generateSummary` for history + optionally `generateTurnPrefixSummary` for split turns. Appends `<read-files>` and `<modified-files>` tags to summary.
- **`estimateTokens(message)`**: Conservative character-based heuristic (chars / 4) with per-role handling.
- **`estimateContextTokens(messages)`**: Uses real usage data from the last assistant message when available, falling back to `estimateTokens`.
- **`shouldCompact(contextTokens, contextWindow, settings)`**: Threshold check: compact when tokens > contextWindow - reserveTokens.
- **Summary format**: Structured Markdown with Goal, Constraints, Progress (Done/In Progress/Blocked), Key Decisions, Next Steps, Critical Context sections.

#### `src/harness/compaction/branch-summarization.ts`
Branch summary generation for tree navigation.

- **`prepareBranchEntries(session, oldLeafId, targetId)`**: Walks both paths to find the common ancestor, collects entries between.
- **`generateBranchSummary(entries, options)`**: Summarizes divergent branch entries using structured format.

#### `src/harness/messages.ts`
AgentMessage type extensions and the harness `convertToLlm` function.

- Custom message types: `bashExecution`, `custom`, `branchSummary`, `compactionSummary` — registered via `CustomAgentMessages` declaration merging.
- **`convertToLlm(messages)`**: Converts AgentMessage[] to Message[]. Transforms `bashExecution` → user message (formatted output), `custom` → user, `branchSummary` → user (with prefix/suffix), `compactionSummary` → user (with prefix/suffix). Filters out `excludeFromContext` bash executions.
- Utility functions: `bashExecutionToText`, `createBranchSummaryMessage`, `createCompactionSummaryMessage`, `createCustomMessage`.

#### `src/harness/skills.ts`
Skill loading from filesystem directories.

- **`loadSkills(env, dirs)`**: Recursively scans directories for `SKILL.md` files and root `.md` files. Honors `.gitignore`/`.ignore`/`.fdignore` files. Parses YAML frontmatter (`name`, `description`, `disable-model-invocation`). Validates skill names (lowercase alphanumeric + hyphens).
- **`loadSourcedSkills(env, inputs)`**: Multi-source loading with provenance-tagged skills and diagnostics.
- **`formatSkillInvocation(skill, additionalInstructions?)`**: Generates `<skill>` XML blocks with relative path resolution.

#### `src/harness/prompt-templates.ts`
Prompt template loading from filesystem.

- **`loadPromptTemplates(env, paths)`**: Loads `.md` files from directories or explicit paths. Parses YAML frontmatter for `description`.
- **`substituteArgs(content, args)`**: Shell-style argument substitution — `$1`, `$@`, `$ARGUMENTS`, `${@:N}`, `${@:N:L}`.
- **`formatPromptTemplateInvocation(template, args)`**: Full formatting pipeline.

#### `src/harness/utils/truncate.ts`
Dual-limit content truncation (lines + bytes).

- **`truncateHead(content, options)`**: Keep first N lines/bytes. For file reads.
- **`truncateTail(content, options)`**: Keep last N lines/bytes. For shell output.
- Format `utf8ByteLength`, `formatSize(bytes)`, `truncateLine(line, maxChars)`.

#### `src/harness/utils/shell-output.ts`
Shell execution with output capture.

- **`executeShellWithCapture(env, command)`**: Runs a command via `ExecutionEnv.exec`, captures stdout/stderr chunks, sanitizes binary output, truncates to ~100KB in memory, spills to temp file for full output.
- **`sanitizeBinaryOutput(str)`**: Strips control characters (except tab, newline, CR), Unicode interlinear annotations.

#### `src/harness/env/nodejs.ts`
Node.js implementation of `ExecutionEnv`.

- **`NodeExecutionEnv`**: Implements all `FileSystem` methods (using `node:fs/promises`) and `Shell.exec` (using `node:child_process.spawn`).
- **Shell discovery**: Locates bash on Windows (Git Bash under `ProgramFiles` or PATH) and Unix (`/bin/bash` or PATH fallback to `sh`).
- **Process tree killing**: `SIGKILL` with process group on Unix; `taskkill /T` on Windows.

---

## Data Flow

### Simple prompt (Agent class)

```
agent.prompt("Hello")
  → normalizePromptInput() → AgentMessage[]
  → runWithLifecycle()
    → runAgentLoop(messages, context, config, emit, signal)
      → emit: agent_start, turn_start, message_start/end (user)
      → runLoop()
        → streamAssistantResponse()
          → transformContext?()       // AgentMessage[] → AgentMessage[]
          → convertToLlm()            // AgentMessage[] → Message[]
          → streamFn(model, context)  // LLM call
          → iterate deltas → emit message_start/update/end
        → emit turn_end
        → emit agent_end
```

### With tool calls

```
runLoop()
  → streamAssistantResponse() → assistant message with toolCalls
  → executeToolCalls()
    → for each toolCall:
        prepareToolCall()          // find, validate, beforeToolCall hook
        executePreparedToolCall()  // tool.execute() + updates
        finalizeExecutedToolCall() // afterToolCall hook
        emit tool_execution_start/update/end
        emit message_start/end (toolResult)
  → emit turn_end
  → prepareNextTurn()  // hook: swap context/model/thinking
  → shouldStopAfterTurn()  // hook: early exit
  → getSteeringMessages()  // drain queue
  → loop again if messages available
```

### AgentHarness prompt (full stack)

```
harness.prompt("Hello")
  → createTurnState()
    → session.buildContext()
      → storage.getPathToRoot(leafId)
      → buildSessionContext(entries)  // handles compaction splits
    → resolve system prompt
  → executeTurn()
    → emitHook("before_agent_start")  → result may add messages/change system prompt
    → runAgentLoop(messages, context, loopConfig, handleAgentEvent, signal, streamFn)
      → handleAgentEvent()  // persists messages to session, emits to subscribers
    → flushPendingSessionWrites()
    → return last AssistantMessage
```

---

## Design Decisions

- **AgentMessage abstraction**: Messages are not raw LLM types. The agent operates on `AgentMessage`, and `convertToLlm` bridges to LLM-compatible `Message[]` only at the call boundary. This allows apps to inject custom message types (notifications, artifacts) without polluting the LLM context.
- **Two-layer public API**: `Agent` is a lightweight stateful wrapper; `AgentHarness` adds session persistence, hooks, compaction, and tree navigation. Users choose based on their needs.
- **Event-driven architecture**: Both `Agent` and `AgentHarness` emit typed lifecycle events. UIs render by subscribing. Listener promises are awaited in order, which means event handlers can serve as barriers (e.g., flush session writes at `turn_end`).
- **Three message queues**: `steer` (inject mid-run after current turn), `followUp` (run after agent would stop), `nextTurn` (prepend to next explicit prompt). Each with configurable drain mode (`"all"` or `"one-at-a-time"`).
- **Parallel tool execution**: Tools execute concurrently by default. Per-tool `executionMode: "sequential"` forces the entire batch to run sequentially. `beforeToolCall` runs sequentially (preflight) even in parallel mode; `afterToolCall` runs after each tool finishes.
- **Session as a tree**: Sessions are append-only trees. Compaction creates summary entries without mutating history. Navigation moves the leaf pointer and optionally generates branch summaries — no history is lost.
- **Result<T, E> pattern**: Harness filesystem and shell operations use `Result` discriminated unions instead of throwing. This forces callers to handle failures explicitly and avoids uncaught rejections in hook chains.
- **Typed error hierarchy**: Every subsystem has its own error class with stable codes — `SessionError.code`, `CompactionError.code`, etc. `AgentHarnessError` wraps them at the top level.
- **Lazy-cut compaction**: `findCutPoint` seeks the nearest valid boundary after accumulating the desired token budget. If the boundary splits a turn, the turn prefix is summarized separately to preserve conversational coherence.
- **Skills and prompt templates as resources**: Both are loaded from the filesystem, exposed via `AgentHarnessResources`, and can be updated at runtime via `setResources()`. System prompt callbacks receive the current resources for dynamic prompt generation.
