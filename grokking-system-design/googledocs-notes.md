# Google Docs — 提炼笔记（169–173，2026-10-06）


## 169. 开场：先定 scope
- 核心差异化：**实时多人协同编辑**；另有自动保存、threaded comments（**回复挂在评论上不是文档位置上**，文字移动评论不丢）、版本历史（snapshot）、view/edit 权限+分享链接、使用分析、自动补全。
- Scope 假设先声明：文档约 50 页（\~100KB）、单文档 <1000 人（同时编辑远少于此）、重媒体功能出 scope。

## 170. 需求：60/40 写重型
- 功能：账户、文档 CRUD/分享、实时协同（多人同编、track changes、评论、建议）、版本历史、细粒度权限、分析、可选聊天。
- 非功能：编辑**强一致**、sub-second 延迟、HA、可靠（不丢数据）、可扩展；**读写比 60/40**——写重型，内容系统里少见。
- 估算（4 亿月活、月 8 亿文档）：日新增 8000 万文档；5000B/文档 → **400GB/天**新文档；并发约 **400 万**；5 年约 **730TB**；**编辑 24TB/天（远超新文档！）**；元数据 8GB/天；2:1 复制 → 日 intake 约 **25TB**。
- 口径：400GB vs 24TB 的对比要讲出来——编辑量主导存储，设计为写优化。

## 171. 设计：微服务拆分
- 组件：WebSocket 客户端、LB、**operation queues**（编辑排序+可追溯，与处理解耦）、**transformers**（并发解决逻辑独立，可单独扩展）、session 管理 + WS 服务器、**offline operation transformer**（断网编辑重放）、metadata DB、object store + CDN。
- **DB 与协同服务分离**：文档数据和协同处理的扩展轴不同，合在一起会互相拖累。
- API：REST 管生命周期/评论/版本/用户；WebSocket 管协同（joinSession、sendOp、broadcastOp）；分析走 GET /documents/{id}/stats。
- Polyglot：RDBMS（用户/权限/分享/threaded 评论/版本）、NoSQL（文档+操作，写快）、**time-series DB**（有序操作历史，并发解决依赖它）、Redis（缓存/session/CRDT 结构）、object store + CDN（媒体）。文档存储分片+复制。

## 172. 并发：OT vs CRDT（必考对比）
- 问题：两处并发编辑发到不同服务器，到达第三个客户端的顺序可能和编辑者意图不一致 → 客户端带 vector clock 让冲突可检测。
- **OT**：中央服务器序列化；每个 op 相对已应用的并发 op 做**变换**再广播（三步：receive → transform → broadcast）。保证：收敛、意图保持、因果保持。毛病：变换逻辑复杂、中央是瓶颈（operation queue 缓解）。
- **CRDT**：去中心化，客户端本地立即应用；数据结构数学上保证并发操作可交换 → 无需中央 authority 收敛。要求操作**可交换、幂等、结合**。毛病：元数据无界增长、内存重。
- 本章选 OT。口径：**OT 适合中心化（Google Docs），CRDT 适合去中心化/P2P（Figma）**——永远按"什么时候选哪个"来答。

## 173. 评估
- 一致性 ← OT/CRDT + 时序库保序 + gossip 复制；延迟 ← 本地 replica + WebSocket + CDN + 就近 zone + 热门文档异步复制（**拿一致性换速度**）；可用 ← 组件复制 + 多 WS 服务器；扩展 ← 微服务独立扩展 + **按文档分片 operation queue**（队列数随活跃文档横向涨）。
- 缺口：**DR 没设计**——主动说出来是加分项。

## 自测 4 问（带答案要点）
1. OT 和 CRDT 的核心区别？各适合什么？
   → OT 中央服务器序列化+变换（receive-transform-broadcast），适合中心化（Google Docs）；CRDT 去中心化本地应用，要求操作可交换/幂等/结合，适合 P2P（Figma）。
2. 两个并发插入 "ABC" 都加字符，怎么收敛到一致？
   → OT：后到的 op 相对已应用的并发 op 做 offset 变换；CRDT：本地应用，可交换的操作数学上保证收敛。
3. 60/40 读写比说明什么？
   → 写重型（少见）；编辑量 24TB/天远超新文档 400GB/天，设计为写优化。
4. Operation queue 成瓶颈怎么扩展？
   → 按文档分片，一文档一队列，队列数随活跃文档横向扩展。
