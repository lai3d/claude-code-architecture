# Claude Code CLI v2.1.88 — Architecture Overview

## 1. Project Overview

Claude Code CLI is a terminal application built with **TypeScript + React (Ink)**, using **Bun** as both its runtime and bundler. The source code comprises approximately **~1900 files**, implementing a fully-featured AI conversational programming assistant.

Core technology stack:

| Technology | Purpose |
|------|------|
| TypeScript | Primary language |
| React (Ink) | Terminal UI framework |
| Bun | Runtime / Bundler |
| Commander.js | CLI argument parsing |
| MCP Protocol | Extensible tool system |
| Yoga (Flexbox) | Terminal layout engine |

---

## 2. Architectural Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│                   User Input (Terminal / SDK / MCP)                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 1: Entry Layer                                                │
│  entrypoints/cli.tsx ─── entrypoints/init.ts ─── entrypoints/mcp.ts │
│  entrypoints/sdk/                                                    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 2: CLI Parsing & Session Initialization                       │
│  main.tsx ─── Commander.js arg parsing ─── Session config            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐             │
│  │ REPL Mode     │  │ Headless Mode │  │ SDK / MCP Mode  │             │
│  └──────┬───────┘  └──────┬───────┘  └───────┬────────┘             │
└─────────┼─────────────────┼──────────────────┼──────────────────────┘
          │                 │                  │
          ▼                 ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 3: Session Engine (QueryEngine)                               │
│  QueryEngine.ts ─── Core conversation loop                           │
│  ┌───────────────────────────────────────────────────────────┐      │
│  │  AsyncGenerator streaming                                   │      │
│  │  Message mgmt → API call → Response handling → Tool exec → Loop  │
│  └───────────────────────────────────────────────────────────┘      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
┌──────────────────┐ ┌────────────────┐ ┌────────────────────────────┐
│  API Layer        │ │  Tool Exec Layer│ │  UI Rendering Layer         │
│  query.ts        │ │  Tool.ts       │ │  components/ (200+)        │
│  services/api/   │ │  tools.ts      │ │  screens/REPL.tsx          │
│                  │ │  tools/ (40+)  │ │  ink/ (custom Ink fork)    │
└──────────────────┘ └────────────────┘ └────────────────────────────┘
```

---

## 3. Source Code Directory Structure

```
src/
├── entrypoints/          # Entry points: cli.tsx (bootstrap), init.ts, mcp.ts, sdk/
├── main.tsx              # CLI arg parsing, session setup, launches REPL or headless query
├── QueryEngine.ts        # Core conversation loop class
├── query.ts              # API call logic
├── Tool.ts               # Tool type definitions, ToolUseContext, permission types
├── tools.ts              # Tool registry with conditional loading via feature flags
│
├── tools/                # 40+ tool implementations
│   ├── AgentTool/        #   Sub-agent tool
│   ├── BashTool/         #   Bash command execution
│   ├── FileEditTool/     #   File editing
│   ├── FileReadTool/     #   File reading
│   ├── FileWriteTool/    #   File writing
│   ├── GlobTool/         #   File pattern matching
│   ├── GrepTool/         #   Content search
│   └── ...               #   More tools
│
├── components/           # 200+ React (Ink) UI components
├── screens/              # Screen components: REPL.tsx, Doctor.tsx, ResumeConversation.tsx
├── ink/                  # Heavily customized Ink framework fork (layout engine, Yoga, focus, mouse)
│
├── services/             # Service layer
│   ├── api/              #   API client
│   ├── mcp/              #   MCP protocol service
│   ├── analytics/        #   Analytics tracking
│   ├── oauth/            #   OAuth authentication
│   ├── compact/          #   Conversation compaction
│   ├── plugins/          #   Plugin service
│   ├── policyLimits/     #   Policy limits
│   ├── SessionMemory/    #   Session memory
│   └── extractMemories/  #   Memory extraction
│
├── state/                # Application state management (AppStateStore)
├── commands/             # Slash commands (/help, /clear, etc.)
├── skills/               # Skills system
├── hooks/                # React Hooks
├── coordinator/          # Multi-agent coordination
├── buddy/                # Buddy system
├── vim/                  # Full Vim mode (motions, operators, text objects)
├── voice/                # Voice input
├── remote/               # Remote mode
├── keybindings/          # Keybinding system
├── memdir/               # Persistent memory filesystem
├── constants/            # Prompts, configuration constants
├── types/                # TypeScript type definitions
├── utils/                # Utility functions (auth, config, git, permissions, models, etc.)
├── bootstrap/            # Bootstrap state management
├── context/              # React Context providers
├── migrations/           # Settings/config migrations
├── schemas/              # Validation schemas
├── plugins/              # Plugin system
└── bridge/               # Bridge mode (remote control)
```

---

## 4. Core Modules in Detail

### 4.1 Entry Layer (`entrypoints/`)

The application provides multiple entry points to accommodate different usage scenarios:

- **`cli.tsx`** -- Standard CLI entry point; runs the bootstrap process and initializes the environment
- **`init.ts`** -- Project initialization entry point (`claude init`)
- **`mcp.ts`** -- Entry point for running as an MCP server
- **`sdk/`** -- Programmatic SDK entry point for external integrations

### 4.2 CLI Parsing & Session Setup (`main.tsx`)

`main.tsx` is the convergence point for all entry paths. It is responsible for:

1. Parsing command-line arguments with Commander.js
2. Loading configuration and user settings
3. Establishing the session context
4. Routing to the appropriate run mode:
   - **REPL Mode** -- Interactive terminal, renders `screens/REPL.tsx`
   - **Headless Mode** -- Single-shot query, calls QueryEngine directly
   - **SDK Mode** -- Programmatic interface

### 4.3 Core Conversation Engine (`QueryEngine.ts`)

`QueryEngine` is the heart of the entire application, implementing the core conversation loop logic:

```
User input → Build messages → Call API → Parse response → Execute tools → Return results
    ↑                                                           │
    └──────────────────── Loop ─────────────────────────────────┘
