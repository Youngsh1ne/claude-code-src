# Claude Code 源码架构全面学习指南

## 1. 项目概览

Claude Code 是 Anthropic 开发的 CLI 编程助手，允许开发者在终端中与 Claude 交互，执行文件编辑、命令运行、代码搜索、工作流编排等软件工程任务。

### 1.1 核心数据

| 项目 | 详情 |
|------|------|
| 规模 | ~1,900 文件，512,000+ 行代码 |
| 语言 | TypeScript (strict mode) |
| 运行时 | Bun (非 Node.js) |
| 终端 UI | React + Ink |
| CLI 解析 | Commander.js (extra-typings) |
| Schema 校验 | Zod v4 |
| 代码搜索 | ripgrep |
| 协议支持 | MCP SDK, LSP |
| API | Anthropic SDK |
| 遥测 | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook + Bun 编译时消除 |
| 认证 | OAuth 2.0, JWT, macOS Keychain |

### 1.2 顶层目录结构

```
src/
├── main.tsx                 # 入口：Commander.js CLI + React/Ink 渲染器初始化
├── query.ts                 # 核心 Agentic Loop（异步生成器驱动）
├── QueryEngine.ts           # LLM API 调用引擎（流式、重试、token 计数）
├── Tool.ts                  # 工具基础类型与接口定义
├── tools.ts                 # 工具注册表
├── commands.ts              # 命令注册表
├── context.ts               # 系统/用户上下文收集（git status, CLAUDE.md）
├── cost-tracker.ts          # Token 费用追踪
│
├── commands/                # ~50 个斜杠命令实现
├── tools/                   # ~40 个 Agent 工具实现
├── components/              # ~140 个 Ink UI 组件
├── hooks/                   # React hooks（权限、输入、状态等）
├── services/                # 外部服务集成层
├── screens/                 # 全屏 UI（REPL, Doctor, Resume）
├── types/                   # TypeScript 类型定义
├── utils/                   # 工具函数（200+ 文件）
│
├── bridge/                  # IDE 双向通信桥（VS Code/JetBrains）
├── coordinator/             # 多智能体协调器
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── state/                   # 全局状态管理
├── query/                   # 查询管道辅助模块
├── context/                 # React Context Providers
├── memdir/                  # 持久化记忆目录
├── tasks/                   # 任务管理
├── remote/                  # 远程会话
├── server/                  # Server 模式
├── voice/                   # 语音输入
├── vim/                     # Vim 模式
└── buddy/                   # 伴侣精灵（趣味功能）
```


---

## 2. 启动流程详解 (`main.tsx`)

### 2.1 启动时序

```
┌─────────────────────────────────────────────────────────┐
│                    main.tsx 启动流程                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 副作用预取（模块顶层，import 之前执行）                  │
│     ├── startMdmRawRead()      // MDM 企业管理设置        │
│     ├── startKeychainPrefetch() // macOS Keychain 预读    │
│     └── apiPreconnect()        // API TCP 预连接          │
│                                                         │
│  2. Commander.js 解析 CLI 参数                            │
│     ├── --model, --permission-mode, --max-turns 等       │
│     └── 子命令路由（bridge, server, mcp 等）               │
│                                                         │
│  3. 初始化阶段 initializeEntrypoint()                     │
│     ├── runMigrations()        // 配置迁移                │
│     ├── eagerLoadSettings()    // 加载设置                │
│     ├── prefetchSystemContext() // 预取 git status 等     │
│     └── startDeferredPrefetches() // GrowthBook 等       │
│                                                         │
│  4. 运行模式分支                                          │
│     ├── 交互式 → React/Ink 渲染 REPL 屏幕                 │
│     ├── 非交互式 (pipe/headless) → 直接 query loop        │
│     └── 子命令 → 对应 handler                             │
│                                                         │
│  5. REPL 屏幕挂载                                        │
│     └── <App> → <REPL> → 消息列表 + 输入框 + 状态栏        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 2.2 并行预取优化

这是一个关键的性能设计。在 `main.tsx` 的模块顶层（所有 import 之前），通过副作用立即启动三个并行 I/O 操作：

```typescript
// main.tsx 顶部 — 作为副作用在其他 import 之前触发
startMdmRawRead()        // 读取企业 MDM 配置
startKeychainPrefetch()  // 预读 macOS Keychain 中的 API Key
```

这些操作返回 Promise 但不 await，让它们在后续模块加载期间并行执行。等到真正需要结果时，Promise 通常已经 resolve。

### 2.3 Feature Flags 编译时消除

使用 Bun 的 `bun:bundle` 特性实现编译时死代码消除：

```typescript
import { feature } from 'bun:bundle'

// 构建时，未启用的分支会被完全移除
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

已知的 Feature Flags：
- `PROACTIVE` — 主动模式
- `KAIROS` — 未知内部功能
- `BRIDGE_MODE` — IDE 桥接模式
- `DAEMON` — 守护进程模式
- `VOICE_MODE` — 语音输入
- `AGENT_TRIGGERS` — 智能体触发器
- `MONITOR_TOOL` — 监控工具
- `REACTIVE_COMPACT` — 响应式压缩
- `CONTEXT_COLLAPSE` — 上下文折叠
- `HISTORY_SNIP` — 历史裁剪
- `CACHED_MICROCOMPACT` — 缓存微压缩
- `TOKEN_BUDGET` — Token 预算
- `BG_SESSIONS` — 后台会话
- `EXPERIMENTAL_SKILL_SEARCH` — 实验性技能搜索
- `TEMPLATES` — 模板系统
- `BREAK_CACHE_COMMAND` — 缓存打破命令
- `TRANSCRIPT_CLASSIFIER` — 转录分类器

---

## 3. 核心 Agentic Loop (`query.ts`)

这是整个系统最核心的部分。理解这个文件就理解了 Claude Code 的运行本质。

### 3.1 架构概览

