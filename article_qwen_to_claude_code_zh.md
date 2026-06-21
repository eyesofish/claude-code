# 从一次千问 AI Coding 测试的失败，看 Claude Code 怎么做 Agent 失败恢复

## 一、现场：阿里 AI Coding 测试

我在阿里参加 AI Coding 测试，被测的是通义千问（Qwen）的 Coding Agent。我观察到一个非常具体的失败序列：

1. 我给 Qwen 一个文件路径
2. `Read` 工具 → 失败；重试 → 失败；第三次 → 失败
3. Qwen 说"Read 似乎坏了，我用 `cat` 试试"
4. `cat` 成功

看起来 fallback 还算合理。真正反常的事情发生在下一轮：**同 session 内**给它另一个路径，Qwen 不是直接用 `cat`，而是**从第 1 步重新开始**：调 `Read` → 失败 → 重试 → 失败 → 重试 → 失败 → 才回到 `cat`。每个新路径都触发完整的 6 步循环。Session 内零跨轮学习。

## 二、为什么反常 + 两层诊断

反常的不是"重试三次"——三次是合理 retry 策略。反常的是 message history 里**明明白白**写着前几轮 `Read` 失败、`cat` 成功，模型却把每一轮都当成全新尝试。这件事让我开始想：**怎么设计一个 Agent，让它从同 session 的失败里把"教训"沉淀下来？** 我带着这个问题去读了 Claude Code 的源码（包发布到 npm 后源码可获取）。

把 Qwen 的失败拆开是两个独立缺失：

**(a) Runtime 层**：没有 session 级"这个工具不靠谱"的显式状态。Qwen 有**单轮** retry 计数器（所以三次后放弃），但不跨轮持久化。

**(b) Model 层**：即使 message history 堆满了过去的 `Read` 失败，模型也不会抽取"这个工具不可靠"作为下一轮先验。这不是 prompt engineering 能修的事——是训练分布问题，模型没有被训练成"从历史 tool_result 提取元知识"。

Claude Code 用了两条互补的防线，正好对应这两层。

## 三、Claude Code 防御 #1：denialTracking 的双阈值熔断

源码：`src/utils/permissions/denialTracking.ts`，整个文件只有 46 行。

第 12-15 行定义两个阈值：

```ts
export const DENIAL_LIMITS = {
  maxConsecutive: 3,
  maxTotal: 20,
} as const
```

`maxConsecutive: 3` 是**可恢复**计数，遇到一次成功就清零（`recordSuccess` 在 32-38 行）。`maxTotal: 20` 是**不可恢复**计数，永远只增不减。

触发条件在 40-45 行：

```ts
export function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  )
}
```

两个细节：**第一**，fallback 到的不是"换一个工具"而是**回去问人**——反复撞墙时正确姿势是承认搞不定，不是自己瞎试。**第二**，所有更新函数都返回 `{ ...state, ... }` 不原地修改——标准不可变状态机，可审计、可测试。

这条防线对应 Qwen 缺的 **(a) runtime 显式状态**，但只覆盖"短时间爆发型"撞墙。要把教训跨轮沉淀下来，需要第二条防线。

## 四、Claude Code 防御 #2：Session Memory 模板

源码：`src/services/SessionMemory/prompts.ts`，第 11-41 行的 `DEFAULT_SESSION_MEMORY_TEMPLATE` 是一个 markdown 模板。直接相关的三节（斜体描述是模板自带的 prompt 指令）：

```markdown
# Workflow
_What bash commands are usually run and in what order?
How to interpret their output if not obvious?_

# Errors & Corrections
_Errors encountered and how they were fixed. What did the user correct?
What approaches failed and should not be tried again?_

# Learnings
_What has worked well? What has not? What to avoid?_
```

这三节合起来把"工具反复失败"捕获干净：**Workflow** 隐含"哪个工具走得通"；**Errors & Corrections** 直接问"哪些尝试失败、下次不该再试"；**Learnings** 兜底总结。把 Qwen 那个 session 喂进来，提取出的 Errors & Corrections 一定会写："`Read` 持续失败，应使用 `cat` 替代"——下一轮加载到 context，问题就消失了。

## 五、关键洞察：提取器里**没有**专门代码

我一开始以为捕获"工具失败"这种元知识肯定有专门子系统在做模式识别。结果看 `buildSessionMemoryUpdatePrompt`（第 226-247 行），这个函数只干三件事：加载模板、算 section 大小加 budget reminder、替换 `{{notesPath}}` / `{{currentNotes}}`。

**完。没有任何一行代码在做"工具可靠性检测"。**

负责实际提取的是一个 fork 出来的 agent，工具被限制到只能 Edit memory 文件。它根据**模板里斜体描述的 prompt 指令**和对话历史，把内容填进对应 section。Anthropic 没有写"工具失败检测器"，他们把"反思失败"编码成了几行斜体 prompt。

## 六、提炼：schema-driven extraction

我给这种模式起个名字：**schema-driven extraction**。

- **结构（markdown 章节）即行为**，不是代码即行为
- **轻**：核心约 30 行 markdown
- **泛化好**：不需要预先枚举"有哪些失败模式"，schema 只要求"反思失败"，具体捕获什么由 extractor 现场判断
- **代价**：依赖 extractor 判断力，没有强保证

对比传统状态机：一个"工具可靠性追踪子系统"要几百行代码、枚举失败类型、决策矩阵——但只能覆盖**设计时想得到**的失败。Schema 驱动反过来：用模型判断换覆盖"设计时想不到的"失败。

模板里那几句斜体话不是注释，不是文档——它**就是这套系统的行为定义**。

## 七、两条防线如何互补

| 层 | 机制 | 覆盖什么 | Qwen |
|---|---|---|---|
| Runtime | denialTracking 显式状态 | 同 session 爆发撞墙 → fallback 给人 | 缺 |
| Schema | Session Memory 模板 | 长期教训 → 下一轮加载回 context | 缺 |

denialTracking 处理**当下**（连续 3 次或累计 20 次熔断）。Session memory 处理**未来**（这轮教训结构化沉淀，下轮自动加载）。两条加起来覆盖 Qwen 失败的两个层面——Qwen 两边都没有，所以每个新路径都触发完整 6 步循环。

## 八、给读者的迁移启发

如果你要给自己的 agent 设计失败恢复：

1. **runtime 层**：给关键失败维护一个显式不可变状态机，触发阈值时 fallback 到"问人"。代码可以很短，46 行就够。
2. **schema 层**：定义 markdown 模板，把"反思什么"用 prompt 写进 section 描述里，fork 一个受限 agent 填表。
3. 不要写"工具可靠性追踪器"。写模板，让 extractor 自己判断。

这个模式不只适用于文件读取失败。API 调用、数据库查询、外部命令——都是同一套思路：runtime 层熔断 + schema 层把教训写回 context。

我不觉得 Qwen 的失败是团队能力问题，更像是设计优先级和训练分布的差异。但作为用户，这次测试给了我一个非常具体的"反例"，让我去读 Claude Code 时知道该看什么。源码就摆在那里。值得读。
