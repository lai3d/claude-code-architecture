# 08 - 设计模式精粹

> 基于 Claude Code CLI v2.1.88 源码分析，提炼值得借鉴的设计模式。

---

## 1. AsyncGenerator 流式模式

Claude Code 的核心对话循环使用 `AsyncGenerator` 实现流式输出，而非回调或 Observable。

```typescript
// 核心签名
class QueryEngine {
  async *submitMessage(
    userMessage: string,
    options: SubmitOptions
  ): AsyncGenerator<SDKMessage, void, unknown> {
    // 调用 Claude API，流式返回结果
    for await (const event of apiStream) {
      const message = transformEvent(event);
      yield message;  // 消费者逐条接收

      // 通过 AbortController 实现取消
      if (options.abortController.signal.aborted) {
        return; // 干净退出
      }
    }
  }
}
```

### 消费端

```typescript
// UI 层消费
for await (const message of engine.submitMessage(input, opts)) {
  renderToTerminal(message);
}

// SDK 层消费——同一个 Generator 两种场景无需适配
for await (const message of engine.submitMessage(input, opts)) {
  sendToClient(JSON.stringify(message));
}
```

### 为什么选 AsyncGenerator

| 对比维度 | Callback | Observable | AsyncGenerator |
|---------|----------|------------|----------------|
| 背压 | 无 | 需手动 | 天然具备 |
| 取消 | 复杂 | unsubscribe | AbortController + return() |
| 错误处理 | try/catch 分散 | onError | 统一 try/catch |
| 组合性 | 差 | 优秀 | 良好（for-await） |

### 取消机制

```typescript
const controller = new AbortController();

for await (const event of submitMessage(prompt, controller.signal)) {
  ui.render(event);
  if (userPressedEscape) {
    controller.abort(); // 触发 AbortError，generator 清理退出
    break;
  }
}
```

**实践要点：**
- `yield` 天然是暂停点，下游消费慢则上游自动等待，无需额外限流
- `AbortController.signal` 同时传给 HTTP 客户端和 generator，一次取消贯穿全链路
- `finally` 块中做资源清理，无论正常结束还是异常中断都能执行

---

## 2. 编译期 Feature Flag 模式

Claude Code 使用 Bun 打包器的 `feature()` 函数实现编译期条件编译，区分内部（Anthropic）和外部用户的构建。

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

### 编译产物对比

```typescript
// 外部构建编译后等价于：
const InternalTool = null;
// require('./internal/InternalTool.js') 这行代码根本不存在于产物中

// 内部构建编译后等价于：
const InternalTool = require('./internal/InternalTool.js').InternalTool;
```

### 构建配置

```typescript
// build.ts
Bun.build({
  define: {
    "feature.internal": isInternalBuild ? "true" : "false",
  },
  // Bun 在 tree-shaking 阶段移除不可达分支
});
```

**与运行时 flag 的区别：**

| 方面 | 编译期 flag | 运行时 flag |
|------|-----------|------------|
| 代码体积 | 更小（DCE 移除不可达代码） | 全部保留 |
| 运行时开销 | 零 | 每次判断 |
| 灵活性 | 需重新构建 | 动态切换 |
| 安全性 | 物理隔离，代码不泄露 | 代码仍在产物中 |
| 适用场景 | internal/external 区分 | A/B 测试、灰度发布 |

**实践要点：**
- 敏感功能（内部调试工具、实验性 API）用编译期 flag，确保外部产物零泄漏
- 本质是常量折叠 + DCE，与 Webpack 的 `DefinePlugin` 原理一致
- 使用 `null` 而非 `undefined` 作为占位值，消费者必须做空检查

---

## 3. 权限分层系统

Claude Code 的权限系统由多层组成：模式 -> 工具规则 -> 运行时检查。

### 分层结构

```
+-----------------------------------------+
|  Permission Mode (全局策略)              |
|  default | plan | autoApprove | bypass  |
+-----------------------------------------+
|  Per-Tool Rules (工具级规则)             |
|  alwaysAllow | alwaysDeny | alwaysAsk   |
+-----------------------------------------+
|  Rule Source (规则来源)                  |
|  user config | project config | MDM     |
+-----------------------------------------+
|  Runtime Context (运行时上下文)          |
|  DeepImmutable<PermissionContext>        |
+-----------------------------------------+
```

### 权限规则按来源组织

