# 08 - Design Pattern Essentials

> Distilled design patterns worth learning from, based on Claude Code CLI v2.1.88 source code analysis.

---

## 1. AsyncGenerator Streaming Pattern

The core conversation loop in Claude Code uses `AsyncGenerator` for streaming output, rather than callbacks or Observables.

```typescript
// Core signature
class QueryEngine {
  async *submitMessage(
    userMessage: string,
    options: SubmitOptions
  ): AsyncGenerator<SDKMessage, void, unknown> {
    // Call the Claude API, streaming results back
    for await (const event of apiStream) {
      const message = transformEvent(event);
      yield message;  // Consumer receives messages one at a time

      // Cancellation via AbortController
      if (options.abortController.signal.aborted) {
        return; // Clean exit
      }
    }
  }
}
```

### Consumer Side

```typescript
// UI layer consumption
for await (const message of engine.submitMessage(input, opts)) {
  renderToTerminal(message);
}

// SDK layer consumption — same Generator, no adaptation needed for different scenarios
for await (const message of engine.submitMessage(input, opts)) {
  sendToClient(JSON.stringify(message));
}
```

### Why AsyncGenerator

| Dimension | Callback | Observable | AsyncGenerator |
|-----------|----------|------------|----------------|
| Backpressure | None | Manual | Built-in |
| Cancellation | Complex | unsubscribe | AbortController + return() |
| Error handling | Scattered try/catch | onError | Unified try/catch |
| Composability | Poor | Excellent | Good (for-await) |

### Cancellation Mechanism

```typescript
const controller = new AbortController();

for await (const event of submitMessage(prompt, controller.signal)) {
  ui.render(event);
  if (userPressedEscape) {
    controller.abort(); // Triggers AbortError, generator cleans up and exits
    break;
  }
}
```

**Practical takeaways:**
- `yield` is a natural pause point — if downstream consumption is slow, upstream automatically waits with no additional throttling needed
- `AbortController.signal` is passed to both the HTTP client and the generator, enabling cancellation across the entire chain in one action
- Resource cleanup in `finally` blocks executes regardless of whether execution ends normally or is interrupted

---

## 2. Compile-Time Feature Flag Pattern

Claude Code uses Bun's bundler `feature()` function to implement compile-time conditional compilation, distinguishing between internal (Anthropic) and external user builds.

```typescript
// Evaluated at compile time, not at runtime
import { feature } from 'bun:bundle';

// Compiles to a true or false literal
const InternalTool = feature('INTERNAL')
  ? require('./internal/InternalTool.js').InternalTool
  : null;

// When the Bun bundler sees feature('INTERNAL') === false,
// the entire require('./internal/InternalTool.js') is eliminated as dead code
```

### Build Output Comparison

```typescript
// External build compiles to the equivalent of:
const InternalTool = null;
// The require('./internal/InternalTool.js') line does not exist in the output at all

// Internal build compiles to the equivalent of:
const InternalTool = require('./internal/InternalTool.js').InternalTool;
```

### Build Configuration

```typescript
// build.ts
Bun.build({
  define: {
    "feature.internal": isInternalBuild ? "true" : "false",
  },
  // Bun removes unreachable branches during tree-shaking
});
```

**Differences from runtime flags:**

| Aspect | Compile-time flag | Runtime flag |
|--------|-------------------|--------------|
| Bundle size | Smaller (DCE removes unreachable code) | Everything included |
| Runtime overhead | Zero | Evaluated every time |
| Flexibility | Requires rebuild | Dynamic toggling |
| Security | Physical isolation, code never leaks | Code remains in the bundle |
| Use case | Internal/external distinction | A/B testing, gradual rollouts |

**Practical takeaways:**
- Use compile-time flags for sensitive features (internal debug tools, experimental APIs) to ensure zero leakage in external builds
- This is essentially constant folding + DCE, the same principle as Webpack's `DefinePlugin`
- Use `null` rather than `undefined` as the placeholder value, forcing consumers to perform null checks

---

## 3. Layered Permission System

Claude Code's permission system consists of multiple layers: mode -> tool rules -> runtime checks.

### Layer Structure

