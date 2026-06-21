# claude-code 带读笔记与面试手册（Agent 专题）

> 路径说明：下面所有源码路径默认以 `D:\Github\claude_code_learning\claude-code` 为仓库根目录。
>
> 使用方式：checkbox 仍表示带读进度，不代表题目有没有答案。这里已经把每个面试题补成“面试官问法 + 代码解释 + 面试者回答 + 追问”。复习时先自己答，再看答案。
>
> 阅读主线：Agent 循环 -> Tool -> Permission -> Task/Sub-Agent -> Skill/MCP -> Memory/Compact -> Prompt/Cache -> Session Resume -> 测试与架构取舍。

## 0. 先把 Claude Code 当成什么系统

Claude Code 不是一个普通 CLI，也不是一个只会调用模型的薄壳。它更像一个能在终端里行动的 Agent runtime。

最小心智模型：

```text
用户输入
  -> QueryEngine 组装上下文
  -> query() 调 Anthropic API
  -> 模型返回文本或 tool_use
  -> 工具系统校验、权限判断、执行
  -> tool_result 写回消息历史
  -> 再调模型
  -> 直到模型不再要工具，输出最终回答
```

面试里不要背“它用了 TypeScript、Bun、Ink”。那只是表层。真正要讲清楚的是：它如何让 LLM 安全地使用本地机器能力。

---

## 1. Agent 核心循环：QueryEngine

一句话：Agent 就是一个循环。模型决定下一步，工具执行，结果再喂回模型。

### 代码地图

- `src/QueryEngine.ts:1186`：`ask()` 入口，一次 headless/SDK 调用从这里进来。
- `src/QueryEngine.ts:218`：`QueryEngine.submitMessage()` 是单轮用户消息的核心入口。
- `src/QueryEngine.ts:318-355`：加载 memory、拼 system prompt、决定 thinking config。
- `src/QueryEngine.ts:675-677`：把 prompt、messages、tools 交给 `query()`。
- `src/query.ts:219-254`：`query()` generator 参数入口。
- `src/query.ts:554-570`：不能只信 `stop_reason === "tool_use"`，要自己扫描 tool_use block。
- `src/query.ts:833-858`：流式响应里收集 tool_use。
- `src/query.ts:1382-1399`：`runTools(...)` 执行工具并生成 tool_result。
- `src/query.ts:1673-1717`：把 assistant message 和 tool_result 带进下一轮模型调用。
- `src/services/api/errors.ts:1060-1079`：tool_use/tool_result 不匹配、重复 tool_use id 的错误分类。
- `src/services/api/errors.ts:1163`：`categorizeRetryableAPIError()` 重试判定入口。

### 1.1 主循环架构

- [x] **前置 1.P1** `src/QueryEngine.ts:1-50` 的依赖图：`query()` / `getSlashCommandToolSkills()` / `loadMemoryPrompt()` / `categorizeRetryableAPIError` 各管什么？拼起来是什么流程？

  **面试官会怎么问**：`QueryEngine.ts` 开头 import 这么多东西，你能从 import 反推出它的职责吗？

  **代码解释**：`query()` 管 LLM 和工具的往返；`loadMemoryPrompt()` 把长期记忆塞进 system prompt；`getSlashCommandToolSkills()` 把 skill 列表变成模型能看到的上下文；`categorizeRetryableAPIError()` 用来判断 API 错误能不能重试。

  **傻瓜理解**：`QueryEngine` 像餐厅前台。它不亲自做菜，但它决定菜单、把客人的话写清楚、把请求交给后厨、后厨出错时决定要不要重做。

  **面试者回答**：我会把 `QueryEngine` 看成会话层。它负责把用户输入、memory、system prompt、tools、permission context 组装好，然后交给 `query()`。`query()` 才是真正跑 LLM/tool loop 的管线。错误分类单独放在 API error 层，避免主流程里到处写 status code 判断。

  **继续追问**：为什么 `query()` 不直接负责 memory？因为 memory 是会话上下文，不是单次 LLM/tool loop 的内核。

- [ ] **主问 1.Q1** 一条用户消息进去，到最终回复出来，Agent 循环的完整步骤是什么？

  **面试官会怎么问**：你不用讲概念，按代码路径讲。用户发一句话以后，Claude Code 到底跑了哪些函数？

  **代码解释**：`ask()` 创建或调用 `QueryEngine`，`submitMessage()` 先准备 system prompt、memory、tools、AppState，再调用 `query()`。`query()` 里请求 Anthropic API，流式解析 assistant 输出。如果发现 `tool_use`，它把 tool_use 收集成执行输入，交给 `runTools(...)`。工具结果变成 `tool_result` message，再和 assistant message 一起作为下一轮输入。循环直到 assistant 输出普通文本，不再产生 tool_use。

  **面试者回答**：完整链路是：`ask()` 进来，`submitMessage()` 拼上下文，`query()` 发模型请求，流式解析响应。如果模型要用工具，就把 tool_use 规范化，走工具执行管线和权限检查，然后把结果作为 tool_result 放回消息历史。下一轮模型看到 tool_result 后继续推理。最终没有 tool_use 时，循环结束，返回文本和 SDK events。

  **继续追问**：如果模型返回多个 tool_use，不能一个个随便跑，要看工具是否并发安全，这会进入 `runTools` 的编排逻辑。

- [ ] **主问 1.Q2** `src/query.ts` 的 `query()` 和 `src/QueryEngine.ts` 的分工边界在哪？为什么 query pipeline 要拆成独立模块？

  **面试官会怎么问**：为什么不把所有逻辑都塞进 `QueryEngine`？拆出 `query.ts` 的收益是什么？

  **代码解释**：`QueryEngine` 维护会话级状态：messages、usage、permission denials、file cache、SDK 状态。`query()` 更像一次可迭代的运行管线，吃 system prompt、messages、tools、tool context，然后产出 message stream。

  **面试者回答**：我会把它们分成 outer shell 和 inner loop。`QueryEngine` 管“这一段对话是谁、在哪个 cwd、有哪些配置”；`query()` 管“这一轮怎么调模型、怎么解析 tool_use、怎么执行工具、怎么继续下一轮”。拆开以后 SDK/headless/REPL 都能复用内层 loop，外层只负责不同入口的状态适配。

  **踩坑点**：不要把 `query()` 说成 HTTP client。它不是裸 API 调用，它包含 tool loop。

- [x] **主问 1.Q3** 工具调用循环的终止条件是什么？LLM 怎么决定“不需要再调工具了”？有没有最大轮次保护？

  **面试官会怎么问**：这个 Agent loop 会不会无限调工具？代码里怎么停？

  **代码解释**：一轮 assistant 响应中如果没有 tool_use block，就进入最终输出路径。`QueryEngineConfig` 里有 `maxTurns`，`query()` 会用这个约束多轮调用。`src/query.ts:554-570` 说明不能只靠 API 的 stop_reason，因为流式场景下需要自己检查 content blocks。

  **面试者回答**：模型不是显式返回“停止循环”命令。它只要不再输出 tool_use，系统就把这轮视为最终回答。工程上还要有 `maxTurns` 这样的保护，避免模型一直“查一点、再查一点”。所以终止有两层：语义上是模型不再请求工具，工程上是 turn budget 兜底。

  **继续追问**：如果最后一轮还有未完成工具，要生成 synthetic tool_result，避免 API 下一次因为 orphan tool_use 报错。

- [ ] **主问 1.Q4** thinking mode 在这一层怎么体现？`ThinkingConfig` 控制的是 prompt 注入还是 API 参数？

  **面试官会怎么问**：extended thinking 是写进 prompt 里，还是发给 API 的参数？

  **代码解释**：`src/QueryEngine.ts:318-355` 附近会决定 `initialThinkingConfig`，随后它进入 `query()` 的参数和 toolUseContext。它不是简单一句“请多思考”的 prompt，而是运行时配置的一部分。子 agent 场景还会选择继承或关闭 thinking。

  **面试者回答**：thinking config 是运行配置，不只是提示词文本。外层 `QueryEngine` 决定默认是否开启，内层 `query()` 和 API 请求会按这个配置处理。它也会影响 prompt cache key，因为 thinking config 是请求形态的一部分。子 agent 默认常常关闭 thinking，是为了降低成本和延迟，除非 fork child 需要精确保留主上下文。

  **踩坑点**：不要说“thinking 就是 system prompt 里加一句话”。那太浅。

### 1.2 错误与中断

- [ ] **前置 1.P2** `src/services/api/errors.ts` 里 `categorizeRetryableAPIError` 的判定逻辑是什么？哪些 StatusCode / error type 是可重试的？

  **面试官会怎么问**：哪些 API 错误该重试，哪些不该？

  **代码解释**：`categorizeRetryableAPIError()` 把 rate limit、网络抖动、5xx 这类临时错误和 tool_use 结构错误分开。结构错误比如 tool_result 顺序不对、重复 tool_use id，不是重试能解决的。

  **面试者回答**：重试只适合临时失败，比如 429、5xx、网络连接问题。协议级错误不能盲目重试，因为同样的 messages 再发一次还是错。这里的关键是错误分类要靠近 API 层，主循环只消费“可重试/不可重试/需要修复消息结构”的结果。

  **继续追问**：如果 tool_use mismatch 是历史消息构造错了，正确做法是修消息，不是 sleep 后重试。

- [ ] **主问 1.Q5** Agent 循环中途 API 挂了，怎么恢复？重试策略是什么？

  **面试官会怎么问**：模型调用到一半 500 了，你怎么保证用户体验不炸？

  **代码解释**：API 错误进入 `categorizeRetryableAPIError()`，可重试错误按策略重发。会话本身由 `QueryEngine` 的 mutable messages 和 session transcript 管着。已经落盘的历史可以恢复，未完成的一轮需要谨慎处理，避免重复提交工具副作用。

  **面试者回答**：我会先区分失败发生在模型调用前、流式输出中、工具执行后。模型调用前失败可以重试；工具已经执行后就不能随便重放同一个 tool_use，否则可能重复写文件或重复跑命令。Claude Code 的思路是分类 API 错误，并通过 transcript/session 恢复尽量保留已确认状态。

  **踩坑点**：重试不等于重跑整个 turn。含副作用的工具必须特别小心。

- [ ] **主问 1.Q6** `OrphanedPermission` 是什么？什么场景下权限请求会变成“孤儿”？

  **面试官会怎么问**：用户还没点权限弹窗，主循环就变了，会发生什么？

  **代码解释**：`src/types/textInputTypes.ts:384` 定义 `OrphanedPermission`。当权限请求挂起，但对应 tool_use 或消息上下文已经被中断、替换或恢复，就会出现“孤儿权限”。`QueryEngine` 有 `orphanedPermission` 配置和处理逻辑，CLI 打印层也会处理这种状态。

  **面试者回答**：孤儿权限就是一个权限请求失去了原来的上下文。比如工具调用在等用户批准，但用户中断了会话或恢复了另一个 session。系统不能再把用户的“允许”套到旧 tool_use 上，否则可能放行错误操作。正确做法是识别它、提示或丢弃，而不是继续执行。

  **继续追问**：权限系统里最怕的不是拒绝，而是把一个批准应用到错的 tool call。

### 1.3 与项目对比

- [ ] **主问 1.Q7** software_reco agent 的主循环和 QueryEngine 比，少了哪些环节？

  **面试官会怎么问**：你自己的推荐系统如果叫 agent，它和 Claude Code 这种 agent 差在哪里？

  **代码解释**：Claude Code 有 prompt 组装、tool schema、权限、tool_result 回灌、session 恢复、compact、memory。普通 RAG/recommendation pipeline 多是固定流程：retrieve -> rerank -> generate。

  **面试者回答**：我的系统更像静态 pipeline：每个 stage 在代码里排好，LLM 主要负责生成。Claude Code 是动态 loop：LLM 可以根据中间结果决定下一步调哪个 tool。差距主要在三块：工具接口化、权限边界、状态恢复。如果我要升级，会先把 retrieval、rerank、profile lookup 封成工具，再加最大轮次和只读权限。

  **踩坑点**：不要为了“Agent 化”把稳定 pipeline 全交给 LLM。高频稳定路径应该保留硬编码。

- [ ] **主问 1.Q8** 如果把推荐系统每个 stage 都变成 tool，LangGraph StateGraph 怎么改？

  **面试官会怎么问**：你怎么用 LangGraph 复刻一个 mini QueryEngine？

  **代码解释**：QueryEngine 的“图”是动态的，LLM 的 tool_use 决定下一步。LangGraph 里可以设计 `llm_node -> tool_router -> tool_node -> llm_node` 的循环，State 里保存 messages、tool_results、turn_count、budget、permission decision。

  **面试者回答**：我会保留一个 LLM decision node，让它输出 tool call 或 final answer。conditional edge 根据输出跳到对应 tool node，tool node 把结果写回 State，再回 LLM node。为了安全，要加 `turn_count >= maxTurns` 的退出边，还要把写操作工具和只读工具分开审批。

  **继续追问**：Graph 是静态拓扑，但工具选择可以是动态路由。

