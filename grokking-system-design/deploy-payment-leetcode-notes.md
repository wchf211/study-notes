# Code Deployment + Payment + LeetCode — 提炼笔记（174–179，2026-10-06）


## Code Deployment（174–175）
- Pipeline 七段：VCS → CI → build → 自动化测试 → staging → deploy → 监控+回滚。
- **五种部署策略**：Basic（直发，最快最险，无回滚）/ Multi-service（多服务协同发，一致但协调复杂）/ Rolling（分批，停机少但慢）/ **Blue-green**（两套环境，秒级回滚，2 倍成本）/ **Canary**（先小流量，抓 bug 早，要流量路由+监控）。
- 估算：200 应用、日 3000 次部署、20GB/包 → **12PB/天**存储；20GB 在 5 分钟内发到单 region 2000 台 → 约 **533Mbps**；3000/天 ÷ 50/台 → **60 台**。
- 两阶段架构：**构建**（VCS→CI→pub-sub 队列→build workers→主 blob）与**部署**（复制服务→区域 blob→全局状态标 ready→全局配置更新版本→区域配置轮询→应用服务器拉取安装）分离。
- **P2P 分发**：区域内机器互传多 GB 二进制 + **read-through**（已在拉就等，不另起 pull）——大文件分发的省带宽答案。
- 容错：pub-sub 解耦 build 请求与 workers（autoscaling）；job 在队列持久化（worker 挂不丢活）；心跳进 SQL DB，miss 则重派。基线用 basic 策略，模块化保留以后接 blue-green/canary。

## Payment（176–177）
- 七实体：持卡人、商户、网店、**发卡行**、**收单行**、支付网关、卡组织。
- **Auth vs Settlement**：auth 同步（网关→发卡行验卡/风控/额度→返回授权码→**额度冻结，钱没动**）；settlement 异步批量（攒一批→processor/收单行→发卡行划钱→对账→商户入账，如每天一批）。
- 估算：日 5000 万笔（均 200 万/时，峰 500 万/时）；存储约 **5GB/天**；峰值 \~1400 tps → 入 1.12Mbps、出 1.46Mbps；约 **782 台**。
- 组件：payment service、fraud detection（实时模式分析）、risk check（设备指纹+历史+位置 → 放行/挑战/拦截）、网关/PSP、卡组织、**wallet**（按商户记余额，多卖家订单拆分）、**ledger**（immutable append-only 审计 trail）、**reconciliation**（每日拿内部账本对 PSP 结算文件）、dispute（chargeback）。
- Kafka 管线：wallet+ledger 写成功才标消息已消费 → **支付事件不丢**。
- 瞬时故障三件套：**指数退避重试**（+最大次数→标失败）、**timeout 歧义**（可能已成功/还在处理/根本没到）、**fallback**（小额交易风控报错就放行，拿小风险换体验）。
- **Idempotency key**：客户端生成唯一请求 ID，服务端对重试去重 → **防重复扣款的标准答案**。

## LeetCode（178–179）
-  workflow：选题 → 写代码 → 提交 → **sandbox 执行** → 判分返回。
- 非功能：可用、低延迟、资源约束（runtime/内存上限）、**一致性**（同代码同题 → 同 verdict，确定性求值）、安全（用户代码隔离）。
- 估算：1000 万 DAU；存储 **57.5GB/天**；带宽入均 6Mbps/峰 80Gbps、出均 47Mbps/峰 800Gbps；约 **157 台**。
- 设计：**双队列 fan-out**——LB→app→pub-sub → **代码执行**与**plagiarism 检查**并行 → results topic → metadata 聚合 → leaderboard/历史。
- 重活全异步：执行和查重走队列，用户路径保持低延迟；查重是**非阻塞后台**（MOSS 式相似度）。
- **沙箱**：ephemeral 容器 + 编排器 = 安全答案；**Redis sorted set** = 实时 leaderboard 答案；SQL 存关系数据，blob 存代码文件（DB 只存引用）；ZooKeeper 服务发现/选主。
- Contest 和练习只是路由不同（contest service vs content service），后面同一条执行管线。

## 自测 4 问（带答案要点）
1. 部署五种策略怎么选？
   → 零停机 → blue-green（2 倍成本）/ canary（要流量路由+监控）；求快求简 → rolling/basic。
2. 大二进制分发怎么省带宽？
   → P2P：区域内机器互传 + read-through（已在拉就等，不另起 pull）。
3. 支付防重复扣款的标准答案？
   → idempotency key：客户端生成唯一请求 ID，服务端对重试去重。
4. Auth 和 settlement 的区别？
   → auth 同步预留额度（钱没动）；settlement 异步批量真转钱（如每天一批）。
