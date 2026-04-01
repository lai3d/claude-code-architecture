# Claude Code CLI Architecture Analysis

> Deep dive into the architecture of [Claude Code](https://claude.ai/code) v2.1.88 — Anthropic's official CLI for Claude.

This repo documents the internal architecture, design patterns, and engineering decisions behind Claude Code CLI. The goal is to learn from a well-engineered, production-grade CLI tool built with TypeScript, React (Ink), and Bun.

## Table of Contents

| # | Document | Description |
|---|----------|-------------|
| 1 | [Architecture Overview](docs/01-architecture-overview.md) | High-level system architecture and module map |
| 2 | [Entry & Startup](docs/02-entry-and-startup.md) | Boot sequence, fast paths, and startup optimizations |
| 3 | [Query Engine & Conversation Loop](docs/03-query-engine.md) | Core conversation loop, message management, context handling |
| 4 | [Tool System](docs/04-tool-system.md) | Tool abstraction, 40+ built-in tools, permission model |
| 5 | [UI Layer (Ink)](docs/05-ui-layer.md) | Custom Ink fork, 200+ React components, terminal rendering |
| 6 | [Services Layer](docs/06-services-layer.md) | MCP, OAuth, compact, analytics, plugin system |
| 7 | [Performance & Build](docs/07-performance-and-build.md) | Startup optimization, feature flags, dead code elimination |
| 8 | [Key Design Patterns](docs/08-design-patterns.md) | Patterns worth adopting in your own projects |

## Tech Stack

- **Language**: TypeScript
- **Runtime**: Node.js >= 18, built with Bun
- **UI**: [Ink](https://github.com/vadimdemedes/ink) (React for terminal) — heavily customized fork
- **CLI Framework**: Commander.js
- **Layout**: Yoga (Flexbox for terminal)
- **Protocol**: MCP (Model Context Protocol) for tool extensibility

## Source Stats

- **~1900 source files** across 50+ modules
- **40+ built-in tools** (Bash, FileEdit, Agent, WebFetch, etc.)
- **200+ React components** for terminal UI
- **Full Vim mode** with motions, operators, and text objects
- **Voice input** support
- **Multi-agent coordination** (coordinator mode, agent swarms)

## Why This Analysis?

Claude Code is one of the most sophisticated terminal applications ever built. It combines:
- A production AI agent loop with tool use and permission control
- A rich terminal UI rivaling desktop apps
- Enterprise-grade auth, policy, and remote management
- Extreme startup performance optimization

There's a lot to learn here regardless of whether you're building AI tools, CLI apps, or complex TypeScript projects.

## License

This analysis is for educational purposes. Claude Code is a product of [Anthropic](https://anthropic.com).
