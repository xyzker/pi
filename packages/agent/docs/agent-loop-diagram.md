# Agent Loop Architecture

> Source: `packages/agent/src/agent-loop.ts`

## Flowchart

```mermaid
flowchart TD
  subgraph Entry["Entry Points"]
    AL["agentLoop(prompts, ctx, config)"]
    ALC["agentLoopContinue(ctx, config)"]
  end

  subgraph Lifecycle["Agent Lifecycle"]
    AS["emit: agent_start"]
    TS["emit: turn_start"]
    MS["emit: message_start/end<br/>(for each prompt)"]
  end

  subgraph RunLoop["runLoop() - Core Loop"]
    OL{"Outer Loop<br/>(follow-up messages?)"}
    IL{"Inner Loop<br/>(hasMoreToolCalls OR<br/>pendingMessages > 0)"}
    PM["Process Pending Messages<br/>(steering / follow-up)"]
    SAR["streamAssistantResponse()"]
    ET["executeToolCalls()"]
    TE["emit: turn_end"]
    PN["prepareNextTurn()"]
    SS{"shouldStopAfterTurn()?"}
    GS["getSteeringMessages()"]
    AE["emit: agent_end"]
    GF["getFollowUpMessages()"]
  end

  subgraph LLM["streamAssistantResponse()"]
    TC["transformContext()?"]
    CTL["convertToLlm()<br/>AgentMessage[] → Message[]"]
    GAK["getApiKey()?"]
    STREAM["streamSimple() LLM call"]
    EVENTS["Stream events:<br/>start → text_delta*<br/>→ toolcall_delta* → done"]
    EMIT_MS["emit: message_start"]
    EMIT_MU["emit: message_update*<br/>(per delta)"]
    EMIT_ME["emit: message_end"]
  end

  subgraph ToolExec["executeToolCalls()"]
    MODE{"Execution Mode?"}
    SEQ["executeToolCallsSequential()"]
    PAR["executeToolCallsParallel()"]
  end

  subgraph ToolLifecycle["Per-Tool Lifecycle"]
    TES["emit: tool_execution_start"]
    PREP["prepareToolCall()"]
    PREP_CHK{"kind?"}
    EXEC["executePreparedToolCall()"]
    TEU["emit: tool_execution_update*"]
    FIN["finalizeExecutedToolCall()"]
    TEE["emit: tool_execution_end"]
    CTR["createToolResultMessage()"]
    ETR["emit: message_start/end<br/>(toolResult)"]
  end

  subgraph Prepare["prepareToolCall() Details"]
    FT["Find tool by name"]
    PA["prepareToolCallArguments()"]
    VA["validateToolArguments()"]
    BTC["beforeToolCall() hook?"]
  end

  subgraph Finalize["finalizeExecutedToolCall() Details"]
    ATC["afterToolCall() hook?"]
    MERGE["Merge overrides:<br/>content, details,<br/>isError, terminate"]
  end

  subgraph Termination["Batch Termination"]
    STB["shouldTerminateToolBatch()<br/>true when ALL results<br/>have terminate=true"]
  end

  AL --> AS --> TS --> MS --> OL
  ALC --> AS --> TS --> OL

  OL -->|first turn or<br/>follow-ups pending| IL
  OL -->|no follow-ups| AE

  IL -->|yes| TS
  TS --> PM
  PM --> SAR
  SAR --> TC
  TC --> CTL
  CTL --> GAK
  GAK --> STREAM
  STREAM --> EVENTS
  EVENTS -->|"start"| EMIT_MS
  EVENTS -->|"text/tool delta"| EMIT_MU
  EMIT_MU --> EVENTS
  EVENTS -->|"done/error"| EMIT_ME
  EMIT_ME -->|stopReason=error/aborted| AE
  EMIT_ME -->|has toolCalls| ET
  EMIT_ME -->|no toolCalls| TE

  ET --> MODE
  MODE -->|"sequential"| SEQ
  MODE -->|"parallel"| PAR

  SEQ --> TES
  PAR --> TES
  TES --> PREP
  PREP --> FT
  FT -->|found| PA
  FT -->|not found| PREP_CHK
  PA --> VA
  VA --> BTC
  BTC -->|blocked| PREP_CHK
  BTC -->|allowed| PREP_CHK

  PREP_CHK -->|"kind=immediate<br/>(error/blocked/aborted)"| TEE
  PREP_CHK -->|"kind=prepared"| EXEC
  EXEC --> TEU
  TEU --> EXEC
  EXEC --> FIN
  FIN --> ATC
  ATC --> MERGE
  MERGE --> TEE

  TEE --> CTR
  CTR --> ETR
  ETR -->|more tools| TES
  ETR -->|last tool| STB
  STB --> TE

  TE --> PN
  PN --> SS
  SS -->|yes| AE
  SS -->|no| GS
  GS -->|has messages| IL
  GS -->|no messages| GF
  GF -->|has messages| OL
  GF -->|no messages| OL

  classDef entry fill:#4a9eff,color:#fff
  classDef lifecycle fill:#6c5ce7,color:#fff
  classDef loop fill:#00b894,color:#fff
  classDef llm fill:#fdcb6e,color:#333
  classDef tool fill:#e17055,color:#fff
  classDef prep fill:#a29bfe,color:#fff
  classDef final fill:#fd79a8,color:#fff
  classDef term fill:#636e72,color:#fff

  class AL,ALC entry
  class AS,TS,MS,TE,AE lifecycle
  class OL,IL,PM,SAR,ET,PN,SS,GS,GF loop
  class TC,CTL,GAK,STREAM,EVENTS,EMIT_MS,EMIT_MU,EMIT_ME llm
  class MODE,SEQ,PAR,TES,EXEC,TEU,TEE,CTR,ETR tool
  class FT,PA,VA,BTC,PREP,PREP_CHK prep
  class ATC,MERGE,FIN final
  class STB term
```

