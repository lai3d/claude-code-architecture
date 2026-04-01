# Entry Point and Startup Flow

> Based on Claude Code CLI v2.1.88 source code analysis

The Claude Code startup flow is driven by three core files: `cli.tsx` (bootstrap entry), `main.tsx` (full CLI initialization), and `init.ts` (runtime initialization). The overarching design philosophy is **lazy loading wherever possible** -- dynamic imports and fast paths are used extensively to ensure that unneeded code is never loaded.

---

## 1. cli.tsx -- Bootstrap Entry

`cli.tsx` is the outermost entry point. Its core design principle: **if a response can be returned immediately, don't load the full application**.

### Fast Paths

Before entering full CLI initialization, `cli.tsx` checks a series of command-line arguments that can be handled immediately. Each fast path avoids the 135ms+ module import overhead required to load the full application:

```typescript
// --version: prints the build-time inlined version number, zero extra imports
if (args.includes("--version") || args.includes("-v")) {
  console.log(MACRO.VERSION);  // inlined by the bundler at build time
  process.exit(0);
}

// --dump-system-prompt: loads only minimal config and model, prints the system prompt
if (args.includes("--dump-system-prompt")) {
  const { dumpSystemPrompt } = await import("./commands/dump-system-prompt");
  await dumpSystemPrompt();
  process.exit(0);
}
```

Full list of fast paths:

| Argument | Purpose | Notes |
|----------|---------|-------|
| `--version` / `-v` | Print version number | Zero imports, uses build-time inlined `MACRO.VERSION` |
| `--dump-system-prompt` | Output system prompt | Loads only minimal config + model |
| `--claude-in-chrome-mcp` | Start Chrome MCP server | Independent subsystem |
| `--chrome-native-host` | Chrome native messaging host | Independent subsystem |
| `--computer-use-mcp` | Computer use MCP server | Feature-gated by `CHICAGO_MCP` |
| `--daemon-worker` | Daemon worker thread | Feature-gated by `DAEMON` |
| `remote-control`/`rc`/`remote`/`sync`/`bridge` | Bridge mode | Feature-gated by `BRIDGE_MODE` |

### Feature Gates

Claude Code uses build-time feature gates to eliminate dead code. The `feature()` function completely removes disabled feature paths at build time:

```typescript
// Feature-gated fast paths -- entire branch eliminated at build time if not enabled
if (feature("CHICAGO_MCP") && args.includes("--computer-use-mcp")) {
  const { startComputerUseMcp } = await import("./mcp/computer-use");
  await startComputerUseMcp();
  process.exit(0);
}

if (feature("DAEMON") && args.includes("--daemon-worker")) {
  const { startDaemonWorker } = await import("./daemon/worker");
  await startDaemonWorker();
  process.exit(0);
}

if (feature("BRIDGE_MODE") && ["remote-control", "rc", "remote", "sync", "bridge"].includes(args[0])) {
  const { startBridge } = await import("./bridge");
  await startBridge(args);
  process.exit(0);
}
```

### Environment Configuration

Before entering any code path, `cli.tsx` sets up necessary environment variables:

```typescript
// Prevent corepack from auto-pinning versions during package installation
process.env.COREPACK_ENABLE_AUTO_PIN = "0";

// NODE_OPTIONS configuration for remote environments
if (isRemoteEnv()) {
  process.env.NODE_OPTIONS = buildRemoteNodeOptions();
}
```

### Ablation Baseline Mode

When the `ABLATION_BASELINE` feature is enabled and the corresponding environment variable is set, a series of advanced features are disabled (used for A/B testing baseline comparisons):

```typescript
if (feature("ABLATION_BASELINE") && process.env.ABLATION_BASELINE) {
  // Disable thinking (extended thinking)
  // Disable compact (context compression)
  // Disable memory (memory system)
  // Disable background tasks
}
```

### Entering Full Startup

Only when no fast path is matched does `cli.tsx` dynamically import the full CLI:

```typescript
// Only load the full application when all fast paths miss
const { main } = await import("./main");
await main(args);
```

---

## 2. main.tsx -- Full CLI Initialization