```
用户输入
    │
    ▼
┌─────────────────────────────────────────────────────┐
│              query() — 外层包装                       │
│  ├── 创建 consumedCommandUuids 追踪                   │
│  ├── yield* queryLoop()  ← 委托给内层循环              │
│  └── 循环结束后通知命令生命周期 completed                │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│           queryLoop() — 核心 while(true) 循环         │
│                                                     │
│  每次迭代：                                           │
│  ┌─────────────────────────────────────────────┐    │
│  │ 1. 上下文预处理                                │    │
│  │    ├── applyToolResultBudget (工具结果预算)     │    │
│  │    ├── snipCompact (历史裁剪)                  │    │
│  │    ├── microcompact (微压缩)                   │    │
│  │    ├── contextCollapse (上下文折叠)             │    │
│  │    └── autoCompact (自动压缩)                  │    │
│  ├─────────────────────────────────────────────┤    │
│  │ 2. 构建请求                                    │    │
│  │    ├── 组装 system prompt                     │    │
│  │    ├── 注入 user context (CLAUDE.md, date)    │    │
│  │    └── 注入 system context (git status)       │    │
│  ├─────────────────────────────────────────────┤    │
│  │ 3. 调用 LLM API (流式)                         │    │
│  │    ├── deps.callModel() → 流式响应              │    │
│  │    ├── 收集 assistant messages                 │    │
│  │    ├── 收集 tool_use blocks                    │    │
│  │    └── 处理 fallback model 切换                │    │
│  ├─────────────────────────────────────────────┤    │
│  │ 4. 工具执行                                    │    │
│  │    ├── StreamingToolExecutor (流式执行)         │    │
│  │    │   或 runTools() (传统执行)                 │    │
│  │    ├── 权限检查 → 执行 → 收集结果               │    │
│  │    └── 生成 tool_result messages               │    │
│  ├─────────────────────────────────────────────┤    │
│  │ 5. 循环决策                                    │    │
│  │    ├── 有 tool_use → continue (下一轮迭代)      │    │
│  │    ├── stop_reason=end_turn → return Terminal  │    │
│  │    ├── max_output_tokens → 恢复重试 (≤3次)     │    │
│  │    └── prompt_too_long → reactive compact      │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 3.2 状态管理

queryLoop 使用一个可变的 `State` 对象在迭代间传递状态：

```typescript
type State = {
  messages: Message[]                    // 完整消息历史
  toolUseContext: ToolUseContext          // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState  // 自动压缩追踪
  maxOutputTokensRecoveryCount: number   // max_output_tokens 恢复计数
  hasAttemptedReactiveCompact: boolean   // 是否已尝试响应式压缩
  maxOutputTokensOverride: number        // 输出 token 上限覆盖
  pendingToolUseSummary: Promise<...>    // 待处理的工具使用摘要
  stopHookActive: boolean               // 停止钩子是否激活
  turnCount: number                     // 当前轮次计数
  transition: Continue | undefined      // 上一次迭代的继续原因
}
```

### 3.3 依赖注入 (`query/deps.ts`)

查询循环通过依赖注入实现可测试性：

```typescript
type QueryDeps = {
  callModel: typeof queryModelWithStreaming  // LLM API 调用
  microcompact: typeof microcompactMessages  // 微压缩
  autocompact: typeof autoCompactIfNeeded    // 自动压缩
  uuid: () => string                         // UUID 生成
}
```

测试时可以注入 fake 实现，避免 `spyOn` 的模块级 mock。

### 3.4 异步生成器模式

整个查询循环使用 `async function*` 实现，每个 `yield` 点都是一个可观察的事件：

```typescript
async function* queryLoop(params, consumedCommandUuids):
  AsyncGenerator<StreamEvent | Message | ..., Terminal> {
  // yield 流式事件给 UI 层
  yield { type: 'stream_request_start' }
  // yield 消息给消息列表
  yield assistantMessage
  // yield 工具结果
  yield toolResultMessage
  // return 终止原因
  return { reason: 'end_turn' }
}
```

这种设计让 UI 层可以实时消费每个事件，实现流式渲染。


---

## 4. 工具系统 (`Tool.ts` + `src/tools/`)

### 4.1 工具接口定义

每个工具实现 `Tool` 接口，这是一个泛型接口，参数化了输入 Schema 和输出类型：

```typescript
interface Tool<Input extends ZodType, Output> {
  // 基础信息
  name: string
  description(options): string           // 动态描述（可根据上下文变化）
  inputSchema: Input                     // Zod schema 定义输入
  
  // 安全分类
  isReadOnly(input): boolean             // 是否只读操作
  isDestructive?(input): boolean         // 是否破坏性操作
  isConcurrencySafe(input): boolean      // 是否可并发执行
  isEnabled(): boolean                   // 是否启用
  
  // 权限
  checkPermissions(input, context): PermissionResult  // 权限检查
  getPath?(input): string                // 获取操作路径（用于权限匹配）
  
  // 执行
  call(input, context): Promise<Output>  // 核心执行逻辑
  
  // UI 渲染
  userFacingName(input): string          // 用户可见名称
  renderToolUseMessage(input): ReactNode // 渲染工具调用消息
  renderToolResultMessage(output): ReactNode // 渲染工具结果消息
  renderToolUseRejectedMessage?(input, reason): ReactNode // 渲染拒绝消息
  renderToolUseErrorMessage?(input, error): ReactNode     // 渲染错误消息
  
