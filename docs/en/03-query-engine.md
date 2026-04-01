# QueryEngine and the Conversation Loop

## Overview

`QueryEngine` is the core conversation engine of Claude Code CLI, responsible for managing the multi-turn dialogue flow between the user and the AI. It encapsulates key capabilities including message management, tool invocation permission control, streaming responses, and cost tracking -- serving as the "brain" of the entire CLI.

---

## The QueryEngine Class (QueryEngine.ts)

### Configuration Structure

`QueryEngine` is initialized with a comprehensive configuration object that covers all tunable parameters of the conversation engine:

```typescript
type QueryEngineConfig = {
  cwd: string                    // Working directory
  tools: Tools                   // Available tool set
  commands: Command[]            // Slash command list
  mcpClients: MCPServerConnection[]  // MCP server connections
  agents: AgentDefinition[]      // Agent definitions
  canUseTool: CanUseToolFn       // Tool permission check function
  getAppState: () => AppState    // Get application state
  setAppState: (f: (prev: AppState) => AppState) => void  // Update application state
  initialMessages?: Message[]    // Initial messages (for session restoration)
  readFileCache: FileStateCache  // File read cache
  customSystemPrompt?: string    // Custom system prompt (replaces default)
  appendSystemPrompt?: string    // Appended system prompt
  userSpecifiedModel?: string    // User-specified model
  fallbackModel?: string         // Fallback model
  thinkingConfig?: ThinkingConfig  // Thinking mode configuration
  maxTurns?: number              // Maximum conversation turns
  maxBudgetUsd?: number          // Maximum budget (USD)
  taskBudget?: { total: number } // Task budget
  jsonSchema?: Record<string, unknown>  // JSON Schema for constrained output
  verbose?: boolean              // Verbose output mode
  replayUserMessages?: boolean   // Whether to replay user messages
  handleElicitation?: ToolUseContext['handleElicitation']  // Interactive confirmation handler
  includePartialMessages?: boolean  // Whether to include partial messages
  setSDKStatus?: (status: SDKStatus) => void  // SDK status callback
  abortController?: AbortController  // Abort controller
  orphanedPermission?: OrphanedPermission  // Orphaned permission handling
  snipReplay?: (                 // Context trimming replay function
    yieldedSystemMsg: Message,
    store: Message[]
  ) => { messages: Message[]; executed: boolean } | undefined
}
```

**Key Configuration Notes:**

- **`canUseTool`**: The permission gatekeeper function -- every tool invocation passes through it before execution to determine whether the call is allowed
- **`readFileCache` (FileStateCache)**: Caches the state of files already read, avoiding redundant reads while also detecting file changes
- **`snipReplay`**: When the context window approaches its limit, this function trims and compresses the conversation history to keep the dialogue sustainable
- **`abortController`**: Allows the user to cancel in-progress API calls or tool executions at any time

---

### Core Method: submitMessage()

`submitMessage()` is the entry point of `QueryEngine`, implemented using an **AsyncGenerator** pattern for streaming output:

```typescript
async *submitMessage(): AsyncGenerator<SDKMessage, void, unknown>
```

This design allows callers to receive messages incrementally, enabling real-time UI updates. The key responsibilities within this method include:

1. **Permission wrapping**: Wraps `canUseTool` with additional tracking of permission denial events (`SDKPermissionDenial`) for analytics and auditing
2. **Message history management**: Maintains the `mutableMessages` array to record the full conversation context
3. **Cost tracking**: Accumulates token usage and API call costs across turns
4. **File state caching**: Maintains a cache for file read operations to improve performance
5. **Skill discovery**: Tracks newly discovered skills during each conversation turn
6. **Memory loading**: Loads persisted memories from memdir and injects them into the context

---

## The Conversation Loop

The `QueryEngine` conversation loop follows a classic **Agent Loop** pattern, completing tasks through repeated API calls and tool executions:

```
User input -> Assemble system prompt -> API call -> Process response -> [Tool call -> Loop] -> Output result
```

### Detailed Flow

#### Step 1: User Submits a Message

The user message enters `submitMessage()` and is appended to the `mutableMessages` history.

#### Step 2: Assemble the System Prompt

The system prompt is dynamically composed from multiple parts:

```typescript
// Collect system prompt fragments from multiple sources
const systemPromptParts = await fetchSystemPromptParts(config)
// Combine with query context to produce the final system prompt
const systemPrompt = buildSystemPrompt(systemPromptParts, queryContext)
```

Sources include:
- Base system prompt (role definition, behavioral guidelines)
- `customSystemPrompt` or `appendSystemPrompt` (user-defined)
- Tool descriptions
- Project context (CLAUDE.md, etc.)
- Persisted memories from memdir

#### Step 3: API Call

A request is sent to the Claude API via the `query()` function:

```typescript
const response = query({
  messages: mutableMessages,
  systemPrompt,
  tools: availableTools,
  model: selectedModel,
  thinkingConfig,
  abortController,
  // ...other parameters
})
```

#### Step 4: Process the Response

The API response contains two types of content blocks:

- **Text blocks (`text`)**: Yielded directly to the caller as `SDKMessage` objects
- **Tool use blocks (`tool_use`)**: Enter the tool execution flow

#### Step 5: Tool Call Execution

For `tool_use` blocks, the following sub-flow is executed:

```
tool_use block -> canUseTool permission check -> Execute tool -> Generate tool_result -> Append to message history
```

The permission check can produce three outcomes:
- **Allowed**: The tool is executed immediately
- **Denied**: An `SDKPermissionDenial` is generated, and the denial information is returned to the model as a `tool_result`
- **Requires user confirmation**: Interaction with the user is initiated via `handleElicitation`