> 带读笔记（已讨论）
>
> - **关于 Agent loop 怎么结束的当前理解（1.Q3 半成品）**：两条退出路径，OR 关系，谁先到谁触发。
>     - **自然终止（99% 走这条）**：assistant response 里**没有任何 `tool_use` block** → 把这次 response 当最终答案。注意 `src/query.ts:554-570` 那条注释——**不能只信 `stop_reason === 'tool_use'`**，流式场景下要自己扫 content blocks 里有没有 `tool_use`，这是易踩坑。
>     - **maxTurns 兜底熔断**：模型还想再调工具，但预算用完了。在 `src/query.ts:1705` 的 `if (maxTurns && nextTurnCount > maxTurns)` 判定。
> - **关于 turn 定义的当前理解**：turn = **模型生成 response 的次数**，不是 tool call 次数、不是用户消息次数。`query.ts:276` 初始 `turnCount: 1`，`query.ts:1679` 每次准备递归前 `nextTurnCount = turnCount + 1`。一个用户消息可能引发 N 个 turn（模型查一点 → 看结果 → 再查一点 → ... 直到不再调工具）。
>
> - **【面试 highlight】maxTurns 是配额，不是开关 —— "谁负责兜底谁负责传"**：
>
>     maxTurns 不是一个固定常量，三种来源对应三种"谁在调 Claude Code"：
>
>     1. **完全不传（undefined）= 你在终端聊天**：人在场能 Ctrl+C，不需要预算。`query.ts:1705` 的 `if (maxTurns && ...)` 自动短路。**类比：自己在家做饭，火大火小自己盯。**
>     2. **SDK / headless（调用方传）= 自动化脚本调 Claude Code**：CI 里凌晨自动 review PR，没人值守。`src/entrypoints/sdk/coreSchemas.ts:1148` 强制 schema 接收 `maxTurns: z.number().int().positive()`。**类比：雇代购去市场，必须给预算上限，否则可能买光整个市场。**
>     3. **AgentDefinition（agent 配置写死）= 主 agent 派子 agent 干活**：`src/tools/AgentTool/loadAgentsDir.ts:89, 648-654` 从 agent .md frontmatter 解析。子 agent 是临时工，预算写在它合同里。**类比：家里请的长期保姆，工作规则写在合同里，不用每次现场嘱咐。**
>
>     **30 秒口播版**：
>
>     > maxTurns 不是固定值，要看谁在调 Claude Code。普通用户在终端聊天，没人盯就不传，让模型自然结束；如果是 SDK 自动化脚本——比如 CI 里跑 Claude——调用方必须传，因为没人能 Ctrl+C；如果是主 agent 派子 agent 干活，子 agent 的上限写在它自己的配置文件里。本质就是"谁有责任兜底，谁就负责传这个数字"。
>
> - **可复用的设计原则**：**自然终止 vs 兜底熔断 是 agent loop 必备的双保险**。自然终止是语义层（模型自己说"我搞完了"），熔断是工程层（防止模型抽风永不收敛）。任何 LLM/tool loop 都得有这两层，缺一不可。这套思路跟 §3.Q2 denialTracking 的 `maxConsecutive: 3 + maxTotal: 20` 双阈值熔断、§8 session memory 的 schema-driven 防线，构成 Claude Code 的**三层防失控体系**：循环退出 / 权限疲劳 / 长期学习。
>
> - **【面试 highlight】流式输出的深层工程价值 —— "流式不只是给人看的，是给系统抢延迟的"**：
>
>     大部分人对流式的理解停留在 UX 层（"用户能看到打字效果"）。但 Claude Code 里流式有 4 层价值，**UX 只是最浅的一层**：
>
>     | # | 价值 | 即使没人看屏幕也成立？ |
>     |---|---|---|
>     | 1 | **边收 tool_use 边执行**（StreamingToolExecutor 并行启动工具） | ✅ **核心** |
>     | 2 | **可中途 abort**（Ctrl+C / maxTurns 一到立刻断流，不再 charge 后面 token） | ✅ |
>     | 3 | **大模型推理本来就是流式生成**（非流式 = 服务端缓冲后一次性返回，反而增加首字节延迟） | ✅ |
>     | 4 | **用户看到打字效果** | ❌（只有人在场才有用） |
>
>     **核心代码证据**：
>     - `src/query.ts:561-568` 的 `useStreamingToolExecution` feature gate + `StreamingToolExecutor`
>     - `src/services/tools/StreamingToolExecutor.ts:34-39` 注释：*"Executes tools as they stream in with concurrency control. Concurrent-safe tools can execute in parallel."*
>     - `src/query.ts:557-558` `toolUseBlocks: ToolUseBlock[]` 数组 + `needsFollowUp` flag 在流式中边收边记
>     - `src/query.ts:832-834` 收到每段 assistant message 就抽 tool_use block，有就 push + 立旗
>
>     **核心场景对比**（模型 5 秒内吐 3 个 tool_use）：
>
>     - **非流式**：等 5 秒拿到整段 → 才开始跑工具 → 假设并行 1 秒 → **总耗时 ≈ 6 秒**
>     - **流式**：t=0.3s 看到第 1 个 tool_use 就开跑 Grep，t=0.4s 看到第 2 个就并行启 Read，t=0.5s 看到第 3 个并行启 Glob → 流结束时工具可能已跑完 → **总耗时 ≈ 5 秒**
>     - **省 1-3 秒，纯靠"边接收边执行"**。子 agent 套娃场景下延迟节省会逐层放大。
>
>     **隐性收益**：流式让 `maxTurns` 能更精确熔断。非流式时模型生成一半 response 才被砍，钱已经花了一半；流式可以在收到第 N 个 tool_use 时立刻 abort，下游 token 完全不 charge。
>
>     **30 秒口播版（面试谈资）**：
>
>     > 大家对流式的认知一般停在 UX——用户能看打字效果。但读 Claude Code 源码会发现一个反直觉的点：**它的子 agent、tool call 这种机器对机器的场景仍然全程流式**。为什么？因为流式在这里不是给人看的，是给系统抢延迟的——`StreamingToolExecutor` 让模型每吐一个 tool_use block，系统就能并行启动那个工具，跟剩余 response 生成同时跑。一个 5 秒的 response 里有 3 个工具调用，流式能把"等响应 + 跑工具"从串行变并行，节省 1-3 秒。这把流式从"UX 特性"升级成了"工程并行原语"。配套收益是可中途 abort——maxTurns 一触发立刻断流，不再 charge 后面的 token，省钱。
>
>     **面试官追问准备**：
>
>     - "那为什么 OpenAI 这些 API 默认非流式？" → 历史原因，早期 chat completion 接口为简单设计；现在主流 LLM API 都把流式作为默认或推荐。
>     - "流式怎么保证 tool_use block 的完整性？" → API 协议里 tool_use 是结构化 content block，stream event 会标 `content_block_start / delta / stop`，本地累积到 `content_block_stop` 才算一个完整 block。**不是把 JSON 字符串拼接到一半就解析**。
>     - "并行执行工具不会有副作用冲突？" → 看 `TrackedTool.isConcurrencySafe`（`StreamingToolExecutor.ts:26`）和工具自带的 `isConcurrencySafe()` 方法。只读工具（Grep/Glob/Read）默认并发安全；Bash/Write/Edit 这种有副作用的会被串行化。
>
> - **1.Q3 A/B/C/D 闭环（答案 C）**：命中 `query.ts:1705` 时**不再调模型**，直接 `yield createAttachmentMessage({ type: 'max_turns_reached', ... })` 给外层 generator，然后 `return { reason: 'max_turns', turnCount }`。三个反直觉点：
>     1. **是 `return` 不是 `throw`**：max_turns 是预设的正常退出路径，不是异常。外层调用者（SDK / REPL / AgentTool）通过 `reason: 'max_turns'` 判断是不是兜底熔断退出，决定要不要给用户报错或继续。
>     2. **不会强制 abort 在跑的 tool**（B 错）：1705 行的判定在**新一轮模型调用前**做，此时上一轮的 tool 已经跑完、tool_result 已经在 messages 里了。abort 是另一条路径（用户 Ctrl+C / AbortController）。
>     3. **不会画蛇添足再调一次模型解释"已达上限"**（A 错）：那纯属浪费 token。max_turns_reached attachment 已经是结构化信号，外层 UI 自己渲染"已达 maxTurns"提示，不需要模型生成话术。D 更是无中生有——根本没有"截断 messages 让 assistant 回答"这条路径。
> - **还没展开的问题**：为什么是 `nextTurnCount > maxTurns` 而不是 `turnCount >= maxTurns`（差一位语义 / off-by-one）；1.Q4 thinking config 怎么影响 cache；StreamingToolExecutor 怎么处理 concurrency-unsafe 工具（串行化策略）。

---

## 2. 工具系统：Tool 的注册、描述与执行

一句话：Tool 是模型能伸出去的手。每只手都有名字、输入 schema、权限检查、执行逻辑和返回格式。

### 代码地图

- `src/Tool.ts:15-20`：`ToolInputJSONSchema`，给 API 和模型看的 JSON schema 外形。
- `src/Tool.ts:158-300`：`ToolUseContext`，工具执行时能拿到的运行上下文。
- `src/Tool.ts:321-330`：`ToolResult`。
- `src/Tool.ts:348-359`：`toolMatchesName()` 和 `findToolByName()`。
- `src/Tool.ts:362-397`：`Tool` 接口核心字段。
- `src/Tool.ts:730-780`：`buildTool()` 和默认行为，`isReadOnly` 默认 false。
- `src/tools.ts:195-229`：默认工具注册表。
- `src/tools.ts:287-341`：`getTools()` 按模式和 deny rule 过滤工具。
- `src/tools.ts:355-383`：`assembleToolPool()` 合并 built-in tools 和 MCP tools。
- `src/services/tools/toolExecution.ts:615-683`：Zod `safeParse()` 与 `validateInput()`。
- `src/services/tools/toolExecution.ts:916-1039`：permission check。
- `src/services/tools/toolExecution.ts:1207`：实际调用 `tool.call(...)`。
- `src/services/tools/toolExecution.ts:1404-1418`：工具返回值包装成 `tool_result`。

### 2.1 Tool 类型体系

- [x] **前置 2.P1** `ToolInputJSONSchema` 为什么是 `[x: string]: unknown` 这种宽松类型？它和 Zod schema 各自负责什么？

  **面试官会怎么问**：既然有 Zod，为什么还要 JSON schema？为什么 JSON schema 类型这么松？

  **代码解释**：`inputSchema` 是给模型/API看的函数参数描述，需要能表达 JSON Schema。TypeScript 无法把所有 JSON Schema 变体写成很严格的静态类型，所以这里用宽松对象承载。真正运行时校验靠 Zod 和每个 tool 的 `validateInput()`。

  **面试者回答**：JSON Schema 负责“告诉模型应该传什么”，Zod 负责“模型真的传了以后能不能过”。前者偏协议和提示，后者偏运行时安全。宽松类型是工程妥协，因为 JSON Schema 本身结构很开放；如果 TypeScript 写太死，工具作者反而不好表达 schema。

  **继续追问**：schema 不能替代权限。参数合法不代表操作安全。

- [ ] **主问 2.Q1** 拿 `GrepTool` 或 `BashTool` 拆一个完整 Tool 模块：必备哪些导出？

  **面试官会怎么问**：你要新增一个 Tool，最少要写哪些东西？

  **代码解释**：一个 Tool 至少要有 `name`、`description()`、`inputSchema`、`call()`。复杂工具还会有 `validateInput()`、`checkPermissions()`、`isReadOnly()`、`isConcurrencySafe()`、渲染方法和 progress 类型。

  **面试者回答**：我会从四件事写起：名字给模型匹配，description 教模型什么时候用，inputSchema 限制输入形状，call 执行真实逻辑。像 BashTool 这种高风险工具还必须重写权限检查和输入校验；像 GrepTool 这种只读工具可以标记 read-only，并提供更简单的权限路径。

  **踩坑点**：不要把 `execute()` 当唯一入口。这个代码里实际接口叫 `call()`。

- [x] **主问 2.Q2** `getTools()` 怎么把工具组装起来？注册、feature flag、MCP 动态注入怎么合并？

  **面试官会怎么问**：这 40 多个工具是不是每次都无脑发给模型？

  **代码解释**：`getAllBaseTools()` 汇总内置工具，带 feature flag、环境变量、用户类型判断。`getTools()` 再按 simple mode、deny rules、REPL mode、`isEnabled()` 过滤。MCP tools 不在 `getTools()` 里直接混，而是通过 `assembleToolPool()` 与 built-in tools 合并、排序、去重。

  **面试者回答**：工具池有三层：先是编译/运行条件决定“可能有哪些工具”，再由权限和模式过滤“这次能给模型看哪些工具”，最后把外部 MCP tool 合并进来。这样做的好处是模型只看到当前可用能力，deny 掉的工具甚至不会暴露 schema。

  **继续追问**：`assembleToolPool()` 还特意保持 built-in tools 的稳定排序，这是为了 prompt cache 稳定。

### 2.2 工具执行生命周期

- [ ] **前置 2.P2** `ToolUseContext` 包含哪些字段？工具执行时能拿到什么“世界信息”？

  **面试官会怎么问**：工具执行不是只拿 input，它还需要哪些上下文？

  **代码解释**：`ToolUseContext` 包含 cwd、permission context、messages、AppState getter/setter、abort controller、progress callback、MCP elicitations、任务预算等。工具靠它知道当前目录、权限模式、会话状态和取消信号。

  **面试者回答**：ToolUseContext 是工具的运行时环境。input 只告诉工具“这次要做什么”，context 告诉它“在哪里做、能不能做、做完怎么报告、被取消怎么办”。没有 context，Bash/Read/AgentTool 这些工具没法安全地运行。

  **踩坑点**：context 越大，工具越强，也越要小心权限边界。

- [x] **主问 2.Q3** 一次 tool_use 从 LLM 返回到 tool_result 回灌，每一步怎么跳转？

  **面试官会怎么问**：模型说要调用 Bash，代码里谁接住这个请求？

  **代码解释**：`query.ts` 收集 tool_use block，然后 `runTools(...)` 编排执行。执行前先通过 `findToolByName()` 或 `toolMatchesName()` 找工具，再进入 `checkPermissionsAndCallTool()`。这里先校验输入、跑 hooks、判权限、调用 `tool.call()`，最后把结果映射成 tool_result。

  **面试者回答**：路径是：LLM 输出 tool_use，query loop 收集它，工具编排层按并发安全性调度，tool execution 层做输入校验和权限判断，真正执行 `tool.call()`，结果统一包装成 Anthropic API 能接受的 tool_result，再放回 messages。下一轮 LLM 就像看到“工具回答了”一样继续推理。

  **继续追问**：不存在的 tool name 也不能让系统崩，要返回结构化错误给模型。

- [ ] **主问 2.Q4** `ToolProgressData` union type 怎么让每种工具有自己的进度通知格式？

  **面试官会怎么问**：Bash 有 stdout，AgentTool 有子任务进度，WebSearch 有搜索状态，怎么统一渲染？

  **代码解释**：`ToolProgressData` 是联合类型，包含 BashProgress、AgentToolProgress、MCPProgress、SkillToolProgress、WebSearchProgress、TaskOutputProgress 等。工具通过 `onProgress` 推送自己的 progress payload，UI 根据 tool 类型和 render 方法展示。

  **面试者回答**：它不是把所有进度压成一个通用字符串，而是给每类工具保留自己的结构。统一的是“进度事件”这个外壳，不统一的是事件内部字段。这样 UI 能对 Bash 流式输出、Agent 子任务、MCP 状态做差异化展示。

  **项目类比**：推荐系统如果有 streaming rerank、LLM generate、background crawl，也可以用 union type 表达不同阶段的进度。

### 2.3 Tool 设计模式

- [ ] **主问 2.Q5** 哪些是“原始能力”工具，哪些是“Agent 能力”工具？

  **面试官会怎么问**：工具列表里 Bash、Read、AgentTool、TaskCreate 都叫 tool，它们是一类东西吗？

  **代码解释**：Bash、Read、Edit、Write、Grep、Glob 是操作环境的原始能力。AgentTool、SkillTool、TaskCreate、TeamCreate、PlanMode 是编排类能力，改变的是 agent 自身的运行方式或任务结构。

  **面试者回答**：我会分成两层：primitive tools 直接接触文件、shell、网络；orchestration tools 管 agent 的工作流，比如 spawn 子 agent、创建任务、进入 plan mode。权限上 primitive tools 更看重副作用，编排工具更看重资源隔离和失控风险。

  **继续追问**：AgentTool 表面上 read-only，但它能启动会写文件的子 agent，所以不能只看外壳。

- [ ] **主问 2.Q6** 如果在自己的 agent 里加“获取实时股价”的工具，需要定义什么？

  **面试官会怎么问**：你能照 Claude Code 的 Tool 模式设计一个外部 API 工具吗？

  **代码解释**：需要 tool name、自然语言 description、JSON/Zod 输入 schema、执行函数、错误映射、权限策略、超时、重试、progress 和结果格式。

  **面试者回答**：我会定义 `GetStockQuote`，input 是 ticker、market、maybe currency。description 说明什么时候用，不要让模型拿它做金融建议。call 里调外部 API，处理 429/5xx/无效 ticker。权限上它是只读，但有网络访问和成本，所以可以按域名或工具名 allow。结果返回结构化 quote，而不是一大段文本。

  **踩坑点**：实时数据类工具一定要标明时间戳，否则模型容易把旧价格当现在价格。

- [ ] **主问 2.Q7** BM25/vector/reranker 改成 Tool，比函数调用多出什么能力、少了什么性能？

  **面试官会怎么问**：所有 stage 都 Tool 化是不是一定更好？

  **代码解释**：Tool 化以后 LLM 可以动态决定调用哪个 retriever、调用几次、用什么 query。代价是多轮模型决策、tool schema token、权限和错误处理复杂度。

  **面试者回答**：收益是灵活性：简单 query 只走 BM25，语义 query 走 vector，比较型 query 可以多次 retrieve。成本是延迟和不可控性。稳定高频路径我会保留静态 pipeline，只把少数需要动态决策的 stage 暴露成 tool。

  **项目取舍**：Agent 化不是替代 pipeline，而是在 pipeline 外加一个可控的决策层。