```

Key design decisions:

- **AsyncGenerator pattern** -- Uses async generators to stream results incrementally, enabling real-time UI rendering
- **Tool execution loop** -- When the model returns a `tool_use` block, the tool is automatically executed and the result is fed back to the model
- **Conversation compaction** -- Via `services/compact/`, automatically compresses context when the conversation grows too long

### 4.4 API Layer (`query.ts` + `services/api/`)

Handles communication with the Claude API:

- Builds request parameters (model selection, system prompt, tool definitions)
- Processes streaming responses (SSE)
- Error handling and retries
- Supports multiple API providers

### 4.5 Tool System (`Tool.ts` / `tools.ts` / `tools/`)

The tool system is the most critical capability layer of Claude Code:

```
┌──────────────────────────────────────────────────────┐
│                   tools.ts (Registry)                  │
│       Feature flag conditional loading + MCP dynamic   │
│                     registration                       │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │
│  │ BashTool │ │ EditTool │ │ ReadTool │ │GrepTool│  │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │
│  │ GlobTool │ │WriteTool │ │AgentTool │ │WebFetch│  │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘  │
│  ┌──────────────────┐ ┌──────────────────────────┐   │
│  │ NotebookEditTool │ │ MCP Dynamic Tools (ext.) │   │
│  └──────────────────┘ └──────────────────────────┘   │
│                    ... 40+ tools                      │
└──────────────────────────────────────────────────────┘
```

**Key type definitions (`Tool.ts`)**:

- **`Tool`** -- Tool interface defining name, description, parameter schema, and execution function
- **`ToolUseContext`** -- Tool execution context containing permission info and session state
- **Permission types** -- Three permission modes:
  - `default` -- Dangerous operations require user confirmation
  - `auto` -- All operations are automatically approved
  - `bypass` -- Permission checks are skipped entirely

**Conditional loading**: `tools.ts` uses the `feature()` macro to achieve compile-time dead code elimination -- disabled tools are excluded from the final build artifact.

### 4.6 UI Layer (`components/` + `screens/` + `ink/`)

The UI is built on a React (Ink) architecture with component-based design and Flexbox layout:

- **`ink/`** -- A heavily customized fork of the Ink framework, featuring a custom layout engine, Yoga integration, focus management, and mouse support
- **`screens/`** -- Top-level screen components
  - `REPL.tsx` -- Main interactive interface
  - `Doctor.tsx` -- Diagnostics interface
  - `ResumeConversation.tsx` -- Session resumption interface
- **`components/`** -- 200+ reusable UI components (message display, input fields, tool output, progress indicators, etc.)

### 4.7 State Management (`state/`)

Uses `AppStateStore` for centralized state management, covering:

- Current conversation state
- User configuration
- Tool state
- UI state

---

## 5. Key Architectural Decisions

### 5.1 React (Ink) for Terminal UI

Adopting the React component model for the terminal interface provides:

- **Componentization** -- Complex UIs are decomposed into reusable components
- **Flexbox layout** -- Flexible terminal layout via the Yoga engine
- **Declarative rendering** -- State changes automatically drive UI updates
- **Deep customization** -- The Ink framework was forked to support advanced features like focus management and mouse events

### 5.2 Bun Bundling & `feature()` Compile-Time Optimization

Leveraging Bun's bundling capabilities along with the `feature()` macro enables:

- **Compile-time dead code elimination** -- Disabled feature modules are excluded from the final build
- **Conditional tool loading** -- Feature flags determine which tools are available
- **Reduced bundle size** -- Only the code that is actually needed gets packaged

### 5.3 MCP Protocol for Tool Extension

The Model Context Protocol (MCP) provides extensibility for the tool system:

- Built-in tools are registered through the `tools/` directory
- External tools are dynamically registered via MCP servers
- A unified tool interface and permission model governs both

### 5.4 AsyncGenerator Streaming

QueryEngine uses the AsyncGenerator pattern to:

- Stream conversation results incrementally
- Enable real-time token-by-token UI rendering
- Naturally support interruption and cancellation
- Gracefully handle tool execution loops

### 5.5 Multi-Layer Permission System

Permission control spans the entire tool execution chain:

```
User action → Permission check → Tool execution
                   │
         ┌─────────┼──────────┐
         ▼         ▼          ▼
      default    auto      bypass
    (confirm)  (auto-     (skip
     required)  approve)   check)
