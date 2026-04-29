面试场景下，讲这块内容的核心是**让面试官感受到你理解设计决策，而不只是背了一遍结构**。

---

**开场定位（30秒）**

先说清楚这是什么：

> "Claude Code 的多智能体系统本质上是让一个 AI 去调度其他 AI 来并行完成任务。它有两种模式——一种是轻量的子智能体，在同一进程内同步执行；另一种是 Swarm 协调器模式，一个 Coordinator 异步管理多个 worker，类似分布式任务调度。"

---

**讲设计亮点（重点，1-2分钟）**

挑 2-3 个有深度的点，别逐一列功能：

**1. 工具权限隔离**
> "不同类型的子智能体有不同的工具白名单。比如 Explore 和 Plan 智能体完全禁用了写入工具，从 system prompt 层面就声明了只读，再加上代码层的 `filterToolsForAgent` 双重保障。这样即使 prompt 被注入攻击，子智能体也物理上无法修改文件。"

**2. verification 智能体的对抗性设计**
> "有一个专门的 verification 智能体，它的 system prompt 里明确列出了两种它自己容易犯的错误——'验证回避'和'被前80%迷惑'。每个检查项必须有实际运行的命令和输出，不能只读代码说'看起来没问题'。这是把 QA 的最佳实践直接编码进 AI 的行为约束里。"

**3. Coordinator 禁止懒委托**
> "Coordinator 的 system prompt 里有一条很有意思的规则：禁止写 'Based on your findings, fix it' 这种话。必须自己理解 worker 的结果，写出包含具体文件路径和行号的 spec 再下发。这是在 prompt 层面强制要求 AI 真正理解问题，而不是把理解责任转嫁给下游。"

---

**收尾（15秒）**

> "整体来看，这套系统的设计思路是：用结构化约束（工具过滤、prompt 规则）来弥补 LLM 行为的不确定性，而不是完全依赖模型自律。"

---

**如果面试官追问**，几个可能的方向准备一下：

- *"怎么防止 worker 无限嵌套？"* → `isAsync` 标志 + `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` 白名单，in-process teammate 可以派发同步子智能体但不能再派发异步后台智能体
- *"worker 之间怎么通信？"* → 不直接通信，全部通过 Coordinator 中转，worker 完成后发 `<task-notification>` XML，Coordinator 读取后决定下一步
- *"为什么 Explore 用 haiku 而不是主模型？"* → 速度和成本，探索任务量大且结果只是中间产物，不需要最强模型