> 带读笔记（已讨论）
>
> - 关于 `ToolInputJSONSchema` 的当前理解：schema 可以先用“点菜单”讲清楚：它不是执行函数本身，而是告诉调用方“你能传哪些字段、字段大概是什么形状”。在 Claude Code 里，`src/Tool.ts:15` 的 `ToolInputJSONSchema` 是给模型/API看的 JSON Schema 外形，`src/Tool.ts:394` 的 `inputSchema` 是工具内部的 Zod schema。
> - 关于 schema 和函数签名的可复用模式 / trade-off：函数签名适合代码内、编译期、同语言调用；schema 适合跨边界、运行时、给非 TypeScript 调用方看。LLM 不是 TypeScript 编译器，所以它需要一份可序列化、可放进 API 请求的“菜单”。Zod 的 `safeParse` 在 `src/services/tools/toolExecution.ts:615` 做运行时兜底，`validateInput()` 在 `src/Tool.ts:489` 处理工具自己的业务校验。
> - 还没展开的问题：schema 只能说明参数形状，不能替代权限；参数合法不代表操作安全。
> - 关于 `tool_use` 的当前理解：`tool_use` 不是模型输出的一段 JSON 文本，而是 assistant message 里的结构化 block。Claude Code 在 `src/query.ts:829-834` 扫 `content.type === 'tool_use'`，扫到后把 block 交给工具执行器；`src/query.ts:553-557` 还说明不能只信 `stop_reason === 'tool_use'`，真正的循环信号是有没有收到 tool_use block。
> - 关于结构化输出完整性的可复用模式 / trade-off：结构化输出要分四层看：格式合法、schema 合法、传输未截断、语义正确。prompt-only JSON 没有工程保证；tool use / strict structured output / constrained decoding 能显著提高前两层可靠性，但仍要处理 token 截断、拒答、content filter、字段内容不真实。生产指标应该记录 `parse_ok`、`schema_ok`、`not_truncated`、`not_refusal`、`semantic_ok`。
> - 关于“要不要用正则抠 JSON”的当前理解：能用协议层 structured block 就不要从自然语言里抠 JSON。正则最多负责定位边界，不能负责理解 JSON；真遇到自由文本 JSON，也要 `JSON.parse` + Zod/Pydantic 校验 + retry-with-error-feedback。
> - 关于当前 agent 与 subagent 的可复用模式 / trade-off：tool call 默认发生在当前 agent 主循环里；subagent 只是 AgentTool/Task 这类编排工具启动出来的特殊任务。直接 tool call 适合小而马上要用的结果；subagent 适合长任务、并行任务、大量读取、上下文隔离。`src/tools/AgentTool/prompt.ts:103-108` 提醒 fresh subagent 默认没有当前上下文，所以必须把任务、已知事实、输出格式和回传粒度写清楚。
> - 关于本地消息链的当前理解：`tool_use_id` 是给模型/API看的配对字段，保证 `tool_result` 对应哪个 `tool_use`；`sourceToolAssistantUUID` 是 Claude Code 本地 transcript 用的父节点指针，保证 `tool_result` 挂回包含该 `tool_use` 的 assistant message。`src/utils/messages.ts:490` 明确说它是“the UUID of the assistant message containing the matching tool_use”。
> - 关于本地消息链的可复用模式 / trade-off：这不是 callback，主要服务 transcript、resume、UI、compact 和并行工具结果归属。简单 append 到最近消息在单工具场景可能没事，但并行/流式 tool_use 会形成 DAG，不是单链表；`src/utils/sessionStorage.ts:2096-2113` 注释说明旧逻辑生产里出现过 sibling assistant orphaned 和 tool_result dropped。正确做法是既保留 API 层 `tool_use_id`，也保留本地层 parent/UUID 拓扑。
>
> - **【面试 highlight】工具池排序的 prompt-cache stability 原则 —— "笨但稳"**：
>
>     `assembleToolPool()` (`src/tools.ts:345-367`) 做的事看着很无聊——把内置工具和 MCP 工具按 `name.localeCompare()` 字典序排序、built-in 一段在前 MCP 一段在后、`uniqBy('name')` 去重。但**这段代码是 cache 命中率的命根子**。
>
>     **反直觉的取舍**：按 LLM 直觉，应该把"模型最爱用的工具"（比如 Bash、Read）排前面让模型更容易选中。但 Claude Code 死活按**字典序**排——为什么？
>
>     **关键事实**（`tools.ts:354-361` 注释明说）：
>     - Anthropic 服务端的 `claude_code_system_cache_policy` 在**最后一个 built-in 工具之后**放了一个 global cache breakpoint
>     - 这意味着 `[system prompt][tools 数组]` 是 prompt cache 的**稳定前缀**
>     - 任何重排——按 recency、按使用频率、按 LLM 选择概率——都会让前缀 hash 变化，**整条 prefix cache 失效**
>     - 字典序的好处：**deterministic**。同一组工具、同一组 MCP server，排出来永远一样
>
>     **为什么 built-in / MCP 必须切两段而不是混排**：MCP 是动态的，用户随时可能装新 MCP server。如果 flat sort，一个新 MCP 工具的 name 排到 built-in 之间（比如 `Awesome_MCP_Tool` 排到 `Bash` 之前），**会把所有 built-in 之后的 cache 全部冲掉**。所以强制 built-in 一段 + MCP 一段，新装 MCP 只影响 MCP 段，built-in 段始终稳定。
>
>     **可复用设计原则**：**cache key 的组成部分必须 deterministic + monotonic**。任何按"用户偏好 / 使用频率 / 时间"排序的"聪明"逻辑，都是在 cache 命中率上自杀。Agent / RAG / 任何带 LLM 的系统都适用。
>
>     **30 秒口播版**：
>
>     > Claude Code 的工具列表为什么按字典序排而不是按模型偏好排？因为 tools 数组是 prompt cache key 的前缀。按字典序排是 deterministic 的，**这次和下次一样**，所以 cache 命中。如果按"模型最爱用的放前面"，每次顺序都在变，每次 cache miss，烧钱。这是一个反直觉的工程取舍：**稳定比聪明重要**。Built-in 和 MCP 还要切成两段不能混排，因为 MCP 是动态的——新装一个 MCP 工具如果插进 built-in 之间，会把整段 built-in 之后的 cache 全冲掉。
>
> - **关于 "cache 命中率到底有没有用" 的当前理解**：常见误解——以为 cache hit rate 衡量"用户输入是否重复"。实际上 prompt cache 是 **prefix cache**，命中率衡量的是 **prefix 跨轮次稳定性**，跟用户当前消息几乎无关。
>     - **prompt 的真实结构**（按拼接顺序）：`[system prompt][tools 数组][memory/context][对话历史][本次用户消息]`
>     - cache 命中是 **longest matching prefix**，**用户本次消息永远在尾部**，永远不可能 cache hit
>     - 主要 cache 贡献者：(1) system prompt（最稳）、(2) tools 数组（assembleToolPool 稳定排序保的就是这层）、(3) **对话历史**（这才是占大头的——多轮 agent loop 里，turn N 的 prompt 把 turn 1~N-1 全装进去，整段都能命中 turn N-1 的 cache）
>     - **命中率到底对谁有用**：不是给终端用户看的（普通 chat 用户确实看不懂也用不上），是给 **API 集成方 / SDK 开发者 / 自己造 agent 的工程师** 看的 debug 工具——它告诉你"**我有没有把上下文结构搞稳定**"。如果你预期 90% hit 但实际 30%，多半是哪个环节在乱动消息——比如把历史消息 patch 了、tools 数组顺序漂了、system prompt 里塞了当前时间戳
>     - **典型多轮 hit rate 曲线**：turn 1 = 0%（冷启动），turn 2 = 90%+（turn 1 全段进 prefix），turn 10 理想仍是 90%+——只要没人乱动历史
>     - **成本意义**：Anthropic cached input token 价格 ≈ 0.1x fresh token，90% hit 直接砍掉约 80% 的输入成本。对长上下文 agent 来说是省钱命脉
>     - **总结一句话**：cache hit rate 不是给"用户碰巧重复了"打分，是给"**应用有没有把 prompt 拼稳**"打分。Claude Code 的 `assembleToolPool` 字典序、stable system prompt、不动历史消息——这些都是为了让这个分数高
>
> - 还没展开的问题：2.Q3 还可以继续 DFS `tool_use -> runTools -> checkPermissionsAndCallTool -> tool_result -> 下一轮 query` 的完整跳转链；`isToolSearchEnabled` 的工具数量阈值（工具池太大时的 escape valve）；为什么 Anthropic 不直接在 API 层支持"动态工具检索"而要在客户端做这层。

---

## 3. 权限系统：工具调用的安全门

一句话：权限系统不是保护 API，它是拦住 LLM 乱动机器的门。

### 代码地图

- `src/types/permissions.ts:24-44`：permission mode 和 allow/deny/ask 行为。
- `src/utils/permissions/PermissionMode.ts:22-43`：external/internal mode 映射。
- `src/hooks/useCanUseTool.tsx:27`：`CanUseToolFn` 类型签名。
- `src/utils/permissions/permissions.ts:473-527`：`hasPermissionsToUseTool` 主入口。
- `src/utils/permissions/permissions.ts:527-676`：accept-edits、安全工具 allowlist、auto classifier 分支。
- `src/utils/permissions/permissions.ts:930-945`：`PermissionRequest` hook 介入。
- `src/utils/permissions/permissions.ts:1158`：`hasPermissionsToUseToolInner()` 内层判定。
- `src/utils/permissions/permissions.ts:1284-1294`：always allow rule 命中。
- `src/utils/permissions/denialTracking.ts:7-40`：连续拒绝状态机。
- `src/utils/permissions/shellRuleMatching.ts:25-40`：exact/prefix/wildcard 规则类型。

### 3.1 权限决策链

- [x] **前置 3.P1** `PermissionMode` 里的 default / plan / auto / bypass 有什么差异？

  **面试官会怎么问**：用户按 Shift+Tab 切权限模式，底层行为到底变了什么？

  **代码解释**：`default` 遇到陌生操作问用户；`acceptEdits` 放宽编辑相关操作和少量 Bash 命令；`plan` 主要靠 prompt 约束；`bypassPermissions` 尽量自动放行，但安全检查仍可能 ask；`auto` 用分类器替用户判断；`dontAsk` 把 ask 转 deny。

  **面试者回答**：这些模式不是 UI 文案不同，而是权限漏斗的默认策略不同。default 保守，bypass 激进，auto 把判断交给分类器，dontAsk 用在无交互场景。最值得注意的是 plan mode：代码里不是强安全边界，它更多依赖 system prompt。

  **踩坑点**：不要说 plan mode 等于只读。源码显示它没有系统性检查 `isReadOnly()`。

- [x] **主问 3.Q1** `CanUseToolFn` 决策流程：settings allowlist、session always allow、弹窗问用户的优先级是什么？

  **面试官会怎么问**：一个 Bash 命令来了，先看配置还是先弹窗？

  **代码解释**：权限先走 deny/ask/allow 规则和 tool 自己的 `checkPermissions()`。命中 deny 直接拒绝，命中 ask 进入询问，命中 allow 或 always allow 可直接放行。没有明确结论时按当前 permission mode 决定 ask、deny 或 classifier。

  **面试者回答**：优先级核心是 fail-closed：明确 deny 优先级最高，其次是强制 ask 和 tool 自己的安全判断，再看 allow 和模式放行。session 的 always allow 是一种临时规则，避免用户重复批准同类操作。最后才是弹窗或 auto classifier。

  **继续追问**：ask rule 不是弱规则，有些 safety check 即使 bypass 也不能绕过。

- [x] **主问 3.Q2** `DenialTrackingState` 跟踪什么？连续拒绝会发生什么？

  **面试官会怎么问**：如果模型一直尝试同一个被拒绝的工具，系统怎么处理？

  **代码解释**：`denialTracking.ts` 记录工具拒绝次数和是否该 fallback 到 prompting。它防止模型在一个被拒绝的操作上反复撞墙。

  **面试者回答**：它跟踪“同类请求被拒绝了几次”。如果连续拒绝，系统应该改变交互方式，比如明确告诉模型或用户，不要继续自动尝试。这个机制在 agent 系统里很重要，因为 LLM 可能不知道自己正在重复犯错。

  **项目类比**：推荐系统里如果外部 API 连续 401，也不该无限重试，应切换到降级路径。

### 3.2 权限持久化与匹配

- [ ] **前置 3.P2** settings.json 里 permissions 的存储结构是什么样的？

  **面试官会怎么问**：用户点了 always allow，这个规则存在哪里？

  **代码解释**：settings 里会保存 allow/deny/ask 规则和 defaultMode。规则表达工具名或带内容的规则，例如 Bash 前缀/通配匹配。session 内也有临时的 always allow 记忆，不一定都持久化到全局配置。

  **面试者回答**：权限配置是 rule-based，不是简单布尔开关。它能表示“允许某个工具”“拒绝某个工具”“允许某类 Bash 命令”。持久化规则适合长期偏好，session 规则适合当前会话临时放行。

  **踩坑点**：权限规则越宽，风险越大，尤其是 Bash wildcard。

- [ ] **主问 3.Q3** Always allow 的匹配粒度是什么？allow `git diff`，`git diff --staged` 会放行吗？

  **面试官会怎么问**：命令匹配是字符串相等、前缀，还是正则？

  **代码解释**：`shellRuleMatching.ts` 区分 exact、prefix、wildcard。`parsePermissionRule()` 会把规则解析成匹配器。是否放行 `git diff --staged` 取决于保存的是 exact 规则还是 prefix 规则，不是所有 allow 都自动前缀匹配。

  **面试者回答**：不能笼统说会或不会。规则有 exact、prefix、wildcard 三种粒度。exact 的 `git diff` 只匹配原命令，prefix 才能覆盖 `git diff --staged`，wildcard 更宽。安全设计上默认应鼓励窄规则，因为 Bash 命令空间太大。

  **继续追问**：compound command、heredoc、环境变量赋值都可能绕过朴素字符串匹配，所以 Bash 权限要先解析命令结构。

- [ ] **主问 3.Q4** 细粒度 tool-call 权限在推荐系统里是不是过度设计？

  **面试官会怎么问**：你的系统要不要每次 retrieval 都问用户？

  **代码解释**：Claude Code 的工具能写文件、跑 shell、联网，风险高，所以每个 tool call 都要过权限。推荐系统的 retrieve/rerank 通常只读，没必要每次弹窗，但写 profile、调用付费 API、访问私有数据时需要审批或 policy。

  **面试者回答**：对纯只读推荐 pipeline 来说，Claude Code 这种粒度可能过重。但一旦 agent 能写用户画像、调用外部付费服务、发邮件或修改业务数据，就需要 tool-call 级别权限。我的原则是：只读高频自动放行，写操作和高成本操作细粒度审批。

> 带读笔记（已讨论）
>
> - **开场题破题（3.P1 之前的认知校准）**：面试官问 "Claude Code 的权限设计"，第一拦截目标不是账号 / API quota / OS chmod，而是 **LLM 发起 tool_use 的那一刻**。开场第一句话定调："Claude Code 的权限系统不是用户鉴权，是 tool-call 级别的守门人——每次 LLM 想调工具，权限层都先判一次。"
> - **关于 PermissionMode 的当前理解（3.P1）**：4 个外部模式 default / acceptEdits / plan / bypassPermissions（加上内部 auto / dontAsk / bubble）。区分维度不是 UI，是"默认策略"。两个最容易讲错的点：
>     - **plan mode 不是 type-level 只读边界**，是 **prompt-level 约束**——靠 system prompt 让模型自己克制，代码里没有系统性 `isReadOnly()` gate。面试里这是"读过代码 vs. 用过 UI"的分水岭。
>     - **bypassPermissions 不是 free-for-all**，它是"VIP 通道"，跳过的是"陌生操作就问"那一层；显式 deny rule、用户配置的 ask rule、敏感路径 safety check、工具自声明 `requiresUserInteraction` 这 4 类还是会拦（见 `permissions.ts:1230-1281`，编号 1d/1e/1f/1g）。整个模式还可被 Statsig gate / settings 全局关掉（`bypassPermissionsKillswitch.ts`）。
> - **可复用的判定链编号体系**：源码注释用 1a/1b/1c/1d/1e/1f/1g/2a/2b/3 给步骤编号。1x 是规则与工具自检（bypass-immune），2x 才是 mode 决策，3 是兜底 ask。这套编号是面试回答时把"优先级"讲清楚的脚手架——下一题 3.Q1 会展开。
> - **关于判定链优先级的当前理解（3.Q1）**：`hasPermissionsToUseToolInner()` (`permissions.ts:1158`) 是一个 **fail-closed early-return pipeline**，10 步连续编号 1a/1b/1c/1d/1e/1f/1g/2a/2b/3。
>     - **编号是注释，不是 TS 语言特性**——TS 给的是 `Promise<PermissionDecision>` 的穷尽返回 + discriminated union 的形状校验，**步骤顺序不在类型系统的管辖范围内**，靠物理顺序 + 注释 + reviewer 自律。
>     - **两层 hierarchy**：大段 1/2/3 切的是"bypass 能不能跳过这步"（1=必跑硬规则 / 2=mode + 默认放行 / 3=兜底 ask）；小段 a/b 只是同段内顺序，**1a 和 2a 没有对应关系**。`checkRuleBasedPermissions()` (`permissions.ts:1071`) 显式复刻 1a-1g 这段，用在"只跑规则、不要 mode 影响"的场景——这是**两层切分的存在证明**。
>     - **bypass (2a) 在第 8 步**——前面 1a-1g 共 7 步都跑完才轮到它。bypass 不是"被前 7 步批准"，是"前 7 步都没出声反对"后的默认放行。
> - **可复用的安全设计原则**：**反对显式（return），允许隐式（fall through）**。这是 fail-closed 的核心——默认拦，必须有人明确说 OK 才放。任何 access control / 安全门设计都得遵守，否则 fail-open，漏一个分支就出事。
> - **关于 denial tracking 的当前理解（3.Q2）**：`denialTracking.ts` 是给 **auto 模式 classifier 装的疲劳熔断器**，不是工具执行层的错误处理。两个阈值（`DENIAL_LIMITS`，line 12-15）：
>     - `maxConsecutive: 3` — 连续 3 次拒绝（**可逆**，一次成功就重置）
>     - `maxTotal: 20` — 全局累计 20 次拒绝（**不可逆**，`recordSuccess` 故意不重置）
>     - 任意触发 → `shouldFallbackToPrompting === true` → 降级到弹窗问真人
>     - 区分意图：**连续**捕捉"暂时误判 / classifier 跟模型杠上"；**总数**捕捉"classifier 整个 session 系统性失灵"
> - **可复用的 Agent 安全原则**：**自动决策永远要带熔断**。Auto-classifier、auto-retry、auto-fallback 都得有"我已经做错太多次了"的退出条件，否则陷入"模型→ classifier 拒→ 模型换写法→ classifier 又拒"的死循环。Claude Code 选了"连续 + 全局，可逆 + 不可逆"的双阈值组合——这是经过实战的成熟模式，可以直接照搬到其他自动决策系统（包括推荐系统的重排器 drift 检测）。
> - **状态机风格**：整个 `denialTracking.ts` 是 **immutable state machine**（所有函数 `return { ...state, ... }`，不 mutate），方便测试和审计回放。
> - **还没展开的问题**：hooks 在外层入口的位置（外层 `hasPermissionsToUseTool` vs. 这个 inner）；权限规则的存储与匹配粒度（3.P2 / 3.Q3）；细粒度 tool-call 权限在推荐系统里的适用边界（3.Q4）。

