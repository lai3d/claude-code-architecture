# 入口与启动流程

> 基于 Claude Code CLI v2.1.88 源码分析

Claude Code 的启动流程由三个核心文件驱动：`cli.tsx`（引导入口）、`main.tsx`（完整 CLI 初始化）、`init.ts`（运行时初始化）。整体设计哲学是**尽可能延迟加载**——大量使用动态 import 和快速路径，确保不需要的代码永远不会被加载。

---

## 1. cli.tsx — 引导入口

`cli.tsx` 是最外层的入口点，其核心设计原则是：**能立刻返回的就不要加载完整应用**。

### 快速路径（Fast Paths）

在进入完整的 CLI 初始化之前，`cli.tsx` 检查一系列可以立即处理的命令行参数，每个快速路径都避免了加载完整应用所需的 135ms+ 模块导入开销：

```typescript
// --version: 直接打印编译时内联的版本号，零额外导入
if (args.includes("--version") || args.includes("-v")) {
  console.log(MACRO.VERSION);  // 构建时由 bundler 内联
  process.exit(0);
}

// --dump-system-prompt: 仅加载最小配置和模型，打印系统提示词
if (args.includes("--dump-system-prompt")) {
  const { dumpSystemPrompt } = await import("./commands/dump-system-prompt");
  await dumpSystemPrompt();
  process.exit(0);
}
```

完整的快速路径列表：

| 参数 | 用途 | 特点 |
|------|------|------|
| `--version` / `-v` | 打印版本号 | 零导入，直接使用构建时内联的 `MACRO.VERSION` |
| `--dump-system-prompt` | 输出系统提示词 | 仅加载最小配置 + 模型 |
| `--claude-in-chrome-mcp` | 启动 Chrome MCP 服务器 | 独立子系统 |
| `--chrome-native-host` | Chrome 原生消息宿主 | 独立子系统 |
| `--computer-use-mcp` | 计算机操控 MCP 服务器 | 特性门控 `CHICAGO_MCP` |
| `--daemon-worker` | 守护进程工作线程 | 特性门控 `DAEMON` |
| `remote-control`/`rc`/`remote`/`sync`/`bridge` | 桥接模式 | 特性门控 `BRIDGE_MODE` |

### 特性门控（Feature Gates）

Claude Code 使用编译时特性门控来消除死代码。通过 `feature()` 函数在构建时将未启用的特性路径完全移除：

```typescript
// 特性门控的快速路径 —— 构建时若未启用，整个分支被消除
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

### 环境配置

在进入任何代码路径前，`cli.tsx` 先设置必要的环境变量：

```typescript
// 阻止 corepack 在包安装时自动 pin 版本
process.env.COREPACK_ENABLE_AUTO_PIN = "0";

// 远程环境下的 NODE_OPTIONS 配置
if (isRemoteEnv()) {
  process.env.NODE_OPTIONS = buildRemoteNodeOptions();
}
```

### Ablation 基线模式

当 `ABLATION_BASELINE` 特性启用且对应环境变量被设置时，会禁用一系列高级功能（用于 A/B 测试基准对比）：

```typescript
if (feature("ABLATION_BASELINE") && process.env.ABLATION_BASELINE) {
  // 禁用 thinking（扩展思考）
  // 禁用 compact（上下文压缩）
  // 禁用 memory（记忆系统）
  // 禁用 background tasks（后台任务）
}
```

### 进入完整启动

当没有命中任何快速路径时，`cli.tsx` 才通过动态导入加载完整的 CLI：

```typescript
// 只有在所有快速路径都未命中时，才加载完整应用
const { main } = await import("./main");
await main(args);
```

---

## 2. main.tsx — 完整 CLI 初始化

`main.tsx` 是真正的 CLI 核心，约 800+ 行代码，负责完整的应用初始化。

### 启动性能优化：并行预取

在 `main.tsx` 最顶部（甚至在模块级别），就启动了多个并行预取操作。这些操作在模块求值阶段就开始执行，不等待任何其他初始化：

```typescript
// 模块顶层 —— 在任何函数调用之前就启动并行预取
// 启动 MDM 设置读取子进程
const mdmPromise = startMdmRawRead();