#### Step 6: Loop Continues

If the response contains tool calls, the tool results are appended to the message history, and execution **returns to Step 3** to call the API again. The model decides the next action based on the tool execution results -- it may continue calling tools or generate a final text response.

#### Step 7: Streaming Output

Throughout the entire process, `SDKMessage` objects are continuously yielded:

```typescript
// Yield messages at various stages of the loop
yield { type: 'text', content: '...' }
yield { type: 'tool_use', id: '...', name: '...', input: {...} }
yield { type: 'tool_result', tool_use_id: '...', content: '...' }
```

### Loop Termination Conditions

The conversation loop terminates under the following conditions:

- The model returns `stop_reason: "end_turn"` with no pending tool calls
- The `maxTurns` turn limit is reached
- The `maxBudgetUsd` cost limit is reached
- The `abortController` fires an abort signal

---

## Key Features in Detail

### 1. Abort Mechanism (AbortController)

```typescript
abortController?: AbortController
```

Users can cancel any in-progress operation at any time via the `AbortController`. The abort signal propagates to:
- In-flight API streaming requests
- Currently executing tool operations
- The entire conversation loop

### 2. Permission Denial Tracking (SDKPermissionDenial)

Each time a tool call is denied, the system generates an `SDKPermissionDenial` event. These events are collected for:
- Showing the user which operations were blocked
- Helping the model understand constraints and adjust subsequent strategies
- Security auditing and logging

### 3. File State Cache (FileStateCache)

```typescript
readFileCache: FileStateCache
```

A caching layer for file read operations with the following core functions:
- Prevents the same file from being read multiple times within a single conversation
- Detects whether a file has been externally modified during the conversation
- Provides the model with a consistent view of file contents

### 4. Context Trimming (snipReplay)

```typescript
snipReplay?: (
  yieldedSystemMsg: Message,
  store: Message[]
) => { messages: Message[]; executed: boolean } | undefined
```

When the conversation history grows too long and approaches the context window limit, `snipReplay` is responsible for:
- Compressing earlier conversation history
- Preserving critical information (such as tool call results)
- Generating summaries to replace original messages
- Ensuring the conversation can continue without losing important context

### 5. Cost Tracking and Budget Control

```typescript
maxBudgetUsd?: number    // Maximum cost per session
maxTurns?: number        // Maximum conversation turns
taskBudget?: { total: number }  // Task-level budget
```

The engine accumulates costs after each conversation turn and gracefully terminates the loop when a limit is reached, notifying the user.

### 6. Session Persistence

Previous conversations can be restored via `initialMessages`:

```typescript
initialMessages?: Message[]
```

Combined with the `replayUserMessages` option, user messages can be re-executed upon session restoration to ensure tool state consistency.

---

## query.ts -- The API Communication Layer

`query.ts` encapsulates the low-level communication logic with the Claude API, serving as the bridge between `QueryEngine` and the API.

### Core Responsibilities

```typescript
async function query({
  messages,
  systemPrompt,
  tools,
  model,
  thinkingConfig,
  abortController,
  // ...
}): Promise<APIResponse>
```

**Key Functions:**

1. **Message formatting**: Converts internal message formats to the format required by the Claude API
2. **Streaming response handling**: Receives incremental responses via SSE (Server-Sent Events)
3. **Retry logic**: Implements automatic retries for transient errors (network timeouts, 429 rate limits, 5xx errors)
4. **Prompt cache break detection**: Detects whether the prompt cache was hit to optimize subsequent requests
5. **Model fallback**: Automatically switches to the `fallbackModel` when the preferred model is unavailable

### Retry Strategy

```
Request fails -> Determine error type -> Transient error? -> Exponential backoff retry -> Throw on final failure
                                         |
                                    Permanent error -> Throw immediately
```

Transient errors include:
- HTTP 429 (rate limiting)
- HTTP 5xx (server errors)
- Network connection timeouts
- Streaming connection interruptions

### Collaboration with QueryEngine

`query()` is called repeatedly by `QueryEngine`'s conversation loop. On each call:
- `QueryEngine` provides the full message history and the latest system prompt
- `query()` returns the model's response (which may contain text and tool calls)
- `QueryEngine` processes the response and decides whether to continue the loop

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────┐
│                   QueryEngine                        │
│                                                      │
│  submitMessage() <- AsyncGenerator streaming output   │
│       |                                              │
│       v                                              │
│  ┌──────────── Conversation Loop ───────────┐        │
│  │                                          │        │
│  │  Assemble system prompt                  │        │
│  │       |                                  │        │
│  │       v                                  │        │
│  │  query() -> Claude API                   │        │
│  │       |                                  │        │
│  │       v                                  │        │
│  │  Process response                        │        │
│  │    |-- Text -> yield SDKMessage          │        │
│  │    `-- tool_use -> Perm check -> Exec -> Loop     │
│  │                                          │        │
│  └──────────────────────────────────────────┘        │
│                                                      │
│  Supporting systems:                                 │
│  * FileStateCache    File state cache                │
│  * AbortController   Abort control                   │
│  * Cost tracking     Budget control                  │
│  * snipReplay        Context trimming                │
│  * SDKPermissionDenial  Permission auditing          │
└─────────────────────────────────────────────────────┘
```

The design of `QueryEngine` reflects several core principles:

- **Streaming-first**: Incremental output via AsyncGenerator -- users do not need to wait for a complete response
- **Safe and controllable**: Multiple layers of permission checks and budget controls prevent runaway behavior
- **Recoverable**: Session persistence and context trimming ensure sustainability for long conversations
- **Observable**: Cost tracking, permission auditing, and status callbacks provide full runtime visibility