---

## 4. Task 系统：Agent 的线程模型

一句话：Task 是后台活的句柄。它不负责“怎么创建”，只负责运行后能追踪、能终止。

### 代码地图

- `src/Task.ts:6-13`：7 种 `TaskType`。
- `src/Task.ts:15-27`：`TaskStatus` 与 terminal status 判断。
- `src/Task.ts:38-42`：`TaskContext`。
- `src/Task.ts:45-58`：共享 `TaskStateBase`。
- `src/Task.ts:69-75`：`Task` 接口只有 `kill()`。
- `src/Task.ts:79-113`：`TASK_ID_PREFIXES`、`generateTaskId()`、`createTaskStateBase()`。
- `src/tasks.ts:3-37`：Task type 到具体实现的分发表。
- `src/tasks/LocalShellTask/LocalShellTask.tsx:173-226`：shell task 的 kill 和 spawn。
- `src/tasks/LocalAgentTask/LocalAgentTask.tsx:478-554`：local agent task state 创建。
- `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:386-448`：remote task 注册。
- `src/tools/shared/spawnMultiAgent.ts:786-798`：in-process teammate task state 创建。

### 4.1 Task 类型与生命周期

- [x] **前置 4.P1** 7 种 `TaskType` 各是什么场景？

  **面试官会怎么问**：为什么 TaskType 不只有 bash 和 agent？

  **代码解释**：`local_bash` 是本地 shell；`local_agent` 是本地子 agent；`remote_agent` 是远端执行；`in_process_teammate` 是同进程 teammate；`local_workflow` 是工作流脚本；`monitor_mcp` 是监控类 MCP 任务；`dream` 是后台 ideation 类任务。

  **面试者回答**：TaskType 把不同后台执行模型统一成 task state。它们的执行方式不同，但 UI 和状态管理需要统一看到 pending/running/completed/failed/killed。

- [ ] **主问 4.Q1** `Task` 接口为什么只有 `kill()`？spawn 逻辑为什么不放在接口里？

  **面试官会怎么问**：一个 Task 接口没有 start/run，是不是设计太贫血？

  **代码解释**：spawn 需要不同参数和环境：shell、agent、remote、workflow 都不一样。统一接口强塞 spawn 会让类型变复杂。运行后统一需要的是可终止能力，所以 `Task` 只保留 `kill()`。

  **面试者回答**：这是典型的生命周期拆分。创建阶段因类型而异，不能抽象得太早；运行阶段共同需求是终止和状态追踪。把 spawn 放在具体模块里，Task 接口只表示“已经活着的东西”，设计更干净。

  **继续追问**：这和进程句柄类似，`Process` 对象不负责决定用什么参数启动自己。

- [ ] **主问 4.Q2** `TaskContext` 的 abortController、getAppState、setAppState 怎么支撑生命周期？

  **面试官会怎么问**：后台任务怎么知道自己被取消了？怎么更新 UI 状态？

  **代码解释**：`abortController` 提供取消信号；`getAppState` 读取当前任务状态；`setAppState` 写入状态变化，比如 running、completed、failed、killed。

  **面试者回答**：AbortController 管“停不停”，AppState getter/setter 管“状态怎么被 UI 和主循环看到”。后台任务不能只在内部变量里改状态，否则 UI 不知道、resume 也不好恢复。

- [ ] **主问 4.Q3** spawn 一个 `local_bash` task 到执行完，AppState 经历哪些状态转换？

  **面试官会怎么问**：Bash 后台跑 5 分钟，中间状态怎么显示？

  **代码解释**：spawn 时创建 task state，通常从 pending/running 开始；进程输出通过 progress 或 state 更新；正常结束变 completed，错误变 failed，用户 kill 变 killed。terminal status 用 helper 判断。

  **面试者回答**：流程是创建 state、放入 AppState、启动子进程、流式收集输出、根据 exit code 更新 terminal status。kill 会通过 task handle 调 `kill()`，同时状态进入 killed。重点是状态变化不只服务 UI，也服务恢复和清理。

- [ ] **主问 4.Q4** `TaskHandle` 的 cleanup 是做什么的？为什么调用方要手动清理？

  **面试官会怎么问**：任务结束了不就完了吗，为什么还要 cleanup？

  **代码解释**：后台任务可能注册 listener、定时器、stream、文件句柄、远端订阅。cleanup 负责释放这些外部资源。调用方知道任务生命周期，所以由调用方在合适时机清理。

  **面试者回答**：cleanup 不是业务结束，而是资源结束。没有 cleanup，任务跑完也可能留下事件监听或流，造成内存泄漏和重复回调。把 cleanup 放在 handle 里，等于把“如何收尾”交还给创建者。

### 4.2 Task 标识与扩展

- [ ] **前置 4.P2** 为什么每种 task type 有单字母前缀？

  **面试官会怎么问**：task id 前缀有什么意义？只是好看吗？

  **代码解释**：`TASK_ID_PREFIXES` 让 id 自带类型提示。日志、UI、调试、恢复时，不用查 state 就能初步判断任务类型。

  **面试者回答**：这是可观测性设计。一个 id 看起来像 `b_...` 或 `a_...`，人和系统都能快速分辨来源。它不是安全边界，但能降低排查成本。

- [ ] **主问 4.Q5** 新增一种 task type 要走哪些步骤？

  **面试官会怎么问**：如果加一个 background crawler task，你要改哪几处？

  **代码解释**：新增 `TaskType`、prefix、state 类型、具体 Task 实现、spawn 函数、`tasks.ts` 分发表、UI 渲染/恢复逻辑，必要时加权限和预算控制。

  **面试者回答**：我会先定义类型和状态，再写 spawn 和 kill，然后接入 `getTaskByType()`。如果它会跨 session 存活，还要写恢复逻辑。如果它有外部副作用，还要加权限和资源限制。

- [ ] **主问 4.Q6** 你的 agent 系统有没有后台任务概念？如果加一个 5 分钟爬虫任务怎么设计？

  **面试者回答**：我会避免让主请求线程阻塞 5 分钟。设计上用 Task 表保存 taskId、status、progress、resultLocation。Agent 只负责创建任务和轮询/订阅结果。爬虫本身跑在 worker 或 queue 里，支持 cancel、timeout、失败重试和结果落盘。

- [ ] **主问 4.Q7** embedding 计算改成异步 task，spawn 逻辑放 agent 内部还是外部调度器？

  **面试者回答**：高耗时、高吞吐任务应该放外部调度器。Agent 可以发起任务，但不要自己承担 worker pool。这样扩缩容、失败重试、幂等和队列背压都更清楚。Agent 内部只保留轻量 orchestration。

> 带读笔记（已讨论）
>
> - 关于 `TaskType` 的当前理解：它不是按抽象层级分类，而是按后台执行载体/来源分类。只要一个工作需要被 AppState、后台任务面板、输出文件和 kill/cleanup 统一追踪，就可以成为 Task。
> - 关于 7 种类型的可复用模式 / trade-off：`local_bash` 是本地命令，`local_agent` 是本地子 agent，`remote_agent` 是云端/远端会话，`in_process_teammate` 是同进程 agent 队友，`local_workflow` 是 feature-gated 的本地工作流脚本，`monitor_mcp` 是 feature-gated 的监控类 MCP 任务，`dream` 是后台 memory consolidation subagent 的 UI 可见入口。统一成 Task 牺牲了分类层级的纯粹性，换来统一展示、状态追踪和终止能力。
> - 还没展开的问题：为什么 `Task` 接口只保留 `kill()`，而不把 spawn/start/run 也做成统一多态接口。

---

## 5. 多 Agent 协作：从单 Agent 到 Swarm

一句话：Claude Code 最激进的点是，启动另一个 agent 本身就是一次 tool call。

### 代码地图

- `src/tools/AgentTool/AgentTool.tsx:196-221`：`AgentTool` 的 buildTool 入口。
- `src/tools/AgentTool/AgentTool.tsx:575`：子 agent 默认 permission mode。
- `src/tools/AgentTool/AgentTool.tsx:1264`：`AgentTool.isReadOnly()` 返回 true。
- `src/tools/AgentTool/runAgent.ts:96-310`：子 agent 运行参数、工具继承和 thinkingConfig。
- `src/tools/AgentTool/runAgent.ts:502`：`resolveAgentTools(...)`。
- `src/tools/AgentTool/agentToolUtils.ts:122`：工具解析函数。
- `src/tools/AgentTool/loadAgentsDir.ts:106-165`：`AgentDefinition` 类型。
- `src/tools/AgentTool/loadAgentsDir.ts:296-359`：内置、用户、插件 agent 合并。
- `src/coordinator/coordinatorMode.ts:120-207`：coordinator system prompt。
- `src/tools/shared/spawnMultiAgent.ts:786-798`：in-process teammate task 创建。

### 5.1 Sub-Agent 机制

- [ ] **前置 5.P1** `AgentTool` 是什么？为什么“启动 Agent”本身是 Tool？

  **面试官会怎么问**：为什么不在代码里直接调用子流程，而要暴露成 Tool？

  **代码解释**：`AgentTool` 有普通 Tool 的外壳：name、description、inputSchema、call。模型可以像调用 Grep 一样调用 AgentTool，把任务描述交给子 agent。

  **面试者回答**：因为 Claude Code 把 agent 当成可组合能力。父 agent 不需要知道子 agent 的内部细节，只要给 description 和 prompt。这样 Explore、Plan、自定义 agent 都能作为统一工具出现。

  **继续追问**：风险是父 agent 可能滥用子 agent，所以要有工具隔离、权限和预算。

- [ ] **主问 5.Q1** `AgentTool` spawn sub-agent 的完整流程是什么？消息怎么隔离、结果怎么回传？

  **代码解释**：父 agent 发起 AgentTool tool_use，`AgentTool.call()` 解析 agent definition 和输入 prompt。`runAgent.ts` 创建独立消息历史、file cache、abort controller、tool context，并运行子 query loop。子 agent 结束后，结果被包装成父 agent 的 tool_result。

  **面试者回答**：子 agent 是一个隔离的对话循环，不是把父 messages 直接共享过去。父 agent 只给任务，子 agent 自己跑工具和模型，最后产出文本结果。父 agent 看到的只是一次工具调用返回。

  **踩坑点**：隔离不是完全安全。子 agent 仍可能共享 cwd 或文件系统能力，权限要单独设计。

- [ ] **主问 5.Q2** sub-agent 有独立 QueryEngine 吗？还是复用主 agent？为什么？

  **代码解释**：运行时会创建隔离的 query loop 和上下文。它可能复用部分配置，比如 model、thinking、tools，但消息历史和 tool context 是新的。

  **面试者回答**：设计上应该看成独立 QueryEngine/独立 loop。这样子 agent 的中间 tool_use 不污染父 agent 历史，父 agent 只得到总结结果。好处是隔离清楚，代价是额外 token 和运行成本。

### 5.2 Team 与 Coordinator

- [ ] **前置 5.P2** Coordinator 模式和普通模式有什么区别？

  **代码解释**：Coordinator 更像项目经理 agent，负责分派、协调多个 teammate/worker。普通模式是单 agent 直接解决问题。

  **面试者回答**：Coordinator 的核心不是多一个 prompt 名字，而是职责变了：它少做具体执行，多做拆分、分配、合并和冲突处理。

- [ ] **主问 5.Q3** TeamCreateTool / TeamDeleteTool 的 agent swarm 是什么？多个 agent 怎么通信？

  **代码解释**：Team tools 创建和销毁多 agent 团队。通信可以通过 coordinator、message tool、shared task state 或 backend 管理。不同 backend 可能是 in-process、tmux、iTerm、remote。

  **面试者回答**：Swarm 是多 worker 并行做事的运行模式。关键问题不是“能开几个 agent”，而是任务怎么分、结果怎么回、权限怎么同步、并发写文件怎么避免冲突。

- [ ] **主问 5.Q4** `AgentDefinition` 包含哪些字段？怎么加载？

  **代码解释**：`AgentDefinition` 包含 agentType、tools、disallowedTools、model、permissionMode、maxTurns、skills、mcpServers、hooks、background、isolation、system prompt 等。来源包括内置、用户目录、插件。

  **面试者回答**：AgentDefinition 是 agent 的配置对象。它定义这个子 agent 是谁、什么时候用、能用哪些工具、用什么模型、有什么权限。加载时要合并多来源并处理覆盖和去重。

### 5.3 并发控制

- [ ] **主问 5.Q5** 多个 sub-agent 同时跑，怎么防止资源竞争？

  **代码解释**：工具层有 `isConcurrencySafe()` 和 orchestration。更高层还可以通过 worktree isolation、permission mode、task budget 限制并发。文件写冲突需要靠隔离工作区或合并策略，不是 tool loop 自动解决一切。

  **面试者回答**：并发最怕两个 agent 改同一个文件。只靠“模型会小心”不够。我会把只读探索和写操作分开，写操作用 worktree 隔离或锁，最后统一 merge。并发安全要由工具属性和执行编排共同保证。

- [ ] **主问 5.Q6** sub-agent 和 in_process_teammate 有什么区别？

  **代码解释**：sub-agent 是通过 AgentTool 创建的一次子任务；in_process_teammate 是 team/swarm 模式中的长期协作实体，作为 TaskType 管理。

  **面试者回答**：sub-agent 更像一次函数调用：给任务，等结果。teammate 更像一个活着的 worker：有状态、有通信、有生命周期。两者都叫子智能体，但进程模型和协作模式不一样。

- [ ] **主问 5.Q7** BM25/vector 并行召回和 sub-agent 并行有什么类比？

  **面试者回答**：BM25 和 vector 并行召回像两个只读 worker，最后 merge。AgentTool 化以后，主 agent 可以动态决定要不要并行、查几轮、是否追加 query reformulation。收益是灵活，风险是延迟、成本和结果不可复现。

---

## 6. 技能系统：可复用的 Agent 工作流

一句话：Skill 不是工具本身，它更像给 agent 的工作手册。

### 代码地图

- `src/tools/SkillTool/SkillTool.ts:291-331`：SkillTool schema 和 buildTool 入口。
- `src/tools/SkillTool/SkillTool.ts:354-424`：skill 名称校验、远程 skill、未知 skill 错误。
- `src/tools/SkillTool/SkillTool.ts:581-774`：普通 inline skill 执行。
- `src/tools/SkillTool/SkillTool.ts:621-650`：forked skill 与 `processPromptSlashCommand(...)`。
- `src/tools/SkillTool/SkillTool.ts:735-766`：skill 生成 messages 并改上下文。
- `src/skills/loadSkillsDir.ts:181-393`：frontmatter 解析和 `createSkillCommand()`。
- `src/skills/loadSkillsDir.ts:404-470`：`/skills/<name>/SKILL.md` 目录格式加载。
- `src/skills/loadSkillsDir.ts:626-799`：managed/user/project/additional/legacy commands 合并去重。
- `src/commands.ts:354-392`：skills、plugin skills、bundled skills 汇总。
- `src/commands.ts:562-604`：`getSkillToolCommands()` 筛出模型可调用 skill。

### 6.1 Skill 本质

- [ ] **前置 6.P1** skill 和 plugin 是什么关系？skill 是 prompt 模板、工作流，还是特殊 tool？

  **面试官会怎么问**：Skill 跟 Tool 有什么本质区别？

  **代码解释**：Tool 是可调用函数接口；Skill 是可复用说明书和流程，可以通过 SkillTool 注入消息、触发 slash command、甚至 fork agent。Plugin 是更大的分发/扩展机制，里面可以带 skill。

  **面试者回答**：Skill 不是直接执行能力，它是让模型按某套流程工作的上下文包。Tool 是手，Skill 是说明书。Plugin 可以携带 skill，相当于扩展包。

- [ ] **主问 6.Q1** `SkillTool` 触发 skill 到执行完，完整链路是什么？

  **代码解释**：模型调用 SkillTool，传 skill name 和 args。SkillTool 查 command，验证名字，展开 skill prompt 或执行对应 slash command。inline skill 会生成 newMessages 改上下文；fork skill 会开隔离 agent 返回结果。

  **面试者回答**：SkillTool 是 router。模型只调用一个 SkillTool，但内部根据 skill 名字找到具体 skill，再把 skill 内容变成对话上下文或子 agent 任务。skill 本身不一定有“代码步骤”，更多是 prompt 和文件契约。

- [ ] **主问 6.Q2** skill 可以动态加载吗？能调用 tool 或 spawn sub-agent 吗？

  **代码解释**：skills 从文件系统、managed/user/project/additional/plugin/bundled 来源加载。skill 的内容进入上下文后，模型可以继续调用普通 tools。fork skill 还能隔离执行。

  **面试者回答**：可以动态发现和加载。skill 不直接拥有工具执行权，但它指导模型使用工具。fork 模式下，它可以通过子 agent 隔离执行复杂流程。

