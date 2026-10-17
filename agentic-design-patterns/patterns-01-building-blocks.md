# Building Blocks（Module 1）：Agent 的零件箱

> 核心图景：agent 不是一个更聪明的 prompt，而是一个循环 —— goal → plan → act with tools → observe → adjust。后面所有 pattern，都是组织这个循环的不同方式。

## 1. Agent vs 一次性 chatbot

**Agent**：以目标为导向的系统，能规划、调工具、记 context、根据结果调整，跑多步才停。**One-shot chatbot**：只回一次，不管之后发生什么。

什么时候值得上 agent：任务多步、条件会变、需要实时数据或工具（查天气、查日历、读写文件）。什么时候别上：单个事实答案、固定流程、确定性计算 —— prompt 或脚本更便宜、更可靠。

例：天气助手不只是回答“明天天气如何”，而是查天气 → 解读 → 结合你的日程给穿衣建议 → 你改了计划它再调整。失败模式：把每个任务都当 agent 做。Fix：先从最简单的 prompt / workflow 起步，真的不够再升级。

## 2. Pattern vs Technique：先分清两层

**Pattern（模式）**：高层架构，回答“系统怎么组织”——比如 routing（先分类再分派）。**Technique（技巧）**：Pattern 里面的具体实现选择 —— 比如 routing 用的分类器 prompt 到底怎么写。

分不清的代价：抄了一个技巧，却丢了它所属的 pattern 上下文，换个场景就失灵。正确顺序：先点名 pattern，再挑 fit 它的 technique。

例：routing 是 pattern；“用一个小模型 + 五个类别 + fallback” 是 technique。

## 3. 五大 Building Blocks：检查清单

设计 agent 时逐项检查，缺了哪块、哪块就会在生产里咬你：

- **Language model（推理引擎）**：解读输入、做决策、选动作。不是越大越好，看任务难度、成本、latency、tool-calling 可靠性。
- **Memory（记忆）**：short-term = 当前 working context（本轮对话，很快但会丢）；long-term = 跨 session 的持久状态（偏好、历史决策、过去的轨迹）。不分层的结果：context 爆掉，或者每轮都从零开始。
- **Tools / APIs（工具）**：agent 摸到真实世界的手段 —— 搜索、计算、读写 DB、调 API。工具必须有清晰的名字、描述、输入输出，否则模型只能靠猜。
- **Planning（规划）**：把 goal 拆成步骤，决定顺序，失败时改计划。没有规划的 agent 只能走一步看一步。
- **Perception（感知）**：从输入（文本、图像、传感器、工具返回）里提取可用意义。感知错了，后面全错。

循环把它们串起来：**reason → act → observe → reflect**（推理 → 行动 → 看结果 → 反思）。例：coding helper 读文件（perception）→ 回忆上次改动（memory）→ 规划改法（planning）→ 跑测试（tools）→ 看测试结果再修。

## 4. Structured Output：让下游代码敢信任模型输出

**Structured output**：强制模型输出符合预定义 schema（类型化字段、校验过的对象），而不是自由散文。任何输出要进工具、路由、存储、UI 的地方，都应该上 structured output。

为什么：下游代码没法靠“希望模型这次格式对了”来运行。自由文本里一个多余的逗号、一个编造的字段，就能打断整条 pipeline。

做法：定义 schema → 校验输出 → 失败则重试或 fallback。例：订单解析器返回校验过的字段（商品、数量、地址），而不是一段话。失败模式：malformed / 编造字段流到下游。Fix：schema validation + retry / fallback，不要裸信模型输出。

Structured output 是后面几乎所有 pattern 的地基：routing 靠它做分类决策，tool calling 靠它做调用参数，parallelization 靠它做结果合并。

## 5. Tools：模型选，代码执行

**Tool calling** 的分工：模型决定调哪个工具、带什么参数；**你的代码**真正执行。模型永远不直接碰系统。

循环：choose-tool → execute → read result → continue。工具设计要点：名字和描述精确（模型只知道描述里写的东西）、输入输出 schema 清晰、工具集保持小（工具越多，选错概率越高）、执行结果要检查。

什么时候用工具：需要实时数据、精确计算、搜索、或对世界做动作。什么时候不用：答案模型已经知道，或静态查表就够。例：旅行助手调天气工具 + 航班工具，再合成计划。失败模式：工具描述含糊 → 调错、重复调。Fix：精确 schema + 有限工具集 + 结果检查。

## 6. Retrieval：给 agent 接外挂知识库

**Retrieval（检索）**：把 agent 连到外部知识（文档、网页、数据库），先搜到相关内容，再基于搜到的内容回答，而不是全靠模型记忆。**Grounding（基于证据回答）**：答案扎在检索到的材料上，而不是凭空生成。

什么时候用：知识是当前的、私有的、或大到塞不进训练记忆。什么时候不用：稳定的通用知识，或检索带来的噪声 / latency 超过收益。例：客服 agent 先检索最新退款政策，再回答。失败模式：搜到不相关 chunk、或来源已过期。Fix：更好的 chunking / indexing、过滤、以及要求答案引用检索到的证据。

Retrieval 本身是 tool / technique，不是独立的魔法：它喂给 planning 和 generation，质量取决于索引和查询，不是模型大小。

## 这门课怎么教的（Module 1）

每课都用一个日常场景开场（天气、旅行、家教），先给动机，再点名概念，再画 loop / diagram 心智模型，最后讲 limits。用的是一课一个新例子 —— 每课都具体，但跨课的联系要读者自己连。最值得偷的：**reason–act–observe–reflect 这张图在全课反复出现**，后面每个 pattern 都往这张图上映射，观众不用每课学一张新图。

## 自测 4 问（带答案要点）
1. Agent 和 one-shot chatbot 的本质区别？什么任务不该用 agent？
   → 循环（goal→plan→act→observe→adjust）+ 多步 + 工具 + 记忆；单答案 / 固定流程用 prompt / 脚本。
2. Pattern vs technique 怎么分？为什么顺序是先 pattern 后 technique？
   → Pattern 是架构组织方式，technique 是里面的实现选择；先点名 pattern 再挑 technique，否则抄技巧丢上下文。
3. 五大 building blocks 各是什么？循环怎么串它们？
   → model / memory（short + long）/ tools / planning / perception；reason→act→observe→reflect。
4. 为什么输出要进下游时必须 structured output？Tool calling 里模型和代码的分工？
   → 下游代码需要可校验的 schema，裸信自由文本会断 pipeline；模型选工具 + 参数，代码执行。