```
+-----------------------------------------+
|  Permission Mode (global policy)        |
|  default | plan | autoApprove | bypass  |
+-----------------------------------------+
|  Per-Tool Rules (tool-level rules)      |
|  alwaysAllow | alwaysDeny | alwaysAsk   |
+-----------------------------------------+
|  Rule Source (rule origins)             |
|  user config | project config | MDM     |
+-----------------------------------------+
|  Runtime Context (runtime context)      |
|  DeepImmutable<PermissionContext>        |
+-----------------------------------------+
```

### Permission Rules Organized by Source

```typescript
type ToolPermissionRulesBySource = {
  user: ToolPermissionRule[];        // User personal configuration
  project: ToolPermissionRule[];     // Project-level .claude configuration
  enterprise: ToolPermissionRule[];  // Enterprise MDM policy
};

type ToolPermissionRule = {
  tool: string;                      // Tool name or wildcard
  permission: 'alwaysAllow' | 'alwaysDeny' | 'alwaysAsk';
  pattern?: string;                  // Optional path/argument matching
};
```

### DeepImmutable Protects Permission Context

```typescript
// Recursive read-only wrapper ensures the permission context cannot be accidentally modified
type DeepImmutable<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepImmutable<T[K]>
    : Readonly<T[K]>;
};

// The context passed to tools is immutable
function executeToolCall(
  context: DeepImmutable<PermissionContext>,
  tool: Tool,
  input: unknown
) { /* ... */ }
```

### Injectable canUseTool

```typescript
// canUseTool as an injectable function, facilitating testing and customization
type CanUseTool = (
  tool: Tool,
  input: unknown,
  context: PermissionContext
) => Promise<PermissionResult>;

async function canUseTool(
  tool: Tool, input: unknown, context: PermissionContext
): Promise<PermissionResult> {
  // 1. Check global mode
  if (context.mode === "bypassPermissions") return { allowed: true };
  if (context.mode === "plan") return { allowed: false, reason: "read_only" };

  // 2. Match tool-level rules (priority: enterprise > project > user)
  const rule = matchRule(tool.name, input, context.rules);
  if (rule) return { allowed: rule.permission === "alwaysAllow" };

  // 3. Default requires user approval
  return { allowed: false, reason: "requires_approval" };
}

// Can be replaced with a mock in tests
const mockCanUseTool: CanUseTool = async () => ({ allowed: true });
```

**Practical takeaways:**
- The layered design enables both coarse-grained control (modes) and fine-grained adjustments (tool rules)
- Multi-source rules have clear priority: enterprise policy > project config > user preferences
- `DeepImmutable` provides type-level protection at zero runtime cost
- The injectable `canUseTool` can be mocked in tests without depending on real configuration files

---

## 4. Circular Dependency Handling

Circular dependencies are nearly unavoidable in large TypeScript projects. Claude Code uses three strategies to break cycles.

### Strategy 1: Lazy require()

```typescript
// Instead of requiring at module top level, defer until function invocation
const getQueryEngine = () => require('./QueryEngine.js').QueryEngine;

function createSession() {
  // By this point the QueryEngine module has finished initializing, so the cycle is safe
  const QueryEngine = getQueryEngine();
  return new QueryEngine(/* ... */);
}
```

### Strategy 2: Dynamic import at Call Site

```typescript
async function handleSpecialCase() {
  // Load only when needed, completely bypassing cycles during module initialization
  const { SpecialHandler } = await import('./SpecialHandler.js');
  return new SpecialHandler();
}
```

### Strategy 3: Type-Only Imports

```typescript
// Import only the type — completely eliminated after compilation, no runtime dependency
import type { QueryEngine } from './QueryEngine.js';

// At runtime, the instance is passed in as a parameter, not imported by this module
function processResult(engine: QueryEngine, result: unknown) {
  // ...
}
```

### Comparison and Selection

| Strategy | Use case | Pros | Cons |
|----------|----------|------|------|
| Lazy require | Synchronous contexts, frequent calls | Simple and direct, module cache works | Deferred loading affects first-call performance |
| Dynamic import | Async contexts, infrequent paths | Most thorough decoupling | Requires async/await |
| Type-only import | Only types needed, not values | Zero runtime overhead | Cannot be used for instantiation |

**Key principle: At the module top level, establish only type relationships. Defer runtime dependencies until they are actually needed.** If circular dependencies appear frequently, it signals that module boundaries need refactoring — extract a shared interface layer.

---

## 5. Parallel Prefetch Pattern

Claude Code's startup path launches async operations in parallel during time windows where it must wait anyway, implementing a "fire-forget-collect later" approach.

### Core Idea