  // 高级
  validateInput?(input): ValidationResult  // 输入校验
  interruptBehavior?(): 'cancel' | 'block' // 中断行为
  extractSearchText?(output): string       // 提取搜索文本
  mapToolResultToToolResultBlockParam(output): ToolResultBlockParam // 结果映射
}
```

### 4.2 工具构建工厂

使用 `buildTool()` 工厂函数构建工具实例：

```typescript
const myTool = buildTool({
  name: 'MyTool',
  inputSchema: z.object({ ... }),
  call: async (input, context) => { ... },
  checkPermissions: (input, context) => { ... },
  // ... 其他方法
})
```

### 4.3 完整工具清单

| 工具名 | 目录 | 功能 | 只读 |
|--------|------|------|------|
| `BashTool` | `tools/BashTool/` | Shell 命令执行 | 取决于命令 |
| `PowerShellTool` | `tools/PowerShellTool/` | PowerShell 执行 (Windows) | 取决于命令 |
| `FileReadTool` | `tools/FileReadTool/` | 文件读取（图片、PDF、Notebook） | ✅ |
| `FileWriteTool` | `tools/FileWriteTool/` | 文件创建/覆盖 | ❌ |
| `FileEditTool` | `tools/FileEditTool/` | 部分文件修改（字符串替换） | ❌ |
| `GlobTool` | `tools/GlobTool/` | 文件模式匹配搜索 | ✅ |
| `GrepTool` | `tools/GrepTool/` | ripgrep 内容搜索 | ✅ |
| `WebFetchTool` | `tools/WebFetchTool/` | URL 内容获取 | ✅ |
| `WebSearchTool` | `tools/WebSearchTool/` | Web 搜索 | ✅ |
| `AgentTool` | `tools/AgentTool/` | 子智能体生成 | ❌ |
| `SkillTool` | `tools/SkillTool/` | 技能执行 | ❌ |
| `MCPTool` | `tools/MCPTool/` | MCP 服务器工具调用 | 取决于工具 |
| `LSPTool` | `tools/LSPTool/` | Language Server Protocol 集成 | ✅ |
| `NotebookEditTool` | `tools/NotebookEditTool/` | Jupyter Notebook 编辑 | ❌ |
| `TaskCreateTool` | `tools/TaskCreateTool/` | 创建后台任务 | ❌ |
| `TaskUpdateTool` | `tools/TaskUpdateTool/` | 更新任务状态 | ❌ |
| `TaskListTool` | `tools/TaskListTool/` | 列出任务 | ✅ |
| `TaskGetTool` | `tools/TaskGetTool/` | 获取任务详情 | ✅ |
| `TaskStopTool` | `tools/TaskStopTool/` | 停止任务 | ❌ |
| `TaskOutputTool` | `tools/TaskOutputTool/` | 获取任务输出 | ✅ |
| `SendMessageTool` | `tools/SendMessageTool/` | 智能体间消息传递 | ❌ |
| `TeamCreateTool` | `tools/TeamCreateTool/` | 创建团队智能体 | ❌ |
| `TeamDeleteTool` | `tools/TeamDeleteTool/` | 删除团队智能体 | ❌ |
| `EnterPlanModeTool` | `tools/EnterPlanModeTool/` | 进入规划模式 | ❌ |
| `ExitPlanModeTool` | `tools/ExitPlanModeTool/` | 退出规划模式 | ❌ |
| `EnterWorktreeTool` | `tools/EnterWorktreeTool/` | 进入 Git Worktree 隔离 | ❌ |
| `ExitWorktreeTool` | `tools/ExitWorktreeTool/` | 退出 Git Worktree | ❌ |
| `ToolSearchTool` | `tools/ToolSearchTool/` | 延迟工具发现 | ✅ |
| `ScheduleCronTool` | `tools/ScheduleCronTool/` | 创建定时触发器 | ❌ |
| `RemoteTriggerTool` | `tools/RemoteTriggerTool/` | 远程触发 | ❌ |
| `SleepTool` | `tools/SleepTool/` | 主动模式等待 | ✅ |
| `SyntheticOutputTool` | `tools/SyntheticOutputTool/` | 结构化输出生成 | ✅ |
| `TodoWriteTool` | `tools/TodoWriteTool/` | TODO 文件写入 | ❌ |
| `REPLTool` | `tools/REPLTool/` | REPL 执行 | 取决于命令 |
| `ConfigTool` | `tools/ConfigTool/` | 配置管理 | ❌ |
| `BriefTool` | `tools/BriefTool/` | 简要模式 | ✅ |
| `AskUserQuestionTool` | `tools/AskUserQuestionTool/` | 向用户提问 | ✅ |
| `ListMcpResourcesTool` | `tools/ListMcpResourcesTool/` | 列出 MCP 资源 | ✅ |
| `ReadMcpResourceTool` | `tools/ReadMcpResourceTool/` | 读取 MCP 资源 | ✅ |
| `McpAuthTool` | `tools/McpAuthTool/` | MCP 认证 | ❌ |

### 4.4 BashTool 深度解析

BashTool 是最复杂的工具，包含 10+ 个子模块：

```
tools/BashTool/
├── BashTool.tsx              # 主实现：call(), checkPermissions()
├── BashToolResultMessage.tsx  # 结果渲染（沙箱违规检测）
├── bashPermissions.ts         # 权限逻辑（命令前缀匹配、分类器）
├── bashSecurity.ts            # 安全校验（引号提取、重定向剥离）
├── bashCommandHelpers.ts      # 命令分段权限检查
├── commandSemantics.ts        # 命令语义分析（退出码解释）
├── destructiveCommandWarning.ts # 破坏性命令警告（rm -rf 等）
├── modeValidation.ts          # 模式校验（plan mode 限制）
├── pathValidation.ts          # 路径安全校验（危险删除检测）
└── commentLabel.ts            # Bash 注释标签提取
```

安全层级：
1. 命令语义分析 — 识别命令类型（只读/写入/破坏性）
2. 路径安全校验 — 检测危险的 `rm` 路径
3. 引号/重定向安全 — 防止注入
4. 权限规则匹配 — 按命令前缀匹配 allow/deny 规则
5. 分类器 — AI 分类器判断命令安全性（auto 模式）
6. 沙箱违规检测 — 运行时检测沙箱逃逸

### 4.5 工具编排 (`services/tools/toolOrchestration.ts`)

工具执行支持串行和并行两种模式：

```typescript
// 分区算法：将工具调用分为可并发批次和串行批次
function partitionToolCalls(toolUseMessages, toolUseContext): Batch[] {
  // 连续的 isConcurrencySafe=true 的工具 → 并发批次
  // isConcurrencySafe=false 的工具 → 单独串行批次
}

// 并发执行：最大并发数由环境变量控制，默认 10
async function* runToolsConcurrently(blocks, ...) {
  yield* all(blocks.map(async function* (toolUse) {
    yield* runToolUse(toolUse, ...)
  }), getMaxToolUseConcurrency())
}
```

### 4.6 流式工具执行 (`StreamingToolExecutor`)

这是一个高级优化：在 API 响应还在流式传输时，就开始执行已经完整接收的工具调用：

```
API 流式响应:  [thinking...][tool_use_1][tool_use_2][tool_use_3]
                              │
                              ▼ tool_use_1 完整接收
                         开始执行 tool_use_1
                                        │
                                        ▼ tool_use_2 完整接收
                                   开始执行 tool_use_2（如果可并发）
