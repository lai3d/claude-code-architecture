# 工具系统 (Tool System)

> 基于 Claude Code CLI v2.1.88 源码分析

Claude Code 的工具系统是其核心架构之一，负责将 LLM 的意图转化为实际的文件操作、命令执行、搜索查询等具体行为。整个系统围绕 `Tool` 类型定义、工具注册表和权限模型三大支柱构建。

---

## Tool 类型定义 (Tool.ts)

`Tool` 类型是整个工具系统的核心抽象，定义在 `src/Tool.ts` 中。以下是关键类型及其职责。

### ToolInputJSONSchema — 输入校验

每个工具通过 JSON Schema 声明其输入参数的类型和约束：

```typescript
type ToolInputJSONSchema = {
  type: "object";
  properties: Record<string, JSONSchemaProperty>;
  required?: string[];
  additionalProperties?: boolean;
};
```

该 schema 在工具调用前用于校验 LLM 生成的参数是否合法，防止无效输入进入执行阶段。

### ToolPermissionContext — 权限上下文

描述工具执行时的权限环境：

```typescript
interface ToolPermissionContext {
  permissionMode: "default" | "auto" | "bypass";
  rules: ToolPermissionRulesBySource;
  additionalWorkingDirs?: string[];
}
```

- **`default`** — 默认模式，敏感操作需用户确认
- **`auto`** — 自动模式，大部分操作自动批准（适合 CI/CD 场景）
- **`bypass`** — 绕过模式，跳过所有权限检查

权限规则按来源组织（`ToolPermissionRulesBySource`），每条规则可设为 `alwaysAllow`、`deny` 或 `ask`，实现细粒度的权限控制。

### ToolUseContext — 运行时上下文

工具执行时传入的运行时上下文，包含当前会话、文件系统状态、配置信息等：

```typescript
interface ToolUseContext {
  // 当前工作目录、会话信息、配置等
  cwd: string;
  session: Session;
  config: Config;
  // ... 其他运行时状态
}
```

工具实现函数通过该上下文获取执行所需的一切环境信息。

### SetToolJSXFn — 自定义渲染

允许工具注册自定义的 React 组件用于 UI 渲染：

```typescript
type SetToolJSXFn = (jsx: React.ReactNode) => void;
```

工具可以通过此函数在终端 UI（基于 Ink）中展示富文本输出，例如文件 diff、搜索结果高亮等。

### ValidationResult — 输入校验结果

工具在执行前可对输入进行自定义校验：

```typescript
type ValidationResult =
  | { valid: true }
  | { valid: false; reason: string };
```

除了 JSON Schema 的静态校验外，工具还可实现 `validateInput` 方法进行业务逻辑层面的校验（如文件路径是否存在、参数组合是否合理等）。

### QueryChainTracking — 嵌套代理链追踪

用于追踪嵌套的代理（Agent）调用链：

```typescript
interface QueryChainTracking {
  depth: number;
  parentToolUseId?: string;
  // ...
}
```

当 `AgentTool` 创建子代理时，系统通过此机制追踪调用层级，防止无限递归并提供调试信息。

---

## Tool Registry — 工具注册表 (tools.ts)

`tools.ts` 是工具注册的中枢，负责根据运行时环境条件加载 40 多个工具。

### 始终加载的工具

以下工具在所有环境下均可用：