```typescript
// Phase 1: Fire async operations immediately (no await)
startMdmRawRead();          // Spawn subprocess to read MDM config
startKeychainPrefetch();    // Start macOS Keychain read

// Phase 2: Do synchronous work that doesn't depend on the above results (~135ms)
import { ConfigManager } from './config/ConfigManager.js';
import { ToolRegistry } from './tools/ToolRegistry.js';
import { UIRenderer } from './ui/UIRenderer.js';
// ... many imports happen here ...

// Phase 3: Collect prefetch results
await ensureKeychainPrefetchCompleted();
// By now the subprocesses have most likely completed, await costs nearly zero
```

### Timeline Comparison

```
Traditional sequential execution:
  [Keychain 80ms] -> [MDM 40ms] -> [imports 135ms] -> [init]
  Total: ~255ms

Parallel prefetch:
  [Keychain 80ms.............]
  [MDM 40ms.....]
  [imports 135ms.........................] -> [await 0ms] -> [init]
  Total: ~135ms
```

### Implementation Details

```typescript
let keychainPromise: Promise<KeychainData> | null = null;

// Start prefetch — just save the Promise reference
function startKeychainPrefetch(): void {
  keychainPromise = readKeychain(); // Returns a Promise immediately
}

// Wait for completion — usually resolves instantly
async function ensureKeychainPrefetchCompleted(): Promise<KeychainData> {
  if (!keychainPromise) {
    throw new Error('Prefetch not started');
  }
  return keychainPromise; // Await an already-resolved Promise
}
```

### Difference from Promise.all

```typescript
// Promise.all: initiate and wait at the same location
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);

// Prefetch pattern: initiate as early as possible + await as late as possible, do other work in between
const promiseA = fetchA();  // Fire immediately
doSyncWork();               // Synchronous work fills the waiting time
const a = await promiseA;   // Collect the result
```

**Practical takeaways:**
- Key insight: although import statements are synchronous, the event loop continues running during their execution, allowing previously started async operations to complete during this time
- Separate initiation and awaiting, inserting work that doesn't depend on the results in between
- Un-awaited Promise rejections become unhandled rejections — add `.catch(noop)` when necessary

---

## 6. Fast Path Pattern

For simple commands (such as `--version`, `--help`), skip the expensive full initialization.

```typescript
// entrypoints/cli.tsx
async function main() {
  const args = process.argv.slice(2);

  // === Fast path: no modules need to be loaded ===
  if (args.includes('--version')) {
    console.log(VERSION);  // VERSION is an inlined constant
    return;
  }

  if (args.includes('--help')) {
    printHelp();  // Pure string output
    return;
  }

  // === Fast path: only partial modules needed ===
  if (args.includes('--resume')) {
    const { resumeSession } = await import('./session/resume.js');
    return resumeSession(args);
  }

  // === Full path: load all modules ===
  const { ConfigManager } = await import('./config/ConfigManager.js');
  const { ToolRegistry } = await import('./tools/ToolRegistry.js');
  const { UIRenderer } = await import('./ui/UIRenderer.js');
  // ... full initialization ...
}
```

### Design Principles

```
Command frequency   Initialization cost   Strategy
---------------------------------------------
--version           Minimal               Direct process.argv check
--help              Minimal               String output, no dependencies
--resume            Medium                Dynamic import of partial modules
Normal startup      Full                  Load all modules and configuration
```

**Practical takeaways:**
- Place argument parsing before all heavyweight initialization
- Fast paths depend only on static data (version number, help text), never touching the filesystem or network
- `claude --version` returns almost instantly, significantly improving perceived CLI responsiveness
- Dynamic `import()` turns module loading into an on-demand behavior rather than a declaration-time behavior

---

## 7. Migration Pattern

Configuration schemas evolve over versions. Claude Code handles this with named, idempotent migration functions.

### Migration Function Design

```typescript
// Each migration is a named function — the name serves as documentation
const migrations = [
  migrateFennecToOpus,
  migrateSonnet45ToSonnet46,
  migrateOldPermissionFormat,
  migrateToolAliases,
];

type MigrationFn = (config: UserConfig) => UserConfig;
```

### Sequential Execution at Startup

```typescript
function runMigrations(config: UserConfig): UserConfig {
  let current = config;
  for (const migration of migrations) {
    current = migration(current);
  }
  return current;
}
```

### Idempotency Guarantees

