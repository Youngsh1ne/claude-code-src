# Claude Code 源码学习指南（零基础版）

## 写在前面

这份文档假设你对以下技术不太熟悉：TypeScript、React、Bun、CLI 开发等。
我会用大白话解释每个概念，然后再对应到源码。

---

## 第一章：这个项目到底是什么？

### 1.1 一句话解释

Claude Code 是一个**终端程序**（就像你用的 `git`、`npm` 一样），你在终端里输入 `claude`，
它就启动了。然后你可以用自然语言告诉它"帮我写个函数"、"帮我修这个 bug"，
它会自动读文件、改代码、跑命令。

### 1.2 它和 ChatGPT 网页版有什么区别？

ChatGPT 网页版只能聊天。Claude Code 能**直接操作你的电脑**：
- 读你的代码文件
- 修改你的代码
- 在终端里执行命令（比如 `npm install`、`git commit`）
- 搜索你的代码库

它本质上是一个"有手有脚的 AI"，不只是"有嘴的 AI"。

### 1.3 技术栈速览（每个都解释）

| 技术 | 它是什么 | 在项目里干什么 |
|------|---------|--------------|
| **TypeScript** | JavaScript 的加强版，加了类型检查 | 整个项目的编程语言 |
| **Bun** | 类似 Node.js 的 JavaScript 运行时，但更快 | 运行这个程序 |
| **React** | 通常用来做网页 UI 的框架 | 这里用来做终端 UI（没错，终端里也能用 React） |
| **Ink** | 让 React 能渲染到终端的库 | 把 React 组件变成终端里的文字和颜色 |
| **Commander.js** | 解析命令行参数的库 | 处理 `claude --model opus` 这样的参数 |
| **Zod** | 数据校验库 | 确保 AI 返回的工具调用参数格式正确 |
| **ripgrep** | 超快的文本搜索工具 | AI 搜索代码时用的 |
| **Anthropic SDK** | Anthropic 公司的 API 客户端 | 和 Claude AI 模型通信 |

---

## 第二章：程序是怎么启动的？

当你在终端输入 `claude` 并回车，发生了什么？

### 2.1 入口文件：`src/main.tsx`

这是整个程序的"大门"。就像一栋大楼的正门，所有人都从这里进。

#### 第一步：抢跑预加载（第 1-20 行）

```typescript
// 这些代码在文件最顶部，比所有其他代码都先执行
profileCheckpoint('main_tsx_entry')  // 记录时间点，用于性能分析

startMdmRawRead()       // 开始读取企业管理配置
startKeychainPrefetch() // 开始读取钥匙串（存密码的地方）
```

**为什么要"抢跑"？**

想象你要做早餐：烧水泡咖啡 + 煎鸡蛋 + 烤面包。
笨办法：先烧水，等水开了泡咖啡，然后煎鸡蛋，然后烤面包。
聪明办法：**同时**把水壶打开、鸡蛋放锅里、面包放烤箱，它们并行进行。

Claude Code 也是这样。启动时需要：
1. 读取企业配置（MDM）— 需要调用系统命令，要等几十毫秒
2. 读取密码/API Key（Keychain）— 也要等几十毫秒
3. 加载一大堆代码模块 — 也要 ~135 毫秒

所以它在加载模块**之前**就先启动了 1 和 2，让它们在后台跑着。
等模块加载完，1 和 2 通常也完成了，总时间就省下来了。

这就是源码注释里说的：
```
// fires MDM subprocesses so they run in parallel with the remaining ~135ms of imports
```

#### 第二步：解析命令行参数（第 884 行 `run()` 函数）

```typescript
const program = new CommanderCommand()
  .name('claude')
  .description('Claude Code - starts an interactive session')
  .argument('[prompt]', 'Your prompt')
  .option('-p, --print', 'Print response and exit')
  .option('--model <model>', 'Model for the session')
  .option('--permission-mode <mode>', 'Permission mode')
  // ... 几十个选项
```

**这是什么？**

就像 `git` 有 `git commit -m "message"` 这样的参数，
Claude Code 也有很多参数：

```bash
claude                          # 启动交互式会话
claude -p "解释这段代码"          # 非交互式：打印结果就退出
claude --model opus             # 指定用哪个模型
claude --resume                 # 恢复上次的对话
claude --dangerously-skip-permissions  # 跳过所有权限检查（危险！）
```