- [ ] **主问 6.Q3** bundled skill 和用户安装 skill 的加载优先级、覆盖规则怎样？

  **代码解释**：`commands.ts` 和 `loadSkillsDir.ts` 负责把 bundled、builtin plugin、skillDir、workflow、plugin、COMMANDS 合并。合并时要处理命名冲突、启用条件和是否能被模型调用。

  **面试者回答**：bundled skill 是编译进二进制的 escape hatch，适合改配置、自省、复杂编排。普通领域流程应该走文件系统 skill。加载顺序决定覆盖和去重，模型可调用列表还会再被筛选。

### 6.2 Skill 设计模式

- [ ] **主问 6.Q4** skill 和写死 system prompt 有什么区别？

  **面试者回答**：写死 system prompt 是常驻规则，所有任务都背着它。Skill 是按需工作手册，只有相关任务才注入。这样省 token，也降低无关指令干扰。Skill 还可以带文件、参数和 fork 执行模式。

- [ ] **主问 6.Q5** software_reco 有没有类似 skill 的概念？推荐策略能不能做成 skill？

  **面试者回答**：可以把“冷启动推荐”“相似项目推荐”“岗位匹配推荐”做成不同 skill。每个 skill 写清楚什么时候用、需要哪些工具、输出格式是什么。稳定的检索算法仍在代码里，skill 只负责调度策略和解释风格。

- [ ] **主问 6.Q6** 运行时动态选 stage 是不是一种 skill 调度？

  **面试者回答**：是类似思想。比如 query 是“比较两个产品”，就用 compare skill；query 是“找相似论文”，就用 semantic retrieval skill。区别是推荐系统里核心 stage 最好仍由代码控制，LLM 只决定高层策略。

---

## 7. MCP 协议：外部工具生态

一句话：MCP 的价值是让外部服务用同一种工具格式接进 agent。

### 代码地图

- `src/services/mcp/types.ts:24-131`：MCP server config schema。
- `src/services/mcp/config.ts:134-205`：stdio 命令和 URL signature 抽取。
- `src/services/mcp/client.ts:595`：`connectToServer` 入口。
- `src/services/mcp/client.ts:626-702`：SSE transport。
- `src/services/mcp/client.ts:734-783`：WebSocket transport。
- `src/services/mcp/client.ts:814-900`：HTTP/claude.ai proxy。
- `src/services/mcp/client.ts:944-950`：stdio transport。
- `src/services/mcp/client.ts:985-1088`：Client 创建、connect timeout、成功/失败处理。
- `src/services/mcp/client.ts:1216-1397`：掉线、清缓存、重连。
- `src/services/mcp/client.ts:1743-1813`：外部 MCP tools 包装成内部 Tool。
- `src/services/mcp/client.ts:3069-3215`：`client.callTool(...)` 与认证/断连错误处理。
- `src/tools/MCPTool/MCPTool.ts:14-45`：MCPTool 通用 passthrough 外壳。
- `src/tools/McpAuthTool/McpAuthTool.ts:49-140`：认证工具。
- `src/services/mcp/auth.ts:244-327`：OAuth server discovery。

### 7.1 MCP Client 架构

- [ ] **前置 7.P1** MCP server 的 transport 怎么建连接？生命周期在哪管？

  **面试官会怎么问**：stdio、SSE、WebSocket 都是 MCP，client 怎么统一？

  **代码解释**：`connectToServer` 根据 config 创建不同 transport，再统一包成 MCP Client。连接成功后拉 tools/resources，失败和掉线走缓存清理、状态更新和重连逻辑。

  **面试者回答**：transport 是连接层差异，MCP Client 是协议层统一。无论底层是本地进程 stdio 还是远端 SSE，最终都变成“列工具、调工具、处理认证和断线”的统一接口。

- [ ] **主问 7.Q1** `McpTool` 怎么把外部 MCP tool 包装成内部 Tool？

  **代码解释**：client 获取 server tool list 后，为每个 tool 生成内部 Tool 对象，名字通常带 server 前缀避免冲突，input schema 来自 MCP server，call 时转发到 `client.callTool(...)`。

  **面试者回答**：包装层做三件事：命名空间隔离、schema 适配、调用转发。对模型来说它就是一个普通 Tool；对执行层来说它是远端 passthrough。

- [ ] **主问 7.Q2** MCP server tool list 变了，Agent 怎么感知？

  **代码解释**：client 有连接状态、缓存清理、重连和工具刷新逻辑。server 重启或连接断开后，需要重新 list tools 并更新 app state/tool pool。

  **面试者回答**：动态工具列表不能当静态常量。MCP 连接变化后，要清掉旧工具缓存，重新发现工具，再让 tool pool 更新。否则模型可能看到已经不存在的 tool，或看不到新工具。

### 7.2 MCP 取舍

- [ ] **主问 7.Q3** MCP tool 和内置 tool 在 Agent 眼里有区别吗？

  **面试者回答**：在 API schema 层面都像 Tool，但描述和名字会暴露来源差异。执行层差别很大：内置工具本地运行，MCP 工具要走外部连接、认证、网络错误和 server 生命周期。

- [ ] **主问 7.Q4** MCP server 为什么需要独立认证？跟 CLI OAuth 是什么关系？

  **面试者回答**：CLI 登录 Anthropic 只证明用户能用 Claude Code，不代表用户授权某个第三方 MCP server 访问数据。每个 MCP server 可能连 GitHub、Slack、DB，所以需要自己的 OAuth 或 token。它是外部服务授权，不是 Claude 登录授权。

- [ ] **主问 7.Q5** 什么时候用 MCP，什么时候自己写 API？

  **面试者回答**：如果工具来自外部生态、多个 client 复用、需要动态发现，MCP 值得用。如果只是项目内部一个稳定 API，直接写 SDK wrapper 更简单、延迟更低、错误面更少。

- [ ] **主问 7.Q6** embedding/LLM/vector DB 都有 MCP server，你愿意迁移吗？

  **面试者回答**：我不会一次性全迁。embedding 和 vector DB 是高频核心路径，性能和稳定性优先，保留 SDK 更稳。低频管理操作、调试工具、跨服务查询可以先接 MCP。统一接口的收益要大过网络、认证和故障复杂度才值得迁移。

### 7.3 启动期 100 个 MCP 的渐进连接（实战题）

- [ ] **前置 7.P2** `src/services/mcp/client.ts:2266-2271` 为什么要把 configEntries 拆成 `localServers` 和 `remoteServers` 两组？分开有什么用？
- [ ] **前置 7.P3** `src/services/mcp/client.ts:2212-2224` 的注释说"previous implementation ran fixed-size sequential batches"，换成 `pMap` 解决了什么具体问题？
- [ ] **主问 7.Q7** 终端启动连 100 个 MCP 怎么做渐进式加载？沿着 `useManageMCPConnections.ts:858` 的 `useEffect` → `getMcpToolsCommandsAndResources` → `processServer` 完整链路答。
- [ ] **主问 7.Q8** `src/services/mcp/client.ts:2307-2322` 跳过 needs-auth 缓存命中的 server，为什么注释里特别强调"print mode 会 await 整个 batch"？这跟渐进加载有什么关系？
- [ ] **主问 7.Q9** stdio MCP 和 HTTP/SSE MCP 在"启动慢"这件事上各自的瓶颈是什么？为什么本地并发要压低、远程可以放高？

> 带读笔记（已讨论）
>
> - **§7.P2 OS 层补充：虚拟内存、MMU、pipe 物理路径**（深挖 stdio transport 底层）
> - **stdio MCP 的 transport 真身**：`src/services/mcp/client.ts:950` 的 `StdioClientTransport` 不传 URL / port，只有 `command / args / env / stderr: 'pipe'`。底层是 `child_process.spawn` → libuv `uv_pipe_t` → OS 匿名 pipe。**spawn 一个子进程，靠 stdin/stdout 通信，没有任何端口监听**。常见误解"本地起进程监听端口"是把 stdio transport 和 SSE/HTTP transport 混了。
> - **同机两进程数据流——pipe 是 2 次拷贝（不是 socket 的 3-4 次）**：MCP server `write(fd, buf, 8KB)` → trap 进内核 → `copy_from_user` 拷到内核 pipe ring buffer → epoll 唤醒 Claude Code → `read(fd, ...)` → `copy_to_user` 拷到 Node Buffer。**两次 syscall + 两次拷贝**。
> - **pipe ring buffer 长什么样**：内核里是 `struct pipe_inode_info`（`include/linux/pipe_fs_i.h`），16 个 `pipe_buffer` 槽位，每个指向一个 page（4KB），Linux 默认总容量 **64 KB**。**超过 64KB 写端在 `write()` 里阻塞** → 这就是 pipe 的天然背压，MCP server 输出比 Claude Code 读得快不会撑爆内存。
> - **为什么 pipe 比 loopback TCP 快很多**：pipe 在 VFS 层直接 `copy_from_user → ring buffer → copy_to_user`，**完全不走 TCP/IP 协议栈**——不算 checksum、不走 netfilter、不进 TCP 拥塞窗口逻辑。loopback socket 即使不出网卡也要走完整协议栈，所以更贵。
> - **8KB pipe 传输的真实开销**：每次 syscall ~100 ns + 8KB `copy_*_user` ~1-2 µs → 单向 ~几 µs。**stdio MCP 启动慢的瓶颈在 spawn + handshake，不在数据传输本身。**
> - **用户空间 vs 内核空间——核心模型**：
>   - 每个进程有自己**独立的虚拟地址空间**（x86-64 上 256 TB，128 TB 用户 + 128 TB 内核）。
>   - **用户空间跟着进程走**：进程死了页表撤销，物理页回收。A 进程的虚拟地址 `0x1000` 和 B 进程的 `0x1000` **指向不同的物理页**。
>   - **内核空间在所有进程间共享**：每个进程的页表里都把高 128 TB 映射到**同一份物理内存**——所以 syscall 不需要换页表、不刷 TLB，这是 syscall 快的物理原因。
>   - **安全靠页表 U/S bit**：用户空间页 `U=1`，内核空间页 `U=0`。用户态访问 `U=0` 直接 page fault。
> - **为什么 IPC 必须借道内核**：A 进程 user space 和 B 进程 user space 物理上独立，A 直接给 B 一个虚拟地址毫无意义（在 B 的页表里要么没映射、要么映射到别的物理页）。**只有内核空间是共享场地**，所以 pipe 必须 user → kernel → user 中转一次。
> - **虚拟内存是懒分配（demand paging）**：256 TB 虚拟空间不会爆物理内存。`mmap(1TB)` 立即返回成功，物理 RAM 一字节没动。**真写入才触发 page fault**，OS 在 handler 里分配物理页填进页表。Linux overcommit + OOM Killer 兜底。
> - **页表（page table）≠ 数据库页表**——纯术语撞车。OS 页表是**虚拟地址 → 物理地址映射表**。x86-64 是 **4 级页表**：PGD → PUD → PMD → PTE，每级 9 bit 索引，48 bit 虚拟地址 = 9+9+9+9+12（页内偏移）。**CR3 寄存器存当前进程顶级页表物理地址**，换进程就换 CR3。源码 `arch/x86/include/asm/pgtable.h`。
> - **真拷贝 vs 改指针（splice/vmsplice）**：
>   - `read/write` 真拷贝：8KB 字节级搬运经过 CPU 寄存器和 cache，~1-2 µs，物理 RAM 里数据**两份**。
>   - `splice` 改指针：内核改 `pipe_buffer.page` 字段指向源 fd 已有的物理页，~10 ns，物理 RAM 里数据**一份**。
>   - **100x 开销差距**，但 zero-copy 要求页对齐、源页能 refcount 锁住，普通 `write(buf, n)` 不满足。Claude Code MCP 走 `read/write`，因为数据要进 V8 做 JSON 解析，零拷贝意义不大。
> - **可复用的认知锚点**：被问"两个进程怎么通信"时，三句话框架 → ①各自用户空间隔离，②必须借共享的内核空间中转，③具体走 pipe / socket / mmap shared / shm 决定中转方式和拷贝次数。pipe 是其中最便宜的"不共享数据但中转一次"方案。
> - **跟 MCP 主线的回扣**：理解这些之后再回头看 `client.ts:2266-2271` 的 local/remote 分组——**local 组的瓶颈是 spawn 子进程的开销（fork+exec+解释器启动），不是 pipe 数据传输**；remote 组的瓶颈是 TLS + OAuth 网络往返。两类瓶颈完全不同，所以必须分两个 pMap 池调度。
>
> - **认知校准 1：CPU 硬件确实"读"虚拟地址，不矛盾**。面试常见反问"硬件不是只能读物理地址吗？"——精确答：DRAM 和内存总线只接受物理地址；但 CPU 芯片内部前段（寄存器、ALU、AGU）操作的就是虚拟地址电信号，**MMU 是芯片内"虚拟世界"和"物理世界"的硬件边界**，地址在芯片内部被 MMU 翻译完才送出芯片到 DRAM。所以"硬件只能读物理地址"对一半——指的是芯片外部的硬件（DRAM/总线）。
>
> - **认知校准 2：MMU + TLB + 页表的最小心智模型**：MMU 是每个 CPU 核里一块翻译电路；TLB 是 MMU 自带的硬件缓存，记最近用过的"虚拟页→物理页"映射，命中 1 cycle；TLB miss 时硬件自动走 4 级页表，~30-100 cycle。换进程换 CR3 寄存器（指向新页表），现代 CPU 用 PCID 避免全刷 TLB。**深挖到这一层即可——面试官也是学软件的，再深就两败俱伤**。
>
> - **面试场口播版（米哈游风格 5 分钟）**：见 §16 模板六。

---

## 8. 记忆与上下文：Agent 如何“记住”

一句话：Memory 是给 agent 用的本地知识库；Compact 是上下文快爆时的摘要机制。

### 代码地图

- `src/QueryEngine.ts:318-321`：`loadMemoryPrompt()` 进入 system prompt。
- `src/memdir/paths.ts:208-254`：auto-memory 目录和 `MEMORY.md` 路径。
- `src/memdir/memdir.ts:188-263`：typed-memory 行为说明。
- `src/memdir/memdir.ts:419-502`：`loadMemoryPrompt()` 主逻辑。
- `src/memdir/findRelevantMemories.ts:18-90`：扫描 memory 文件并筛选相关项。
- `src/memdir/memoryTypes.ts:38-181`：user/feedback/project/reference memory 类型。
- `src/services/extractMemories/extractMemories.ts:296-421`：自动记忆提取初始化。
- `src/services/extractMemories/extractMemories.ts:594-611`：query loop 结束后的 extraction/drain。
- `src/services/compact/autoCompact.ts:160-313`：自动 compact 触发条件。
- `src/services/compact/compact.ts:387-655`：传统 compact 主流程。
- `src/services/compact/compact.ts:1125-1133`：compaction agent 禁止 tool use。
- `src/services/compact/sessionMemoryCompact.ts:514-617`：session memory compact。

### 8.1 记忆注入

- [ ] **前置 8.P1** `src/memdir/` 存什么？memory 文件格式为什么是 frontmatter + body？

  **面试官会怎么问**：memory 为什么不直接塞数据库？为什么是 Markdown 文件？

  **代码解释**：memory 按 git root 隔离，文件里 frontmatter 存类型、元数据，body 存自然语言内容。类型包括 user、feedback、project、reference。

  **面试者回答**：Markdown 文件对 CLI agent 很友好：可读、可编辑、可用普通文件工具检索。frontmatter 让系统能按类型筛选，body 让模型直接读懂。它牺牲了复杂查询能力，换来透明和易调试。

- [ ] **主问 8.Q1** `loadMemoryPrompt()` 怎么被调用？记忆在哪一步注入 system prompt？

  **代码解释**：`QueryEngine.submitMessage()` 组装 system prompt 时调用 `loadMemoryPrompt()`。返回的 memory prompt 被加入 system prompt 数组，再传给 `query()`。

  **面试者回答**：memory 注入发生在模型调用之前，不是工具调用之后。每轮用户消息进来，QueryEngine 先加载相关 memory，把它作为 system prompt 的一部分交给模型。模型从第一轮推理就能看到这些长期信息。

- [ ] **主问 8.Q2** 记忆分类对 prompt 和缓存有什么影响？

  **代码解释**：user/project/feedback/reference 会影响组织方式和筛选。因为 memory 是动态内容，它通常属于每个 session 会变化的部分，放进缓存前缀会破坏 global prompt cache。

  **面试者回答**：分类的价值是减少噪音。用户偏好、项目事实、纠错反馈、外部引用不是一种东西，混在一起会让模型难判断优先级。缓存上，memory 是动态段，不能和万年不变的 system prompt 混在一起。

### 8.2 自动记忆提取

- [ ] **主问 8.Q3** 自动提取什么时候触发？提取什么内容？

  **代码解释**：query loop 结束后进入 extraction/drain 逻辑。系统检查主 agent 是否已经写过 memory，没写时可以 fork 一个 extract agent，从对话里提取用户偏好、项目约束、决策和 reference。

  **面试者回答**：它是事后提取，不阻塞主回答太多。提取对象不是所有聊天内容，而是长期有用的信息，比如用户偏好、项目事实、重复纠正。它像 RAG 的自动入库。