```typescript
function migrateSonnet45ToSonnet46(config: UserConfig): UserConfig {
  // Idempotent: only modify when the old value is present
  if (config.defaultModel === 'claude-sonnet-4-5') {
    return { ...config, defaultModel: 'claude-sonnet-4-6' };
  }
  return config; // Already migrated or no migration needed, return as-is
}

function migrateOldPermissionFormat(config: UserConfig): UserConfig {
  // Idempotent: check whether the old format exists
  if (!config.permissions && config.legacyPermissions) {
    const { legacyPermissions, ...rest } = config;
    return {
      ...rest,
      permissions: convertLegacyPermissions(legacyPermissions),
    };
  }
  return config;
}
```

### Design Highlights

| Feature | Explanation |
|---------|-------------|
| **Named functions** | `migrateFennecToOpus` is more self-explanatory than `migration_001` |
| **Sequential execution** | Migrations may have dependencies; ordering guarantees correctness |
| **Idempotent safety** | Repeated runs produce no side effects; safe to re-run after a crash |
| **Pure functions** | Input config -> output config, no external state dependencies |

**Practical takeaways:** Applicable to any long-lived project that needs schema evolution. Always append new migrations; never modify historical ones.

---

## 8. MCP Extensibility Pattern

The Model Context Protocol (MCP) is an open tool extension mechanism that replaces plugin APIs with a standard protocol.

### Architecture Overview

```
Claude Code CLI
  +-- Built-in tools (Read, Edit, Bash, Grep, Glob, ...)
  +-- MCP Client
        +-- GitHub MCP Server   -> git operations
        +-- Postgres MCP Server -> database queries
        +-- Custom MCP Server   -> arbitrary functionality
```

### Server Configuration

```typescript
// .claude/settings.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/mcp-server"],
      "env": { "GITHUB_TOKEN": "..." }
    },
    "database": {
      "command": "python",
      "args": ["db_server.py"],
      "allowedTools": ["query", "schema"],   // Tool whitelist
      "deniedTools": ["drop_table"]          // Tool blocklist
    }
  }
}
```

### Lifecycle

```
Discovery          Approval          Execute           Result
  Scan config  ->  Permission    ->  JSON-RPC      ->  Type conversion
  List tools       system            protocol          Result validation
                   Enterprise
                   policy filter
```

### Enterprise Policy Filtering

```typescript
interface EnterprisePolicy {
  allowedMcpServers: string[];
  blockedMcpTools: string[];
  requireServerSignature: boolean;
}

function filterMcpTools(
  tools: McpTool[],
  policy: EnterprisePolicy
): McpTool[] {
  return tools.filter(t => !policy.blockedMcpTools.includes(t.name));
}
```

**Practical takeaways:**
- MCP is an inter-process communication protocol (stdio/SSE) — tools run in separate processes, so crashes do not affect the main process
- A unified JSON Schema description enables LLMs to understand the inputs and outputs of any external tool
- Protocol over API: any language, any process can become a tool provider
- The permission system seamlessly covers MCP tools, applying the same security controls as built-in tools

---

## 9. State Management Pattern

Claude Code uses a lightweight state management approach with functional updates, without relying on external libraries like Redux.

### Core API

```typescript
// Get current state (read-only snapshot)
const getAppState: () => Readonly<AppState>;

// Functional update (similar to React's setState)
const setAppState: (updater: (prev: AppState) => AppState) => void;
```

### Functional Updates Guarantee Atomicity

```typescript
// Correct: update based on prev, avoiding race conditions
setAppState(prev => ({
  ...prev,
  turnCount: prev.turnCount + 1,
  lastActivity: Date.now(),
}));

// Wrong: read then set — may overwrite other updates
const state = getAppState();
setAppState(_ => ({
  ...state,  // state here may already be stale!
  turnCount: state.turnCount + 1,
}));
```

### createStore Abstraction

```typescript
function createStore<T>(initialState: T) {
  let state = initialState;
  const listeners = new Set<(state: T) => void>();

  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      state = updater(state);
      listeners.forEach(fn => fn(state));
    },
    subscribe: (fn: (state: T) => void) => {
      listeners.add(fn);
      return () => listeners.delete(fn); // Return unsubscribe function
    },
  };
}
```

### Why "Just Enough" Works