```

```typescript
class StreamingToolExecutor {
  addTool(block: ToolUseBlock, assistantMessage): void {
    // 工具块完整接收后立即加入执行队列
    if (this.canExecuteTool(isConcurrencySafe)) {
      this.executeTool(tool)  // 立即开始执行
    }
  }
  
  *getCompletedResults(): Generator<MessageUpdate> {
    // 在流式传输期间，获取已完成的工具结果
  }
  
  async *getRemainingResults(): AsyncGenerator<MessageUpdate> {
    // 流式传输结束后，等待剩余工具完成
  }
}
```


---

## 5. 权限系统

### 5.1 权限模式

```typescript
type PermissionMode = 
  | 'default'           // 默认：逐次确认
  | 'acceptEdits'       // 自动接受编辑
  | 'plan'              // 规划模式：只允许只读操作
  | 'bypassPermissions' // 跳过所有权限检查
  | 'dontAsk'           // 不询问，直接拒绝不确定的
  | 'auto'              // 自动模式：AI 分类器决定
  | 'bubble'            // 冒泡：向上级智能体请求
```

### 5.2 权限决策流程

```
工具调用请求
    │
    ▼
┌─────────────────────────────────────┐
│ 1. Pre-Tool-Use Hooks               │
│    (用户自定义钩子，可拦截)            │
├─────────────────────────────────────┤
│ 2. 权限规则匹配                      │
│    ├── alwaysAllowRules → allow      │
│    ├── alwaysDenyRules  → deny       │
│    └── alwaysAskRules   → ask        │
├─────────────────────────────────────┤
│ 3. 模式检查                          │
│    ├── bypassPermissions → allow     │
│    ├── plan + !isReadOnly → deny     │
│    ├── auto → 分类器判断              │
│    └── default → ask                 │
├─────────────────────────────────────┤
│ 4. 交互式确认 (如果 behavior=ask)     │
│    ├── 用户 approve → allow          │
│    ├── 用户 deny → deny              │
│    └── 用户修改输入 → allow(modified) │
├─────────────────────────────────────┤
│ 5. Post-Tool-Use Hooks               │
│    (工具执行后的钩子)                  │
└─────────────────────────────────────┘
```

### 5.3 权限规则来源

```typescript
type PermissionRuleSource =
  | 'userSettings'     // ~/.claude/settings.json
  | 'projectSettings'  // .claude/settings.json
  | 'localSettings'    // .claude/settings.local.json
  | 'flagSettings'     // CLI --allowedTools 参数
  | 'policySettings'   // 企业策略
  | 'cliArg'           // CLI 参数
  | 'command'          // 命令级别
  | 'session'          // 会话级别（用户在会话中批准的）
```

### 5.4 Auto 模式分类器

Auto 模式使用 AI 分类器（YoloClassifier）判断工具调用是否安全：

```typescript
type YoloClassifierResult = {
  shouldBlock: boolean        // 是否应该阻止
  reason: string              // 原因
  thinking?: string           // 思考过程
  model: string               // 使用的分类模型
  usage?: ClassifierUsage     // Token 使用量
  stage?: 'fast' | 'thinking' // 两阶段分类：快速 → 深度思考
}
```

分类器采用两阶段设计：
1. Fast stage — 快速判断，大部分请求在此通过
2. Thinking stage — 对不确定的请求进行深度分析

---

## 6. 上下文管理与压缩

Claude Code 有一套精密的多层上下文管理系统，这是保证长会话可用性的关键。

### 6.1 压缩层级

```
消息历史
    │
    ▼
┌─────────────────────────────────────┐
│ Layer 1: Tool Result Budget          │
│ 对单条工具结果的大小进行预算控制         │
│ 超大结果被截断或替换为引用              │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ Layer 2: Snip Compact                │
│ 历史裁剪：移除旧的、不重要的消息        │
│ 保留"受保护尾部"（最近的交互）          │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ Layer 3: Microcompact                │
│ 微压缩：压缩单条消息内的冗余内容        │
│ 支持缓存编辑（cache editing）优化      │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ Layer 4: Context Collapse            │
│ 上下文折叠：将多轮交互折叠为摘要        │
│ 读时投影（read-time projection）      │
│ 折叠存储在独立的 commit log 中         │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ Layer 5: Auto Compact                │
│ 自动压缩：当 token 超过阈值时          │
│ 调用 LLM 生成整个会话的摘要            │
│ 替换所有历史消息为摘要 + 附件           │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ Layer 6: Reactive Compact            │
│ 响应式压缩：API 返回 prompt_too_long   │
│ 时的紧急压缩，作为最后的安全网          │
└─────────────────────────────────────┘
```

### 6.2 Auto Compact 触发条件

```typescript
function shouldAutoCompact(tokenCount, model): boolean {
  const threshold = getAutoCompactThreshold(model)
  // 阈值通常是上下文窗口的 ~80%
  return tokenCount > threshold && isAutoCompactEnabled()
}

function getAutoCompactThreshold(model): number {
  const windowSize = getEffectiveContextWindowSize(model)
  // 为手动 /compact 保留空间
  return windowSize * 0.8
}
```

### 6.3 Token 警告状态

```typescript
function calculateTokenWarningState(tokenCount, model) {
  // 三级警告：
  // 1. warning — 接近阈值，显示黄色警告
  // 2. critical — 非常接近，显示红色警告
  // 3. blocking — 达到硬限制，阻止发送
}
```

---

## 7. API 层 (`services/api/`)

### 7.1 核心文件

```
services/api/
├── client.ts          # Anthropic SDK 客户端创建
├── claude.ts          # 核心 API 调用（流式/非流式）
├── bootstrap.ts       # API 引导（认证、预连接）
├── errors.ts          # 错误类型定义
├── withRetry.ts       # 重试逻辑
├── usage.ts           # 使用量追踪
├── dumpPrompts.ts     # Prompt 转储（调试用）
├── filesApi.ts        # 文件 API
├── logging.ts         # API 日志
└── promptCacheBreakDetection.ts  # Prompt 缓存中断检测
```

### 7.2 API 调用流程 (`claude.ts`)

```typescript
async function* queryModel({
  messages, systemPrompt, tools, signal, options, ...
}) {
  // 1. 构建 system prompt blocks（带缓存控制）
  const systemBlocks = buildSystemPromptBlocks(systemPrompt)
  
  // 2. 添加缓存断点
  addCacheBreakpoints(messages, systemBlocks)
  
  // 3. 配置 effort 参数（thinking budget）
  configureEffortParams(params, options.effortValue)
  
  // 4. 配置 task budget（agentic turn 预算）
  configureTaskBudgetParams(params, options.taskBudget)
  
  // 5. 流式调用
  yield* queryModelWithStreaming({ messages, system: systemBlocks, ... })
  
  // 6. Fallback：流式失败时切换到非流式
  yield* executeNonStreamingRequest({ ... })
}
```

### 7.3 Prompt 缓存策略

```typescript
function addCacheBreakpoints(messages, systemBlocks) {
  // 在 system prompt 末尾添加 cache_control
  // 在最后几条消息添加 cache_control
  // 使用 ephemeral 或 1h TTL
}