Commander.js 帮你把这些参数解析成程序能用的数据。

#### 第三步：初始化（`preAction` 钩子）

```typescript
program.hook('preAction', async () => {
  // 等待之前"抢跑"的任务完成
  await Promise.all([
    ensureMdmSettingsLoaded(),      // 确保企业配置读完了
    ensureKeychainPrefetchCompleted() // 确保密码读完了
  ])
  
  await init()          // 核心初始化（认证、配置等）
  runMigrations()       // 配置文件版本升级
  loadRemoteManagedSettings() // 加载远程企业设置
  loadPolicyLimits()    // 加载使用限制
})
```

**什么是 `preAction` 钩子？**

Commander.js 的功能：在真正执行命令之前，先跑一段准备代码。
就像你进餐厅，先洗手（preAction），再点菜（action）。

#### 第四步：分支 — 交互式 vs 非交互式

```
用户输入 `claude`
    │
    ├── 有 -p 参数？ ──是──→ 非交互式模式
    │                        直接调用 query loop
    │                        打印结果，退出
    │
    └── 没有 ──→ 交互式模式
                 启动 React/Ink 渲染
                 显示 REPL 界面
                 等待用户输入
```

**交互式模式**就是你看到的那个可以一直聊天的界面。
**非交互式模式**就是 `echo "解释这段代码" | claude -p`，适合脚本调用。

#### 第五步：setup() — 会话准备

```typescript
// setup.ts
async function setup(cwd, permissionMode, ...) {
  // 1. 检查 Node.js 版本
  // 2. 找到项目的 git 根目录
  // 3. 初始化会话 ID
  // 4. 加载权限配置
  // 5. 启动文件变更监听
  // 6. 初始化插件
}
```

#### 第六步：渲染 REPL 界面

```
┌─────────────────────────────────────────────┐
│ Claude Code                                  │
│                                              │
│ > 你好，我能帮你什么？                         │
│                                              │
│ 用户: 帮我看看这个项目的结构                    │
│                                              │
│ Claude: 让我先看看目录结构...                   │
│ [工具调用: GlobTool - 搜索文件]                 │
│ [工具调用: FileReadTool - 读取 package.json]    │
│                                              │
│ 这个项目是一个 React 应用，结构如下...           │
│                                              │
├─────────────────────────────────────────────┤
│ Token: 12,345 | Cost: $0.03 | Model: Sonnet  │
└─────────────────────────────────────────────┘
│ > 输入你的消息...                              │
└─────────────────────────────────────────────┘
```

这个界面就是用 React + Ink 渲染的。是的，React 不只能做网页，
配合 Ink 库，它可以把组件渲染成终端里的彩色文字。

### 2.2 启动流程总结（时间线）

```
0ms    ─── 程序启动
         ├── 启动 MDM 读取（后台）
         ├── 启动 Keychain 读取（后台）
         └── 开始加载代码模块

135ms  ─── 模块加载完成
         ├── MDM 读取完成 ✓（在后台已经跑完了）
         ├── Keychain 读取完成 ✓
         └── 解析命令行参数

150ms  ─── preAction 钩子
         ├── init() 初始化
         ├── 运行配置迁移
         └── 加载远程设置（后台）

200ms  ─── setup() 会话准备
         ├── 找 git 根目录
         ├── 初始化会话
         └── 加载插件

300ms  ─── REPL 界面显示
         └── 等待用户输入
```


---

## 第三章：核心循环 — AI 是怎么"思考和行动"的？

这是整个项目最重要的部分。文件：`src/query.ts`

### 3.1 先理解"Agentic Loop"是什么

普通聊天机器人：你问一句，它答一句，结束。

Claude Code 不一样。你说"帮我修这个 bug"，它会：
1. 先搜索相关代码（调用 GrepTool）
2. 读取几个文件（调用 FileReadTool）
3. 思考问题在哪
4. 修改代码（调用 FileEditTool）
5. 运行测试（调用 BashTool）
6. 如果测试失败，再看错误信息，再修改...

这个"思考 → 行动 → 观察 → 再思考"的循环，就叫 **Agentic Loop**。

### 3.2 用做菜来类比

想象你请了一个厨师（Claude），你说"做一道红烧肉"：