| Choice | Rationale |
|--------|-----------|
| No middleware | A CLI tool doesn't need devtools or time-travel |
| Lightweight subscriptions | React/Ink manages its own render cycle |
| No action types | Functional updates carry their own semantics, no extra categorization needed |
| Minimal implementation | A few dozen lines of code, zero dependencies |

**Practical takeaways:** Functional updates avoid race conditions; `Readonly` prevents consumers from mutating state directly; pub-sub lets the UI layer subscribe to changes and re-render automatically. For non-web TypeScript projects, this "just enough" approach is preferable to introducing a heavyweight state library.

---

## 10. React Terminal UI Pattern

Claude Code uses React + Ink to bring component-based UI development to the terminal, with Flexbox layout powered by the Yoga engine.

### Component Structure

```typescript
import React from 'react';
import { Box, Text } from 'ink';

function MessageBubble({ role, content }: MessageProps) {
  return (
    <Box flexDirection="column" borderStyle="round" paddingX={1}>
      <Text bold color={role === 'assistant' ? 'blue' : 'green'}>
        {role === 'assistant' ? 'Claude' : 'You'}
      </Text>
      <Text>{content}</Text>
    </Box>
  );
}
```

### Flexbox Layout

```typescript
function AppLayout() {
  return (
    <Box flexDirection="column" height="100%">
      {/* Message area: automatically fills remaining space */}
      <Box flexGrow={1} flexDirection="column" overflow="hidden">
        <MessageList />
      </Box>

      {/* Status bar: fixed height */}
      <Box height={1} borderStyle="single" borderTop borderBottom={false}>
        <StatusBar />
      </Box>

      {/* Input area: fixed at the bottom */}
      <Box>
        <InputArea />
      </Box>
    </Box>
  );
}
```

### Terminal-Specific Hooks

```typescript
function useTerminalSize() {
  const [size, setSize] = useState({
    columns: process.stdout.columns,
    rows: process.stdout.rows,
  });

  useEffect(() => {
    const handler = () => setSize({
      columns: process.stdout.columns,
      rows: process.stdout.rows,
    });
    process.stdout.on('resize', handler);
    return () => process.stdout.off('resize', handler);
  }, []);

  return size;
}
```

**Practical takeaways:**
- `<Box>` corresponds to `<div>`, `<Text>` corresponds to `<span>` — web developers face nearly zero learning curve
- Virtual DOM diffing is equally valuable in the terminal — only changed character regions are updated, avoiding full-screen redraw flicker
- Especially critical for streaming output (displaying AI responses character by character): declarative updates are far cleaner than manual cursor manipulation
- Component snapshot testing is possible with `ink-testing-library`

---

## 11. File State Cache

`FileStateCache` avoids redundant reads of the same file within a single conversation turn — the AI may read the same file multiple times (read, then edit, then verify).

### Cache Design

```typescript
interface FileStateCache {
  files: Map<string, FileState>;
}

interface FileState {
  content: string;
  hash: string;
  lastRead: number;
}
```

### Read Path

```typescript
async function readFileWithCache(
  path: string,
  cache: FileStateCache
): Promise<string> {
  const cached = cache.files.get(path);

  if (cached) {
    const currentHash = await quickHash(path);
    if (currentHash === cached.hash) {
      return cached.content;  // Cache hit
    }
  }

  // Cache miss or stale
  const content = await fs.readFile(path, 'utf-8');
  cache.files.set(path, {
    content,
    hash: await quickHash(path),
    lastRead: Date.now(),
  });
  return content;
}
```

### Cache Cloning for Isolation

```typescript
// Each tool invocation gets an independent cache copy
function cloneCache(source: FileStateCache): FileStateCache {
  return { files: new Map(source.files) };
}

// Tool A's file modifications won't pollute Tool B's cache view
const cacheForToolA = cloneCache(masterCache);
const cacheForToolB = cloneCache(masterCache);
```

**Practical takeaways:**
- Using file hash for cache validity is more reliable than mtime
- Proactively `invalidate` after write operations to ensure consistency
- Cache granularity is per-file, and invalidation is also per-file — a single write doesn't clear the entire cache
- Cloning provides isolation between parallel tool invocations

---

## 12. Session Persistence

Claude Code supports session recording (transcript), resumption (resume), and cross-environment transfer (teleport).

### Transcript Recording