`main.tsx` is the true CLI core -- roughly 800+ lines of code responsible for full application initialization.

### Startup Performance Optimization: Parallel Prefetching

At the very top of `main.tsx` (at module level), multiple parallel prefetch operations are initiated. These begin executing during module evaluation, without waiting for any other initialization:

```typescript
// Module top level -- parallel prefetching starts before any function call
// Spawn MDM settings read subprocess
const mdmPromise = startMdmRawRead();

// Spawn Keychain credential prefetch subprocess
const keychainPromise = startKeychainPrefetch();

// Initiate OAuth token prefetch
const oauthPromise = prefetchOAuthTokens();
```

This design leverages Node.js module evaluation behavior -- the moment `import("./main")` is called, these subprocesses are already spawned, and subsequent code can simply `await` the existing Promises.

### Startup Time Measurement

`profileCheckpoint()` calls are distributed throughout the startup flow to precisely measure the duration of each phase:

```typescript
profileCheckpoint("main_start");
// ... Commander parsing ...
profileCheckpoint("args_parsed");
// ... Authentication flow ...
profileCheckpoint("auth_complete");
// ... Tool initialization ...
profileCheckpoint("tools_ready");
```

### Anti-Debugging Protection

In external release builds, Node.js Inspector attachment is blocked:

```typescript
if (isExternalBuild() && isBeingDebugged()) {
  console.error("Debugging is not supported in this build.");
  process.exit(1);
}
```

### Commander.js CLI Configuration

`main.tsx` uses Commander.js to define dozens of command-line options and flags:

```typescript
const program = new Command()
  .name("claude")
  .description("Claude Code CLI")
  .option("-m, --model <model>", "specify model")
  .option("--fast", "use fast model")
  .option("-p, --print", "non-interactive mode, print and exit")
  .option("--resume <sessionId>", "resume session")
  .option("--teleport <url>", "remote session teleport")
  .option("--verbose", "verbose output")
  // ... dozens of other options ...
```

### Model Migration

As Anthropic model versions evolve, the code includes automatic migration logic:

```typescript
// Automatically migrate old model references to new versions
migrateFennecToOpus();        // fennec -> opus
migrateSonnet45ToSonnet46();  // sonnet-4.5 -> sonnet-4.6
// ... more migrations ...
```

### Session Management

Multiple session modes are supported:

```typescript
// Resume existing session
if (options.resume) {
  session = await resumeSession(options.resume);
}
// Remote session teleport
else if (options.teleport) {
  session = await teleportToSession(options.teleport);
}
// Remote session
else if (options.remote) {
  session = await connectRemoteSession(options.remote);
}
// New session
else {
  session = createNewSession();
}
```

### Authentication Flow

Multiple authentication methods are checked by priority:

```typescript
// Authentication check flow (simplified)
const credentials = await resolveCredentials({
  // 1. API Key from environment variable
  apiKey: process.env.ANTHROPIC_API_KEY,
  // 2. OAuth token (prefetched in parallel)
  oauthToken: await oauthPromise,
  // 3. Credentials stored in Keychain (prefetched in parallel)
  keychainCredentials: await keychainPromise,
});

// Subscription check
if (!credentials) {
  await startOAuthFlow();  // Guide user through OAuth login
}
```

### Model and Thinking Configuration

```typescript
// Model selection logic
const model = options.model
  ?? (options.fast ? getFastModel() : getDefaultModel());

// Thinking (extended thinking) configuration
const thinkingConfig = resolveThinkingConfig({
  model,
  budget: options.thinkingBudget,
  disabled: isAblationBaseline,
});
```

### Tool and MCP Initialization

```typescript
// Initialize built-in tools + user-configured MCP servers
const tools = await initializeTools({
  builtinTools: getBuiltinTools(),
  mcpServers: await loadMcpConfig(),
  permissions: await resolvePermissions(),
});
```

### Launching REPL or Headless Mode

The final branch of `main.tsx` -- determines the launch mode based on arguments:

```typescript
if (options.print || stdinHasData) {
  // Headless mode: execute a single query and exit
  await runHeadlessQuery({ query, tools, model, session });
} else {
  // Interactive REPL mode
  await startRepl({ tools, model, session });
}
```

---

## 3. init.ts -- Runtime Initialization

`init.ts` handles runtime initialization logic, executing before REPL or headless mode actually begins:

### Telemetry Initialization

```typescript
// Initialize telemetry system (anonymous usage data collection)
await initTelemetry({
  sessionId: session.id,
  model,
  installId: getInstallId(),
});
```

### Settings Loading

```typescript
// Load user settings (~/.claude/settings.json)
// Load project settings (.claude/settings.json)
// Load MDM managed settings (enterprise configuration, prefetched in parallel)
const settings = await loadSettings({
  mdmSettings: await mdmPromise,
});
```

### Trust Dialog

When running in a new project for the first time, a trust dialog is displayed for user confirmation:

```typescript
// Trust confirmation when running in a project for the first time
if (!isTrustedProject(cwd)) {
  const trusted = await showTrustDialog(cwd);
  if (!trusted) {
    process.exit(0);
  }
}
```

---

## Key Startup Optimizations Summary

### Dynamic Imports Everywhere

Throughout the startup flow, virtually all module loading uses dynamic `import()`, ensuring that only modules needed for the current code path are loaded:

```typescript
// Not this (static import, loaded regardless of whether it's used):
import { startBridge } from "./bridge";

// But this (dynamic import, loaded only when needed):
if (needsBridge) {
  const { startBridge } = await import("./bridge");
}
```

### Parallel Subprocess Prefetching

MDM and Keychain reads are spawned as subprocesses during module evaluation, which means:

```
Timeline:
  0ms   |-- import("./main") triggers module evaluation
        |   |-- spawn MDM read subprocess          -+
        |   |-- spawn Keychain prefetch subprocess  -+-- parallel execution
        |   +-- initiate OAuth token prefetch      -+
 50ms   |-- main() function begins execution
        |-- Commander parses arguments
        |-- ... other initialization ...
150ms   |-- await mdmPromise      <-- subprocess typically done by now
        |-- await keychainPromise <-- zero wait
```

### Fast Paths Skip Heavy Imports

The fast path design in `cli.tsx` ensures that simple operations (like `--version`) can return in milliseconds, completely bypassing the 135ms+ module import chain in `main.tsx`.

### Build-Time Feature Gates

The `feature()` function is evaluated at build time. Code branches for disabled features are completely eliminated by the bundler's tree-shaking and do not appear in the final output:

```typescript
// If feature("DAEMON") evaluates to false at build time,
// the entire if block (including the dynamic import) is removed by tree-shaking
if (feature("DAEMON") && args.includes("--daemon-worker")) {
  const { startDaemonWorker } = await import("./daemon/worker");
  await startDaemonWorker();
  process.exit(0);
}
```

---

## Startup Flow Diagram

```
cli.tsx
  |
  |-- --version -------------> Print version number, exit immediately
  |-- --dump-system-prompt --> Print system prompt, exit
  |-- --claude-in-chrome-mcp > Chrome MCP server
  |-- --chrome-native-host --> Chrome native messaging host
  |-- --computer-use-mcp ----> Computer use MCP (requires CHICAGO_MCP)
  |-- --daemon-worker -------> Daemon worker thread (requires DAEMON)
  |-- remote-control/rc/... -> Bridge mode (requires BRIDGE_MODE)
  |
  +-- no match --> import("./main")
                    |
                    main.tsx
                      |
                      |-- [parallel] MDM prefetch / Keychain prefetch / OAuth prefetch
                      |-- Anti-debugging check
                      |-- Commander.js argument parsing
                      |-- Model migration
                      |-- Session management (new/resume/teleport/remote)
                      |-- Authentication flow
                      |-- Model + Thinking configuration
                      |-- Tool + MCP initialization
                      |
                      |-- init.ts
                      |     |-- Telemetry initialization
                      |     |-- Settings loading
                      |     +-- Trust dialog
                      |
                      +-+-- --print / stdin --> Headless query mode
                        +-- otherwise -------> Interactive REPL
```