```
你: "做红烧肉"

厨师思考: 我需要五花肉、酱油、糖...
厨师行动: 打开冰箱看看有什么（= FileReadTool 读文件）
厨师观察: 有五花肉，没有冰糖
厨师行动: 去超市买冰糖（= BashTool 执行命令）
厨师观察: 买到了
厨师行动: 开始切肉、焯水、炒糖色...（= FileEditTool 改代码）
厨师行动: 尝一口味道（= BashTool 跑测试）
厨师观察: 有点淡
厨师行动: 加点盐（= FileEditTool 再改）
厨师: "红烧肉做好了！"（循环结束）
```

### 3.3 代码里是怎么实现的？

```typescript
// query.ts 简化版（去掉了 90% 的细节）

async function* queryLoop(params) {
  let messages = params.messages  // 对话历史
  
  while (true) {  // 无限循环，直到 AI 说"我做完了"
    
    // ===== 第一步：准备上下文 =====
    // 把对话历史整理好，如果太长就压缩
    let messagesForQuery = prepareMessages(messages)
    
    // ===== 第二步：调用 AI =====
    // 把消息发给 Claude API，获取流式响应
    for await (const chunk of callClaudeAPI(messagesForQuery)) {
      yield chunk  // 实时把 AI 的回复传给 UI 显示
      
      // 收集 AI 的回复
      if (chunk.type === 'text') {
        // AI 在说话，显示给用户
      }
      if (chunk.type === 'tool_use') {
        // AI 想要使用工具！记录下来
        toolUseBlocks.push(chunk)
      }
    }
    
    // ===== 第三步：AI 想用工具吗？ =====
    if (toolUseBlocks.length === 0) {
      // AI 没有要用工具，说明它觉得任务完成了
      return { reason: 'end_turn' }  // 退出循环
    }
    
    // ===== 第四步：执行工具 =====
    for (const toolUse of toolUseBlocks) {
      // 比如 AI 说：我要读取 src/main.tsx
      // 那就真的去读这个文件，把内容返回给 AI
      const result = await executeTool(toolUse)
      messages.push(result)  // 把工具结果加入对话历史
    }
    
    // ===== 第五步：回到第一步，继续循环 =====
    // AI 会看到工具的结果，然后决定下一步做什么
  }
}
```

### 3.4 `yield` 是什么？（异步生成器）

你可能注意到了 `async function*` 和 `yield`。这是 JavaScript 的"生成器"语法。

**普通函数**：做完所有事，一次性返回结果。
```typescript
function getNumbers() {
  return [1, 2, 3, 4, 5]  // 一次性返回所有数字
}
```

**生成器函数**：做一点，吐一个结果，再做一点，再吐一个。
```typescript
function* getNumbers() {
  yield 1  // 先给你 1
  yield 2  // 再给你 2
  yield 3  // 再给你 3
  // 调用者可以随时停止，不用等全部完成
}
```

**为什么 Claude Code 要用生成器？**

因为 AI 的回复是**流式**的（一个字一个字蹦出来），不是一次性返回的。
用生成器，每收到一个字就可以 `yield` 给 UI 显示，用户就能看到 AI 在"打字"。

```
AI 回复流: "让" → "我" → "看" → "看" → "这" → "个" → "文" → "件"
           ↓      ↓      ↓      ↓      ↓      ↓      ↓      ↓
UI 显示:   让     让我   让我看  让我看看 ...（实时更新）
```

### 3.5 一次完整的循环示例

用户输入："帮我在 utils.ts 里加一个 add 函数"

