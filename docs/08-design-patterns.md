# Claude Code CLI v2.1.88 — 设计模式分析

本文档从源码中提炼出 14 个值得学习的设计模式，涵盖性能优化、架构解耦、状态管理等多个维度。

---

## 1. AsyncGenerator 流式模式

`QueryEngine.submitMessage()` 的返回类型为 `AsyncGenerator<SDKMessage, void, unknown>`，这是整个系统最核心的数据流动机制。

### 为什么用 AsyncGenerator 而非回调或 Observable？

```typescript
class QueryEngine {
  async *submitMessage(
    userMessage: string,
    options: SubmitOptions
  ): AsyncGenerator<SDKMessage, void, unknown> {
    // 调用 Claude API，流式返回结果
    for await (const event of apiStream) {
      // 处理每个事件
      const message = transformEvent(event);
      yield message;  // 消费者可以逐条接收

      // 检查是否被取消
      if (options.abortController.signal.aborted) {
        return; // 干净地退出
      }
    }
  }
}
```

### 消费端代码

```typescript
// UI 层消费
for await (const message of engine.submitMessage(input, opts)) {
  renderToTerminal(message);
}

// SDK 层消费
for await (const message of engine.submitMessage(input, opts)) {
  sendToClient(JSON.stringify(message));
}
```

### 三大优势

| 特性 | 说明 |
|------|------|
| **流式输出** | `yield` 逐条推送，UI 和 SDK 消费者实时接收 |
| **天然背压** | 消费者未调用 `next()` 时生产者自动暂停，无需额外流控 |
| **干净取消** | 通过 `AbortController` 信号退出，无残留的 listener 或 subscription |

这种模式的精妙之处在于：同一个 Generator 同时服务终端 UI（React/Ink 渲染）和 SDK 消费者（JSON 流输出），不需要适配层。

---

## 2. 编译期 Feature Flag 模式

Claude Code 使用 Bun 打包器的 `feature()` 函数实现编译期条件编译，区分内部（Anthropic）和外部用户的构建。

### 核心模式

```typescript
// 编译期求值，不是运行时判断
import { feature } from 'bun:bundle';

// 编译为 true 或 false 的字面量
const InternalTool = feature('INTERNAL')
  ? require('./internal/InternalTool.js').InternalTool
  : null;

// Bun 打包器看到 feature('INTERNAL') === false 时，
// 整个 require('./internal/InternalTool.js') 被死代码消除
```

### 条件 require 实现 Tree-shaking

```typescript
// 外部构建编译后等价于：
const InternalTool = null;
// require('./internal/InternalTool.js') 这行代码根本不存在于产物中

// 内部构建编译后等价于：
const InternalTool = require('./internal/InternalTool.js').InternalTool;
```

### 设计要点

| 方面 | 说明 |
|------|------|
| **零运行时成本** | 编译器直接替换为常量，无 `if` 分支 |
| **代码不泄露** | 外部构建物理上不包含内部功能代码 |
| **类型安全** | 使用 `null` 而非 `undefined`，消费者必须做空检查 |
| **多构建目标** | 同一份源码生成 internal (ant) 和 external 两种产物 |

这比运行时 `process.env.FEATURE_FLAG` 更强——不仅是功能开关，更是物理级别的代码隔离。

---

## 3. 权限分层系统

Claude Code 的权限系统是一个多层决策模型，确保 AI 工具调用在安全边界内执行。

### 分层结构

```
┌─────────────────────────────────────┐
│  Permission Mode (全局策略)          │
│  default | auto | bypass            │
├─────────────────────────────────────┤
│  Per-Tool Rules (工具级规则)         │
│  alwaysAllow | alwaysDeny | alwaysAsk│
├─────────────────────────────────────┤
│  Rule Source (规则来源)              │
│  ToolPermissionRulesBySource        │
│  - user config                      │
│  - project config                   │
│  - enterprise policy                │
├─────────────────────────────────────┤
│  Runtime Context (运行时上下文)      │
│  DeepImmutable<PermissionContext>    │
└─────────────────────────────────────┘
```

### 权限规则按来源组织

```typescript
type ToolPermissionRulesBySource = {
  user: ToolPermissionRule[];        // 用户个人配置
  project: ToolPermissionRule[];     // 项目级 .claude 配置
  enterprise: ToolPermissionRule[];  // 企业策略
};

type ToolPermissionRule = {
  tool: string;                      // 工具名或通配符
  permission: 'alwaysAllow' | 'alwaysDeny' | 'alwaysAsk';
  pattern?: string;                  // 可选的路径/参数匹配
};
```

