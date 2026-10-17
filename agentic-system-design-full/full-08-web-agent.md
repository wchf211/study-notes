# Web Agent（Modules 7–8）：WebVoyager 设计 + ADK 实现

> 这两章是一对：Module 7 讲设计（为什么多模态、ReAct 循环、怎么评），Module 8 讲实现（ADK 代码怎么组织、浏览器状态放哪、工具怎么分工）。读法：先看设计，再看实现 —— 实现里的每个文件都对应设计里的一个责任。

## 1. Web Agent 的挑战：真实网站没有干净 API（Module 7.1）

**Web agent（网页 agent）**：把自然语言目标变成真实网站上的动作。网站是动态、非结构化、充满弹窗、登录、变布局的，没有干净 API。

三个核心问题：
- **Grounding（接地）**：把指令映射到页面上正确的 element（元素），而页面每次访问可能长得不一样。
- **Navigation / recovery（导航/恢复）**：选路径，失败时回退。
- **Evaluation（评估）**：成功是 end state（最终状态），不是文本答案；页面随时间变。

**Text-only agent（纯文本 agent）**：读 DOM / accessibility tree（可访问性树）—— 结构精确，但看不见布局和视觉。**Multimodal agent（多模态 agent）**：还读 screenshot（截图）—— 看得见布局、图标、视觉状态，像人。

没有 API、任务要浏览/比较/提取真实网站时用 web agent。API 或脚本能确定性做、或不可逆/高 stakes 动作没确认时别用。例：跨几个零售商找某商品价格，每个站布局不同。失败模式：只信文本，点了同名错按钮。Fix：视觉 + 结构观察融合 + 行动前验证。

## 2. WebVoyager 架构：多模态 ReAct 循环（Module 7.2）

**WebVoyager**：单个 agent 的多模态 ReAct 循环，跑在大 multimodal model 上。每轮：observe（观察：当前 screenshot + 清洗过的、带编号的 accessibility tree，只含可交互元素 + 短历史）→ reason（推理：写短 thought，陈述下一 sub-goal，选一个 action）→ act（行动：一个 grounded action —— click / type / scroll / select / navigate / go back / wait / answer）→ 再观察、再规划。

**Grounding 用 Set-of-Marks**：accessibility tree 给可交互元素分配稳定数字 ID，模型引用 ID 而不是猜坐标，click 落在目标元素上。导航辅助：go back 恢复、回到早期 URL 重开分支、直接导航卡住时用已知捷径（搜索引擎）。每步一张 screenshot，成本可控。

开放式浏览、恢复重要时用这个 pattern。别在不重新观察的情况下链多个 action。例：“找日期 X 最便宜航班”—— 观察结果页 → 推理要 filter → 点 filter → 再观察 → 回答。失败模式：在 click 改变布局后的 stale page（过期页面）上行动。Fix：一次 action 一次观察，每步 fresh screenshot + tree。

## 3. Text vs Multimodal：同一任务，两次跑（Module 7.3）

Side-by-side trace（并排追踪）同一任务（在出版商站找某论文 DOI）：text-only agent 看到干净 DOM 列表，但分不清哪个是论文、哪个是广告/相关链接，modal（弹窗）或布局变化打乱 tree 时卡住。Multimodal agent 看 screenshot，认出论文标题、作者、DOI 区块，视觉上直接导航。

Takeaway：screenshot 加布局、salience（显著性）、视觉确认，而纯文本缺这些，尤其 DOM 噪声大或误导时。页面视觉结构化、有广告/modal、或要区分相似元素时用多模态输入。视觉密集页面别假设文本够用。例：两个链接文本都是“Download”—— 只有 screenshot 能分清哪个是论文、哪个是广告。失败模式：text agent 点第一个匹配元素。Fix：click 前把文本 ID 和 screenshot 交叉核对。

## 4. 评估 WebVoyager：End-State + Trajectory（Module 7.4）

评估：643 个任务、15 个 live 网站（购物、旅行、新闻、GitHub 等），success 由 GPT-4V evaluator 判（检查最终 screenshot + trajectory 对照任务）+ 人工抽查。结果：live 任务约 59.1% 成功，失败多在 navigation（迷路、循环）和 recovery（不回退），其次 grounding 错误和 site block。

设计课：fuse modality（融合模态）、context 保持短且新、显式数字 grounding、加 recovery primitive（back、重启分支）、评估用 end-state + trajectory，不只最终答案。任何 web agent 都用这种评估风。别按流利度或单张 snapshot 判。例：agent 到达正确 checkout 页但从没点“confirm”，不算成功。失败模式：优化看起来合理的推理 trace，却从不完成任务。Fix：测 end-state 成功 + 失败原因（navigation、grounding、recovery）。

## 5. ADK 实现：责任分工 + Artifact 审计（Module 8.1）

把 WebVoyager 设计变成可跑 ADK 代码，固定 demo 任务（找某公司最近 LinkedIn 帖子，返回文本 + URL）。项目按责任拆：agent.py（entry point，调 build_root_agent）、tools/browser_runtime.py（Playwright 包装，持浏览器状态）、tools/web_tools.py（LLM 可调的所有工具）、adk_agent/web_agent.py（ADK 绑定、prompt、agent 模式）。

每次运行写 artifact 审计：process_log.jsonl（每阶段一条 JSON —— observation、action、outcome、blocker）、raw screenshot、labeled screenshot（带数字 ID）。Demo 节奏：observe（screenshot + labeled element + VLM 总结）→ 一个 action → 再观察，搜索被 block 或登录墙出现时恢复，诚实答“blocked”而不是编内容。