| 工具名称 | 分类 | 说明 |
|---------|------|------|
| `AgentTool` | 代理 | 创建子代理执行复杂任务 |
| `BashTool` | 命令执行 | 执行 shell 命令 |
| `FileEditTool` | 文件操作 | 编辑文件（基于字符串替换） |
| `FileReadTool` | 文件操作 | 读取文件内容 |
| `FileWriteTool` | 文件操作 | 写入/创建文件 |
| `GlobTool` | 搜索 | 文件名模式匹配搜索 |
| `GrepTool` | 搜索 | 文件内容正则搜索（基于 ripgrep） |
| `NotebookEditTool` | 文件操作 | 编辑 Jupyter Notebook |
| `WebFetchTool` | 网络 | 抓取网页内容 |
| `WebSearchTool` | 网络 | 网络搜索 |
| `SkillTool` | 扩展 | 调用已注册的技能 |
| `AskUserQuestionTool` | 交互 | 向用户提问 |
| `LSPTool` | 开发 | 与语言服务器交互 |
| `TodoWriteTool` | 任务管理 | 管理待办事项 |
| `TaskCreate/Get/Update/List/Stop/Output` | 任务管理 | 后台任务的完整生命周期管理 |
| `EnterPlanMode` / `ExitPlanMode` | 模式切换 | 进入/退出计划模式 |
| `EnterWorktree` / `ExitWorktree` | 模式切换 | 进入/退出 git worktree |
| `ConfigTool` | 配置 | 读写配置 |
| `ToolSearchTool` | 元工具 | 搜索可用工具的 schema |
| `ListMcpResources` / `ReadMcpResource` | MCP | 操作 MCP 资源 |
| `BriefTool` | 输出控制 | 控制输出详细程度 |
| `SendMessageTool` | 通信 | 发送消息 |
| `TeamCreate` / `TeamDelete` | 团队 | 团队管理 |

### 按特性标志加载的工具

部分工具仅在特定特性标志（feature flag）开启时加载：

| 工具名称 | 加载条件 | 说明 |
|---------|---------|------|
| `REPLTool` | `USER_TYPE === 'ant'` | REPL 执行环境（仅 Anthropic 内部） |
| `SleepTool` | `PROACTIVE \|\| KAIROS` | 等待/休眠（主动式代理场景） |
| `CronCreate/Delete/List` | `AGENT_TRIGGERS` | 定时任务管理 |
| `RemoteTriggerTool` | `AGENT_TRIGGERS_REMOTE` | 远程触发器 |
| `MonitorTool` | `MONITOR_TOOL` | 系统监控 |
| `SendUserFileTool` | `KAIROS` | 向用户发送文件 |
| `PushNotificationTool` | `KAIROS \|\| KAIROS_PUSH_NOTIFICATION` | 推送通知 |
| `SubscribePRTool` | `KAIROS_GITHUB_WEBHOOKS` | 订阅 GitHub PR 事件 |
| `VerifyPlanExecutionTool` | 环境变量 `CLAUDE_CODE_VERIFY_PLAN` | 验证计划执行结果 |

### 条件加载模式

工具的条件加载采用 `require()` + `feature()` 的模式：

```typescript
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null;
```

这种模式的设计意图：

1. **按需加载** — 未启用的工具不会被 `require()`，减少启动时间和内存占用
2. **编译时死代码消除** — 打包工具可在编译期移除不可达的 `require()` 分支
3. **运行时灵活性** — 通过 `feature()` 函数在运行时决定加载哪些工具

最终，注册表将所有非 `null` 的工具收集到数组中：

```typescript
export function getTools(): Tool[] {
  return [
    AgentTool,
    BashTool,
    FileEditTool,
    // ... 始终加载的工具
    SleepTool,       // 可能为 null
    REPLTool,        // 可能为 null
    // ... 其他条件工具
  ].filter(Boolean); // 过滤掉 null 值
}
```

---

## 权限模型 (Permission Model)

工具的权限控制是 Claude Code 安全架构的关键组成部分。

### 权限规则来源

权限规则按来源分层组织（`ToolPermissionRulesBySource`），优先级从高到低：

1. **命令行参数** — 启动时通过 `--allow` / `--deny` 指定
2. **项目配置** — `.claude/settings.json` 中的规则
3. **用户配置** — `~/.claude/settings.json` 中的全局规则
4. **默认规则** — 工具自身定义的默认行为

### 权限判定流程

```
用户/LLM 请求调用工具
        │
        ▼
  检查 permissionMode
        │
   ┌────┼────┐
   │    │    │
bypass auto default
   │    │    │
   │    │    ▼
   │    │  遍历规则来源
   │    │    │
   │    │  ┌─┴──────────┐
   │    │  │ alwaysAllow │→ 直接执行
   │    │  │ deny        │→ 拒绝执行
   │    │  │ ask         │→ 提示用户确认
   │    │  └─────────────┘
   │    │
   │    ▼
   │  自动批准大部分操作
   │  （危险操作仍需确认）
   │
   ▼
  跳过所有检查，直接执行
```