```

---

## 6. Data Flow Overview

A typical user interaction follows this flow:

```
1. User input
   │
2. REPL.tsx captures input
   │
3. Check if it's a slash command (/commands)
   │     Yes → handled by commands/
   │
4. QueryEngine.query() starts the conversation loop
   │
5. Build message context (system prompt + history + user message)
   │
6. query.ts → API call (streaming)
   │
7. Parse response
   │
   ├── Text content → streamed to UI for rendering
   │
   └── tool_use → permission check → tool execution → result fed back → return to step 6
   │
8. Conversation turn ends, state updated, awaiting next input
```

---

## 7. Supporting Subsystems

| Subsystem | Directory | Responsibility |
|--------|------|------|
| Multi-agent coordination | `coordinator/` | Manages parallel/sequential collaboration across multiple agents |
| Buddy system | `buddy/` | Auxiliary agent functionality |
| Vim mode | `vim/` | Full Vim editing support (motions, operators, text objects) |
| Voice input | `voice/` | Speech-to-text input |
| Remote mode | `remote/` | Remote connection and execution |
| Bridge mode | `bridge/` | External program remote control interface |
| Keybinding system | `keybindings/` | Customizable key bindings |
| Memory system | `memdir/` + `services/SessionMemory/` | Persistent memory and session memory |
| Plugin system | `plugins/` + `services/plugins/` | Plugin loading and management |
| Skills system | `skills/` | Reusable skill templates |

---

## 8. Summary

The architecture of Claude Code CLI is centered on the **conversation loop** as its core, using a layered design to cleanly separate entry points, parsing, the engine, tools, and the UI. Adopting React (Ink) brings modern frontend componentization to the terminal UI, while the MCP protocol and plugin system ensure the tool layer remains extensible. Two key design patterns run throughout the entire system: AsyncGenerator streaming -- which ensures a real-time user experience -- and the multi-layer permission system -- which ensures operational safety.