### DeepImmutable 保护权限上下文

```typescript
// 递归只读包装，确保权限上下文不被意外修改
type DeepImmutable<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepImmutable<T[K]>
    : Readonly<T[K]>;
};

// 传递给工具的上下文是不可变的
function executeToolCall(
  context: DeepImmutable<PermissionContext>,
  tool: Tool,
  input: unknown
) { /* ... */ }
```

### 可注入的权限判断函数

```typescript
// canUseTool 作为可注入函数，方便测试
type CanUseTool = (
  tool: Tool,
  input: unknown,
  context: PermissionContext
) => Promise<PermissionResult>;

// 测试中可以替换为 mock
const mockCanUseTool: CanUseTool = async () => ({ allowed: true });
```

这种设计将"策略定义"与"策略执行"分离，允许不同来源的规则叠加，同时保证运行时不可变性。

---

## 4. 循环依赖处理

大型 TypeScript 项目中循环依赖几乎不可避免。Claude Code 使用了多种模式来优雅地打破循环。

### 模式一：惰性 require 函数

```typescript
// 不在模块顶层 require，而是包装成函数
const getQueryEngine = () => require('./QueryEngine.js').QueryEngine;

// 使用时才触发加载，此时依赖图已经完成初始化
function createSession() {
  const QueryEngine = getQueryEngine();
  return new QueryEngine(/* ... */);
}
```

### 模式二：调用点动态 import

```typescript
async function handleSpecialCase() {
  // 只在需要时加载，完全跳过模块初始化阶段的循环
  const { SpecialHandler } = await import('./SpecialHandler.js');
  return new SpecialHandler();
}
```

### 模式三：类型-only 导入

```typescript
// 只导入类型，不产生运行时依赖
import type { QueryEngine } from './QueryEngine.js';

// 运行时通过参数传入实例
function processResult(engine: QueryEngine, result: unknown) {
  // engine 是从调用者传入的，不是本模块 import 的
}
```

### 对比与选择

| 模式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| 惰性 require | 同步场景、频繁调用 | 简单直接、缓存生效 | 延迟加载可能影响首次性能 |
| 动态 import | 异步场景、低频路径 | 最彻底的解耦 | 需要 async/await |
| 类型-only 导入 | 只需类型不需值 | 零运行时开销 | 不能用于实例化 |

关键原则：**在模块顶层只建立类型关系，运行时依赖延迟到实际调用时建立**。

---

## 5. 并行预取模式

Claude Code 的启动路径展示了一个精巧的性能优化——在必须等待的时间窗口内并行启动异步操作。

### 核心思路

```typescript
// ===== 启动阶段 1: 发射异步操作（不等待） =====
startMdmRawRead();          // 副作用：启动子进程读取 MDM 配置
startKeychainPrefetch();    // 副作用：启动 macOS Keychain 读取

// ===== 启动阶段 2: 同步初始化（~135ms 的 import 和解析） =====
import { ConfigManager } from './config/ConfigManager.js';
import { ToolRegistry } from './tools/ToolRegistry.js';
import { UIRenderer } from './ui/UIRenderer.js';
// ... 大量 import 在这里发生 ...

// ===== 启动阶段 3: 收集预取结果 =====
await ensureKeychainPrefetchCompleted();
// 此时子进程大概率已经完成，await 几乎零等待
```

### 时间线对比

```
传统顺序执行:
  [Keychain 80ms] → [MDM 40ms] → [imports 135ms] → [init]
  总计: ~255ms

并行预取:
  [Keychain 80ms.............]
  [MDM 40ms.....]
  [imports 135ms.........................] → [await 0ms] → [init]
  总计: ~135ms
```

### 实现要点

```typescript
let keychainPromise: Promise<KeychainData> | null = null;

// 启动预取——只是保存 Promise 引用
function startKeychainPrefetch(): void {
  keychainPromise = readKeychain(); // 立即返回 Promise
}

// 等待完成——通常瞬间完成
async function ensureKeychainPrefetchCompleted(): Promise<KeychainData> {
  if (!keychainPromise) {
    throw new Error('Prefetch not started');
  }
  return keychainPromise; // await 已经 resolved 的 Promise
}
```

