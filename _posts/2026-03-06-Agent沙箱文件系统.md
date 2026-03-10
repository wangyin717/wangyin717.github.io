# 构建具备“长期记忆”的 AI 智能体：沙箱文件系统与上下文工程实践

在 AI Agent 的开发中，让大语言模型（LLM）陷入死循环或者逐渐“变笨”的罪魁祸首往往不是模型智力不足，而是**[上下文腐退（Context Rot）](https://research.trychroma.com/context-rot)**。Chroma 的研究与 Anthropic 的[阐述](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)都指出：随着上下文不断膨胀，LLM 的注意力预算（Attention Budget）会被逐渐耗尽，性能随之下降。

当一个智能体需要完成复杂的、多步骤的分析任务时，它通常需要调用数十次工具。如果在传统架构下，把所有工具调用的原始结果、中间数据和报错信息都粗暴地塞进 LLM 的上下文窗口里，模型很快就会被“撑爆”。正如 [Karpathy 所言](https://x.com/karpathy/status/1937902205765607626)：

> 上下文工程是一门精细的艺术与科学：在 Agent 的轨迹中，用恰到好处的信息填满上下文窗口，以支撑下一步决策。

为了解决这个问题，许多顶尖的 Agent 系统（例如 [Manus](https://en.wikipedia.org/wiki/Manus_(AI_agent))）都引入了一种优雅的解决方案：**为 Agent 配备一个具备文件系统的隔离沙箱（Virtual Computer）**。Manus 的每次会话都使用[云端虚拟机与 E2B 沙箱](https://e2b.dev/blog/how-manus-uses-e2b-to-provide-agents-with-virtual-computers)，赋予 Agent 文件系统、Shell 及命令行工具。这与 Anthropic 所倡导的[有效上下文工程](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)中的三种策略——隔离、卸载、缩减——高度契合。通过对当前项目的研究，我想分享一种高度结构化、轻量级的沙箱文件系统设计范式，看看我们如何通过“文件系统”来实现高效的上下文工程。

<figure>
<img src="/assets/e2b_sandbox.png" width="100%">
<figcaption>
</figcaption>
</figure>


## 1. 为什么我们需要沙箱文件系统？

在传统的 Agent 交互中，开发者习惯将大模型作为“内存中心”。数据被加载进来，经过处理，结果以文本形式直接返回给模型。这种方式在处理简单问答时无可厚非，但在处理真实世界的大型文件或持续多轮的复杂任务时，会导致灾难性的 Token 消耗和极差的稳定性。若把大量工具一股脑绑定给 LLM，[工具描述本身也会占用宝贵 Token，并可能造成模型混淆](https://www.anthropic.com/news/context-management)。

引入一个云端沙箱文件系统，本质上是为了实现**上下文卸载（Context Offloading）**。我们让 Agent 拥有在虚拟环境中读写硬盘的权限。这样，大批量的数据处理、中间变量的产生，都发生在沙箱的计算和存储层，而不再需要穿越网络回到大模型的上下文窗口中。模型从“亲自搬运每一块砖”，变成了“拿着图纸指挥施工的工头”。业界已有共识：给 Agent 一台“电脑”（文件系统 + 终端 + 基础工具），用少量通用能力即可覆盖大量动作，是[常见且可扩展的模式](https://simonwillison.net/2025/Oct/16/claude-skills/#skills-depend-on-a-coding-environment)；Claude 的 [Skills 设计](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)也把技能放在文件系统中，通过渐进式披露按需加载，而非全部塞进上下文。

## 2. 结构化设计：给 Agent 打造一张“标准办公桌”

只是给 Agent 一个能读写的目录是不够的。如果不加约束，几次迭代后，沙箱里就会充满混乱的临时文件。为此，我们在文件系统中设计了一套清晰、层次分明的标准目录结构。这就像是给 Agent 规划了一张功能明确的办公桌：

- **输入层（Inputs）**：这里是“收件箱”。用户上传的原始物料（如数据集、文档）统一放置于此。它对 Agent 而言是只读的数据源。
- **工作区（Workspace）**：这是 Agent 的“草稿本”。在处理数据的过程中，Agent 过滤出的子集、计算出的临时矩阵等中间变量，都会被序列化保存在这里。这些数据不需要被人类看到，也不需要全量塞给大模型。
- **输出层（Outputs）**：这是“成品陈列室”。最终生成的图表、分析报告、或者供用户下载的最终产物，都会被妥善地放置在这个目录，等待被提取展示给用户。
- **上下文层（Context）**：这是整个沙箱的“大脑索引”。这是连接物理硬盘和大模型认知的桥梁，也是整个架构中最具巧思的部分。

## 3. 状态持久化：用轻量索引替代沉重记忆

这种分层设计的核心魔力，在于实现了完美的**上下文缩减（Context Reduction）**。

试想一下，当 Agent 在第一轮对话中，从数百万行的原始数据里筛选出了一份核心数据。在传统模式下，Agent 可能需要在内存中保持这个对象，或者试图总结它。但在这种基于沙箱的架构中，Agent 的操作是：

1. 将这份核心数据持久化写入到 **工作区（Workspace）** 的硬盘里。
2. 在 **上下文层（Context）** 中生成或更新一个轻量级的注册表（例如一个 JSON 格式的元数据文件）。

在这个注册表中，Agent 只记录极其精简的元信息：  
“这里有一个经过筛选的数据集，它的路径在工作区的某个位置，它的维度是 X 行 Y 列，包含了这些字段特征……”

到了下一轮对话时，大模型不需要读取几百兆的真实数据，它只需要阅读这个微小的注册表，就能精准地知道自己当前拥有哪些“资产”。当它需要基于这些资产绘制图表时，只需让沙箱从对应路径加载文件即可。

这种设计将信息的**完整表示（Full Representation）**留在了沙箱硬盘中，而将**紧凑表示（Compact Representation）**放进了大模型的上下文中：既极大节省 Token，又保证了任务执行状态的可追溯与可恢复。Manus 在[上下文工程实践](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)中同样采用“完整版 + 紧凑版”的二元表示，对陈旧工具结果做压缩（只保留路径等引用），需要时再从沙箱按需读取。Anthropic 的 [Context Editing](https://www.anthropic.com/news/context-management) 也在接近 token 上限时自动清理陈旧工具调用与结果，同时保留对话流，与上述思路异曲同工。

## 4. 多轮对话的无缝连续性

有了上述机制，Agent 就打破了单次会话或单台服务器内存的枷锁。

多轮对话的连续性不再依赖于主程序的内存常驻。哪怕上一个任务已经结束，哪怕 Agent 暂时休眠，所有的物理状态都安全地保存在沙箱的文件系统中，所有的认知状态都记录在上下文索引文件中。

当新的指令到来，Agent 被重新唤醒，它只需“看一眼”上下文目录下的索引文件，就能瞬间恢复记忆：  
“哦，原来我的工作区里已经准备好了一份清洗过的数据，现在我可以基于它直接生成报告并保存到输出层了。”

这种**上下文隔离（Context Isolation）**的策略，确保了不论经历多少次复杂的工具调用，大模型始终能在清爽、精简的信息流中做出更稳定、更清晰的决策。在多 Agent 场景下，规划器与子 Agent 之间的[上下文共享与隔离](https://cognition.ai/blog/dont-build-multi-agents)同样是关键设计点；用文件系统作为共享状态载体，可以让“只传指令”与“共享完整上下文”两种模式清晰并存。

## 5. 总结：顺应模型演进的极简哲学

人工智能领域有一个著名的**[“苦涩的教训”（The Bitter Lesson）](http://www.incompleteideas.net/IncIdeas/BitterLesson.html)**：人类依靠自身经验手工设计的复杂规则，短期内可能会带来性能提升，但在长期算力（和模型能力）的指数级增长面前，最终都会成为限制模型发挥的瓶颈。Lance Martin 曾[探讨过它对 AI 工程的含义](https://rlancemartin.github.io/2025/07/30/bitter_lesson/)；Claude Code 的创造者 Boris Cherny 也提到，[The Bitter Lesson 促使他保持 Claude Code 的“无倾向性”](https://www.youtube.com/watch?v=Lue8K2jqfKk)，以便更好地随模型进化而适配。

在构建 Agent 时，如果我们采用复杂的、高度定制化的内存管理代码，很可能会限制未来更强模型的发挥。相反，提供一个标准的、无倾向性（Unopinionated）的文件系统架构——划分出清晰的输入、工作、输出和索引区——是一种更通用、更可持续的底层赋能。

我们赋予模型一台虚拟电脑和一套文件管理规范，其余的分析、决策与状态管理交由模型自己编排与调度：将重度状态卸载到沙箱物理层，用轻量索引维持逻辑连贯性。这是通向强大、稳定且低成本的生产级 AI Agent 的一条优雅路径。

---

*写作时参考了 [Lance Martin 的 Manus 上下文工程笔记](https://rlancemartin.github.io/2025/10/15/manus/)与 [Anthropic 的 Agent 定义与上下文工程论述](https://www.anthropic.com/engineering/building-effective-agents)。*