// 启动 Keychain 凭证预取子进程
const keychainPromise = startKeychainPrefetch();

// 启动 OAuth token 预取
const oauthPromise = prefetchOAuthTokens();
```

这种设计利用了 Node.js 模块求值的特性——`import("./main")` 的那一刻，这些子进程就已经启动了，后续代码可以直接 `await` 已有的 Promise。

### 启动时间测量

整个启动流程中分布着 `profileCheckpoint()` 调用，用于精确测量各阶段耗时：

```typescript
profileCheckpoint("main_start");
// ... Commander 解析 ...
profileCheckpoint("args_parsed");
// ... 认证流程 ...
profileCheckpoint("auth_complete");
// ... 工具初始化 ...
profileCheckpoint("tools_ready");
```

### 反调试保护

在外部发布版本中，阻止 Node.js Inspector 附加：

```typescript
if (isExternalBuild() && isBeingDebugged()) {
  console.error("Debugging is not supported in this build.");
  process.exit(1);
}
```

### Commander.js CLI 配置

`main.tsx` 使用 Commander.js 定义了数十个命令行选项和标志：

```typescript
const program = new Command()
  .name("claude")
  .description("Claude Code CLI")
  .option("-m, --model <model>", "指定模型")
  .option("--fast", "使用快速模型")
  .option("-p, --print", "非交互模式，输出后退出")
  .option("--resume <sessionId>", "恢复会话")
  .option("--teleport <url>", "远程会话传送")
  .option("--verbose", "详细输出")
  // ... 数十个其他选项 ...
```

### 模型迁移

随着 Anthropic 模型版本更新，代码中包含自动迁移逻辑：

```typescript
// 自动将旧模型引用迁移到新版本
migrateFennecToOpus();        // fennec -> opus
migrateSonnet45ToSonnet46();  // sonnet-4.5 -> sonnet-4.6
// ... 更多迁移 ...
```

### 会话管理

支持多种会话模式：

```typescript
// 恢复已有会话
if (options.resume) {
  session = await resumeSession(options.resume);
}
// 远程会话传送
else if (options.teleport) {
  session = await teleportToSession(options.teleport);
}
// 远程会话
else if (options.remote) {
  session = await connectRemoteSession(options.remote);
}
// 新会话
else {
  session = createNewSession();
}
```

### 认证流程

按优先级检查多种认证方式：

```typescript
// 认证检查流程（简化）
const credentials = await resolveCredentials({
  // 1. 环境变量中的 API Key
  apiKey: process.env.ANTHROPIC_API_KEY,
  // 2. OAuth token（已并行预取）
  oauthToken: await oauthPromise,
  // 3. Keychain 中存储的凭证（已并行预取）
  keychainCredentials: await keychainPromise,
});

// 订阅检查
if (!credentials) {
  await startOAuthFlow();  // 引导用户完成 OAuth 登录
}
```

### 模型与 Thinking 配置

```typescript
// 模型选择逻辑
const model = options.model
  ?? (options.fast ? getFastModel() : getDefaultModel());