- [ ] **主问 8.Q4** 记忆冲突或过时怎么办？

  **代码解释**：memory 文件可被 Write/Edit 更新或删除；自动提取逻辑也会避免重复写。冲突处理更多靠 prompt 规则和后续编辑，不是强事务系统。

  **面试者回答**：文件 memory 的弱点是冲突治理不如数据库。我的理解是它靠透明性解决一部分问题：用户或 agent 能直接修改文件。生产系统如果要强一致，需要版本、时间戳、置信度和过期策略。

### 8.3 Context 压缩

- [ ] **前置 8.P2** compact 在 Agent 上下文中是什么意思？

  **面试官会怎么问**：compact 是压缩文件吗？

  **代码解释**：compact 是压缩对话历史。上下文窗口快满时，用 LLM 生成摘要，把旧消息替换成 compact summary，同时保留必要尾部消息。

  **面试者回答**：它压的不是磁盘文件，而是 prompt 里的历史消息。目标是保留任务意图、关键决策、已改文件、下一步，同时释放 context window。

- [ ] **主问 8.Q5** 什么条件触发 compact？策略是什么？

  **代码解释**：`shouldAutoCompact()` 用 token 预算判断：当前消息 token 超过 contextWindow - maxOutputTokens - buffer。compact 会 fork agent 生成结构化摘要，Session Memory 路径会保留尾部 10K-40K token，Legacy 路径更激进。

  **面试者回答**：触发不是模型觉得“该总结了”，而是工程硬阈值。压缩时用同模型生成摘要，摘要要覆盖原始请求、用户消息、完成进展、剩余任务。压缩后后续 tool call 只能看到摘要和保留尾部，所以摘要质量会直接影响继续工作的能力。

- [ ] **主问 8.Q6** Prompt Cache 和 compact 是什么关系？cache 命中高还需要 compact 吗？

  **代码解释**：Prompt Cache 降低重复前缀计算成本，但不扩大 context window。Compact 是为了解决上下文长度限制。两者不同层面：一个省钱省延迟，一个释放窗口。

  **面试者回答**：需要。cache 命中高，只说明相同前缀重算便宜，不代表模型能无限读历史。上下文满了还是要 compact。更有意思的是 compact fork 会利用已有 cache，把原历史作为热前缀发送，减少摘要调用成本。

### 8.4 与项目对比

- [ ] **主问 8.Q7** vector index 和 file memory 各有什么优劣？

  **面试者回答**：vector index 适合大量文档和语义检索，但不透明、更新和调试成本高。file memory 适合少量高价值事实，透明、可编辑、易审查，但检索能力弱。Agent 的长期偏好和项目约束更适合 file memory；大规模知识库更适合向量索引。

- [ ] **主问 8.Q8** 推荐系统能不能从用户行为日志自动提取 user profile memory？

  **面试者回答**：可以，但要谨慎。点击、收藏、停留时长能提取偏好，但容易误判。explicit feedback 置信度高但稀疏；implicit behavior 覆盖广但噪声大。生产里应该加置信度、过期时间和用户可编辑机制。

> 带读笔记（已讨论）
>
> - 关于 session memory 是 **schema-driven extraction** 的当前理解：extractor agent 没有任何针对"工具可靠性"的专用代码。能不能捕到"工具反复失败"这种 meta-knowledge，**完全取决于 template 的 section 写没写**。`src/services/SessionMemory/prompts.ts:11-41` 的 `DEFAULT_SESSION_MEMORY_TEMPLATE` 里有三栏直接接住这种 fact：
>     - **Workflow**：什么 bash 命令有效，按什么顺序
>     - **Errors & Corrections**：什么方法失败了，不要再试（原文 *"What approaches failed and should not be tried again?"*）
>     - **Learnings**：什么有效、什么无效、要避开什么
> - 关于 extractor 工作方式的当前理解：`buildSessionMemoryUpdatePrompt()`（`prompts.ts:226-247`）只做三件事——装载 template、按 token 预算提醒压缩、变量替换。**没有任何检测工具失败的代码**。extractor 是普通 fork agent，靠 `createMemoryFileCanUseTool()`（`sessionMemory.ts:460-481`）限制只能 Edit memory 文件。
> - 可复用模式 / trade-off：**结构决定行为，不是代码决定行为**。Anthropic 没写 runtime 显式的"工具可靠性追踪器"，他们把对失败的反思当成**普通会话学习**，用 markdown section 的 schema 编码进 extraction prompt。
>     - **优势**：轻、通用、能覆盖训练时没预想到的失败模式
>     - **代价**：依赖 extractor agent 的判断力，没有强保证；section 写得不好，meta-knowledge 就抓不到
> - 与千问 AI Coding case 的对照：千问 Agent 每次给新路径都要重头试 Read tool 三次再退 bash——同一 session 内学不会。如果它有 Claude Code 这种 session memory 模板，Errors & Corrections 栏会自动落 "Read tool 反复失败，从第一次就用 cat"，下一轮主 agent 加载 memory 就避开了。
> - **跟 §3.Q2 denialTracking 串起来看**：runtime 层（`denialTracking.ts` 的 3/20 双阈值）+ schema 层（session memory 的 Workflow/Errors/Learnings）是**互补的双层防线**——runtime 层用显式状态机捕熔断时机，schema 层用 prompt-driven 提取捕长期教训。Qwen 那次缺的是这两层全部。
> - 还没展开的问题：truncateSessionMemoryForCompact() 里对 section 的截断会不会优先压掉 Errors & Corrections？如果会，是不是反而稀释了这层防线？

---

## 9. 高压追问：测试、安全与错误恢复

一句话：Agent 测试不是测函数，是测一串 LLM 决策和工具副作用。

### 代码地图

- `src/query.ts:554-557`：tool_use 检测不能只信 stop_reason。
- `src/query.ts:1003-1017`：interrupted/queued/in-progress tool 生成 synthetic tool_result。
- `src/services/tools/toolExecution.ts:338-413`：不存在 tool name 返回 `<tool_use_error>`。
- `src/services/tools/toolExecution.ts:615-683`：Zod validation 失败变成 `InputValidationError`。
- `src/services/tools/toolExecution.ts:916-1039`：permission deny/ask 变成 tool_result。
- `src/services/api/errors.ts:666-729`：tool_use mismatch、unexpected tool_result、duplicate id。
- `src/utils/bash/ast.ts:9-18`：AST walker 不是 sandbox，只是权限检查输入抽取。
- `src/utils/bash/heredoc.ts:333-451`：heredoc 命令走私防护。
- `src/tools/BashTool/bashPermissions.ts:161-172`：命令 token 和环境变量赋值处理。
- `src/tools/BashTool/bashPermissions.ts:778-930`：exact/prefix/wildcard 规则匹配。

### 9.1 测试 Agent

- [ ] **主问 9.Q1** Agent 循环怎么测试？mock Anthropic API 响应模拟 tool_use 往返怎么做？

  **面试者回答**：我会 mock 模型 stream：第一轮返回 assistant text + tool_use，断言系统调用了正确 tool；tool 返回后，第二轮 mock 模型看到 tool_result 并返回 final answer。断言 messages 顺序、tool_use id、tool_result id、最大轮次和中断路径。重点不是测试模型聪不聪明，而是测试 loop 状态机。

- [ ] **主问 9.Q2** Tool 的单元测试怎么写？以 GrepTool 为例。

  **面试者回答**：构造临时目录和文件，传入合法 input，断言返回匹配结果；传入非法 pattern/path，断言校验错误；模拟权限 deny，断言不会执行实际 grep。只读工具还要测 `isReadOnly()` 和并发安全属性。

### 9.2 安全边界

- [ ] **主问 9.Q3** Bash tool 的 sandbox 怎么防命令注入和 `rm -rf /`？

  **代码解释**：源码明确 AST walker 不是 sandbox。它的作用是解析命令结构，抽取权限判断输入。安全来自多层：命令解析、heredoc 防走私、规则匹配、危险 argv 检查、permission prompt、外部 sandbox。

  **面试者回答**：不能说“AST parser 就是 sandbox”。它只是看懂命令，帮助权限系统判断。真正安全要靠权限规则、危险命令识别、用户审批和运行环境 sandbox。`rm -rf /` 这类命令要被危险参数检查和权限层拦住。

- [ ] **主问 9.Q4** Prompt injection 攻击入口有哪些？防护是什么？

  **面试者回答**：入口包括用户文件、网页抓取、MCP server 返回、命令输出、剪贴板和历史消息。防护思路是分层标注来源、不要把外部内容当 system prompt、工具权限独立于模型文本、危险操作必须过代码级 permission。最关键的是：外部文本不能改变工具权限。

### 9.3 容错

- [ ] **主问 9.Q5** tool call 中途 crash，resume 能恢复到什么程度？

  **面试者回答**：已经写入 transcript 的消息能恢复，已经落盘的工具结果可能能恢复。内存里的临时进度、未落盘 stdout、未完成 permission prompt 可能丢。含副作用工具最难恢复，因为系统未必知道外部世界已经改变到哪一步。

- [ ] **主问 9.Q6** LLM 返回不存在 tool name 或参数校验失败怎么办？

  **代码解释**：不存在 tool name 会返回 `<tool_use_error>`；Zod 校验失败转成 `InputValidationError`；结果仍包装为 tool_result 让模型有机会自我修正。

  **面试者回答**：不要 crash，也不要静默忽略。把错误作为 tool_result 回给模型，让模型知道“这个工具不存在”或“参数错了”。这样 loop 能自愈一部分错误。

- [ ] **主问 9.Q7** 你的 software_reco agent 生产最怕什么故障？

  **面试者回答**：我最怕三类：外部 API 慢或挂、检索结果质量差但生成仍很自信、用户画像被错误更新。对应措施是超时和降级、检索指标和答案引用、profile 写入审批或置信度门槛。

---

## 10. 反向提问：架构决策压力测试

一句话：这章不是知识检查，是逼你做取舍。

### 10.1 工具设计

- [ ] **反向 10.Q1** Claude Code 有 42 个 tool。如果砍到只剩 10 个，保留哪些？

  **面试者回答**：我会保留 Read、Grep、Glob、Edit、Write、Bash、TodoWrite、AgentTool、SkillTool、WebFetch。标准是覆盖“看代码、搜代码、改代码、跑验证、规划任务、委托子任务、按流程工作、查外部资料”。砍掉低频、模式切换类、实验性和高度场景化工具。Bash 虽危险但不可替代，所以保留但加强权限。

  **追问回答**：如果是只读审查模式，我会砍掉 Edit/Write/Bash，保留 Read/Grep/Glob/WebFetch/AgentTool 这类只读能力。

- [ ] **反向 10.Q2** AgentTool 很危险，你会加什么限制？

  **面试者回答**：限制四件事：最大子 agent 数、最大 turns、工具白名单、工作区隔离。默认只读探索，写操作必须 worktree 隔离或父 agent 审批。还要记录子 agent 的完整 transcript，方便追责。

### 10.2 Agent 循环

- [ ] **反向 10.Q3** QueryEngine tool loop 的致命弱点是什么？10 轮还没收敛怎么办？

  **面试者回答**：弱点是不可控成本和循环漂移。模型可能不断查工具但不推进目标。保护手段是 maxTurns、maxBudget、重复 tool call 检测、同类错误熔断、要求模型在 N 轮后总结阻塞点并请求用户确认。

- [ ] **反向 10.Q4** software_reco 加 tool calling，先给哪几个 tool？循环怎么设计？

  **面试者回答**：我会先给只读工具：keyword_search、vector_search、rerank、get_user_profile、get_item_detail。写工具如 update_profile 先不开放或加审批。循环设计成 LLM decision -> tool router -> tool result -> LLM，最多 3 到 5 轮，最终必须输出推荐理由和引用来源。

### 10.3 MCP 取舍

- [ ] **反向 10.Q5** 什么场景值得用 MCP，什么场景不如写死在 agent 里？

  **面试者回答**：跨团队、跨产品、工具经常变、需要第三方生态时用 MCP。核心高频低延迟路径别用 MCP，直接 SDK 或内部 API。MCP 省胶水代码，但引入认证、连接、版本和远端故障。

### 10.4 RAG 到 Agent

- [ ] **反向 10.Q6** RAG pipeline Agent 化最大收益和风险是什么？保留哪些静态部分？

  **面试者回答**：收益是能按 query 类型动态选择策略。风险是延迟、成本、不可复现和评估变难。我会保留 ingest、chunk、embed、index build、基础 retrieve/rerank 这些静态部分，只把策略选择和多轮澄清交给 agent。

- [ ] **反向 10.Q7** Recall@K / NDCG 能直接搬到 Agent 评估吗？

  **面试者回答**：不能直接搬。Recall@K 测检索器，Agent 还要测任务完成率、工具调用成本、事实正确率、用户满意度、是否遵守权限。可以保留 retrieval metrics 作为子指标，但总评估要看 end-to-end outcome。

- [ ] **反向 10.Q8** 用 LangGraph 复刻 mini QueryEngine，几个 node、几个 edge？

  **面试者回答**：最小图：`prepare_context -> llm -> route_tool_or_final -> execute_tool -> llm`。conditional edge 根据 LLM 输出决定 final 还是 tool。State 里放 messages、tool_results、turn_count、budget、permission_context。再加错误边：validation_error、permission_denied、max_turns。

---

## 11. Hooks 系统：Agent 生命周期的拦截器

一句话：Hook 不回答用户问题，只在关键事件前后插一脚：拦、改、记日志。

### 代码地图

- `src/entrypoints/sdk/coreTypes.ts:25`：HOOK_EVENTS 来源。
- `src/schemas/hooks.ts:31-128`：command/prompt/http/agent hook schema。
- `src/schemas/hooks.ts:176-212`：Hooks 配置结构。
- `src/types/hooks.ts:49-171`：hook JSON 输出协议。
- `src/utils/hooks/hooksConfigManager.ts:29-104`：每种事件的说明和 exit code 行为。
- `src/utils/hooks/hooksConfigManager.ts:269-399`：按 event 和 matcher 分组。
- `src/utils/hooks.ts:418-651`：hook 输出解析成 blocking、permission decision、updatedInput。
- `src/utils/hooks.ts:1338-1378`：matcher 匹配 tool/event。
- `src/utils/hooks.ts:1952-2968`：`executeHooks(...)` 核心执行器。
- `src/utils/hooks.ts:3410-3448`：PreToolUse。
- `src/utils/hooks.ts:3460-3490`：PostToolUse。
- `src/utils/hooks.ts:3826-3847`：UserPromptSubmit。
- `src/utils/hooks.ts:3867-3884`：SessionStart。
- `src/utils/hooks.ts:4097-4131`：SessionEnd。
- `src/services/tools/toolHooks.ts:322-638`：PreToolUse hook 影响 permission decision。

### 11.1 Hook 事件类型

- [ ] **前置 11.P1** settings.json 里 hook 配置长什么样？

  **面试者回答**：hooks 配置按事件名分组，每个 hook 有 matcher、command 或其他执行方式、timeout、是否允许 block 等字段。matcher 决定它对哪些工具或事件生效。

- [ ] **主问 11.Q1** 有哪些 hook 事件？各在生命周期哪触发？

  **面试者回答**：核心事件有 PreToolUse、PostToolUse、UserPromptSubmit、Stop、SessionStart、SessionEnd 等。PreToolUse 在工具执行前，PostToolUse 在工具结果后，UserPromptSubmit 在用户输入进入主流程前，SessionStart/End 在会话边界。

- [ ] **主问 11.Q2** PreToolUse hook 怎么影响 tool 是否执行？能改 input 吗？能静默拒绝吗？

  **代码解释**：hook 输出会被解析成 blocking、permission decision、updatedInput。PreToolUse 可以阻断、要求 deny/ask/allow，也可以修改输入。

  **面试者回答**：PreToolUse 是最危险也最有用的 hook。它能在工具真正执行前看见 tool name 和 input，然后决定放行、阻断或改写。安全上必须限制它的能力和超时。

### 11.2 执行模型

- [ ] **前置 11.P2** hook command 怎么执行？有没有 sandbox？

  **面试者回答**：hook command 本质是外部命令或脚本执行，有 timeout 和输出协议。它不天然等于安全 sandbox。能跑 shell 就意味着有副作用，所以配置权限和执行环境很重要。

- [ ] **主问 11.Q3** hook 是同步阻塞还是异步？超时怎么办？

  **面试者回答**：PreToolUse 这类影响决策的 hook 必须阻塞当前工具执行，否则来不及拦截。异步 hook 可用于不阻塞的观察或后台响应。超时要按配置处理，安全敏感场景应 fail-closed。

- [ ] **主问 11.Q4** hook 能拿到什么上下文？跟 middleware 类比是什么？

  **面试者回答**：它能拿到事件名、tool name、tool input、上下文 JSON 等，类似 API middleware 拿 request headers/body。区别是这里的 request 是 LLM 发起的 tool call，不是用户 HTTP 请求。