function getCacheControl({ isLastSystemBlock, isLastMessage, ... }) {
  // ephemeral: 短期缓存（默认）
  // 1h TTL: 长期缓存（compact/session_memory 查询）
}
```

---

## 8. 命令系统 (`commands.ts` + `src/commands/`)

### 8.1 命令注册

```typescript
async function getCommands(cwd: string): Promise<Command[]> {
  // 1. 加载内置命令
  // 2. 加载插件命令
  // 3. 加载 MCP 技能命令
  // 4. 加载用户自定义技能命令
  // 5. 按可用性过滤
  return commands
}
```

### 8.2 命令接口

```typescript
interface Command {
  name: string                    // 命令名（如 "commit"）
  description: string             // 描述
  isEnabled: boolean              // 是否启用
  isAvailable: boolean            // 是否可用（环境检查）
  progressMessage?: string        // 执行中的进度消息
  
  // 执行方式（二选一）
  getPromptForCommand?(args): string  // 转换为 prompt 发送给 LLM
  handler?(args, context): void       // 直接执行（不经过 LLM）
  
  // UI
  argNames?: string[]             // 参数名列表
  argDescription?: string         // 参数描述
}
```

### 8.3 主要命令分类

| 类别 | 命令 | 说明 |
|------|------|------|
| Git | `/commit`, `/diff`, `/pr_comments` | Git 操作 |
| 会话 | `/compact`, `/clear`, `/resume`, `/share` | 会话管理 |
| 配置 | `/config`, `/theme`, `/color`, `/vim` | 设置 |
| 诊断 | `/doctor`, `/cost`, `/context` | 诊断信息 |
| 智能体 | `/agents`, `/tasks` | 智能体管理 |
| 插件 | `/mcp`, `/skills` | 扩展管理 |
| 认证 | `/login`, `/logout` | 认证 |
| 记忆 | `/memory` | 持久化记忆 |
| 模式 | `/plan`, `/effort` | 工作模式 |


---

## 9. UI 层 (React + Ink)

### 9.1 架构概览

Claude Code 使用 React + Ink 将 React 组件渲染到终端。这是一个非常规但强大的选择。

```
┌─────────────────────────────────────────────┐
│                  App.tsx                      │
│  ┌─────────────────────────────────────────┐ │
│  │ FpsMetricsProvider                       │ │
│  │  ┌───────────────────────────────────┐   │ │
│  │  │ StatsProvider                      │   │ │
│  │  │  ┌─────────────────────────────┐   │   │ │
│  │  │  │ AppStateProvider             │   │   │ │
│  │  │  │  ┌───────────────────────┐   │   │   │ │
│  │  │  │  │ REPL Screen           │   │   │   │ │
│  │  │  │  │  ├── Messages         │   │   │   │ │
│  │  │  │  │  ├── StatusLine       │   │   │   │ │
│  │  │  │  │  ├── TextInput        │   │   │   │ │
│  │  │  │  │  └── Dialogs          │   │   │   │ │
│  │  │  │  └───────────────────────┘   │   │   │ │
│  │  │  └─────────────────────────────┘   │   │ │
│  │  └───────────────────────────────────┘   │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### 9.2 状态管理

使用自定义的轻量级 Store（类似 zustand）：

```typescript
// state/store.ts — 极简实现
function createStore<T>(initialState, onChange?): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  
  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 引用相等跳过
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

配合 React 的 `useSyncExternalStore` 使用：

```typescript
function useAppState(selector) {
  const store = useAppStore()
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
  )
}
```

### 9.3 React Context Providers

```
src/context/
├── fpsMetrics.tsx        # FPS 性能指标
├── stats.tsx             # 统计数据（token 数、费用等）
├── mailbox.tsx           # 消息邮箱（智能体间通信）
├── modalContext.tsx       # 模态对话框
├── overlayContext.tsx     # 覆盖层
├── promptOverlayContext.tsx # Prompt 覆盖层
├── notifications.tsx     # 通知
├── QueuedMessageContext.tsx # 排队消息
└── voice.tsx             # 语音状态
```

### 9.4 关键组件

| 组件 | 文件 | 功能 |
|------|------|------|
| `REPL` | `screens/REPL.tsx` | 主交互屏幕 |
| `Messages` | `components/Messages.tsx` | 消息列表 |
| `Message` | `components/Message.tsx` | 单条消息 |
| `MessageResponse` | `components/MessageResponse.tsx` | 助手响应 |
| `TextInput` | `components/TextInput.tsx` | 文本输入框 |
| `VimTextInput` | `components/VimTextInput.tsx` | Vim 模式输入 |
| `StatusLine` | `components/StatusLine.tsx` | 底部状态栏 |
| `Markdown` | `components/Markdown.tsx` | Markdown 渲染 |
| `HighlightedCode` | `components/HighlightedCode.tsx` | 代码高亮 |
| `StructuredDiff` | `components/StructuredDiff.tsx` | 结构化 Diff |
| `Spinner` | `components/Spinner.tsx` | 加载动画 |
| `TokenWarning` | `components/TokenWarning.tsx` | Token 警告 |
| `ModelPicker` | `components/ModelPicker.tsx` | 模型选择器 |
| `VirtualMessageList` | `components/VirtualMessageList.tsx` | 虚拟滚动消息列表 |

### 9.5 React Compiler 优化

源码中可见 React Compiler 的编译产物：

```typescript
// 编译前（原始 TSX）
export function App({ getFpsMetrics, stats, initialState, children }) {
  return (
    <FpsMetricsProvider getFpsMetrics={getFpsMetrics}>
      <StatsProvider store={stats}>
        <AppStateProvider initialState={initialState}>
          {children}
        </AppStateProvider>
      </StatsProvider>
    </FpsMetricsProvider>
  )
}