## Key Design Decisions

1. **Two-layer loop**: The outer loop handles follow-up messages (queued while the agent runs),
   the inner loop processes tool calls and steering messages within a single "conversation turn".

2. **AgentMessage abstraction**: All messages flow as `AgentMessage` throughout the loop.
   Conversion to LLM-native `Message[]` happens only in `convertToLlm()` right before
   the provider call, keeping the loop provider-agnostic.

3. **Dual tool execution modes**:
   - **Sequential**: each tool is prepared → executed → finalized one at a time, with events
     emitted in order.
   - **Parallel**: all tools are prepared sequentially, then allowed tools execute concurrently.
     `tool_execution_end` is emitted in completion order; tool-result messages are emitted in
     assistant source order.

4. **Hook pipeline per tool**: `beforeToolCall` → `execute` → `afterToolCall`, with each hook
   able to short-circuit (block) or override results.

5. **Streaming**: Assistant responses stream delta-by-delta via `message_update` events, while
   tool execution streams via `tool_execution_update` events. The partial message is kept
   in-place in `context.messages` and atomically replaced on each delta.

## Entry Points

### `agentLoop(prompts, ctx, config)`

Starts a new agent run with one or more prompt messages. Creates a new `EventStream`, spawns
`runAgentLoop` in the background (fire-and-forget), and returns the stream immediately.

Events emitted upfront: `agent_start` → `turn_start` → `message_start/end` (× prompts).

### `agentLoopContinue(ctx, config)`

Resumes an agent run from existing context without adding new messages. Used for retries.
Validates that the last context message is not `assistant`, then spawns `runAgentLoopContinue`.

## Event Stream Protocol

| Event | When Emitted |
|---|---|
| `agent_start` | Run begins |
| `turn_start` | Each inner-loop iteration starts |
| `message_start` | A message enters the transcript |
| `message_update` | Delta on the in-progress assistant message |
| `message_end` | A message is finalized |
| `tool_execution_start` | Tool execution begins |
| `tool_execution_update` | Partial result from a running tool |
| `tool_execution_end` | Tool execution completes |
| `turn_end` | Inner-loop iteration completes |
| `agent_end` | Run terminates (carries final `messages[]`) |

## Hook and Callback Summary

| Config Hook | When Called | Purpose |
|---|---|---|
| `transformContext` | Before `convertToLlm`, each inner turn | AgentMessage-level transforms (pruning, injection) |
| `convertToLlm` | Before every LLM call | AgentMessage[] → Message[] translation |
| `getApiKey` | Before every LLM call | Resolve fresh API key (e.g. OAuth) |
| `beforeToolCall` | After validation, before execution | Block or allow tool execution |
| `afterToolCall` | After execution, before `tool_execution_end` | Override tool results |
| `prepareNextTurn` | After `turn_end` | Swap model/context/thinking level for next turn |
| `shouldStopAfterTurn` | After `prepareNextTurn` | Request graceful stop after current turn |
| `getSteeringMessages` | After `shouldStopAfterTurn` returns false | Inject mid-run steering messages |
| `getFollowUpMessages` | After inner loop drains (no more tool calls) | Inject follow-ups that wait for agent to finish |