这个模式的关键洞察是：**import 语句虽然是同步的，但执行期间 event loop 仍在运转，此前启动的异步操作可以在这段时间内完成**。

---

## 6. 快速路径模式

对于不需要完整功能的简单命令，通过短路优化跳过昂贵的初始化流程。

### 模式实现

```typescript
// entrypoints/cli.tsx — 主入口
async function main() {
  const args = process.argv.slice(2);

  // ===== 快速路径：不需要加载任何模块 =====
  if (args.includes('--version')) {
    console.log(VERSION);  // VERSION 是内联常量
    return;
  }

  if (args.includes('--help')) {
    printHelp();  // 纯字符串输出
    return;
  }

  // ===== 快速路径：只需要部分模块 =====
  if (args.includes('--resume')) {
    const { resumeSession } = await import('./session/resume.js');
    return resumeSession(args);
  }

  // ===== 完整路径：加载全部模块 =====
  const { ConfigManager } = await import('./config/ConfigManager.js');
  const { ToolRegistry } = await import('./tools/ToolRegistry.js');
  const { UIRenderer } = await import('./ui/UIRenderer.js');
  // ... 完整初始化 ...
}
```

### 设计原则

```
命令频率       初始化成本       策略
─────────────────────────────────────────
--version      ⬛ 极低          process.argv 直接检查
--help         ⬛ 极低          字符串输出，无依赖
--resume       ⬛⬛ 中等         动态 import 部分模块
正常启动       ⬛⬛⬛⬛ 完整     加载全部模块和配置
```

核心理念：**在检查完参数之前不加载任何不必要的模块**。动态 `import()` 是实现这一点的关键工具——它让模块加载变成了按需行为而非声明时行为。

---

## 7. Migration 模式

随着产品迭代，配置格式和模型名称不断变化。Claude Code 实现了一套有序的迁移系统来处理这种演化。

### 迁移函数设计

```typescript
// 每个迁移是一个具名函数，名称即文档
const migrations = [
  migrateFennecToOpus,
  migrateSonnet45ToSonnet46,
  migrateOldPermissionFormat,
  migrateToolAliases,
];

// 迁移函数签名：接收配置，返回可能修改的配置
type MigrationFn = (config: UserConfig) => UserConfig;
```

### 启动时顺序执行

```typescript
function runMigrations(config: UserConfig): UserConfig {
  let current = config;
  for (const migration of migrations) {
    current = migration(current);
  }
  return current;
}
```

### 幂等性保证

```typescript
function migrateSonnet45ToSonnet46(config: UserConfig): UserConfig {
  // 幂等：只在旧值存在时才修改
  if (config.defaultModel === 'claude-sonnet-4-5') {
    return {
      ...config,
      defaultModel: 'claude-sonnet-4-6',
    };
  }
  // 已迁移或无需迁移，原样返回
  return config;
}

function migrateOldPermissionFormat(config: UserConfig): UserConfig {
  // 幂等：检查旧格式是否存在
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

### 设计要点

| 特性 | 说明 |
|------|------|
| **具名函数** | `migrateFennecToOpus` 比 `migration_001` 更具自解释性 |
| **顺序执行** | 迁移之间可能有依赖关系，顺序保证正确性 |
| **幂等安全** | 重复运行不会产生副作用，断电重启后安全 |
| **纯函数** | 输入配置 → 输出配置，无外部状态依赖 |

这个模式适用于任何需要演化配置 schema 的长期维护项目。

---

## 8. MCP 可扩展性模式

Model Context Protocol (MCP) 是 Claude Code 的开放式工具扩展机制，允许第三方通过标准协议提供工具。

### 生命周期管理

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  发现     │ ──→ │  审批     │ ──→ │  执行     │ ──→ │  结果     │
│ Discovery │     │ Approval │     │ Execute  │     │ Result   │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
     │                │                │                │
  扫描配置         权限系统          协议通信          结果验证
  列出工具        企业策略过滤       JSON-RPC         类型转换
```

### 服务器配置与发现

```typescript
// .claude/settings.json 中声明 MCP 服务器
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
      "allowedTools": ["query", "schema"],  // 工具白名单
      "deniedTools": ["drop_table"]         // 工具黑名单
    }
  }
}
```

### 企业策略过滤