```
┌─ 循环第 1 轮 ─────────────────────────────────────────┐
│                                                        │
│  AI 收到: "帮我在 utils.ts 里加一个 add 函数"            │
│                                                        │
│  AI 思考: 我需要先看看 utils.ts 现在长什么样              │
│                                                        │
│  AI 输出:                                               │
│    文本: "让我先看看 utils.ts 的内容"                     │
│    工具调用: FileReadTool({ path: "src/utils.ts" })     │
│                                                        │
│  → 有工具调用，执行工具，继续循环                          │
│                                                        │
│  工具结果: "export function multiply(a, b) { ... }"     │
│                                                        │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─ 循环第 2 轮 ─────────────────────────────────────────┐
│                                                        │
│  AI 收到: 之前的对话 + utils.ts 的内容                   │
│                                                        │
│  AI 思考: 好的，我看到文件了，现在加上 add 函数            │
│                                                        │
│  AI 输出:                                               │
│    文本: "我来添加 add 函数"                              │
│    工具调用: FileEditTool({                              │
│      path: "src/utils.ts",                              │
│      old_string: "export function multiply",             │
│      new_string: "export function add(a, b) {\n         │
│        return a + b\n}\n\nexport function multiply"      │
│    })                                                   │
│                                                        │
│  → 有工具调用，执行工具，继续循环                          │
│                                                        │
│  工具结果: "文件已修改"                                   │
│                                                        │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─ 循环第 3 轮 ─────────────────────────────────────────┐
│                                                        │
│  AI 收到: 之前的对话 + 文件修改成功的结果                  │
│                                                        │
│  AI 输出:                                               │
│    文本: "已经在 utils.ts 中添加了 add 函数。"            │
│    （没有工具调用）                                       │
│                                                        │
│  → 没有工具调用，AI 认为任务完成                           │
│  → return { reason: 'end_turn' }                        │
│  → 循环结束                                              │
│                                                        │
└────────────────────────────────────────────────────────┘
```


---

## 第四章：工具系统 — AI 的"手和脚"

AI 本身只能生成文字。工具系统让它能**真正做事**。

### 4.1 什么是"工具"？

工具就是 AI 可以调用的函数。就像手机上的 App：
- 想看文件？打开 FileReadTool（相当于"文件管理器" App）
- 想改代码？打开 FileEditTool（相当于"编辑器" App）
- 想跑命令？打开 BashTool（相当于"终端" App）
- 想搜代码？打开 GrepTool（相当于"搜索" App）

### 4.2 AI 怎么调用工具？

AI 不是直接执行代码。它是在回复中说"我想用某个工具"，然后程序帮它执行。

AI 的回复格式（简化）：
```json
{
  "content": [
    { "type": "text", "text": "让我搜索一下相关代码" },
    { 
      "type": "tool_use",
      "name": "GrepTool",
      "input": { "pattern": "function add", "path": "src/" }
    }
  ]
}
```

程序看到 `tool_use`，就知道 AI 想用 GrepTool，于是：
1. 找到 GrepTool 的实现代码
2. 检查权限（用户允许吗？）
3. 执行搜索
4. 把结果返回给 AI

### 4.3 每个工具长什么样？（以 FileEditTool 为例）

```
src/tools/FileEditTool/
├── FileEditTool.ts    ← 核心逻辑：怎么编辑文件
├── prompt.ts          ← 告诉 AI 这个工具怎么用的说明文字
├── UI.tsx             ← 在终端里怎么显示这个工具的调用和结果
└── utils.ts           ← 辅助函数（字符串匹配等）
```

每个工具都要实现这些关键方法：

```typescript
// 简化版，帮助理解
const FileEditTool = {
  name: 'FileEditTool',
  
  // 输入格式定义（用 Zod 校验）
  inputSchema: z.object({
    path: z.string(),        // 要编辑的文件路径
    old_string: z.string(),  // 要替换的旧内容
    new_string: z.string(),  // 替换成的新内容
  }),
  
  // 这个操作是只读的吗？不是，它会修改文件
  isReadOnly: () => false,
  
  // 权限检查：用户允许修改这个文件吗？
  checkPermissions: (input) => {
    // 检查文件路径是否在允许范围内
    // 检查权限模式（是否需要用户确认）
  },
  
  // 真正执行编辑
  call: async (input) => {
    const content = readFile(input.path)
    const newContent = content.replace(input.old_string, input.new_string)
    writeFile(input.path, newContent)
    return { success: true }
  },
}
```

### 4.4 工具分类

```
只读工具（安全，不需要确认）：
├── FileReadTool   — 读文件
├── GlobTool       — 搜索文件名
├── GrepTool       — 搜索文件内容
├── LSPTool        — 代码智能（跳转定义等）
└── WebSearchTool  — 网页搜索

写入工具（会改东西，通常需要确认）：
├── FileWriteTool  — 创建/覆盖文件
├── FileEditTool   — 修改文件的一部分
└── NotebookEditTool — 编辑 Jupyter Notebook

执行工具（最危险，一定要确认）：
├── BashTool       — 执行任意 Shell 命令
└── PowerShellTool — 执行 PowerShell 命令

高级工具：
├── AgentTool      — 创建子 AI 助手
├── MCPTool        — 调用外部 MCP 服务器的工具
└── TaskCreateTool — 创建后台任务
```