```typescript
type ToolPermissionRulesBySource = {
  user: ToolPermissionRule[];        // 用户个人配置
  project: ToolPermissionRule[];     // 项目级 .claude 配置
  enterprise: ToolPermissionRule[];  // 企业 MDM 策略
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

### 可注入的 canUseTool

```typescript
// canUseTool 作为可注入函数，方便测试和定制
type CanUseTool = (
  tool: Tool,
  input: unknown,
  context: PermissionContext
) => Promise<PermissionResult>;

async function canUseTool(
  tool: Tool, input: unknown, context: PermissionContext
): Promise<PermissionResult> {
  // 1. 检查全局模式
  if (context.mode === "bypassPermissions") return { allowed: true };
  if (context.mode === "plan") return { allowed: false, reason: "read_only" };

  // 2. 匹配工具级规则（优先级：enterprise > project > user）
  const rule = matchRule(tool.name, input, context.rules);
  if (rule) return { allowed: rule.permission === "alwaysAllow" };

  // 3. 默认需要用户确认
  return { allowed: false, reason: "requires_approval" };
}

// 测试中可以替换为 mock
const mockCanUseTool: CanUseTool = async () => ({ allowed: true });
```

**实践要点：**
- 分层设计使权限既可粗粒度控制（模式），也可细粒度调整（工具规则）
- 多来源规则有明确优先级，企业策略 > 项目配置 > 用户偏好
- `DeepImmutable` 在类型层面防御，零运行时成本
- `canUseTool` 可注入意味着测试时可以 mock，不依赖真实配置文件

---

## 4. 循环依赖处理

大型 TypeScript 项目中循环依赖几乎不可避免。Claude Code 使用三种策略打破循环。

### 策略一：惰性 require()

```typescript
// 不在模块顶层 require，而是推迟到函数调用时
const getQueryEngine = () => require('./QueryEngine.js').QueryEngine;

function createSession() {
  // 此时 QueryEngine 模块已完成初始化，循环安全
  const QueryEngine = getQueryEngine();
  return new QueryEngine(/* ... */);
}
```

### 策略二：调用点动态 import

```typescript
async function handleSpecialCase() {
  // 只在需要时加载，完全跳过模块初始化阶段的循环
  const { SpecialHandler } = await import('./SpecialHandler.js');
  return new SpecialHandler();
}
```

### 策略三：type-only 导入

```typescript
// 只导入类型，编译后完全消除，不产生运行时依赖
import type { QueryEngine } from './QueryEngine.js';

// 运行时通过参数传入实例，不是本模块 import 的
function processResult(engine: QueryEngine, result: unknown) {
  // ...
}
```

### 对比与选择

| 策略 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| 惰性 require | 同步场景、频繁调用 | 简单直接、模块缓存生效 | 延迟加载影响首次性能 |
| 动态 import | 异步场景、低频路径 | 最彻底的解耦 | 需要 async/await |
| type-only 导入 | 只需类型不需值 | 零运行时开销 | 不能用于实例化 |

**关键原则：在模块顶层只建立类型关系，运行时依赖延迟到实际调用时建立。** 如果循环依赖频繁出现，说明模块边界需要重构，应提取公共接口层。

---

## 5. 并行预取模式

Claude Code 的启动路径在必须等待的时间窗口内并行启动异步操作，实现"发射-忘记-稍后收集"。

### 核心思路

```typescript
// 阶段 1: 立即发射异步操作（不 await）
startMdmRawRead();          // 启动子进程读取 MDM 配置
startKeychainPrefetch();    // 启动 macOS Keychain 读取

// 阶段 2: 做不依赖上述结果的同步工作（~135ms）
import { ConfigManager } from './config/ConfigManager.js';
import { ToolRegistry } from './tools/ToolRegistry.js';
import { UIRenderer } from './ui/UIRenderer.js';
// ... 大量 import 在这里发生 ...

// 阶段 3: 收集预取结果
await ensureKeychainPrefetchCompleted();
// 此时子进程大概率已完成，await 几乎零等待
```

### 时间线对比

```
传统顺序执行:
  [Keychain 80ms] -> [MDM 40ms] -> [imports 135ms] -> [init]
  总计: ~255ms

并行预取:
  [Keychain 80ms.............]
  [MDM 40ms.....]
  [imports 135ms.........................] -> [await 0ms] -> [init]
  总计: ~135ms
```

### 实现细节

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

### 与 Promise.all 的区别

```typescript
// Promise.all: 同一位置发起 + 同一位置等待
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);