// Thinking（扩展思考）配置
const thinkingConfig = resolveThinkingConfig({
  model,
  budget: options.thinkingBudget,
  disabled: isAblationBaseline,
});
```

### 工具与 MCP 初始化

```typescript
// 初始化内置工具 + 用户配置的 MCP 服务器
const tools = await initializeTools({
  builtinTools: getBuiltinTools(),
  mcpServers: await loadMcpConfig(),
  permissions: await resolvePermissions(),
});
```

### 启动 REPL 或 Headless 模式

`main.tsx` 的最终分支——根据参数决定启动方式：

```typescript
if (options.print || stdinHasData) {
  // Headless 模式：执行单次查询后退出
  await runHeadlessQuery({ query, tools, model, session });
} else {
  // 交互式 REPL 模式
  await startRepl({ tools, model, session });
}
```

---

## 3. init.ts — 运行时初始化

`init.ts` 处理应用运行时的初始化逻辑，在 REPL 或 headless 模式真正开始前执行：

### 遥测初始化

```typescript
// 初始化遥测系统（匿名使用数据收集）
await initTelemetry({
  sessionId: session.id,
  model,
  installId: getInstallId(),
});
```

### 设置加载

```typescript
// 加载用户设置（~/.claude/settings.json）
// 加载项目设置（.claude/settings.json）
// 加载 MDM 托管设置（企业级配置，已并行预取）
const settings = await loadSettings({
  mdmSettings: await mdmPromise,
});
```

### 信任对话框

首次在新项目中运行时，显示信任对话框让用户确认：

```typescript
// 首次在项目中运行时的信任确认
if (!isTrustedProject(cwd)) {
  const trusted = await showTrustDialog(cwd);
  if (!trusted) {
    process.exit(0);
  }
}
```

---

## 关键启动优化总结

### 动态导入无处不在

整个启动流程中，几乎所有模块加载都使用动态 `import()`，确保只加载当前代码路径所需的模块：

```typescript
// 不是这样（静态导入，无论是否使用都会加载）：
import { startBridge } from "./bridge";

// 而是这样（动态导入，仅在需要时加载）：
if (needsBridge) {
  const { startBridge } = await import("./bridge");
}
```

### 并行子进程预取

MDM 和 Keychain 的读取通过子进程在模块求值阶段就启动，这意味着：

```
时间线：
  0ms   ├── import("./main") 触发模块求值
        │   ├── spawn MDM 读取子进程        ─┐
        │   ├── spawn Keychain 预取子进程    ─┤── 并行执行
        │   └── 发起 OAuth token 预取       ─┘
 50ms   ├── main() 函数开始执行
        ├── Commander 解析参数
        ├── ... 其他初始化 ...
150ms   ├── await mdmPromise      ← 此时子进程通常已完成
        ├── await keychainPromise ← 零等待
```

### 快速路径跳过大量导入

`cli.tsx` 中的快速路径设计确保简单操作（如 `--version`）能在毫秒级返回，完全跳过 `main.tsx` 中 135ms+ 的模块导入链。

### 构建时特性门控

`feature()` 函数在构建时求值，未启用的特性对应的代码分支会被 bundler 的 tree-shaking 完全消除，不会出现在最终产物中：

```typescript
// 构建时 feature("DAEMON") 如果为 false，
// 整个 if 块（包括动态 import）会被 tree-shaking 移除
if (feature("DAEMON") && args.includes("--daemon-worker")) {
  const { startDaemonWorker } = await import("./daemon/worker");
  await startDaemonWorker();
  process.exit(0);
}
```

---

## 启动流程图

```
cli.tsx
  │
  ├─ --version ──────────────► 打印版本号，立即退出
  ├─ --dump-system-prompt ──► 打印系统提示词，退出
  ├─ --claude-in-chrome-mcp ► Chrome MCP 服务器
  ├─ --chrome-native-host ──► Chrome 原生消息宿主
  ├─ --computer-use-mcp ────► 计算机操控 MCP（需 CHICAGO_MCP）
  ├─ --daemon-worker ───────► 守护进程工作线程（需 DAEMON）
  ├─ remote-control/rc/... ─► 桥接模式（需 BRIDGE_MODE）
  │
  └─ 无匹配 ──► import("./main")
                  │
                  main.tsx
                    │
                    ├── [并行] MDM 预取 / Keychain 预取 / OAuth 预取
                    ├── 反调试检查
                    ├── Commander.js 参数解析
                    ├── 模型迁移
                    ├── 会话管理（新建/恢复/传送/远程）
                    ├── 认证流程
                    ├── 模型 + Thinking 配置
                    ├── 工具 + MCP 初始化
                    │
                    ├── init.ts
                    │     ├── 遥测初始化
                    │     ├── 设置加载
                    │     └── 信任对话框
                    │
                    └─┬─ --print / stdin ──► Headless 查询模式
                      └─ 否则 ────────────► 交互式 REPL
```