### 4.5 工具执行的并行优化

如果 AI 一次要用多个工具，程序会智能地决定串行还是并行：

```
AI 说：我要同时读 3 个文件

程序判断：FileReadTool 是只读的，可以并行！

┌─ 并行执行 ──────────────────────┐
│  FileReadTool("a.ts") ──→ 结果1  │
│  FileReadTool("b.ts") ──→ 结果2  │  同时进行，更快！
│  FileReadTool("c.ts") ──→ 结果3  │
└─────────────────────────────────┘

AI 说：先写文件，再跑测试

程序判断：写文件不能并行（可能冲突），必须串行！

┌─ 串行执行 ──────────────────────┐
│  FileWriteTool("a.ts") ──→ 完成  │
│          ↓                       │
│  BashTool("npm test") ──→ 完成   │  一个接一个
└─────────────────────────────────┘
```

这个逻辑在 `services/tools/toolOrchestration.ts` 里：

```typescript
function partitionToolCalls(toolUseMessages) {
  // 把工具调用分成批次：
  // - 连续的只读工具 → 放一批，并行执行
  // - 写入工具 → 单独一批，串行执行
}
```

---

## 第五章：权限系统 — 防止 AI 搞破坏

AI 能执行命令、修改文件，这很强大但也很危险。
权限系统就是"安全锁"。

### 5.1 权限模式（你可以选择信任程度）

```
最严格 ◄──────────────────────────────────► 最宽松

plan        default      acceptEdits    auto    bypassPermissions
│            │              │             │           │
│ 只能看     │ 每次都问     │ 自动接受    │ AI判断    │ 什么都不问
│ 不能改     │ 你来确认     │ 代码编辑    │ 是否安全  │ 直接执行
```

- **plan 模式**：AI 只能读文件、搜索，不能修改任何东西。适合"先让我看看你打算怎么做"。
- **default 模式**：每次 AI 要做危险操作，都会弹出确认框让你选 Yes/No。
- **acceptEdits 模式**：代码编辑自动通过，但执行命令还是要确认。
- **auto 模式**：用另一个 AI 分类器来判断操作是否安全，安全的自动通过。
- **bypassPermissions 模式**：完全不检查权限。只在沙箱环境里用！

### 5.2 权限检查流程（每次工具调用都会经过）

```
AI 想执行: BashTool("rm -rf node_modules")
                    │
                    ▼
        ┌─ 检查权限规则 ─┐
        │                │
        │ 有 allow 规则？ │──是──→ 直接允许
        │                │
        │ 有 deny 规则？  │──是──→ 直接拒绝
        │                │
        └───────┬────────┘
                │ 没有匹配的规则
                ▼
        ┌─ 检查权限模式 ─┐
        │                │
        │ bypass 模式？   │──是──→ 直接允许
        │                │
        │ plan 模式？     │──是──→ 是只读操作吗？
        │                │        ├── 是 → 允许
        │                │        └── 否 → 拒绝
        │                │
        │ auto 模式？     │──是──→ 调用 AI 分类器判断
        │                │        ├── 安全 → 允许
        │                │        └── 不确定 → 问用户
        │                │
        │ default 模式？  │──是──→ 弹框问用户
        │                │
        └────────────────┘
                │
                ▼
        ┌─ 用户确认 ──────┐
        │                  │
        │  [允许] [拒绝]    │
        │  [允许并记住]     │  ← 下次同样的操作自动通过
        │                  │
        └──────────────────┘
```

### 5.3 BashTool 的特殊安全检查

BashTool 是最危险的工具（能执行任意命令），所以有额外的安全层：

```
AI 想执行: BashTool("rm -rf /")
                    │
                    ▼
    ┌─ 破坏性命令检测 ─┐
    │ rm -rf / ？      │──→ 警告！这会删除整个系统！
    └──────────────────┘
                    │
                    ▼
    ┌─ 路径安全检查 ──┐
    │ 路径在项目外？   │──→ 拒绝！不能操作项目外的文件
    └──────────────────┘
                    │
                    ▼
    ┌─ 命令语义分析 ──┐
    │ 这是读操作？     │──→ 安全级别：低
    │ 这是写操作？     │──→ 安全级别：中
    │ 这是删除操作？   │──→ 安全级别：高
    └──────────────────┘
```


---

