# QueryEngine 与对话循环

## 概述

`QueryEngine` 是 Claude Code CLI 的核心对话引擎，负责管理用户与 AI 之间的多轮对话流程。它封装了消息管理、工具调用权限控制、流式响应、费用追踪等关键能力，是整个 CLI 的"大脑"。

---

## QueryEngine 类 (QueryEngine.ts)

### 配置结构

`QueryEngine` 通过一个庞大的配置对象进行初始化，涵盖了对话引擎的所有可调参数：

```typescript
type QueryEngineConfig = {
  cwd: string                    // 工作目录
  tools: Tools                   // 可用工具集合
  commands: Command[]            // 斜杠命令列表
  mcpClients: MCPServerConnection[]  // MCP 服务器连接
  agents: AgentDefinition[]      // Agent 定义
  canUseTool: CanUseToolFn       // 工具权限判定函数
  getAppState: () => AppState    // 获取应用状态
  setAppState: (f: (prev: AppState) => AppState) => void  // 更新应用状态
  initialMessages?: Message[]    // 初始消息（恢复会话用）
  readFileCache: FileStateCache  // 文件读取缓存
  customSystemPrompt?: string    // 自定义系统提示词（替换默认）
  appendSystemPrompt?: string    // 追加系统提示词
  userSpecifiedModel?: string    // 用户指定模型
  fallbackModel?: string         // 回退模型
  thinkingConfig?: ThinkingConfig  // 思考模式配置
  maxTurns?: number              // 最大对话轮次
  maxBudgetUsd?: number          // 最大预算（美元）
  taskBudget?: { total: number } // 任务预算
  jsonSchema?: Record<string, unknown>  // JSON Schema 约束输出
  verbose?: boolean              // 详细输出模式
  replayUserMessages?: boolean   // 是否回放用户消息
  handleElicitation?: ToolUseContext['handleElicitation']  // 交互式确认处理
  includePartialMessages?: boolean  // 是否包含部分消息
  setSDKStatus?: (status: SDKStatus) => void  // SDK 状态回调
  abortController?: AbortController  // 取消控制器
  orphanedPermission?: OrphanedPermission  // 孤立权限处理
  snipReplay?: (                 // 上下文裁剪回放函数
    yieldedSystemMsg: Message,
    store: Message[]
  ) => { messages: Message[]; executed: boolean } | undefined
}
```

**关键配置说明：**

- **`canUseTool`**：权限守门函数，每次工具调用前都会经过它的判定，决定是否允许执行
- **`readFileCache` (FileStateCache)**：缓存已读文件的状态，避免重复读取，同时用于检测文件变更
- **`snipReplay`**：当上下文窗口接近上限时，对历史消息进行裁剪压缩，保持对话可持续进行
- **`abortController`**：支持用户随时取消正在进行的 API 调用或工具执行

---

### 核心方法：submitMessage()

`submitMessage()` 是 `QueryEngine` 的入口方法，采用 **AsyncGenerator** 模式实现流式输出：

```typescript
async *submitMessage(): AsyncGenerator<SDKMessage, void, unknown>
```

这种设计使调用方可以逐步接收消息，实现实时 UI 更新。方法内部的核心职责包括：

1. **权限包装**：对 `canUseTool` 进行二次封装，追踪权限拒绝事件（`SDKPermissionDenial`），用于统计和审计
2. **消息历史管理**：维护 `mutableMessages` 数组，记录完整对话上下文
3. **费用追踪**：跨轮次累计 token 使用量和 API 调用费用
4. **文件状态缓存**：为文件读取操作维护缓存，提升性能
5. **技能发现**：每轮对话中追踪新发现的技能（Skill）
6. **记忆加载**：从 memdir 加载持久化记忆，注入到上下文中

---

## 对话循环

`QueryEngine` 的对话循环是一个经典的 **Agent Loop** 模式，通过反复调用 API 和执行工具来完成任务：

```
用户输入 → 组装系统提示词 → API 调用 → 处理响应 → [工具调用 → 循环] → 输出结果
```

### 详细流程

#### 第一步：用户提交消息

用户消息进入 `submitMessage()`，被添加到 `mutableMessages` 历史记录中。

#### 第二步：组装系统提示词

系统提示词由多个部分动态拼装：

```typescript
// 从多个来源收集系统提示词片段
const systemPromptParts = await fetchSystemPromptParts(config)
// 结合查询上下文生成最终的系统提示词
const systemPrompt = buildSystemPrompt(systemPromptParts, queryContext)
```

来源包括：
- 基础系统提示词（角色定义、行为规范）
- `customSystemPrompt` 或 `appendSystemPrompt`（用户自定义）
- 工具描述信息
- 项目上下文（CLAUDE.md 等）
- memdir 中的持久化记忆

#### 第三步：API 调用

通过 `query()` 函数向 Claude API 发送请求：

```typescript
const response = query({
  messages: mutableMessages,
  systemPrompt,
  tools: availableTools,
  model: selectedModel,
  thinkingConfig,
  abortController,
  // ...其他参数
})
```

#### 第四步：处理响应

API 返回的响应包含两种类型的内容块：

- **文本块（text）**：直接作为 `SDKMessage` yield 给调用方
- **工具调用块（tool_use）**：进入工具执行流程

#### 第五步：工具调用执行

对于 `tool_use` 块，执行以下子流程：

```
tool_use 块 → canUseTool 权限检查 → 执行工具 → 生成 tool_result → 追加到消息历史
```

权限检查可能产生三种结果：
- **允许**：立即执行工具
- **拒绝**：生成 `SDKPermissionDenial`，将拒绝信息作为 `tool_result` 返回给模型
- **需要用户确认**：通过 `handleElicitation` 与用户交互