建可调试 web agent 时用这个结构。别让 agent 在没有 logged observation（日志观察）的情况下行动。例：LinkedIn 任务 —— 搜索、打开结果、关 banner、滚 feed、答 URL 或承认登录 block。失败模式：agent 编一个从没见过的帖子。Fix：每个 claim（断言）都要 artifact / log 证据。

## 6. Browser State：Stateless 工具 + Stateful Runtime（Module 8.2）

**ADK tool function 是 stateless（无状态）**，所以浏览器状态必须放在工具外面，一个 module-level singleton BrowserRuntime（单例浏览器运行时）里。它持有 Playwright browser / context / page、当前 URL / title、visited URL、action history、screenshot、last labeled element。

关键 helper：ensure_page()（创建/复用 page，从空白页起，固定 viewport）、navigate(url)（跳转、等待、记录、处理 timeout）、observation bookkeeping（observation_streak、no-progress_streak、search_loop_streak，检测循环、触发恢复或退出）。固定 viewport 让 screenshot 和元素坐标跨步骤稳定。

工具要共享 live browser 时用 singleton runtime。别每次 tool 调用新建浏览器。例：一个 Playwright page 在 ReAct 循环里活过几十个 tool 调用。失败模式：调用之间状态丢了，click 指向死 page。Fix：单个 runtime + 显式 ensure / reuse + streak 计数器。

## 7. Multimodal Web Tools：三组工具（Module 8.3）

- **Observation（观察）**：capture_multimodal_observation（融合 DOM + screenshot + 可选 VLM 总结）、analyze_screenshot_with_vlm（让 vision model 描述 goal、区域、下一步）。
- **Annotation（标注）**：_annotate_screenshot（按 accessibility tree 在 screenshot 上画数字框/ID）、accessibility snapshot 提取（过滤到可见、可交互元素，带 tag、role、text、坐标）。
- **Execution（执行）**：click_element、type_text_element、scroll、navigate、go_back、wait、answer —— 每个取数字 element ID，按最新 snapshot 解析，执行一个 action，再返回 fresh observation。

Fusion（融合）重要，因为 DOM 给精确 ID，VLM 给布局/salience；合起来降 grounding 错误。用 typed、single-action（类型化、单动作）工具 + fresh observation。别暴露原始坐标或多动作批。例：click_element(3) → 工具把 ID 3 解析成按钮，点，返回新 screenshot + tree。失败模式：按猜的坐标点，布局一变就点偏。Fix：数字 ID 绑定最新 snapshot。

## 8. 组装：三种模式，先用 Single-Agent（Module 8.4）

组装是 factory（build_root_agent），返回三种模式之一：
- **Single-agent**：一个 LlmAgent + 所有工具 + ReAct prompt —— 最简单，大多数任务最好用。
- **Strict**：LoopAgent 强制 observe → act → observe，防止模型链未观察的 action。
- **Multi-agent**：Planner（选 sub-goal）、Vision（总结 screenshot）、Browser（执行）—— 更能干，但更慢更贵。

Prompt 编码循环纪律：一次一个 action、永远再观察、用 ID、回答前验证、承认 block。退出/循环控制用 iteration cap、no-progress streak、显式 ANSWER / finish 信号。

先用 single-agent；loop 或 grounding 错误持续时再加 strict / multi-agent。别给模型自由批 action。例：Planner 说“打开搜索”，Browser 执行，Vision 确认结果页，Planner 再选下一步。失败模式：模型一轮发五个 click 在 stale page 上。Fix：strict loop + one-action-per-turn prompt + streak exit。

## 这门课怎么教的（Modules 7–8）

Module 7 先给真实世界 messiness（弹窗、登录、变布局）引出多模态动机，再按 WebVoyager 循环顺序：挑战 → 架构 → 并排比较 → 证据。一个 running task type（在真实站上找/提取）贯穿，traced comparison（追踪比较）是核心教具。心智模型：ReAct loop + Set-of-Marks grounding。每课都让“多模态有帮助”这个抽象断言变得可见。难处：评估课把 evaluator 设计和失败分类混在一起，headline 数字（59.1%）需要上下文。最值得偷的：**同一任务用两种设计各跑一遍，标注到底哪一步一个失败、一个成功**。要避免：不先讲清什么算成功就给成功率。

Module 8 先给一个 fixed 任务的 live demo 锚定代码，再自底向上走：runtime → tools → assembly。同一 LinkedIn 任务和同一批 artifact 文件反复出现，代码映射到可观察行为。心智模型：stateless 工具 + stateful runtime + one-action-per-observation 纪律。每课点名文件和责任，好跟。难处：raw process_log 摘录很长，会埋住 pattern。最值得偷的：**讲工具代码前，先给一次真实运行的 artifact / log —— 学习者先看到每个文件为什么存在**。要避免：不标 stage transition（阶段转换）就贴完整 log dump。

## 自测 4 问（带答案要点）
1. Web agent 三个核心问题？Text-only 和 multimodal 各缺什么？
   → grounding / navigation-recovery / evaluation；text-only 看不见布局视觉，multimodal 补布局/salience/视觉确认。
2. WebVoyager 一轮是什么？Grounding 怎么做的？
   → observe → reason → act → 再观察；Set-of-Marks：accessibility tree 给数字 ID，模型引 ID 不猜坐标。
3. ADK 里 browser state 放哪？为什么？
   → module-level singleton BrowserRuntime；tool function 是 stateless，状态必须放工具外面。
4. 三种组装模式怎么选？循环纪律是什么？
   → 先 single-agent；loop / grounding 错误持续再 strict / multi-agent；一次一个 action、永远再观察、用 ID、回答前验证。
