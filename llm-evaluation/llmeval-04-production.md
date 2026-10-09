# 生产评估（Ch4–5）：多轮、Agent、RAG 排障、评估即开发

> 真实用户进来后，故障长得像"坏掉的交互"，不像"答错的题"。评估单位从单条回答变成整段旅程。

## 1. 多轮对话：评旅程不评单句

多轮里故障是交互性的：一个烂追问带偏整段对话；几轮前的不完整 tool 结果，几轮后变成"不相关的幻觉"。
**评估单位变成整个 session**，问题只有一个：用户目标达成了吗？

工作流：**按用户看到的样子**从头读 transcript（不看 tool 调用细节）→ 整体 pass/fail → 找到**轨迹第一次偏的 turn（first failure point）**→ 压成最小复现（很多多轮失败能压成单轮 prompt）→ 定修哪里（prompt/检索/tool/流程）。

两个测试技巧：
- **Prefix replay**：前 N−1 轮冻结原样，从同一点生成新 continuation，对比；
- **模型扮演用户**：便宜可 scale，但和真人有差，结果只看方向。

Handoff（bot 转 bot/转人工）是 session 的一部分——trace 和评估**不许在 handoff 处断**。

## 2. Agent workflow：评步骤不评结局

**Agentic workflow**：规划→调工具→看结果→迭代。评法：把 trace 拆成步，**每步的正确性都评**。
为什么：结局对但路径错（wrong-but-lucky）——用错工具蒙对答案，下次条件一变就炸。**最终答案是滞后且有损的指标**。

中间产物是一级评估对象：plan 本身、tool query、tool 参数。
典型失败四件套：**tool misuse**（调错工具/参数错）、**plan drift**（跑着跑着忘了目标）、**error cascade**（一步烂步步烂）、**premature termination**（没达成就停了）。

做法：从观察到的失败建 step taxonomy；能确定性查的确定性查（如 tool 参数 JSON 合法），要判断的用 judge；汇总成 workflow 健康度。

## 3. RAG 排障四连查（按顺序）

RAG 答案错了，按这个顺序查——**大多数 RAG 故障是检索问题，不是模型问题**：
1. **Retrieval miss**：对的文档根本没检回来；
2. **Ranking/context**：检回来了但排在 cutoff 之下/被截出 context；
3. **Grounding failure**：检索没问题，答案没被文档支持（幻觉）；
4. **Synthesis failure**：文档都对，模型组织表达烂了。

修在诊断出的那层：miss→重切块/换 embedding；ranking→rerank/context 管理；grounding→更严的 grounding prompt/强制引用；synthesis→答案模板。**别一上来就换大模型**。

## 4. 评估即开发 + 离线/在线

**Evaluation-driven loop**：记 trace → 抽样 review → error analysis → 从失败写新 eval → 修 → eval 当回归跑。**每个修好的失败变成永久测试**——eval suite 跟着产品长大，可靠性复利。

纪律：eval 进 CI（prompt/模型/代码改动触发，确定性失败 block 发布）；eval suite 有 owner（没人管的 eval 会烂）；定期 trace review，dashboard 看 pass rate 趋势——**趋势比单点重要**。Eval debt = tech debt，欠的评估迟早以神秘回归收利息。

**Offline（离线）**：事后判 log——安全、可复现、迭代快；弱点是**分布漂移**（数据集悄悄过时，测的是昨天的世界）。
**Online（在线）**：真实流量——A/B、shadow mode（新版本跑真实流量但输出不展示）、用户反馈（显式评分+隐式重试/改问法）；弱点是噪声、慢、有风险、要统计功底。

桥：生产 trace 持续回填离线数据集（抗 stale）；抽样人工 review 线上 trace 当常驻质量信号。
**离线管迭代速度，在线管验证**，两者用生产 trace 连起来。

## 自测 4 问（带答案要点）
1. 多轮评估为什么看"旅程"？first failure point 干嘛的？
   → 故障是交互性的；定位轨迹第一次偏的点，修机制。
2. Agent 评估为什么拆步骤？wrong-but-lucky 是什么？
   → 结局对路径错，下次必炸；中间产物（plan/tool 参数）一级评估。
3. RAG 排障四连查的顺序和逻辑？
   → miss→ranking→grounding→synthesis；多数是检索问题，别先换模型。
4. Offline vs online 各自弱点？怎么连起来？
   → 离线怕分布漂移，在线怕噪声慢贵；生产 trace 回填离线集。