- [ ] **主问 11.Q5** 多个 hook 同一 event，执行顺序和失败行为怎样？

  **面试者回答**：系统按配置分组和顺序执行。一个 hook 失败是否短路取决于事件类型和输出/exit code 行为。设计上要避免多个 hook 同时修改同一 input 造成不可预测。

### 11.3 Hook 设计模式

- [ ] **主问 11.Q6** Hook 跟 Tool 有什么区别？为什么日志要写 hook 而不是 tool？

  **面试者回答**：Tool 是模型主动调用的能力；Hook 是系统在生命周期自动触发的拦截器。日志、审计、格式化、策略检查不应该让模型决定要不要调用，所以应该做成 hook。

- [ ] **主问 11.Q7** 怎么判断需求是 hook 还是 tool？

  **面试者回答**：如果需求是“模型需要一种新能力”，做 tool。如果需求是“每次某事件发生都要自动做”，做 hook。比如查股价是 tool，记录每次 Bash 命令是 hook。

- [ ] **主问 11.Q8** RAG pipeline 改成 Hook 模式，哪些环节受益？风险是什么？

  **面试者回答**：受益点是审计、query rewrite、结果过滤、A/B 实验、权限检查。风险是用户脚本影响核心路径、延迟不可控、数据泄露。生产里要限制 hook API，而不是直接给 shell。

- [ ] **主问 11.Q9** 推荐系统 hook 能力边界怎么限制？

  **面试者回答**：我会默认只读，只允许访问白名单上下文和输出结构化 JSON。需要写操作的 hook 走审核或沙箱。不要给业务用户任意 shell，这等于把生产系统交出去。

---

## 12. 提示词工程：写给 LLM 的系统文档

一句话：Claude Code 的 prompt 不是一段话，是一个带缓存边界的动态文档系统。

### 代码地图

- `src/constants/prompts.ts:114`：`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`。
- `src/constants/prompts.ts:175-184`：Intro 段。
- `src/constants/prompts.ts:199-253`：Doing Tasks 段。
- `src/constants/prompts.ts:269-314`：Using Your Tools 段。
- `src/constants/prompts.ts:402-428`：Output Efficiency 段。
- `src/constants/prompts.ts:430-442`：Tone and Style 段。
- `src/constants/prompts.ts:444-577`：`getSystemPrompt()`。
- `src/constants/prompts.ts:491-554`：dynamicSections。
- `src/constants/systemPromptSections.ts:20-37`：cacheable vs uncached sections。
- `src/tools/GrepTool/prompt.ts:6-18`：工具 description 写法。
- `src/tools/AgentTool/prompt.ts:66-80`：AgentTool 动态 prompt。
- `src/constants/outputStyles.ts:41-135`：OutputStyle 配置。
- `src/utils/systemPrompt.ts:41-121`：effective system prompt 优先级。
- `src/utils/api.ts:321-435`：`splitSysPromptPrefix()` 切缓存前缀。
- `src/utils/claudemd.ts:790`：CLAUDE.md 作为 user message 注入。

### 12.1 System Prompt 结构化写作

- [ ] **前置 12.P1** Intro 第一话为什么是身份声明？

  **面试者回答**：第一句话定 agent 的角色边界：它是 interactive coding agent，不是普通聊天助手。后续工具、代码修改、终端交互规则都建立在这个身份上。删掉会让模型行为更泛化。

- [ ] **主问 12.Q1** `getSystemPrompt()` 为什么返回 `string[]` 而不是大字符串？

  **面试者回答**：段数组方便独立维护、按条件插入、做缓存边界切分和测试。一个大字符串很难知道哪段静态、哪段动态，也不利于 prompt cache。

- [ ] **主问 12.Q2** Doing Tasks 段大量否定式指令为什么有效？

  **面试者回答**：因为 coding agent 最常见风险是过度发挥。`Don't add features`、`Don't refactor unrelated code` 这种否定式能明确边界，比“写干净代码”更可执行。它是在限制破坏面。

- [ ] **主问 12.Q3** 为什么简洁输出要单独成段？

  **面试者回答**：输出风格是全局行为，不只影响 coding task。把它拆成独立段，便于不同用户类型和 output style 覆盖，也减少和任务规则混在一起的冲突。

### 12.2 静态 vs 动态：缓存边界

- [ ] **前置 12.P2** `cacheBreak` 控制什么？

  **面试者回答**：它控制某段 system prompt 是否切断缓存前缀。稳定内容可以放在 boundary 前并打 cache_control，动态内容要放后面，避免每次变化导致前缀缓存失效。

- [ ] **主问 12.Q4** 哪些内容放 boundary 前，哪些放后？

  **面试者回答**：产品级固定规则、工具通用规则、语气规范适合放前面。当前日期、cwd、memory、MCP 状态、用户配置、session guidance 必须放后面。把动态内容放前面会让 cache key 碎片化。

- [ ] **主问 12.Q5** `getSessionSpecificGuidanceSection()` 为什么放 boundary 后？

  **面试者回答**：它有很多条件分支。放在 static 段会造成 2^N 种前缀组合，global cache 几乎命不中。放后面牺牲一点动态 token，但保住大块静态 prompt 的复用。

### 12.3 Tool Description 写法

- [ ] **前置 12.P3** `GrepTool/prompt.ts` 的 description 最终去了哪里？

  **面试者回答**：它被序列化成 API tools 数组里的 tool description。模型在决定是否调用工具时看到它。

- [ ] **主问 12.Q6** ALWAYS/NEVER 模式为什么有效？和 inputSchema 分工是什么？

  **面试者回答**：description 教模型什么时候用工具，inputSchema 限制参数怎么填。ALWAYS/NEVER 是策略指令，schema 是形状约束。两者不能互相替代。

- [ ] **主问 12.Q7** AgentTool description 为什么动态，GrepTool 可以静态？

  **面试者回答**：GrepTool 能力固定。AgentTool 可用 agent 列表、fork/explore guidance、feature flag 都可能变，所以 description 或附件必须动态生成。

- [ ] **主问 12.Q8** agent list 从 tool description 搬到 attachment message 跟 cache 有什么关系？

  **面试者回答**：动态 agent list 如果塞进 tool description，会污染 tools 数组缓存，导致大量 cache_creation tokens。搬到 message attachment，可以把稳定 tool schema 留在缓存前缀里。

### 12.4 Agent Prompt 特化

- [ ] **前置 12.P4** AgentDefinition 的 prompt 字段有哪些？

  **面试者回答**：核心是 `whenToUse`、`getSystemPrompt()`、tools、model、permissionMode、skills。`whenToUse` 帮父 agent 选择，`getSystemPrompt()` 定义子 agent 行为，tools 限制能力边界。

- [ ] **主问 12.Q9** 子 agent system prompt 怎么构建？为什么不直接吃主 prompt？

  **面试者回答**：子 agent 有自己的任务和角色，直接继承主 prompt 会带来噪音和过宽能力。它用 agentDefinition 的 system prompt，再增强环境细节。这样职责更窄，成本更低。

- [ ] **主问 12.Q10** Explore agent 砍 CLAUDE.md 和 git status 节省什么？

  **面试者回答**：节省输入 token 和 cache creation/read 成本，也减少无关上下文干扰。Explore 只需要快速看代码，不一定需要全部项目记忆和 git 状态。

- [ ] **主问 12.Q11** 子 agent 默认关闭 thinking，为什么？

  **面试者回答**：子 agent 通常是短任务 worker，默认开启 thinking 会增加成本和延迟。只有需要精确继承主上下文的 fork child 才值得继承 thinking config。

### 12.5 Output Style

- [ ] **前置 12.P5** Explanatory / Learning / Default 替换什么？

  **面试者回答**：OutputStyle 改的是 agent 的交互风格和部分 intro/行为指导。Default 偏高效执行，Explanatory 偏解释，Learning 偏教学。

- [ ] **主问 12.Q12** outputStyle 可切换，会不会 bust cache？

  **面试者回答**：会影响 effective prompt，所以设计上要尽量把可变 style 放在动态或可控段。否则每个 style 都会形成不同 cache 前缀。取舍是个性化和 cache 命中率之间的平衡。

- [ ] **主问 12.Q13** OutputStyle 和 Skill 有什么区别？

  **面试者回答**：OutputStyle 是皮肤，改变长期说话和教学风格；Skill 是工作手册，告诉模型在特定任务中按某流程做事。一个常驻，一个按需。

### 12.6 CLAUDE.md

- [ ] **前置 12.P6** CLAUDE.md 最终去了哪里？

  **面试者回答**：它不是 system prompt 的静态部分，而是作为 user message/上下文注入。这样项目级说明可以动态变化，不污染全局 system prompt cache。

- [ ] **主问 12.Q14** CLAUDE.md 放 user message 和 system prompt 有什么区别？

  **面试者回答**：system prompt 优先级更高、更稳定；user message 更像项目上下文。CLAUDE.md 作为 user message 可以被项目覆盖和滚动，但不能当安全边界。

- [ ] **主问 12.Q15** CLAUDE.md 支持 include 有什么风险？

  **面试者回答**：include 提升复用，但会扩大上下文和引入隐性依赖。风险是 include 太多导致 token 膨胀，或者把不该给模型看的内容带进 prompt。

---

## 13. Session 恢复与 JSONL Transcript

一句话：Claude Code 用 JSONL 给每句话写日记，恢复时沿 UUID 指针重建对话链。

### 代码地图

- `src/utils/sessionStorage.ts:993`：`insertMessageChain()` 写入 session 元数据。
- `src/utils/sessionStorage.ts:1128`：`appendEntry()` 按 UUID 去重追加。
- `src/utils/sessionStorage.ts:1408`：`recordTranscript()` 主入口。
- `src/utils/sessionStorage.ts:2069`：`buildConversationChain()` 从叶子沿 parentUuid 回溯。
- `src/utils/sessionStorage.ts:2118`：`recoverOrphanedParallelToolResults()` 补并行 tool_use DAG 分支。
- `src/utils/sessionStorage.ts:3472`：`loadTranscriptFile()` 解析 JSONL。
- `src/utils/conversationRecovery.ts:456`：`loadConversationForResume()`。
- `src/utils/sessionRestore.ts:409`：`processResumedConversation()`。
- `src/commands/rewind/rewind.ts`：`/rewind`，别名 checkpoint。

### 13.1 Event Sourcing vs Snapshot

- [ ] **前置 13.P1** LangGraph checkpoint 和 JSONL 事件溯源有什么优劣？

  **面试者回答**：Snapshot 恢复快，因为直接读状态；缺点是每次存全量，空间大。JSONL 事件溯源写入便宜、可回溯任意时间点，但恢复要解析和重建链，速度慢。Claude Code 选择 JSONL，是因为对话天然是事件流。

- [ ] **主问 13.Q1** `parentUuid` 怎么写入？为什么不用数组索引？

  **面试者回答**：每条 transcript message 带 UUID 和 parentUuid，形成链。数组索引在并发写、插入、compact、分支恢复时不稳定；UUID 指针更适合事件日志和分叉。

- [ ] **主问 13.Q2** `buildConversationChain` 遇到 cycle 怎么办？

  **面试者回答**：健壮实现应该维护 visited set，遇到重复 UUID 就停止或报错，避免无限循环。面试里要强调：事件日志恢复不能默认数据永远干净。

- [ ] **主问 13.Q3** 并行 tool_use 的 DAG 分支怎么补？`message.id` 和 UUID 分工是什么？

  **代码解释**：UUID 是 transcript 记录自己的身份和 parent 指针；`message.id` 是 API 消息层面的 id。并行 tool_use 可能产生兄弟分支，`recoverOrphanedParallelToolResults()` 通过 message.id 找回同组兄弟。

  **面试者回答**：UUID 管日志链，message.id 管 API 语义上的同一条 assistant 消息。单链回溯会漏掉并行分支，所以要按 message.id 补兄弟节点。

- [ ] **主问 13.Q4** JSONL 文件删了或损坏怎么办？

  **面试者回答**：删了就无法完整 resume，只能从现有内存或其他副本恢复。损坏时可以尽量跳过坏行、保留可解析部分，但 tool side effects 无法凭空恢复。这里体现 transcript 是恢复基础设施，不是万能事务日志。

### 13.2 与项目对比

- [ ] **主问 13.Q5** 推荐系统刷新页面后继续，是用 Event Sourcing 还是 Snapshot？

  **面试者回答**：如果只是恢复一次推荐结果，用 snapshot 存 query、retrieved items、ranking scores、answer 更简单。如果需要审计用户多轮决策和 A/B 分支，event sourcing 更合适。

- [ ] **主问 13.Q6** `/rewind` 分叉对 RAG 有什么价值？

  **面试者回答**：它适合比较不同 retrieval 策略。用户可以回到同一 query 点，换 BM25-only、vector-only、hybrid 继续，比较结果。实现上要保存 checkpoint 或事件分支。

### 13.3 Compact-aware Resume

- [x] **前置 13.P2** `getMessagesAfterCompactBoundary()` 是怎么区分“原样保留全部历史”和“只保留 compact 后视图”的？

  **面试官会怎么问**：resume 时到底是把 JSONL 里的所有消息原样喂回模型，还是从 compact boundary 后开始？

  **代码解释**：`src/utils/messages.ts:4643-4655` 先找最后一个 compact boundary；没有 boundary 就返回原 messages，有 boundary 就从 boundary 开始 slice。`src/query.ts:365` 把这个 compact-aware view 作为模型请求的 messages。

  **面试者回答**：as-is 不是一个神秘模式，它就是“没有 compact boundary 时沿 parentUuid 重建出的链基本原样进入上下文”。compact-aware resume 则把最后一个 compact boundary 当切点：boundary 之前的原始长历史不再原样进入模型，改由摘要和必要保留段代表。

- [ ] **主问 13.Q7** compact 后 resume 到底保留了什么、丢掉了什么？`preservedSegment` 为什么需要 relink？

  **面试官会怎么问**：compact 不是删除历史吗？为什么还要 preservedSegment、anchor/head/tail 这些元数据？

  **代码解释**：`src/utils/sessionStorage.ts:1824-1955` 会把 compact 时保留的尾段 splice 回链里，并删除最后一个 compact boundary 之前不在 preserved segment 里的旧消息。这样 resume 不会重新加载完整 pre-compact 历史。

  **面试者回答**：compact 后保留三类东西：compact summary、boundary 后的新消息、必要的 preserved tail segment。丢掉的是 boundary 前被摘要覆盖的逐条原始消息。`preservedSegment` 是为了让尾部原消息继续可恢复，但它们在 JSONL 里还带着 pre-compact parentUuid，所以恢复时要在内存里重接链。

> 带读笔记（已讨论）
>
> - 关于 session memory / compact / resume 的当前理解：从空 session 开始，所有 user/assistant/tool 消息先正常进入 `messages` 并写入 JSONL transcript；JSONL 是完整日记本，负责恢复、审计、debug，不等于模型每轮都会完整重读。
> - 关于 session memory 的当前理解：当上下文达到门槛后，`src/services/SessionMemory/sessionMemory.ts:134-181` 的 `shouldExtractMemory()` 判断是否该抽取记忆；默认门槛在 `src/services/SessionMemory/sessionMemoryUtils.ts:32-36`：初始化至少 10K token，后续至少增长 5K token，并结合 tool call 数量。抽取时会用 `runForkedAgent()`（`sessionMemory.ts:315-325`）启动后台 fork agent 更新 session memory markdown，且 `createMemoryFileCanUseTool()`（`sessionMemory.ts:460-481`）只允许 Edit 那个 memory 文件。
> - 关于 `lastSummarizedMessageId` 的当前理解：session memory 成功更新后，`updateLastSummarizedMessageIdIfSafe()`（`sessionMemory.ts:488-494`）把“记忆已经总结到哪条消息”记录下来；它不会在最后一轮还含 tool call 时标记，避免切断 tool_use/tool_result。
> - 关于 auto-compact 的当前理解：`src/services/compact/autoCompact.ts:225-238` 用 token 估算和模型阈值判断是否快满；快满后 `autoCompactIfNeeded()` 会先尝试 `trySessionMemoryCompaction()`（`autoCompact.ts:287-292`）。这不是等窗口完全爆掉，而是留余量后主动压缩。
> - 关于第一次 compact 的傻瓜时间线：假设 messages 是 0..650，session memory 已经总结到 500。`trySessionMemoryCompaction()` 读取 `lastSummarizedMessageId` 和 session memory（`sessionMemoryCompact.ts:529-530`），然后 `calculateMessagesToKeepIndex()` 先从 `lastSummarizedIndex + 1` 开始，也就是 501（`sessionMemoryCompact.ts:334-339`）。含义是：0..500 的重点已经在 memory summary 里，先尝试只原样保留 501..650。
> - 关于保留尾段策略的当前理解：默认 `minTokens: 10_000`、`minTextBlockMessages: 5`、`maxTokens: 40_000`（`sessionMemoryCompact.ts:57-60`）。如果 501..650 太短，就从 startIndex 往前扩（`sessionMemoryCompact.ts:364-393`），直到满足最小 token + 最小文本消息数，或达到最大 token。最后通过 `adjustIndexToPreserveAPIInvariants()`（`sessionMemoryCompact.ts:232-314`）避免切断 tool_use/tool_result 以及同一 API message.id 的 thinking/tool_use 分片。
> - 关于 compact 后模型看到什么：`createCompactionResultFromSessionMemory()` 会创建 compact boundary（`sessionMemoryCompact.ts:447-451`），把 session memory 包装成 compact summary message（`sessionMemoryCompact.ts:464-482`），再加上 `messagesToKeep` 原文尾段（`sessionMemoryCompact.ts:487-497`）。所以模型桌面上是：compact boundary + session memory summary + 最近原文 tail + 后续新消息。
> - 关于第二次 compact 的傻瓜时间线：如果后台 session memory 又更新到了 800，那么第二次 compact 默认从 801 开始保留原文，0..800 由 summary 表达；如果后台 memory 还只总结到 500，那么系统不会假装 501..800 已经被记忆覆盖，会从 501 之后的未总结段和近期尾段里计算保留范围。
> - 可复用模式 / trade-off：JSONL = 完整日记本；session memory = 持续更新的复习笔记；compact = 把模型桌面清空，只留下复习笔记 + 最近几页原文；`lastSummarizedMessageId` = 复习笔记已经写到第几页；`messagesToKeep` = 这次还要摊在桌面上的最近几页原文。
> - 还没展开的问题：`preservedSegment` 的 head/anchor/tail 如何在 resume 时把保留尾段重新接回 parentUuid 链；传统 compact 和 session-memory compact 的差异；`loadTranscriptFile()` 如何跳过 pre-boundary 旧内容但保留 metadata。