// 编译后（自动 memoization）
export function App(t0) {
  const $ = _c(9)  // 9 个缓存槽位
  const { getFpsMetrics, stats, initialState, children } = t0
  let t1
  if ($[0] !== children || $[1] !== initialState) {
    t1 = <AppStateProvider ...>{children}</AppStateProvider>
    $[0] = children; $[1] = initialState; $[2] = t1
  } else {
    t1 = $[2]  // 缓存命中
  }
  // ... 类似的缓存逻辑
}
```

---

## 10. 服务层 (`services/`)

### 10.1 MCP 服务 (`services/mcp/`)

Model Context Protocol 的完整实现：

```
services/mcp/
├── client.ts              # MCP 客户端（3000+ 行，核心）
│   ├── 连接管理            # 批量连接、重连、超时
│   ├── 工具调用            # callMCPTool, transformMCPResult
│   ├── 认证               # OAuth, 会话过期处理
│   └── 资源管理            # 资源列表、预取
├── config.ts              # 配置加载（mcp.json）
├── MCPConnectionManager.tsx # React 连接管理器
├── auth.ts                # 认证流程
├── types.ts               # 类型定义
├── InProcessTransport.ts  # 进程内传输
├── SdkControlTransport.ts # SDK 控制传输
├── channelPermissions.ts  # 频道权限
├── elicitationHandler.ts  # 信息征询处理
└── envExpansion.ts        # 环境变量展开
```

### 10.2 压缩服务 (`services/compact/`)

```
services/compact/
├── autoCompact.ts         # 自动压缩（阈值检测、触发）
├── compact.ts             # 核心压缩逻辑（调用 LLM 生成摘要）
├── microCompact.ts        # 微压缩（单消息级别）
├── apiMicrocompact.ts     # API 级微压缩
├── sessionMemoryCompact.ts # 会话记忆压缩
├── prompt.ts              # 压缩 prompt 模板
├── grouping.ts            # 消息分组（按 API 轮次）
└── postCompactCleanup.ts  # 压缩后清理
```

### 10.3 工具执行服务 (`services/tools/`)

```
services/tools/
├── toolExecution.ts       # 工具执行核心（权限检查 + 调用）
├── toolOrchestration.ts   # 工具编排（串行/并行分区）
├── StreamingToolExecutor.ts # 流式工具执行器
└── toolHooks.ts           # 工具钩子（pre/post）
```

### 10.4 分析服务 (`services/analytics/`)

基于 GrowthBook 的 Feature Flags 和实验系统：

```typescript
// 缓存的 feature gate 检查（可能过期）
checkStatsigFeatureGate_CACHED_MAY_BE_STALE('gate_name')

// 获取 feature value
getFeatureValue_CACHED_MAY_BE_STALE('feature_name')
```

### 10.5 其他服务

| 服务 | 目录 | 功能 |
|------|------|------|
| `oauth/` | OAuth 2.0 认证流程 | 浏览器回调、token 管理 |
| `lsp/` | Language Server Protocol | 代码智能（定义跳转、引用等） |
| `plugins/` | 插件加载器 | 内置 + 第三方插件 |
| `extractMemories/` | 记忆提取 | 从对话中自动提取记忆 |
| `policyLimits/` | 策略限制 | 企业级使用限制 |
| `tokenEstimation.ts` | Token 估算 | 快速估算 token 数 |
| `teamMemorySync/` | 团队记忆同步 | 跨智能体记忆共享 |
| `toolUseSummary/` | 工具使用摘要 | 生成工具调用的摘要 |
| `AgentSummary/` | 智能体摘要 | 子智能体执行摘要 |
| `SessionMemory/` | 会话记忆 | 跨会话持久化记忆 |

---

## 11. Bridge 系统 (`src/bridge/`)

### 11.1 架构

Bridge 是 Claude Code CLI 与 IDE 扩展（VS Code/JetBrains）之间的双向通信层。

```
┌──────────────┐                    ┌──────────────┐
│   IDE 扩展    │ ◄── WebSocket ──► │  Claude Code  │
│  (VS Code)   │     /HTTP SSE      │    CLI        │
│              │                    │              │
│  bridgeUI    │ ◄── 消息协议 ────► │ bridgeMain   │
│  bridgeAPI   │                    │ bridgeMsg    │
└──────────────┘                    └──────────────┘
```

### 11.2 核心模块

```
bridge/
├── bridgeMain.ts              # Bridge 主循环
├── bridgeMessaging.ts         # 消息协议（序列化/反序列化）
├── bridgeApi.ts               # Bridge API 端点
├── bridgeUI.ts                # UI 集成
├── bridgeConfig.ts            # 配置
├── bridgeEnabled.ts           # 启用检测
├── bridgePermissionCallbacks.ts # 权限回调（IDE 侧确认）
├── bridgePointer.ts           # 指针/光标同步
├── bridgeStatusUtil.ts        # 状态工具
├── bridgeDebug.ts             # 调试工具
│
├── replBridge.ts              # REPL 会话桥接
├── replBridgeHandle.ts        # REPL 桥接句柄
├── replBridgeTransport.ts     # REPL 传输层
│
├── sessionRunner.ts           # 会话执行管理
├── createSession.ts           # 会话创建
├── codeSessionApi.ts          # 代码会话 API
│
├── jwtUtils.ts                # JWT 认证
├── workSecret.ts              # 工作密钥
├── trustedDevice.ts           # 可信设备
│
├── inboundMessages.ts         # 入站消息处理
├── inboundAttachments.ts      # 入站附件
├── flushGate.ts               # 刷新门控
│
├── pollConfig.ts              # 轮询配置
├── pollConfigDefaults.ts      # 轮询默认值
├── capacityWake.ts            # 容量唤醒
│
├── remoteBridgeCore.ts        # 远程桥接核心
├── envLessBridgeConfig.ts     # 无环境桥接配置
├── sessionIdCompat.ts         # 会话 ID 兼容
├── debugUtils.ts              # 调试工具
└── types.ts                   # 类型定义
```

---

## 12. 多智能体系统

### 12.1 AgentTool — 子智能体

AgentTool 是最复杂的工具之一（20 个子模块），支持生成独立的子智能体：

```
AgentTool/
├── AgentTool.tsx              # 主实现（1400+ 行）
├── agentToolUtils.ts          # 工具过滤、解析
├── agentMemory.ts             # 智能体记忆管理
├── agentMemorySnapshot.ts     # 记忆快照
├── agentColorManager.ts       # 颜色管理
├── agentDisplay.ts            # 显示逻辑
│
├── built-in/                  # 内置智能体类型
│   ├── generalPurposeAgent.ts # 通用智能体
│   ├── exploreAgent.ts        # 探索智能体（只读）
│   ├── planAgent.ts           # 规划智能体
│   └── claudeCodeGuideAgent.ts # Claude Code 指南智能体
```

### 12.2 智能体类型

| 类型 | 说明 | 工具集 |
|------|------|--------|
| General Purpose | 通用子智能体 | 完整工具集 |
| Explore | 探索型（只读） | 只读工具 |
| Plan | 规划型 | 只读 + 规划工具 |
| Claude Code Guide | 使用指南 | 只读工具 |
| Custom | 用户自定义 | 可配置 |

### 12.3 团队智能体 (Swarm)

```typescript
// TeamCreateTool — 创建团队级并行工作
// 多个智能体可以同时工作在不同任务上