```typescript
interface TranscriptMessage {
  role: 'user' | 'assistant' | 'tool';
  content: unknown;
  timestamp: number;
  tokenUsage?: TokenUsage;
}

// JSONL format: one JSON object per line, append-only writes
function recordTranscript(entry: TranscriptMessage): void {
  const line = JSON.stringify(entry) + "\n";
  fs.appendFileSync(transcriptPath, line);
}
```

### Resume

```typescript
// --resume flag continues from the previous session
async function resumeSession(sessionId: string): Promise<void> {
  // 1. Load historical transcript
  const transcript = await loadTranscript(sessionId);

  // 2. Rebuild message history
  const messages = transcript.messages.map(toAPIMessage);

  // 3. Create a new QueryEngine, injecting the history
  const engine = new QueryEngine({ initialMessages: messages });

  // 4. Wait for new user input
  await startInteractiveLoop(engine);
}
```

### Teleport (Cross-Environment Transfer)

```typescript
interface TeleportPayload {
  transcript: SessionTranscript;
  fileContext: FileContextSnapshot;  // Snapshot of relevant files
  workingDirectory: string;
}

// Serialize -> transmit -> deserialize -> restore
async function teleportSession(payload: TeleportPayload): Promise<void> {
  const serialized = JSON.stringify(payload);
  await transmitToEnvironment(serialized, targetEnv);
}
```

**Advantages of the JSONL format:**
- Append-only writes, no need to read and rewrite the entire file
- Single-line JSON is easy to parse in a streaming fashion; at most the last line is lost in a crash
- Can be analyzed directly with `grep`/`jq`

**Practical takeaways:** Session persistence allows long-running tasks to resume after interruption without losing context. Resumption only requires rebuilding the message array — there is no need to replay every tool invocation.

---

## 13. Cost Control Pattern

Claude Code has multiple built-in layers of cost control to prevent runaway expenses when the AI agent operates autonomously.

### Budget Configuration

```typescript
interface BudgetConfig {
  maxBudgetUsd: number;    // Hard dollar limit per session
  maxTurns: number;        // Maximum conversation turns
  taskBudget: number;      // Budget allocation for subtasks
}
```

### Per-Turn Tracking and Checking

```typescript
interface UsageTracker {
  totalInputTokens: number;
  totalOutputTokens: number;
  totalCostUsd: number;
  turnsUsed: number;
}

function checkBudget(
  tracker: UsageTracker,
  config: BudgetConfig
): BudgetStatus {
  if (tracker.totalCostUsd >= config.maxBudgetUsd) {
    return { exceeded: true, reason: 'dollar_limit', spent: tracker.totalCostUsd };
  }
  if (tracker.turnsUsed >= config.maxTurns) {
    return { exceeded: true, reason: 'turn_limit', turns: tracker.turnsUsed };
  }
  return { exceeded: false };
}
```

### Prompt Cache Break Detection

```typescript
// A prompt cache break causes a cost spike (the full context must be reprocessed)
function detectCacheBreak(
  currentUsage: TokenUsage,
  expectedCachedTokens: number
): boolean {
  const cacheHitRate = currentUsage.cache_read_input_tokens
    / expectedCachedTokens;

  if (cacheHitRate < 0.5) {
    console.warn(
      `Cache break detected: hit rate ${(cacheHitRate * 100).toFixed(1)}%`
      + ` — this turn will cost significantly more`
    );
    return true;
  }
  return false;
}
```

**Practical takeaways:**
- `maxBudgetUsd` is a hard ceiling, ideal for CI/automation scenarios to prevent unexpected high charges
- `maxTurns` prevents the agent from getting stuck in infinite loops
- `taskBudget` allows distributing the total budget across subtasks (e.g., sub-agents), enabling hierarchical budget management
- Cache break detection provides early warning of cost anomalies — when changes in the AI conversation context cause cache invalidation, a single turn's cost can double

---

## 14. Error Classification Pattern

`categorizeRetryableAPIError` classifies API errors into retryable and non-retryable categories, driving different handling strategies.

### Error Classification Function