#### 第六步：循环继续

如果响应中包含工具调用，工具结果会被追加到消息历史，然后**回到第三步**重新调用 API。模型会基于工具执行结果决定下一步操作——可能继续调用工具，也可能生成最终文本回复。

#### 第七步：流式输出

整个过程中，`SDKMessage` 通过 `yield` 持续输出：

```typescript
// 在循环的各个环节持续 yield 消息
yield { type: 'text', content: '...' }
yield { type: 'tool_use', id: '...', name: '...', input: {...} }
yield { type: 'tool_result', tool_use_id: '...', content: '...' }
```

### 循环终止条件

对话循环在以下情况下终止：

- 模型返回 `stop_reason: "end_turn"`，且没有待处理的工具调用
- 达到 `maxTurns` 轮次上限
- 达到 `maxBudgetUsd` 费用上限
- `abortController` 触发取消信号

---

## 关键特性详解

### 1. 取消机制 (AbortController)

```typescript
abortController?: AbortController
```

用户可以随时通过 `AbortController` 取消正在进行的操作。取消信号会传播到：
- 进行中的 API 流式请求
- 正在执行的工具操作
- 整个对话循环

### 2. 权限拒绝追踪 (SDKPermissionDenial)

每次工具调用被拒绝时，系统会生成 `SDKPermissionDenial` 事件。这些事件被收集用于：
- 向用户展示哪些操作被阻止
- 帮助模型理解约束，调整后续策略
- 安全审计和日志记录

### 3. 文件状态缓存 (FileStateCache)

```typescript
readFileCache: FileStateCache
```

文件读取操作的缓存层，核心作用：
- 避免同一文件在一次对话中被重复读取
- 检测文件在对话过程中是否被外部修改
- 为模型提供文件内容的一致性视图

### 4. 上下文裁剪 (snipReplay)

```typescript
snipReplay?: (
  yieldedSystemMsg: Message,
  store: Message[]
) => { messages: Message[]; executed: boolean } | undefined
```

当对话历史过长、接近上下文窗口限制时，`snipReplay` 负责：
- 压缩早期对话历史
- 保留关键信息（如工具调用结果）
- 生成摘要替代原始消息
- 确保对话可以继续进行而不丢失重要上下文

### 5. 费用追踪与预算控制

```typescript
maxBudgetUsd?: number    // 单次会话最大费用
maxTurns?: number        // 最大对话轮次
taskBudget?: { total: number }  // 任务级别预算
```

引擎在每轮对话后累计费用，到达限制时优雅终止循环并通知用户。

### 6. 会话持久化

通过 `initialMessages` 恢复之前的对话：

```typescript
initialMessages?: Message[]
```

配合 `replayUserMessages` 选项，可以在恢复会话时重新执行用户消息，确保工具状态的一致性。

---

## query.ts — API 调用层

`query.ts` 封装了与 Claude API 的底层通信逻辑，是 `QueryEngine` 和 API 之间的桥梁。

### 核心职责

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

**主要功能：**

1. **消息格式化**：将内部消息格式转换为 Claude API 要求的格式
2. **流式响应处理**：通过 SSE（Server-Sent Events）接收增量响应
3. **重试逻辑**：对瞬态错误（网络超时、429 限流、5xx 错误）实施自动重试
4. **Prompt Cache 断裂检测**：检测 prompt cache 是否命中，优化后续请求
5. **模型回退**：当首选模型不可用时，自动切换到 `fallbackModel`

### 重试策略

```
请求失败 → 判断错误类型 → 瞬态错误？ → 指数退避重试 → 最终失败则抛出
                           ↓
                       永久错误 → 立即抛出
```

瞬态错误包括：
- HTTP 429（限流）
- HTTP 5xx（服务端错误）
- 网络连接超时
- 流式连接中断

### 与 QueryEngine 的协作

`query()` 被 `QueryEngine` 的对话循环反复调用。每次调用时：
- `QueryEngine` 提供完整的消息历史和最新的系统提示词
- `query()` 返回模型的响应（可能包含文本和工具调用）
- `QueryEngine` 处理响应并决定是否继续循环

---

## 架构总结

```
┌─────────────────────────────────────────────────────┐
│                   QueryEngine                        │
│                                                      │
│  submitMessage() ← AsyncGenerator 流式输出            │
│       │                                              │
│       ▼                                              │
│  ┌──────────────── 对话循环 ────────────────┐        │
│  │                                          │        │
│  │  组装系统提示词                            │        │
│  │       │                                  │        │
│  │       ▼                                  │        │
│  │  query() → Claude API                    │        │
│  │       │                                  │        │
│  │       ▼                                  │        │
│  │  处理响应                                 │        │
│  │    ├── 文本 → yield SDKMessage           │        │
│  │    └── tool_use → 权限检查 → 执行 → 循环  │        │
│  │                                          │        │
│  └──────────────────────────────────────────┘        │
│                                                      │
│  辅助系统:                                            │
│  • FileStateCache    文件状态缓存                     │
│  • AbortController   取消控制                         │
│  • 费用追踪          预算控制                          │
│  • snipReplay        上下文裁剪                       │
│  • SDKPermissionDenial  权限审计                      │
└─────────────────────────────────────────────────────┘
```

`QueryEngine` 的设计体现了几个核心原则：

- **流式优先**：通过 AsyncGenerator 实现增量输出，用户无需等待完整响应
- **安全可控**：多层权限检查和预算控制，防止失控
- **可恢复**：会话持久化和上下文裁剪确保长对话的可持续性
- **可观测**：费用追踪、权限审计、状态回调提供完整的运行时可见性
