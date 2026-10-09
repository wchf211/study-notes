# 评估工作流（Ch2）：trace 采集、合成数据、pass/fail

> 地基三件套：把 trace 记全、把 edge case 造出来、把判分标准写死。顺序不能反。

## 1. Trace 采集：第一天就做

完整 trace 五件套：请求头（ID/timestamp）+ 完整输入（system prompt + user 消息）+ 中间事件（tool 调用及结果）+ 最终输出 + 模型元数据（模型版本、**temperature** 随机性旋钮、token 用量）。

三个 trade-off：
- **可追溯 vs 隐私**：trace 里有个人数据——敏感字段 mask/redact，审计谁看过；
- **存储索引**：append-only 不可变存储，按 request/session/user ID 索引；热冷分层 retention；
- **完整 vs 噪声**：默认全记——**缺了的事后补不回来**。

血泪：**只记 user 消息的 trace 是撒谎**（system prompt 藏了决定性步骤）。instrument 第一上线的第一天，别等"你觉得需要"的时候——那时历史已经没了。

## 2. Synthetic Data：把 edge case 造出来

**Synthetic data（合成数据）**：AI 生成的测试输入，不是真实用户来的。结构化生成法：
- **Dimensions（维度）**：你关心的变化类别（请求类型、账户年龄、地区）；
- **Tuples（组合）**：维度的具体组合（退款 × 新客 × 跨境单）；
- 生成模型按 tuple 写多样逼真的输入；judge 模型检查自然度+去重。

为什么：真实用户数据扎堆在常见 case，**稀有但重要的组合永远测不到**，scale 越大欠账越贵。例：booking bot 在"升舱+国际+忌口"组合上翻车。

Trade-off：合成数据便宜买覆盖，但会**过拟合到生成器自己的盲区**——永远和真实 trace 混着用，人工抽查真实性。
**维度从你真实的失败模式里长出来**，别拍脑袋。

## 3. Pass/Fail 碾压 1–5 分

**Inter-rater agreement（评分者一致性）**：独立评审给同一标签的频率。pass/fail 的一致性吊打 1–5 分——没人能可靠说出 3 和 4 的区别。

数字分还制造**虚假精度**：均分 4.2 可能藏着 30% 失败；"70% pass"直接说可靠性事实。
前提：标准必须 explicit 且**从真实失败里长出来**——含糊的二值标准和含糊的量表一样烂。LLM judge 也要先和人类对 agreement 再大规模用。

写法：**每个 eval 标准写成 yes/no 问题**，汇报 pass rate 不汇报均分。

## 自测 4 问（带答案要点）
1. Trace 五件套？为什么"第一天就做"？
   → 请求头/完整输入/中间事件/最终输出/模型元数据；缺了事后补不回来。
2. Synthetic data 的 dimensions/tuples 方法解决什么问题？
   → 真实数据扎堆常见 case；稀有组合系统性造出来；防过拟合到生成器盲区。
3. 为什么 pass/fail 比 1–5 分诚实？
   → 一致性更高；均分藏失败率，pass rate 直接说可靠性。
4. Trace 的隐私 trade-off 怎么处理？
   → 敏感字段脱敏+审计查看；append-only 存储+冷热分层。