### 工具自定义权限逻辑

每个工具可实现 `isReadOnly()` 和 `needsPermission()` 方法：

```typescript
// BashTool 示例：根据命令内容判断是否需要权限
needsPermission(input: BashInput, context: ToolPermissionContext): boolean {
  // 只读命令（如 ls, cat）不需要额外权限
  if (isReadOnlyCommand(input.command)) return false;
  // 写入/删除操作需要权限确认
  return true;
}
```

---

## MCP 工具集成

Claude Code 通过 Model Context Protocol (MCP) 支持外部工具服务器的集成。

### MCP 工具的加载

MCP 工具在运行时动态发现并注册，与内置工具共存于同一工具列表中：

- `ListMcpResources` — 列出 MCP 服务器提供的资源
- `ReadMcpResource` — 读取特定 MCP 资源

MCP 工具的权限同样受工具权限模型管控，外部工具默认需要用户确认。

### 配置示例

MCP 服务器在项目或用户配置中声明：

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-figma"],
      "env": {}
    }
  }
}
```

---

## 工具实现模式 (Tool Implementation Pattern)

每个工具以独立目录的形式组织在 `src/tools/` 下。

### 目录结构

```
src/tools/
├── AgentTool/
│   └── AgentTool.ts
├── BashTool/
│   ├── BashTool.ts
│   └── bashHelpers.ts
├── FileEditTool/
│   └── FileEditTool.ts
├── GrepTool/
│   └── GrepTool.ts
└── ...
```

### 标准工具定义

每个工具导出一个符合 `Tool` 接口的对象：

```typescript
// src/tools/GrepTool/GrepTool.ts
export const GrepTool: Tool = {
  name: "Grep",
  description: "基于 ripgrep 的强力搜索工具...",

  // JSON Schema 定义输入参数
  inputSchema: {
    type: "object",
    properties: {
      pattern: { type: "string", description: "正则表达式模式" },
      path: { type: "string", description: "搜索路径" },
      glob: { type: "string", description: "文件名过滤" },
    },
    required: ["pattern"],
  },

  // 输入校验（可选）
  validateInput(input): ValidationResult {
    if (!input.pattern) {
      return { valid: false, reason: "必须提供搜索模式" };
    }
    return { valid: true };
  },

  // 是否只读（影响权限判定）
  isReadOnly(): boolean {
    return true;
  },

  // 核心执行逻辑
  async call(input, context: ToolUseContext): Promise<ToolResult> {
    // 1. 解析参数
    // 2. 执行 ripgrep
    // 3. 格式化并返回结果
  },

  // 自定义 UI 渲染（可选）
  renderToolUse(input, setJSX: SetToolJSXFn) {
    setJSX(<GrepResultView results={...} />);
  },
};
```

### 工具生命周期

一个工具调用的完整生命周期如下：

1. **Schema 校验** — 根据 `inputSchema` 校验 LLM 生成的参数
2. **自定义校验** — 调用 `validateInput()` 进行业务逻辑校验
3. **权限检查** — 根据 `needsPermission()` 和权限规则决定是否放行
4. **执行** — 调用 `call()` 方法执行实际操作
5. **渲染** — 通过 `renderToolUse()` 在终端 UI 中展示结果
6. **结果返回** — 将执行结果返回给 LLM 作为后续推理的输入

---

## 总结

Claude Code 的工具系统通过以下设计实现了高度的灵活性和安全性：

- **类型安全** — 完整的 TypeScript 类型定义确保工具接口的一致性
- **条件加载** — 基于特性标志的按需加载减少不必要的资源消耗
- **分层权限** — 多来源、多级别的权限规则实现细粒度访问控制
- **可扩展性** — MCP 协议支持无限扩展外部工具
- **独立封装** — 每个工具独立目录，职责清晰，易于维护和测试