// 预取模式: 尽早发起 + 尽晚等待，中间可以做其他事
const promiseA = fetchA();  // 立即发射
doSyncWork();               // 同步工作填充等待时间
const a = await promiseA;   // 收集结果
```

**实践要点：**
- 关键洞察：import 语句虽是同步的，但执行期间 event loop 仍在运转，此前启动的异步操作可在这段时间完成
- 发起和等待分离，中间插入不依赖结果的工作
- 未 await 的 Promise rejection 会变成 unhandled rejection，必要时加 `.catch(noop)`

---

## 6. 快速路径模式

对于简单命令（如 `--version`、`--help`），跳过耗时的完整初始化。

```typescript
// entrypoints/cli.tsx
async function main() {
  const args = process.argv.slice(2);

  // === 快速路径：不需要加载任何模块 ===
  if (args.includes('--version')) {
    console.log(VERSION);  // VERSION 是内联常量
    return;
  }

  if (args.includes('--help')) {
    printHelp();  // 纯字符串输出
    return;
  }

  // === 快速路径：只需要部分模块 ===
  if (args.includes('--resume')) {
    const { resumeSession } = await import('./session/resume.js');
    return resumeSession(args);
  }

  // === 完整路径：加载全部模块 ===
  const { ConfigManager } = await import('./config/ConfigManager.js');
  const { ToolRegistry } = await import('./tools/ToolRegistry.js');
  const { UIRenderer } = await import('./ui/UIRenderer.js');
  // ... 完整初始化 ...
}
```

### 设计原则

```
命令频率       初始化成本       策略
---------------------------------------------
--version      极低             process.argv 直接检查
--help         极低             字符串输出，无依赖
--resume       中等             动态 import 部分模块
正常启动       完整             加载全部模块和配置
```

**实践要点：**
- 将参数解析放在所有重量级初始化之前
- 快速路径只依赖静态数据（版本号、帮助文本），不触碰文件系统或网络
- `claude --version` 几乎瞬间返回，CLI 体感响应速度大幅提升
- 动态 `import()` 让模块加载变成按需行为而非声明时行为

---

## 7. Migration 模式

配置 schema 随版本演进。Claude Code 用具名的幂等迁移函数处理。

### 迁移函数设计

```typescript
// 每个迁移是具名函数，名称即文档
const migrations = [
  migrateFennecToOpus,
  migrateSonnet45ToSonnet46,
  migrateOldPermissionFormat,
  migrateToolAliases,
];

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
    return { ...config, defaultModel: 'claude-sonnet-4-6' };
  }
  return config; // 已迁移或无需迁移，原样返回
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
| **纯函数** | 输入配置 -> 输出配置，无外部状态依赖 |

**实践要点：** 适用于任何需要演化 schema 的长期维护项目。只追加不修改历史迁移。

---

## 8. MCP 可扩展性模式

Model Context Protocol (MCP) 是开放式工具扩展机制，用标准协议代替插件 API。

### 架构概览

```
Claude Code CLI
  +-- 内置工具 (Read, Edit, Bash, Grep, Glob, ...)
  +-- MCP 客户端
        +-- GitHub MCP Server   -> git 操作
        +-- Postgres MCP Server -> 数据库查询
        +-- 自定义 MCP Server   -> 任意功能
```

### 服务器配置

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
      "allowedTools": ["query", "schema"],   // 工具白名单
      "deniedTools": ["drop_table"]          // 工具黑名单
    }
  }
}
```

### 生命周期

```
发现 Discovery     审批 Approval     执行 Execute     结果 Result
  扫描配置      ->   权限系统      ->   JSON-RPC    ->   类型转换
  列出工具          企业策略过滤       协议通信          结果验证
```

### 企业策略过滤

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

**实践要点：**
- MCP 是进程间通信协议（stdio/SSE），工具运行在独立进程中，崩溃不影响主进程
- 统一的 JSON Schema 描述使 LLM 能理解任意外部工具的输入输出
- 用协议代替 API：任何语言、任何进程都可以成为工具提供者
- 权限系统无缝覆盖 MCP 工具，与内置工具同等安全管控

---

## 9. 状态管理模式

Claude Code 采用函数式更新的轻量状态管理，不依赖 Redux 等外部库。

### 核心 API

```typescript
// 获取当前状态（只读快照）
const getAppState: () => Readonly<AppState>;