```typescript
type ErrorCategory =
  | { retryable: true;  strategy: RetryStrategy; userMessage: string }
  | { retryable: false; fatal: boolean;          userMessage: string };

type RetryStrategy = {
  maxRetries: number;
  backoffMs: number[];     // Wait time for each retry
  jitter: boolean;         // Whether to add random jitter
};

function categorizeRetryableAPIError(error: APIError): ErrorCategory {
  // 429: Rate limited — retryable, exponential backoff
  if (error.status === 429) {
    return {
      retryable: true,
      strategy: {
        maxRetries: 5,
        backoffMs: [1000, 2000, 4000, 8000, 16000],
        jitter: true,
      },
      userMessage: 'API rate limited, waiting to retry...',
    };
  }

  // 529: Service overloaded — retryable, longer backoff
  if (error.status === 529) {
    return {
      retryable: true,
      strategy: {
        maxRetries: 3,
        backoffMs: [5000, 10000, 20000],
        jitter: true,
      },
      userMessage: 'API service overloaded, waiting...',
    };
  }

  // 5xx: Server error — retryable, shorter backoff
  if (error.status >= 500 && error.status < 600) {
    return {
      retryable: true,
      strategy: { maxRetries: 3, backoffMs: [500, 1000, 2000], jitter: true },
      userMessage: 'API service temporarily unavailable, retrying...',
    };
  }

  // 401: Authentication failure — not retryable, fatal
  if (error.status === 401) {
    return { retryable: false, fatal: true,
      userMessage: 'API key is invalid or expired. Please check your credentials.' };
  }

  // 400: Bad request — not retryable, non-fatal
  if (error.status === 400) {
    return { retryable: false, fatal: false,
      userMessage: 'Malformed request. Please check your input.' };
  }

  // Unknown error — handle conservatively, do not retry
  return { retryable: false, fatal: false,
    userMessage: `Unexpected API error (${error.status})` };
}
```

### Retry Executor

```typescript
async function executeWithRetry<T>(
  fn: () => Promise<T>,
  category: ErrorCategory & { retryable: true }
): Promise<T> {
  const { strategy } = category;

  for (let attempt = 0; attempt <= strategy.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === strategy.maxRetries) throw error;

      let delay = strategy.backoffMs[attempt] ?? strategy.backoffMs.at(-1)!;
      if (strategy.jitter) {
        delay += Math.random() * delay * 0.3; // 30% random jitter
      }
      await sleep(delay);
    }
  }
  throw new Error('unreachable');
}
```

**Practical takeaways:**
- Error classification is centralized in one function — all callers share a unified retry strategy
- `retryAfterMs` prefers the server's `Retry-After` header to avoid blind retries that add load to the server
- Exponential backoff + jitter prevents the "thundering herd" effect (multiple clients retrying simultaneously)
- Authentication errors and malformed request errors should not be retried — retrying only wastes time
- Decoupling "how to classify errors" from "how to handle errors" means adding a new error type only requires adding one branch

---

## Summary: Core Engineering Principles

The design patterns found in the Claude Code source code can be distilled into the following principles:

### Compile-Time over Runtime

Compile-time elimination of feature flags, zero-overhead type-only imports — problems that can be solved at compile time should never be deferred to runtime.

### Defer Everything Non-Essential

Parallel prefetch, fast paths, lazy require, dynamic import — at startup, only do what must be done; load everything else on demand. A CLI tool's cold start speed directly impacts user experience.

### Immutability as a Security Boundary

`DeepImmutable<>` protects permission context, functional `setState` guarantees update atomicity, cache cloning provides isolation — immutability is not merely a code style choice but a core technique for defensive programming.

### Protocol over Interface

MCP uses a standard protocol (JSON-RPC) instead of a programming interface; AsyncGenerator uses language primitives instead of a custom event system — protocol-level design is more flexible and longer-lived than API-level design.

### Explicit Classification Drives Behavior

Error classification (retryable vs. fatal), permission layering (mode -> rule -> source), cost classification (dollar vs. turn vs. task) — classify first, then decide behavior based on the classification. This eliminates if-else chains and makes logic auditable and extensible.

### Idempotency Throughout

Migration functions are idempotent, cache reads are idempotent, permission checks have no side effects — in an environment where you cannot be certain how many times code will execute (CLI restarts, network retries, repeated user actions), idempotency is the foundation of correctness.

### Just-Enough Abstraction

State management doesn't use Redux, the UI framework uses only a minimal subset of React, the tool system only abstracts registration and execution — the cost of over-abstraction is cognitive overhead and debugging difficulty. Just enough is the optimal solution.

---

These patterns are not inventions unique to Claude Code, but their combined application in a real, large-scale TypeScript CLI project demonstrates how to strike a balance between performance, maintainability, and security. Every pattern solves a concrete engineering problem rather than existing for the sake of "architectural correctness."