```typescript
// 企业管理员可以限制允许的 MCP 服务器和工具
interface EnterprisePolicy {
  allowedMcpServers: string[];           // 白名单服务器
  blockedMcpTools: string[];             // 全局禁用的工具
  requireServerSignature: boolean;       // 是否需要签名验证
}

// 过滤流程
function filterMcpTools(
  tools: McpTool[],
  policy: EnterprisePolicy
): McpTool[] {
  return tools.filter(tool =>
    !policy.blockedMcpTools.includes(tool.name)
  );
}
```

### 服务器签名验证

```typescript
// 验证 MCP 服务器的完整性
interface ServerSignature {
  hash: string;          // 服务器代码哈希
  signedBy: string;      // 签名者
  timestamp: number;     // 签名时间
}

// 企业环境要求验证签名后才允许连接
async function verifyServer(
  server: McpServerConfig
): Promise<boolean> {
  if (!enterprisePolicy.requireServerSignature) return true;
  const signature = await fetchServerSignature(server);
  return verifySignature(signature, trustedKeys);
}
```

MCP 的精妙之处在于：**用协议代替插件 API**。任何语言、任何进程都可以成为工具提供者，只要它实现了 JSON-RPC 协议。

---

## 9. 状态管理模式

Claude Code 采用类 Redux 的函数式状态管理，但更加轻量。

### 核心 API

```typescript
// 获取当前状态（只读快照）
const getAppState: () => AppState;

// 更新状态（函数式更新，类似 React 的 setState）
const setAppState: (updater: (prev: AppState) => AppState) => void;
```

### 函数式更新保证原子性

```typescript
// 正确：基于 prev 更新，避免竞态
setAppState(prev => ({
  ...prev,
  turnCount: prev.turnCount + 1,
  lastActivity: Date.now(),
}));

// 错误：读取后再设置，可能覆盖其他更新
const state = getAppState();
setAppState(_ => ({
  ...state,  // 这里的 state 可能已过时！
  turnCount: state.turnCount + 1,
}));
```

### createStore 抽象

```typescript
function createStore<T>(initialState: T) {
  let state = initialState;

  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      state = updater(state);
    },
  };
}

// 使用
const appStore = createStore<AppState>({
  turnCount: 0,
  messages: [],
  isStreaming: false,
  // ...
});
```

### 设计选择

为什么没有用 Redux、MobX 或 Zustand？

| 选择 | 理由 |
|------|------|
| **无中间件** | CLI 工具不需要 devtools 或 time-travel |
| **无订阅机制** | React/Ink 自身管理渲染周期 |
| **无 action 类型** | 函数式更新自带语义，无需额外分类 |
| **极简实现** | 几十行代码，零依赖 |

这种"刚好够用"的状态管理非常适合非 Web 的 TypeScript 项目。

---

## 10. React 终端 UI 模式

Claude Code 使用 React + Ink 将组件化 UI 开发模式带入终端。

### 组件结构

```typescript
import React from 'react';
import { Box, Text } from 'ink';

// 终端组件和 Web 组件写法一致
function MessageBubble({ role, content }: MessageProps) {
  return (
    <Box flexDirection="column" borderStyle="round" paddingX={1}>
      <Text bold color={role === 'assistant' ? 'blue' : 'green'}>
        {role === 'assistant' ? '🤖 Claude' : '👤 You'}
      </Text>
      <Text>{content}</Text>
    </Box>
  );
}
```

### Flexbox 布局（Yoga 引擎）

```typescript
function AppLayout() {
  return (
    <Box flexDirection="column" height="100%">
      {/* 消息区域：自动填充剩余空间 */}
      <Box flexGrow={1} flexDirection="column" overflow="hidden">
        <MessageList />
      </Box>

      {/* 状态栏：固定高度 */}
      <Box height={1} borderStyle="single" borderTop borderBottom={false}>
        <StatusBar />
      </Box>

      {/* 输入区域：固定在底部 */}
      <Box>
        <InputArea />
      </Box>
    </Box>
  );
}
```

### 终端专用 Hooks

```typescript
// 自定义 Hook：监听终端尺寸变化
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

// 焦点管理
function InputArea() {
  const { isFocused } = useFocus();
  return (
    <Box borderStyle={isFocused ? 'bold' : 'single'}>
      <TextInput
        placeholder={isFocused ? '输入消息...' : '按 Tab 切换焦点'}
      />
    </Box>
  );
}
```

### Virtual DOM 的终端优势

React 的 Virtual DOM diffing 在终端中同样有价值——只更新变化的字符区域，避免全屏重绘导致的闪烁。这对于流式输出（逐字显示 AI 回复）尤其关键。