## 第六章：上下文管理 — 解决"AI 记不住"的问题

### 6.1 问题是什么？

AI 模型有"上下文窗口"限制。就像人的短期记忆有限：
- Claude 的上下文窗口大约 200K tokens（大约 15 万字）
- 一次长对话可能产生几十万字的内容（代码文件、命令输出等）
- 超过限制，AI 就"记不住"了，会报错

### 6.2 Claude Code 的解决方案：多层压缩

想象你的书桌（= 上下文窗口）只能放 10 本书，但你需要参考 50 本：

```
第 1 层：工具结果预算（Tool Result Budget）
  "这本书太厚了，只保留目录和关键章节"
  → 把超大的工具输出截断或替换为摘要

第 2 层：历史裁剪（Snip Compact）
  "把最早的几本书收起来，只留最近的"
  → 删除旧的、不重要的消息

第 3 层：微压缩（Microcompact）
  "把每本书里的废话删掉，只留干货"
  → 压缩单条消息内的冗余内容

第 4 层：上下文折叠（Context Collapse）
  "把 3 本相关的书合并成 1 本笔记"
  → 将多轮交互折叠为摘要

第 5 层：自动压缩（Auto Compact）
  "把所有书的内容写成一份总结报告"
  → 调用 AI 生成整个会话的摘要，替换所有历史

第 6 层：响应式压缩（Reactive Compact）
  "书桌实在放不下了，紧急清理！"
  → API 报错 prompt_too_long 时的最后手段
```

### 6.3 自动压缩是怎么工作的？

```
每次调用 AI 之前：
    │
    ▼
计算当前 token 数量
    │
    ├── < 80% 上下文窗口 → 正常，不压缩
    │
    ├── 80% ~ 95% → 触发自动压缩
    │   │
    │   ├── 把所有对话历史发给另一个 AI
    │   ├── 让它生成一份摘要
    │   └── 用摘要替换所有历史消息
    │
    └── > 95% → 阻止发送，提示用户手动 /compact
```

这就是为什么你在长对话中有时会看到"Context compacted"的提示。

---

## 第七章：API 调用 — 和 Claude 模型通信

### 7.1 调用流程

```
程序                              Anthropic 服务器
  │                                     │
  │  POST /v1/messages                  │
  │  {                                  │
  │    model: "claude-sonnet-4-6",      │
  │    messages: [...对话历史...],        │
  │    tools: [...可用工具列表...],       │
  │    system: "你是一个编程助手...",      │
  │    stream: true                     │
  │  }                                  │
  │ ──────────────────────────────────→ │
  │                                     │
  │  data: {"type":"content_block_start"}│
  │ ←────────────────────────────────── │
  │  data: {"type":"content_block_delta",│
  │         "text":"让"}                 │
  │ ←────────────────────────────────── │
  │  data: {"type":"content_block_delta",│
  │         "text":"我"}                 │
  │ ←────────────────────────────────── │
  │  ... (一个字一个字传回来)             │
  │ ←────────────────────────────────── │
  │  data: {"type":"message_stop"}      │
  │ ←────────────────────────────────── │
```

### 7.2 Prompt 缓存

每次调用 API 都要发送完整的对话历史，这很浪费。
Anthropic 的 API 支持 **Prompt Caching**：

```
第 1 次调用：发送 system prompt + 10 条消息 → 服务器缓存了
第 2 次调用：发送 system prompt + 10 条消息 + 1 条新消息
             → 服务器发现前 11 条和缓存一样，只处理新的 1 条
             → 更快、更便宜！
```

代码里通过 `cache_control` 标记来告诉服务器哪些内容可以缓存：

```typescript
// claude.ts
function addCacheBreakpoints(messages, systemBlocks) {
  // 在 system prompt 末尾标记：这部分可以缓存
  // 在最近的消息标记：到这里为止可以缓存
}
```

### 7.3 重试和降级

```
调用 API
  │
  ├── 成功 → 返回结果
  │
  ├── 429 (限流) → 等一会儿再试
  │
  ├── 500 (服务器错误) → 重试几次
  │
  ├── 模型过载 → 切换到 fallback 模型
  │   比如 Opus 过载了，自动切到 Sonnet
  │
  └── 流式失败 → 切换到非流式模式重试
```

---

## 第八章：UI 层 — 终端里的 React

### 8.1 为什么在终端里用 React？