// SendMessageTool — 智能体间通信
// 智能体可以互相发送消息协调工作

// coordinator/coordinatorMode.ts — 协调器模式
// 管理多智能体的生命周期和通信
```

### 12.4 智能体工具过滤

```typescript
function filterToolsForAgent({ tools, agentType, ... }) {
  // 根据智能体类型过滤可用工具
  // explore 智能体只能使用只读工具
  // plan 智能体不能使用写入工具
  // 自定义智能体按配置过滤
}
```


---

## 13. 消息系统

### 13.1 消息类型

```typescript
type Message =
  | UserMessage           // 用户消息（文本、tool_result）
  | AssistantMessage       // 助手消息（文本、thinking、tool_use）
  | AttachmentMessage      // 附件消息（CLAUDE.md、记忆文件）
  | SystemMessage          // 系统消息（compact boundary 等）
  | ToolUseSummaryMessage  // 工具使用摘要
  | TombstoneMessage       // 墓碑消息（标记已删除的消息）

// 流式事件
type StreamEvent =
  | { type: 'stream_request_start' }  // 请求开始
  | { type: 'stream_delta', ... }     // 流式增量
  | { type: 'stream_end', ... }       // 流式结束
```

### 13.2 消息工具函数 (`utils/messages.ts`)

```typescript
// 创建消息
createUserMessage({ content, toolUseResult, ... })
createAssistantAPIErrorMessage({ content, error })
createSystemMessage({ content })
createUserInterruptionMessage()
createToolUseSummaryMessage(summary)
createMicrocompactBoundaryMessage()

// 消息处理
normalizeMessagesForAPI(messages)        // 标准化为 API 格式
getMessagesAfterCompactBoundary(messages) // 获取压缩边界后的消息
stripSignatureBlocks(messages)           // 剥离签名块
```

### 13.3 消息队列 (`utils/messageQueueManager.ts`)

```typescript
// 消息队列管理器 — 处理斜杠命令的排队执行
remove(uuid)                    // 从队列移除
getCommandsByMaxPriority()      // 按优先级获取命令
isSlashCommand(text)            // 检测是否为斜杠命令
```

---

## 14. 配置系统

### 14.1 配置层级

```
优先级从低到高：
1. 默认值
2. ~/.claude/settings.json          (用户全局设置)
3. .claude/settings.json            (项目设置)
4. .claude/settings.local.json      (本地设置，不提交)
5. CLI 参数 (--model, --permission-mode 等)
6. 环境变量 (CLAUDE_CODE_*, ANTHROPIC_*)
7. 企业 MDM 策略
```

### 14.2 CLAUDE.md 系统

CLAUDE.md 是 Claude Code 的项目级指令文件，类似于 `.editorconfig` 但面向 AI：

```typescript
// context.ts — getUserContext()
const claudeMd = getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))
// 注入到每次对话的 user context 中
```

加载顺序：
1. `~/.claude/CLAUDE.md` — 全局
2. 项目根目录 `CLAUDE.md` — 项目级
3. `--add-dir` 指定的额外目录

### 14.3 Schema 校验 (`schemas/`)

使用 Zod v4 定义所有配置的 schema，确保类型安全。

---

## 15. 插件与技能系统

### 15.1 插件系统 (`plugins/`)

```
plugins/
├── builtinPlugins.ts    # 内置插件注册
└── bundled/             # 打包的内置插件
```

插件可以：
- 注册新的工具
- 注册新的命令
- 扩展 system prompt

### 15.2 技能系统 (`skills/`)

```
skills/
├── bundledSkills.ts     # 内置技能
├── loadSkillsDir.ts     # 从目录加载技能
├── mcpSkillBuilders.ts  # MCP 技能构建器
└── bundled/             # 打包的内置技能
```

技能是可复用的工作流，通过 `SkillTool` 执行。用户可以在 `~/.claude/skills/` 或项目 `.claude/skills/` 中添加自定义技能。

---

## 16. 关键设计模式总结

### 16.1 异步生成器驱动

整个系统大量使用 `async function*`，从 API 调用到工具执行到查询循环：

```typescript
// 查询循环
async function* queryLoop(...) { yield* ... }

// API 调用
async function* queryModelWithStreaming(...) { yield ... }

// 工具执行
async function* runTools(...) { yield ... }

// 工具钩子
async function* runPreToolUseHooks(...) { yield ... }
```

这种模式的优势：
- 天然支持流式处理
- 每个 yield 点都是可观察的事件
- 调用者可以随时 `.return()` 中断
- 内存友好（不需要缓冲所有结果）

### 16.2 依赖注入

通过 `deps` 参数实现可测试性：

```typescript
// 生产环境
const deps = productionDeps()