---

## 11. 文件状态缓存

FileStateCache 模式用于避免在同一个对话轮次中重复读取相同文件。

### 设计模式

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

### 缓存克隆实现隔离

```typescript
// 每个上下文（tool call）拿到独立的缓存副本
function cloneCache(source: FileStateCache): FileStateCache {
  return {
    files: new Map(source.files),  // 浅克隆 Map
  };
}

// 工具 A 的文件修改不会污染工具 B 的缓存视图
const cacheForToolA = cloneCache(masterCache);
const cacheForToolB = cloneCache(masterCache);
```

### 读取路径

```typescript
async function readFileWithCache(
  path: string,
  cache: FileStateCache
): Promise<string> {
  const cached = cache.files.get(path);

  if (cached) {
    // 检查文件是否被外部修改
    const currentHash = await quickHash(path);
    if (currentHash === cached.hash) {
      return cached.content;  // 缓存命中
    }
  }

  // 缓存未命中或已过期
  const content = await fs.readFile(path, 'utf-8');
  cache.files.set(path, {
    content,
    hash: await quickHash(path),
    lastRead: Date.now(),
  });
  return content;
}
```

核心目的：**在一个对话轮次中，AI 可能多次读取同一个文件（先读取、再编辑、再验证），缓存消除了冗余的文件系统操作**。

---

## 12. 会话持久化

Claude Code 支持会话的保存、恢复和跨环境传输。

### Transcript 记录

```typescript
interface SessionTranscript {
  sessionId: string;
  startTime: number;
  messages: TranscriptMessage[];
  metadata: {
    model: string;
    totalTokens: number;
    totalCost: number;
  };
}

// 每条消息都被记录
interface TranscriptMessage {
  role: 'user' | 'assistant' | 'tool';
  content: unknown;
  timestamp: number;
  tokenUsage?: TokenUsage;
}
```

### Resume 恢复机制

```typescript
// --resume 标志从上次会话继续
async function resumeSession(sessionId: string): Promise<void> {
  // 1. 加载历史 transcript
  const transcript = await loadTranscript(sessionId);

  // 2. 重建消息历史
  const messages = transcript.messages.map(toAPIMessage);

  // 3. 创建新的 QueryEngine，注入历史
  const engine = new QueryEngine({
    initialMessages: messages,
    // ...
  });

  // 4. 等待用户新输入
  await startInteractiveLoop(engine);
}
```

### Teleport 跨环境传输

```typescript
// 将会话从一个环境"传送"到另一个
interface TeleportPayload {
  transcript: SessionTranscript;
  fileContext: FileContextSnapshot;  // 相关文件快照
  workingDirectory: string;
}

// 序列化 → 传输 → 反序列化 → 恢复
async function teleportSession(
  payload: TeleportPayload,
  targetEnv: string
): Promise<void> {
  const serialized = JSON.stringify(payload);
  await transmitToEnvironment(serialized, targetEnv);
}
```

会话持久化让 AI 对话不再是一次性的——**你可以中断、恢复、甚至跨机器接续之前的工作上下文**。

---

## 13. 成本控制模式

Claude Code 内置了多维度的预算管理系统，防止 API 调用费用失控。

### 预算配置

```typescript
interface BudgetConfig {
  maxBudgetUsd: number;    // 硬性美元上限
  maxTurns: number;        // 最大对话轮次
  taskBudget: number;      // 单任务预算（子任务场景）
}
```

### 逐次追踪

```typescript
interface UsageTracker {
  totalInputTokens: number;
  totalOutputTokens: number;
  totalCostUsd: number;
  turnsUsed: number;
}

function trackUsage(tracker: UsageTracker, apiResponse: APIResponse): void {
  tracker.totalInputTokens += apiResponse.usage.input_tokens;
  tracker.totalOutputTokens += apiResponse.usage.output_tokens;
  tracker.totalCostUsd += calculateCost(apiResponse.usage);
  tracker.turnsUsed += 1;
}
```

### 预算检查点