---

## 14. Skills 披露机制：模型怎么知道有哪些 Skill

一句话：built-in tools 进 API `tools` 数组，skill 列表进 `<system-reminder>` 文本，所以 skill 可能被上下文滚掉。

### 代码地图

- `src/tools/SkillTool/prompt.ts:70-171`：`formatCommandsWithinBudget()` 技能列表预算截断。
- `src/tools/SkillTool/prompt.ts:173-196`：Skill tool description。
- `src/utils/attachments.ts:2607`：`sentSkillNames` 追踪已发送技能。
- `src/utils/attachments.ts:2661`：`getSkillListingAttachments()` 生成技能列表 attachment。
- `src/utils/messages.ts:1797`：`ensureSystemReminderWrap()` 包 `<system-reminder>`。
- `src/utils/api.ts:119-266`：`toolToAPISchema()` 把 Tool 序列化成 API function definition。

### 14.1 披露路径

- [ ] **主问 14.Q1** built-in tool 和 skill 的披露路径有什么不同？

  **面试者回答**：built-in tool 每次作为 API tools 数组发送，模型稳定看到 schema。Skill 只有一个 SkillTool router 在 tools 数组里，具体 skill 列表通过 system-reminder 文本注入 messages。它会受上下文滚动和预算截断影响。

- [ ] **主问 14.Q2** 为什么装了很多 skill 后模型会忘？

  **面试者回答**：三个原因：system-reminder 是 message，会滚出上下文；技能列表有预算，太多会截断描述；`sentSkillNames` 避免重复注入，导致后面不一定再次提醒。

- [ ] **主问 14.Q3** 为什么不把每个 skill 都做成 API tool？

  **面试者回答**：因为 skill 数量多、描述长、动态来源复杂。全放 tools 数组会膨胀 schema token，破坏 prompt cache，也让模型面对太多函数。一个 SkillTool router 更省，但牺牲稳定披露和强 schema 约束。

- [ ] **主问 14.Q4** SkillTool 的 `skill` 参数为什么不像 built-in tool 那样强枚举？

  **面试者回答**：skill 是动态加载的，可能来自用户、项目、插件、远端。强枚举会频繁变化并污染 tool schema。它用文本列表告诉模型有哪些 skill，再在运行时校验名字。

---

## 15. System Prompt 拼接与 Prompt Cache

一句话：不变内容放缓存边界前，可变内容放边界后。Prompt cache 省计算，不省上下文窗口。

### 代码地图

- `src/constants/prompts.ts:114`：`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`。
- `src/constants/prompts.ts:444-577`：`getSystemPrompt()`。
- `src/utils/api.ts:72-77`：`cache_control` 类型。
- `src/utils/api.ts:321-435`：`splitSysPromptPrefix()`。
- `src/utils/api.ts:119-266`：`toolToAPISchema()`。
- `src/services/api/claude.ts:601-604`：user message 上打 cache_control。
- `src/services/api/claude.ts:3228-3234`：system prompt block 上打 cache_control。
- `src/utils/forkedAgent.ts:46-68`：`CacheSafeParams`。
- `src/utils/forkedAgent.ts:524`：fork context messages 拼接。

### 15.1 Cache 控制

- [ ] **主问 15.Q1** `cache_control` 不是布尔值，它有哪些字段？

  **面试者回答**：它有 `type: 'ephemeral'`，可选 `scope: 'global' | 'org'`，可选 ttl。scope 决定缓存共享范围，ttl 决定有效时间。不是“开缓存/关缓存”这么简单。

- [ ] **主问 15.Q2** global cache 为什么能跨用户命中？

  **面试者回答**：服务端按相同前缀字节算 hash。Claude Code 的静态 system prompt 对所有用户都一样，所以可以 global scope。只要前缀完全一致且未过期，就能读缓存。

- [ ] **主问 15.Q3** KV Cache 底层到底缓存什么？

  **面试者回答**：缓存的是前缀 token 经过模型计算后的 K/V 投影状态，不是最终答案，也不是用户本地 GPU 显存。命中后服务端不用重算这段前缀的 K/V，后续 token 仍要参与 attention。

- [ ] **主问 15.Q4** 缓存写入更贵，为什么还值得？

  **面试者回答**：首次 miss 写入大概更贵，但后续命中很便宜。Claude Code 这种全世界都在用的固定前缀，命中率很高，所以把大块静态 prompt 缓存起来非常划算。

### 15.2 Compaction 与 cache

- [ ] **主问 15.Q5** Compaction fork 为什么把历史全塞进 prompt，而不是让子 agent 读文件？

  **代码解释**：`createCompactCanUseTool()` 禁止 tool use，`runForkedAgent` 传 `cacheSafeParams` 和 `maxTurns: 1`。fork 请求复用相同 system prompt、tools、model、messages prefix，服务端能命中已有 KV cache。

  **面试者回答**：因为历史消息本来就是热前缀。全塞 prompt 看起来大，但 cache 命中后大部分只按 cache read 计费。拆文件让子 agent 用 Read 反而会产生新的冷启动请求、工具往返和语义碎片。

- [ ] **主问 15.Q6** prompt cache 和 context compact 的边界是什么？

  **面试者回答**：cache 解决重复计算成本，compact 解决窗口容量。cache 再强也不能让模型读超过 context window 的内容。compact 再好也会丢细节。所以两者都要。

- [ ] **主问 15.Q7** 写 CLAUDE.md 或 skill 时，怎么提高 cache 友好性？

  **面试者回答**：高复用、稳定、不随项目变的规则放前面；项目路径、日期、用户偏好、临时状态放后面。不要在稳定段夹动态值。skill 描述也要短，动态列表不要污染 tools schema。

---

## 16. 最后给面试者的答题模板

### 模板一：讲主循环

> Claude Code 的核心是 LLM/tool loop。外层 QueryEngine 管会话上下文，比如 system prompt、memory、tools、permission context。内层 query() 负责调模型、解析 tool_use、执行工具、把 tool_result 回灌，再进入下一轮。循环结束不是靠一个特殊 stop 命令，而是 assistant 不再输出 tool_use，同时还有 maxTurns 和预算保护。

### 模板二：讲安全

> 这个系统的安全边界不能靠 prompt。Prompt 可以提醒模型，但真正 enforce 的是 permission layer、tool validation、Bash 命令解析、hooks 和 sandbox。尤其是 plan mode 的问题说明了：只写“不要编辑”不等于代码级禁止编辑。

### 模板三：讲取舍

> Claude Code 的设计不是所有地方都追求最简单，而是在本地副作用很强的场景下选择可控性。Tool schema、permission、hooks、session log、compact、cache 都是为了让 LLM 能行动，但行动不能失控。

### 模板四：把它迁移到自己的项目

> 我不会把推荐系统整条 pipeline 都交给 LLM。稳定高频的 retrieve/rerank/generate 保持静态，少数需要动态决策的部分封成 tool。再加只读默认放行、写操作审批、maxTurns、成本预算和 end-to-end 评估。

### 模板五：用真实失败案例切入 Agent 工程谈资（千问 Read tool 案例）

**适用场景**：面试官问"你对 Agent / LLM 工具调用的理解"或想找一个具体故事切入工程深度时。

**Bullet（可以直接背）**：

> 我做阿里 AI Coding 测试时发现千问的 Agent 每次遇到新路径都要重头试 Read tool 三次再退 bash——同一个 session 里学不会。后来读 Claude Code 源码发现它的 session memory template（`src/services/SessionMemory/prompts.ts:11-41`）里专门有三栏：Workflow、Errors & Corrections、Learnings，分别记"什么 bash 命令有用"、"什么方法失败了不要再试"、"什么有效什么无效"。Anthropic 没写专门的工具可靠性追踪器，他们用 markdown section 的 schema 把"对失败的反思"编码进了 session memory 的提取流程。我会把这看成 agent 设计里一个反直觉的杠杆点——把 meta-knowledge 的捕获做成 prompt-driven 的 schema，比写 runtime 显式状态机更轻、更通用、还能覆盖训练时没预想到的失败模式。

**配套追问准备**：

- "你说的双层防线，runtime 层指什么？" → 答 §3.Q2 的 `src/utils/permissions/denialTracking.ts:12-15`，`maxConsecutive: 3` + `maxTotal: 20` 双阈值熔断。
- "千问那个失败，根因是 runtime 还是模型？" → 答两层都有：runtime 层没有 session-scoped 工具可靠性状态；模型层不会从 in-context 失败历史里归纳 meta-knowledge。**关键失败必须落到显式 state，不能赌模型自己悟**。
- "schema-driven extraction 这个说法是什么意思？" → 答 extractor agent 没有任何针对工具可靠性的专用代码，`buildSessionMemoryUpdatePrompt()`（`prompts.ts:226-247`）只装载 template、做预算提醒、变量替换。模型行为完全由 markdown section 的设计驱动。

**这条 bullet 的杀伤力**：第一层是真实故事，第二层是源码行号定位（证明读过源码），第三层是把别人不一定看见的设计模式抽象命名（"schema-driven extraction"）。

**进阶追问"那阿里为什么不修？"的应对（组织结构视角）**：

如果面试官追问"那像阿里这种大厂为什么修不了这种问题"，**不要归因到"程序员不努力"或"大厂技术差"**——这两种回答都很初级。用组织结构视角回答：

> "我后来想这个 case，发现真正阻塞 Qwen 修复的不是工程难度，是组织结构——模型团队、agent 团队、产品团队的 KPI 不在一条线上，跨层改进的协调成本远大于工程成本。Anthropic 之所以能做出这种细节，是因为 Claude Code 团队同时影响模型训练、运行时、UX 三层。这让我对加入哪种公司有了更清楚的判断标准——看是不是允许一个团队穿透三层做产品。"

**这句话在面试里有两个用处**：

1. 表明你**看大厂技术差距时不会归因到"程序员不努力"**，而是看到组织结构——这是 senior 的视角
2. 表明你**对加入什么样的团队有判断标准**——面试官会觉得这是个有思考的候选人，不是简单的"我想进大厂"

**支撑材料（如果对方再追问"那 Anthropic 为什么能做到"）**：

- Anthropic ~1000 人，所有人为同一个 Claude 产品工作，模型团队/Claude Code 团队/infra 团队物理上和 KPI 上耦合
- 大厂 AI 部门规模大，但每一层之间隔着汇报线、KPI、季度目标。任何跨层改进需要跨团队对齐，**协调成本远大于工程成本**
- 这就是为什么硅谷小公司能在垂直产品上吊打大厂——**不是 outwork，是 outcoordinate**

**注意不要踩的雷**：

- 不要说"阿里程序员能力不行"——立刻 lose 面试
- 不要说"应该直接抄 Claude Code 源码"——proprietary 代码有版权（不是 GitHub 上的开源 MIT 代码），抄了会进 SEC 财报，法务一关就过不去
- 正经路径叫 **clean room implementation**：A 团队读源码写自然语言架构文档，B 团队没看过源码从零实现。合法但贵且慢，大厂权衡后通常不走这条

### 模板六：CPU + OS 这块的 5 分钟口播版（米哈游风格）

**适用场景**：米哈游这类喜欢问 CPU/OS 底层的面试官，问到"MCP stdio 是怎么通信的"或者"两个进程之间怎么传数据"。

**关键策略**：
- **对面也是学软件的**——不要主动抛"MMU / TLB / 页表 / 虚拟地址" 这些硬件名词
- **用"地址空间 / 内存视图 / 内核小本子"这种偏抽象的说法**，给自己留模糊空间
- **故事化讲，5 分钟内收尾**，不给对方深挖的钩子
- 全程把话题拉回 MCP 主线（这是工程问题不是 OS 课）

**口播版（约 5 分钟）**：

> 这个题我从 MCP stdio 的真身讲起。Claude Code 源码 `src/services/mcp/client.ts:950` 的 `StdioClientTransport`，构造参数只有 `command / args / env`，没有 URL 没有端口——它本质就是 spawn 一个子进程，靠 stdin/stdout 跟 Claude Code 主进程通信。这是 Unix 经典的 pipe。
>
> 那 pipe 在 OS 里到底是什么？我用一个类比讲：两个进程像住在两套封闭公寓里，墙上没有窗，互相看不见对方的内存。**每个进程都有自己独立的内存视图**，这是 OS 给的隔离保证。它们唯一的共同接触面是 OS 内核——内核像房东，每套公寓的门都通向它。所以两个进程想传话，必须把话先交给内核，对方再从内核取。
>
> pipe 就是内核里一块小缓冲区，Linux 默认 64 KB。子进程 `write(fd, buf)` 把 JSON-RPC 响应写进去——这一步实际是一次系统调用，CPU 切到内核态，**把字节从子进程的内存拷到内核 pipe 缓冲区**。父进程这边 `read(fd, buf)`——又一次系统调用，**把字节从内核 pipe 拷到父进程内存**。一来一回两次拷贝、两次系统调用，加起来微秒级。
>
> 为什么这种方式快？因为完全不走 TCP/IP 协议栈。同机进程通信如果硬走 socket，哪怕是 loopback `127.0.0.1`，数据也要经过 TCP 分段、IP 路由、netfilter 这些层，每层都是 CPU 时间。pipe 直接在 OS 文件系统层面 read/write，省掉整个网络栈。**这就是 MCP 协议把 stdio 设成本地默认 transport 的物理原因**。
>
> 但 pipe 有个上限：64 KB 写满了，写端的 `write()` 会被 OS 阻塞住，等读端把数据消费掉才继续——这其实是天然的背压机制，MCP server 输出再快也不会撑爆 Claude Code 的内存。
>
> 所以回到面试官最初的问题，"100 个 MCP 启动慢"——**慢的不是 pipe 通信本身，pipe 几微秒就够了**。慢的是 spawn 子进程那一步：fork 一个新进程、exec 加载 Node 解释器、Node 再加载 MCP server 的 JS 代码、做 JSON-RPC handshake。一个 stdio MCP 启动 50-200 ms 是常态。100 个一起 spawn 会同时吃 CPU 调度 + 内存 + 文件 IO，所以 Claude Code 在 `client.ts:2266` 把本地 MCP 和远程 MCP 分两组、各自用 pMap 限并发，本地组并发上限压得比远程组低，就是这个原因。

**为什么这版"刚刚好"**：

- 讲了 **OS 隔离**、**内核中转**、**两次拷贝**、**协议栈对比**、**背压**、**回扣 MCP 主线**——足够体现你看过底层
- 没提 **MMU / TLB / 虚拟地址 / 页表 / CR3 / VIPT / copy-on-write**——这些词一旦出口，懂的面试官会追问，自己挖坑
- 落点回到工程决策（pMap 分组），把主导权拉回到你熟的领域

**如果面试官追问"那 OS 内核的隔离是怎么做的"**：

> 简单说每个进程有自己独立的地址空间，A 进程的地址 `0x1000` 和 B 进程的 `0x1000` 物理上指向不同的内存页，是 CPU 和 OS 配合做的隔离。底层是页表机制，OS 负责维护，CPU 有专门的硬件单元做地址翻译。**这块我做软件的角度理解到这里就够用了，再往下是芯片设计的事**。

**最后这句话是主动收尾**——表态"我懂边界，不装"，让面试官知道再追也是浪费时间，他自然会换题。**这比你硬扛 TLB / MMU 细节安全得多**。