终端 UI 看起来简单，但实际上很复杂：
- 消息列表要能滚动
- 代码要有语法高亮
- 工具调用要有进度动画
- 要处理键盘快捷键
- 要响应终端大小变化

React 的组件化思想非常适合管理这种复杂 UI。
Ink 库把 React 的 `<div>` 变成终端里的文字块。

### 8.2 组件树

```
<App>                          ← 最外层，提供全局状态
  <FpsMetricsProvider>         ← 性能监控
    <StatsProvider>            ← 统计数据（token 数、费用）
      <AppStateProvider>       ← 全局应用状态
        <REPL>                 ← 主界面
          <Messages>           ← 消息列表
            <Message>          ← 单条消息
              <Markdown>       ← Markdown 渲染
              <HighlightedCode>← 代码高亮
              <StructuredDiff> ← 文件修改的 diff 显示
            </Message>
          </Messages>
          <StatusLine>         ← 底部状态栏（token/费用/模型）
          <TextInput>          ← 输入框
          <各种 Dialog>        ← 权限确认框、设置框等
        </REPL>
      </AppStateProvider>
    </StatsProvider>
  </FpsMetricsProvider>
</App>
```

### 8.3 状态管理

Claude Code 没有用 Redux 或 MobX 这些大型状态管理库。
它自己写了一个极简的 Store（只有 30 行代码！）：

```typescript
// state/store.ts — 完整代码就这么多
function createStore(initialState, onChange) {
  let state = initialState           // 当前状态
  const listeners = new Set()        // 监听者列表
  
  return {
    getState: () => state,           // 获取状态
    
    setState: (updater) => {         // 更新状态
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 没变就不通知
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()  // 通知所有监听者
    },
    
    subscribe: (listener) => {       // 订阅变化
      listeners.add(listener)
      return () => listeners.delete(listener)  // 返回取消订阅函数
    },
  }
}
```

这个模式叫"发布-订阅"（Pub-Sub）：
- 状态变了 → 通知所有订阅者 → React 组件重新渲染

---

## 第九章：其他重要子系统

### 9.1 Bridge — IDE 集成

Claude Code 不只能在终端里用，还能嵌入 VS Code 和 JetBrains。
Bridge 就是连接 IDE 和 CLI 的"桥梁"。

```
VS Code 扩展                    Claude Code CLI
    │                                │
    │  "用户在编辑器里选中了代码"       │
    │ ──────── WebSocket ──────────→ │
    │                                │
    │  "AI 想修改这个文件"             │
    │ ←──────── WebSocket ────────── │
    │                                │
    │  在编辑器里显示 diff             │
    │  用户点击"接受"                  │
    │ ──────── WebSocket ──────────→ │
```

### 9.2 MCP — 扩展 AI 的能力

MCP（Model Context Protocol）是一个标准协议，
让 AI 可以连接外部工具服务器。

比如你可以连接一个数据库 MCP 服务器，
AI 就能直接查询数据库了。

```
Claude Code ←→ MCP 协议 ←→ 数据库 MCP 服务器 ←→ PostgreSQL
Claude Code ←→ MCP 协议 ←→ GitHub MCP 服务器 ←→ GitHub API
Claude Code ←→ MCP 协议 ←→ 你自己写的 MCP 服务器 ←→ 任何东西
```

### 9.3 子智能体 — AI 的分身

AgentTool 可以创建"子 AI"，让它们并行工作：

```
主 AI: "我需要同时做 3 件事"
  │
  ├── 创建子智能体 A: "你去调研技术方案"
  ├── 创建子智能体 B: "你去写单元测试"
  └── 创建子智能体 C: "你去更新文档"
  
  （三个子智能体同时工作）
  
  子智能体 A: "调研完了，推荐用方案 X"
  子智能体 B: "测试写好了，全部通过"
  子智能体 C: "文档更新完了"
  
主 AI: "好的，所有任务都完成了"
```

### 9.4 CLAUDE.md — 项目级指令

就像 `.gitignore` 告诉 Git 忽略哪些文件，
`CLAUDE.md` 告诉 Claude Code 关于你项目的信息：

```markdown
# CLAUDE.md 示例

## 项目说明
这是一个 React + TypeScript 项目。

## 代码规范
- 使用 4 空格缩进
- 组件用 PascalCase 命名
- 工具函数用 camelCase 命名

## 测试
运行测试：npm test
测试框架：Jest

## 注意事项
- 不要修改 src/legacy/ 目录下的文件
- API Key 存在 .env 文件中，不要提交
```