// 测试环境
const deps = {
  callModel: fakeCallModel,
  microcompact: noopMicrocompact,
  autocompact: noopAutocompact,
  uuid: () => 'test-uuid',
}
```

### 16.3 Feature Flag 编译时消除

```typescript
// 构建时，feature('X') 被替换为 true/false
// false 分支被 tree-shaking 完全移除
if (feature('VOICE_MODE')) {
  // 这段代码在未启用时不会出现在产物中
  const voice = require('./voice.js')
}
```

### 16.4 懒加载

重模块延迟到首次使用时加载：

```typescript
// 顶层使用 require() 包装在 feature() 守卫中
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null

// 或使用动态 import()
const otel = await import('@opentelemetry/api')
```

### 16.5 Memoize 缓存

系统上下文等昂贵计算使用 memoize 缓存：

```typescript
export const getSystemContext = memoize(async () => {
  const gitStatus = await getGitStatus()
  return { gitStatus }
})

export const getUserContext = memoize(async () => {
  const claudeMd = getClaudeMds(...)
  return { claudeMd, currentDate: ... }
})
```

### 16.6 并行预取

启动时并行执行多个 I/O 操作：

```typescript
// main.tsx 顶部
startMdmRawRead()         // 并行 1
startKeychainPrefetch()   // 并行 2

// query.ts 中
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(...)
const pendingSkillPrefetch = startSkillDiscoveryPrefetch(...)
// 这些在 API 流式传输期间并行执行
```

---

## 17. 数据流全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户输入                                  │
│  终端输入 / IDE Bridge / 非交互式 pipe / 语音                     │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    REPL / 输入处理                                │
│  TextInput → handlePromptSubmit → 斜杠命令检测 → 消息队列         │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    QueryEngine.submitMessage()                   │
│  消息历史管理 → 上下文注入 → query() 调用                          │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    query() / queryLoop()                         │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ 上下文    │→│ API 调用  │→│ 工具执行  │→│ 循环决策  │       │
│  │ 预处理    │  │ (流式)    │  │ (串行/   │  │          │       │
│  │          │  │          │  │  并行)    │  │          │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│       │              │              │              │             │
│       │         yield events   yield results  continue/return   │
│       │              │              │              │             │
└───────┼──────────────┼──────────────┼──────────────┼─────────────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    UI 渲染 (React/Ink)                           │
│  Messages → MessageResponse → Markdown/Code/Diff → 终端输出      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 18. 学习路径建议

### 18.1 入门路径（理解核心流程）

1. `src/main.tsx` — 理解启动流程和模式分支
2. `src/screens/REPL.tsx` — 理解交互式会话的 UI 结构
3. `src/QueryEngine.ts` — 理解 LLM 调用引擎
4. `src/query.ts` — **最重要**，理解核心 agentic loop
5. `src/Tool.ts` — 理解工具接口定义
6. `src/tools/BashTool/BashTool.tsx` — 理解一个具体工具的实现

### 18.2 进阶路径（理解子系统）

7. `src/services/api/claude.ts` — API 调用细节（缓存、重试、流式）
8. `src/services/compact/` — 上下文压缩策略
9. `src/services/tools/toolOrchestration.ts` — 工具编排
10. `src/services/tools/StreamingToolExecutor.ts` — 流式工具执行
11. `src/hooks/toolPermission/` — 权限系统
12. `src/types/permissions.ts` — 权限类型定义

### 18.3 深入路径（理解高级功能）

13. `src/tools/AgentTool/` — 子智能体系统
14. `src/bridge/` — IDE 集成
15. `src/services/mcp/client.ts` — MCP 协议实现
16. `src/state/` — 全局状态管理
17. `src/utils/` — 工具函数库（200+ 文件）

### 18.4 关键代码量参考

| 文件 | 行数 | 重要性 |
|------|------|--------|
| `QueryEngine.ts` | ~46,000 | ⭐⭐⭐⭐⭐ |
| `Tool.ts` | ~29,000 | ⭐⭐⭐⭐⭐ |
| `commands.ts` | ~25,000 | ⭐⭐⭐⭐ |
| `query.ts` | ~1,730 | ⭐⭐⭐⭐⭐ |
| `services/api/claude.ts` | ~3,400 | ⭐⭐⭐⭐⭐ |
| `services/mcp/client.ts` | ~3,300 | ⭐⭐⭐⭐ |
| `tools/AgentTool/AgentTool.tsx` | ~1,400 | ⭐⭐⭐⭐ |
| `tools/BashTool/BashTool.tsx` | ~300+ | ⭐⭐⭐⭐ |
| `services/tools/toolOrchestration.ts` | ~200 | ⭐⭐⭐⭐ |
| `state/store.ts` | ~30 | ⭐⭐⭐ |

---

## 19. 值得关注的工程实践

### 19.1 性能优化

- **并行预取**：启动时 MDM/Keychain/API 并行初始化
- **流式工具执行**：API 响应还在传输时就开始执行工具
- **编译时死代码消除**：未启用的 feature 在构建时完全移除
- **懒加载**：重模块延迟到首次使用
- **React Compiler**：自动 memoization 优化渲染
- **虚拟滚动**：长消息列表使用虚拟滚动
- **Prompt 缓存**：利用 Anthropic API 的 prompt caching 减少重复计算

### 19.2 安全设计

- **多层权限系统**：规则 → 模式 → 分类器 → 用户确认
- **BashTool 安全**：命令语义分析 + 路径校验 + 沙箱
- **JWT 认证**：Bridge 通信使用 JWT
- **可信设备**：设备级信任管理
- **企业策略**：MDM 策略支持

### 19.3 可测试性

- **依赖注入**：query loop 的 deps 参数
- **Store 模式**：可预测的状态管理
- **类型安全**：TypeScript strict + Zod schema
- **模块化**：每个工具/命令是独立模块

### 19.4 可扩展性

- **插件系统**：第三方插件支持
- **技能系统**：可复用工作流
- **MCP 协议**：标准化的工具扩展协议
- **自定义智能体**：用户可定义智能体类型
- **Feature Flags**：渐进式功能发布
