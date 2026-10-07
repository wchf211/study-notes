# Ch17: Rate Limiter — 提炼笔记（75–78，2026-10-06）


## 75. 限流器是什么
- 按时间窗口 cap 请求数（如 500 请求/分钟），是 API 前的**防御层**，不只防攻击：
  - 防"友军误伤"：bug/误配置导致的 runaway 进程拖垮资源
  - 多租户公平与配额、流量整形、成本封顶（freemium/失控实验）
  - 防 DoS、暴力破解
- 口径：先讲"防御层"定位（防内也防外），再走需求→设计→算法的答题结构。

## 76. 需求与关键决策
- 功能：按窗口限流、限额可配置、超限通知客户端。非功能：高可用（自己不能是单点）、低延迟（检查不能拖慢请求）、可扩展。
- **三种节流**：hard（硬 cap，超了丢）/ soft（500 配额 +5% 缓冲 → 525）/ elastic（有空闲就放行，无固定 cap；公平性换突发吸收）。
- **放哪**：client-side（易被篡改）/ server-side / **middleware**（API 网关，所有流量必经，最常用）。
- **集中式 DB vs 分布式计数器**：集中式（Redis/PG）严格但延迟高、有 race/lock 竞争；分布式本地计数快但不精确，sticky session 会伤容错和扩展。
- Global vs per-user 计数器：全局桶简单，per-user 公平但复杂。

## 77. 详细设计
- 独立服务，挡在 client 和 server 之间。五组件：**rule DB**（持久化规则）→ **rules retriever**（后台同步 DB 到缓存）→ **throttle-rules cache**（内存查规则）→ **decision-maker**（跑算法判放行）→ **client identifier builder**（按 IP/login ID 生成 key）。
- 拒绝 → **HTTP 429**，然后丢弃或排队（临时过载可排队）。
- **Race condition**：朴素 read-modify-write 会少计；锁会串行化成瓶颈 → 用 **Redis INCR 原子操作**，写竞争大再用 **sharded counters**（读要 sum 各分片，稍慢）。
- **关键路径优化**：在线查缓存先放行，计数器**异步离线更新**——换延迟，代价是最终一致（少量超限请求会漏过去）。
- 讨论题：limiter 自己挂了是 fail-open（放行，保可用）还是 fail-closed（拒绝，保防护）？

## 78. 五种算法（对照表背下来）
| 算法 | 内存 | 突发 | 要点 |
|---|---|---|---|
| Token bucket | 省 | 允许（≤容量 C） | 按 1/R 补 token；C、R 难调 |
| Leaking bucket | 省 | 不允许，恒定 R_out 平滑 | 满了丢弃；突发会排队延迟 |
| Fixed window counter | 省 | 允许 | **边界漏洞**：10/分钟可在 01:29 十个 + 01:30 十个 = 20 短时超限 |
| Sliding window log | 费（存每个请求时间戳） | 精确 | 无边界问题，但内存贵 |
| Sliding window counter | 省 | 平滑 | 加权公式 Rate = Rp×(Tf−Ot)/Tf + Rc；例：88×45/60+12=78 < 100 放行；假设上一窗口均匀分布（近似） |

- 选型口径：API 限流要容突发 → token bucket；保护下游要恒定速率 → leaking bucket；折中实用 → sliding window counter。

## 自测 4 问（带答案要点）
1. Fixed window counter 的边界漏洞是什么？
   → 10/分钟的配额可以在 01:29 用 10 个、01:30 再用 10 个，短时间内 20 个超限。
2. Token bucket vs leaking bucket 怎么选？
   → 要允许突发（API 限流）用 token bucket；要平滑恒定速率保护下游用 leaking bucket。
3. 分布式计数器的 race condition 怎么解？
   → Redis INCR 原子操作；写竞争大用 sharded counters（读要 sum 分片，稍慢）。
4. 关键路径上怎么把延迟压下来？代价是什么？
   → 在线查缓存先放行，计数器异步离线更新；代价是最终一致，少量超限请求会漏过去。
