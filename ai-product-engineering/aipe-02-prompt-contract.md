# Prompt 即 Contract（Ch3）：模型与应用的接口

> 核心观点：prompt 不是聊天消息，是**下游代码依赖的规格说明书**。prompt 里含糊一句，生产环境就是一个 bug。

## 1. 把 Prompt 写成 Contract

Contract 五件套：**任务**（exactly 做什么）、**输入**（给什么、什么格式）、**规则**（约束和 edge case）、**输出格式**（精确的形状）、**停止条件**（什么时候 abstain、什么绝不能做）。

反面教材："Read this support ticket and categorize it appropriately"——"appropriately"逼模型猜你的意图。

正面例子：把"summarize this"换成"返回 category（仅限 billing/technical/account 三选一）+ 1–2 句摘要 + confidence（high/medium/low）；工单为空返回 category='unknown'"。

检验标准：**如果另一个工程师光看你的 prompt 写不出同样的行为，它还不是 contract**。

## 2. 四种 Prompt 技巧：对症下药

每种技巧治一种病，先诊断再选，别全堆上（堆料 = prompt 膨胀 = 烧钱）：

1. **Explicit contract**：治含糊。把意图写死。
2. **Few-shot examples**：给 2–3 个输入→输出对。治**格式漂移**和 edge case 误标。例："I love the product"被误标为流失——加它当标注样本，比重写指令管用。
3. **Step-by-step sequencing**：拆成有序步骤。治多步任务的推理错误。
4. **Fallbacks**：告诉模型不确定时怎么办。治**静默瞎猜**。
5. **Delimiters**：把用户内容和指令用分隔符划开（指令是代码，用户输入是数据，混了就等着被注入）。

## 3. Structured Output：用代码强制，不用嘴请求

**Structured output** = 模型返回机器可读的数据（固定字段的 JSON），而不是散文。两层：
1. Contract 里精确要求；
2. **代码里校验**：parse JSON、查类型和枚举值、畸形就 reject/repair，失败重试一次再转人工。

**畸形的模型输出绝不能静默流进应用下游**。例：摘要器返回缺 category 字段 → 重试一次 → 再失败走人工审核。

金句：**把模型输出当不可信的用户输入——全部校验**。"模型说了什么"和"应用做了什么"之间，是生产事故最高发的地带。

## 自测 4 问（带答案要点）
1. Prompt contract 五件套？
   → 任务/输入/规则/输出格式/停止条件；检验：别人光看 prompt 能否复现行为。
2. "I love the product"被误标流失，用哪种技巧修？为什么不用重写指令？
   → Few-shot：加标注样本；指令重写治不了 edge case 分布问题。
3. Structured output 为什么要代码校验？
   → prompt 里的要求是请求不是保证；畸形输出静默流入=事故。
4. Delimiter 防的是什么？
   → 指令和用户数据混淆 → prompt 注入（见生产篇）。