```typescript
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

### Prompt Cache 中断检测

```typescript
// Prompt cache 中断会导致成本突增（需要重新处理完整上下文）
function detectCacheBreak(
  currentUsage: TokenUsage,
  expectedCachedTokens: number
): boolean {
  const cacheHitRate = currentUsage.cache_read_input_tokens
    / expectedCachedTokens;

  // 如果缓存命中率骤降，说明缓存被破坏
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

成本控制模式的意义：**当 AI agent 自主执行多步操作时，没有预算控制就等于开着水龙头不关**。

---

## 14. 错误分类模式

`categorizeRetryableAPIError` 将 API 错误分为可重试和不可重试两类，驱动不同的处理策略。

### 错误分类

```typescript
type ErrorCategory =
  | { retryable: true;  strategy: RetryStrategy; userMessage: string }
  | { retryable: false; fatal: boolean;          userMessage: string };

type RetryStrategy = {
  maxRetries: number;
  backoffMs: number[];     // 每次重试的等待时间
  jitter: boolean;         // 是否添加随机抖动
};

function categorizeRetryableAPIError(error: APIError): ErrorCategory {
  // 429: 速率限制 — 可重试，指数退避
  if (error.status === 429) {
    return {
      retryable: true,
      strategy: {
        maxRetries: 5,
        backoffMs: [1000, 2000, 4000, 8000, 16000],
        jitter: true,
      },
      userMessage: 'API 速率受限，正在等待重试...',
    };
  }

  // 500/502/503: 服务器错误 — 可重试，较短退避
  if (error.status >= 500 && error.status < 600) {
    return {
      retryable: true,
      strategy: {
        maxRetries: 3,
        backoffMs: [500, 1000, 2000],
        jitter: true,
      },
      userMessage: 'API 服务暂时不可用，正在重试...',
    };
  }

  // 401: 认证失败 — 不可重试，致命
  if (error.status === 401) {
    return {
      retryable: false,
      fatal: true,
      userMessage: 'API 密钥无效或已过期，请检查认证信息。',
    };
  }

  // 400: 请求错误 — 不可重试，非致命
  if (error.status === 400) {
    return {
      retryable: false,
      fatal: false,
      userMessage: '请求格式错误，请检查输入。',
    };
  }

  // 未知错误 — 保守处理，不重试
  return {
    retryable: false,
    fatal: false,
    userMessage: `未预期的 API 错误 (${error.status})`,
  };
}
```

### 重试执行器

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
        delay += Math.random() * delay * 0.3; // 30% 随机抖动
      }
      await sleep(delay);
    }
  }
  throw new Error('unreachable');
}
```

错误分类模式将"如何判断错误"与"如何处理错误"解耦。添加新的错误类型只需在 `categorizeRetryableAPIError` 中增加一个分支。

---

## 总结：值得借鉴的工程实践

Claude Code 源码中体现的工程实践可以归纳为以下几条核心原则：

### 1. 编译期 > 运行时

Feature flag 的编译期消除、类型-only 导入的零开销、Bun 打包器的深度利用——**能在编译期解决的问题绝不留到运行时**。

### 2. 延迟一切非必要操作

并行预取、快速路径、惰性 require、动态 import——**启动时只做必须做的事，其余按需加载**。对于 CLI 工具来说，冷启动速度直接影响用户体验。

### 3. 不可变性作为安全边界

`DeepImmutable<>` 保护权限上下文、函数式 `setState` 保证状态更新原子性、缓存克隆实现上下文隔离——**不可变性不只是代码风格，而是防御性编程的核心手段**。

### 4. 协议优于接口

MCP 用标准协议（JSON-RPC）代替编程接口（plugin API），AsyncGenerator 用语言原语代替自定义事件系统——**协议级别的设计比 API 级别的设计更灵活、更长寿**。

### 5. 显式分类驱动行为

错误分类（retryable vs fatal）、权限分层（mode → rule → source）、成本分类（dollar vs turn vs task）——**先分类，再根据分类决定行为**。这消除了 `if-else` 链条，让逻辑可审计、可扩展。

### 6. 幂等性贯穿始终

迁移函数幂等、缓存读取幂等、权限检查无副作用——**在不确定代码会被执行几次的环境中（CLI 重启、网络重试、用户重复操作），幂等性是正确性的基础**。

### 7. 刚好够用的抽象

状态管理没用 Redux，UI 框架用 React 但只用了最小子集，工具系统只抽象了注册和执行两步——**过度抽象的代价是认知负担和调试困难，刚好够用才是最优解**。

---

这些模式不是 Claude Code 独有的发明，但它们在一个真实的大型 TypeScript CLI 项目中的组合应用方式，展示了如何在性能、可维护性和安全性之间取得平衡。每一个模式都解决了具体的工程问题，而非为了"架构正确"而存在。