// 函数式更新（类似 React 的 setState）
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
  const listeners = new Set<(state: T) => void>();

  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      state = updater(state);
      listeners.forEach(fn => fn(state));
    },
    subscribe: (fn: (state: T) => void) => {
      listeners.add(fn);
      return () => listeners.delete(fn); // 返回取消订阅函数
    },
  };
}
```

### 为什么"刚好够用"

| 选择 | 理由 |
|------|------|
| 无中间件 | CLI 工具不需要 devtools 或 time-travel |
| 轻量订阅 | React/Ink 自身管理渲染周期 |
| 无 action 类型 | 函数式更新自带语义，无需额外分类 |
| 极简实现 | 几十行代码，零依赖 |

**实践要点：** 函数式更新避免竞态条件、`Readonly` 防止消费方直接修改状态、发布-订阅让 UI 层订阅变更后自动重渲染。对非 Web 的 TypeScript 项目来说，这种"刚好够用"的方案优于引入重量级状态库。

---

## 10. React 终端 UI 模式

Claude Code 使用 React + Ink 将组件化 UI 开发带入终端，通过 Yoga 引擎实现 Flexbox 布局。

### 组件结构

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

### Flexbox 布局

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

**实践要点：**
- `<Box>` 对应 `<div>`，`<Text>` 对应 `<span>`，Web 开发者几乎零学习成本
- Virtual DOM diffing 在终端同样有价值——只更新变化的字符区域，避免全屏重绘闪烁
- 对流式输出（逐字显示 AI 回复）尤其关键：声明式更新比手动光标操作简洁得多
- 可用 `ink-testing-library` 做组件快照测试

---

## 11. 文件状态缓存

`FileStateCache` 在同一对话轮次中避免重复读取相同文件——AI 可能多次读取同一文件（先读、再编辑、再验证）。

### 缓存设计

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

### 读取路径

```typescript
async function readFileWithCache(
  path: string,
  cache: FileStateCache
): Promise<string> {
  const cached = cache.files.get(path);

  if (cached) {
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

### 缓存克隆实现隔离

```typescript
// 每个工具调用拿到独立的缓存副本
function cloneCache(source: FileStateCache): FileStateCache {
  return { files: new Map(source.files) };
}

// 工具 A 的文件修改不会污染工具 B 的缓存视图
const cacheForToolA = cloneCache(masterCache);
const cacheForToolB = cloneCache(masterCache);
```

**实践要点：**
- 以文件 hash 作为缓存有效性判断，比 mtime 更可靠
- 写操作后主动 `invalidate`，保证一致性
- 缓存粒度是单文件，失效也是单文件级别，不会因一次写入清空全部缓存
- 克隆实现并行工具调用间的隔离

---

## 12. 会话持久化

Claude Code 支持会话的录制（transcript）、恢复（resume）和跨环境传输（teleport）。

### Transcript 记录

```typescript
interface TranscriptMessage {
  role: 'user' | 'assistant' | 'tool';
  content: unknown;
  timestamp: number;
  tokenUsage?: TokenUsage;
}

// JSONL 格式：每行一条 JSON，追加写入
function recordTranscript(entry: TranscriptMessage): void {
  const line = JSON.stringify(entry) + "\n";
  fs.appendFileSync(transcriptPath, line);
}
```

### Resume 恢复

```typescript
// --resume 标志从上次会话继续
async function resumeSession(sessionId: string): Promise<void> {
  // 1. 加载历史 transcript
  const transcript = await loadTranscript(sessionId);

  // 2. 重建消息历史
  const messages = transcript.messages.map(toAPIMessage);

  // 3. 创建新 QueryEngine，注入历史
  const engine = new QueryEngine({ initialMessages: messages });

  // 4. 等待用户新输入
  await startInteractiveLoop(engine);
}
```

### Teleport 跨环境传输

```typescript
interface TeleportPayload {
  transcript: SessionTranscript;
  fileContext: FileContextSnapshot;  // 相关文件快照
  workingDirectory: string;
}

// 序列化 -> 传输 -> 反序列化 -> 恢复
async function teleportSession(payload: TeleportPayload): Promise<void> {
  const serialized = JSON.stringify(payload);
  await transmitToEnvironment(serialized, targetEnv);
}
```

**JSONL 格式的优势：**
- 追加写入，不需要读取再写入整个文件
- 单行 JSON 便于流式解析，崩溃时最多丢失最后一行
- 可以用 `grep`/`jq` 直接分析

**实践要点：** 会话持久化使长任务中断后可恢复，不丢失上下文。恢复时重建消息数组即可，不需要重放每个工具调用。

---

## 13. 成本控制模式

Claude Code 内置多层成本控制，防止 AI agent 自主执行时费用失控。

### 预算配置

```typescript
interface BudgetConfig {
  maxBudgetUsd: number;    // 单次会话硬性美元上限
  maxTurns: number;        // 最大对话轮次
  taskBudget: number;      // 子任务预算分配
}
```

### 逐次追踪与检查

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

### Prompt Cache Break 检测

```typescript
// Prompt cache 中断会导致成本突增（需重新处理完整上下文）
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

**实践要点：**
- `maxBudgetUsd` 是硬上限，适合 CI/自动化场景防止意外高额消费
- `maxTurns` 防止 agent 陷入无限循环
- `taskBudget` 允许将总预算分配给子任务（如 sub-agent），实现预算层级管理
- Cache break 检测可提前预警成本异常——当 AI 对话上下文变化导致缓存失效时，单轮成本可能翻倍

---

## 14. 错误分类模式

`categorizeRetryableAPIError` 将 API 错误分为可重试和不可重试两类，驱动不同的处理策略。

### 错误分类函数

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

  // 529: 服务过载 — 可重试，较长退避
  if (error.status === 529) {
    return {
      retryable: true,
      strategy: {
        maxRetries: 3,
        backoffMs: [5000, 10000, 20000],
        jitter: true,
      },
      userMessage: 'API 服务过载，正在等待...',
    };
  }

  // 5xx: 服务器错误 — 可重试，较短退避
  if (error.status >= 500 && error.status < 600) {
    return {
      retryable: true,
      strategy: { maxRetries: 3, backoffMs: [500, 1000, 2000], jitter: true },
      userMessage: 'API 服务暂时不可用，正在重试...',
    };
  }

  // 401: 认证失败 — 不可重试，致命
  if (error.status === 401) {
    return { retryable: false, fatal: true,
      userMessage: 'API 密钥无效或已过期，请检查认证信息。' };
  }

  // 400: 请求错误 — 不可重试，非致命
  if (error.status === 400) {
    return { retryable: false, fatal: false,
      userMessage: '请求格式错误，请检查输入。' };
  }

  // 未知错误 — 保守处理，不重试
  return { retryable: false, fatal: false,
    userMessage: `未预期的 API 错误 (${error.status})` };
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

**实践要点：**
- 错误分类集中在一个函数中，所有调用方共享统一的重试策略
- `retryAfterMs` 优先使用服务端 `Retry-After` 头，避免盲目重试加重服务端压力
- 指数退避 + jitter 防止"惊群效应"（多客户端同时重试）
- 认证错误、请求格式错误等不应重试——重试只会浪费时间
- 将"如何判断错误"与"如何处理错误"解耦，新增错误类型只需加一个分支

---

## 总结：核心工程原则

Claude Code 源码中的设计模式可以归纳为以下原则：

### 编译期 > 运行时

Feature flag 的编译期消除、type-only 导入的零开销——能在编译期解决的问题绝不留到运行时。

### 延迟一切非必要操作

并行预取、快速路径、惰性 require、动态 import——启动时只做必须做的事，其余按需加载。CLI 工具的冷启动速度直接影响用户体验。

### 不可变性作为安全边界

`DeepImmutable<>` 保护权限上下文、函数式 `setState` 保证更新原子性、缓存克隆实现隔离——不可变性不只是代码风格，而是防御性编程的核心手段。

### 协议优于接口

MCP 用标准协议（JSON-RPC）代替编程接口，AsyncGenerator 用语言原语代替自定义事件系统——协议级别的设计比 API 级别更灵活、更长寿。

### 显式分类驱动行为

错误分类（retryable vs fatal）、权限分层（mode -> rule -> source）、成本分类（dollar vs turn vs task）——先分类，再根据分类决定行为，消除了 if-else 链条，让逻辑可审计、可扩展。

### 幂等性贯穿始终

迁移函数幂等、缓存读取幂等、权限检查无副作用——在不确定代码会被执行几次的环境中（CLI 重启、网络重试、用户重复操作），幂等性是正确性的基础。

### 刚好够用的抽象

状态管理没用 Redux，UI 框架只用 React 最小子集，工具系统只抽象注册和执行——过度抽象的代价是认知负担和调试困难，刚好够用才是最优解。

---

这些模式不是 Claude Code 独有的发明，但它们在一个真实的大型 TypeScript CLI 项目中的组合应用，展示了如何在性能、可维护性和安全性之间取得平衡。每个模式都解决了具体的工程问题，而非为了"架构正确"而存在。