Claude Code 每次对话都会自动读取这个文件，
让 AI 了解你的项目规范。

---

## 第十章：Feature Flags — 功能开关

### 10.1 什么是 Feature Flag？

就像你手机上的"开发者选项"，可以打开/关闭某些功能。

Claude Code 用两种 Feature Flag：

**编译时 Flag**（`bun:bundle` 的 `feature()`）：
```typescript
if (feature('VOICE_MODE')) {
  // 语音模式的代码
  // 如果 VOICE_MODE 没开，这段代码在构建时就被删掉了
  // 最终的程序里根本不包含这段代码
}
```

**运行时 Flag**（GrowthBook）：
```typescript
if (checkFeatureGate('new_ui_experiment')) {
  // 新 UI 实验
  // 代码还在，但运行时根据服务器配置决定是否执行
  // 可以对不同用户开放不同功能（A/B 测试）
}
```

### 10.2 已知的 Feature Flags

| Flag | 功能 | 说明 |
|------|------|------|
| `VOICE_MODE` | 语音输入 | 用语音和 AI 对话 |
| `PROACTIVE` | 主动模式 | AI 主动监控文件变化 |
| `BRIDGE_MODE` | IDE 桥接 | 连接 VS Code/JetBrains |
| `DAEMON` | 守护进程 | 后台常驻运行 |
| `KAIROS` | 助手模式 | 未知的内部功能 |
| `REACTIVE_COMPACT` | 响应式压缩 | API 报错时自动压缩 |
| `CONTEXT_COLLAPSE` | 上下文折叠 | 智能折叠历史消息 |
| `TOKEN_BUDGET` | Token 预算 | 控制单次任务的 token 用量 |

---

## 第十一章：项目文件导航地图

当你想了解某个功能时，去哪里找代码：

```
"我想了解 AI 怎么调用 API"
  → src/services/api/claude.ts

"我想了解核心循环怎么工作"
  → src/query.ts

"我想了解某个工具怎么实现"
  → src/tools/[工具名]/

"我想了解权限怎么检查"
  → src/hooks/toolPermission/
  → src/types/permissions.ts

"我想了解 UI 怎么渲染"
  → src/components/
  → src/screens/REPL.tsx

"我想了解命令怎么注册"
  → src/commands.ts
  → src/commands/[命令名]/

"我想了解上下文压缩"
  → src/services/compact/

"我想了解 IDE 集成"
  → src/bridge/

"我想了解 MCP 协议"
  → src/services/mcp/

"我想了解状态管理"
  → src/state/

"我想了解配置系统"
  → src/utils/config.ts
  → src/schemas/

"我想了解消息类型定义"
  → src/types/
```

---

## 第十二章：建议的学习顺序

### 阶段 1：理解大局（1-2 天）
1. 读这份文档
2. 浏览 `src/main.tsx` — 不用看懂每一行，理解流程就行
3. 读 `src/query.ts` — 核心循环，最重要的文件

### 阶段 2：理解工具系统（2-3 天）
4. 读 `src/Tool.ts` — 工具接口定义
5. 读一个简单工具：`src/tools/GrepTool/` — 最简单的工具之一
6. 读一个复杂工具：`src/tools/BashTool/` — 最复杂的工具

### 阶段 3：理解 API 和 UI（2-3 天）
7. 读 `src/services/api/claude.ts` — API 调用
8. 读 `src/state/store.ts` — 状态管理（只有 30 行）
9. 读 `src/screens/REPL.tsx` — 主界面

### 阶段 4：深入子系统（按兴趣选择）
10. 权限系统：`src/types/permissions.ts` + `src/hooks/toolPermission/`
11. 上下文压缩：`src/services/compact/`
12. 子智能体：`src/tools/AgentTool/`
13. MCP 协议：`src/services/mcp/`
14. IDE 集成：`src/bridge/`

### 前置知识建议

如果你对某些技术完全不了解，建议先学：
- **TypeScript 基础**：类型注解、接口、泛型、async/await
- **React 基础**：组件、props、state、hooks（useState, useEffect）
- **异步编程**：Promise、async/await、生成器（function*）
- **命令行基础**：什么是 CLI、什么是参数解析
