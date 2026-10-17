# Security Protocols + Case Study 1：GFS（Modules 13–15）

> Module 13 是 Module 12 目标的协议版 —— 每个协议按同一弧线讲：解决什么问题 → 谁信任谁 → 消息流 → 哪一步跳过会坏。Module 14 是桥：后面不再是新理论，是把前 13 章的原语在真实系统里认出来。Module 15 是第一个 case study：GFS，记住它的 workload（大文件、append 为主、多读者）—— 每个设计选择都是它推出来的。

## 1. TLS：非对称认证/协商 + 对称批量数据（Module 13.1）

**TLS（Transport Layer Security，传输层安全）**：在 TCP 上给 confidential（机密）、authenticated（认证）、integrity-protected（完整性保护）的通道。Handshake（握手）：client 和 server 协商协议版本、cipher suite（密码套件）、密码参数；server 出示 certificate（证书）并证明持有 private key（私钥）；双方协商 shared secret（如经 Diffie-Hellman），导出对称 session key（会话密钥）；之后 app 数据被加密 + MAC。

原则：非对称 crypto 做认证/协商，对称做批量数据。不可信网络上用 TLS（服务间最好 mutual authentication，双向认证）。敏感数据别用 plain TCP，别跳过 certificate verification（证书校验）。例：浏览器 → 银行，padlock（锁）意味着 server 身份被检查过。失败模式：man-in-the-middle（中间人）拿未校验证书读/改流量。Fix：proper certificate validation + 强 cipher + 当前 TLS 版本。

## 2. PKI：信任经 CA 层级传递（Module 13.2）

**PKI（Public-Key Infrastructure，公钥基础设施）** 回答“这个公钥是谁的？”—— 可信 CA（certificate authority，证书颁发机构）把公钥和身份绑进签名的 certificate；verifier（验证方）信任 CA root（根），检查 chain（链）、expiry（过期）、revocation（吊销）。原则：信任经 CA 层级传递。

中央 trust anchor（信任锚）可接受的地方用 PKI（web、企业）。没有 out-of-band verification（带外验证）别信 self-signed certificate（自签名证书）。例：浏览器信某站证书，因为 root CA 签了链。失败模式：CA 被攻破/错发，或吊销证书仍被接受。Fix：短命证书 + revocation 检查（CRL / OCSP）+ certificate transparency（证书透明度）。

## 3. Web of Trust（PGP）：去中心化的信任图（Module 13.3）

**Web of Trust（信任网，PGP）**：PKI 的去中心化替代 —— 用户互相签名对方 key，建成信任图；你信任一个 key，如果你信任的人（直接或经短链）签过它。PGP 用它做加密/签名邮件和文件。原则：没有中央权威 —— 信任是社会化的、可传递的。

中央 CA 不可用或不想要时用。别指望它像 PKI 一样 scale（扩展）—— key discovery（密钥发现）和 revocation 更难。例：开发者 key 被两个你已信任的同事签过。失败模式：恶意签名者给假 key 背书，或吊销不传播。Fix：谨慎的 key-signing 实践 + 带外 fingerprint 验证 + expiry。

## 4. OAuth：Delegation，不是 Authentication（Module 13.4）

**OAuth** 是 delegation（委托）：用户让 app 代它在某服务上行动，不给 app 密码。角色：resource owner（资源所有者，用户）、client（app）、authorisation server（授权服务器）、resource server（资源服务器）。Flow（流程）：client 把用户重定向到授权 scope 请求；授权服务器返回 code / token；client 换 short-lived access token（短命访问令牌，可选 refresh token），拿它给资源服务器看。

原则：token 是 scoped（限定范围）、time-limited（限时）、revocable（可吊销）—— 和密码不同。第三方访问和服务委托用 OAuth。别把 OAuth 当 authentication（那是上层的 OpenID Connect），别把 token 漏进 URL / log。例：日历 app 用 read-only token 读你日历，不拿你密码。失败模式：被盗 token 带着 broad scope（宽范围）。Fix：最小权限 scope + 短命 + PKCE + 安全 token 存储。

## 5. From Theory to Practice：Case Study 是比较，不是百科（Module 14）

桥：基础（partitioning、replication、quorum、consistency、consensus、time、networking、security）现在要应用到真实系统 —— file system、coordination service、data store、messaging、cluster management、ledger、data processing —— 再到可复用 pattern。原则：真实系统是同一批原语的组合，按 workload 做 trade-off。

用 case study 看 guarantee（保证）怎么选，别当孤立产品背。别指望任何一个系统普遍“最好”。例：GFS 为 append throughput（吞吐）trade 强一致性，因为 workload 是 append-heavy。失败模式：不看 workload 假设就抄某系统设计。Fix：先点名 workload 和 guarantee。

## 6. GFS / HDFS：元数据集中，数据分布（Module 15.1）

**GFS（Google）** 和 **HDFS（Hadoop）** 同架构：一个 manager / master（namespace、chunk placement、lease）+ 很多 chunk / data server（存大固定大小 chunk，经典 GFS 是 64 MB），replication（典型 3×）。设计是 workload-driven（工作负载驱动）：huge file（巨大文件）、sequential read / append（顺序读/追加）、很多 client、会坏的 commodity hardware（廉价硬件）。原则：元数据集中，数据分布；优化 throughput（吞吐）而非 latency（延迟）。

Batch / analytical 文件存储用这个 pattern。小文件、随机写、低延迟 workload 别用。例：1 TB 数据集拆成 chunk 分散到几百节点。失败模式：manager 变 bottleneck / single point of failure。Fix：metadata scaling、standby manager、manager 不上数据路径。

## 7. 创建 + 读：Manager 在控制路径，数据在直达路径（Module 15.2）

**Create（创建）**：client 问 manager 要 chunk 位置；manager 分配 chunk、把 metadata 记进 operation log（操作日志）、返回 handle；client 直接写 chunk server。**Read（读）**：client 问 manager 哪个 chunk server 持有 chunk，再从最近 replica 读，manager 不再参与。原则：manager 在 control path（控制路径），数据在 direct path（直达路径）。

要 scale 用直接读。别把 bulk data（批量数据）经 manager 路由。例：读文件只碰 manager 一次，然后从 chunk server 流式读。失败模式：replica 移动后 manager 信息 stale（过期）。Fix：lease、heartbeat、client 按 fresh metadata 重试。

## 8. 写 + 删：靠 Leased Primary 排序写（Module 15.3）

**Namespace mutation（命名空间变更：create / delete / rename）**：atomic（原子）且 linearizable（线性化），经单个 manager，用 read-write lock（读写锁）串行化 + operation log 里 total order（全序）。**Data write（数据写）**：client 沿 replica 链 push 数据（chain replication，链式复制，用带宽高效），再请 primary replica（lease holder，租约持有者）分配 serial number（序列号）并排序写；primary 应用并转发给 secondary，再 ack。

**Delete（删）**：manager 把文件移到 hidden namespace（隐藏命名空间，undelete 窗口），再 lazy garbage-collect（惰性回收）chunk。原则：经 leased primary 给写排序；删要可恢复。并发写用 chain replication + primary ordering。数据写别指望强一致性 —— 见下一课。例：两个写同一 chunk 被 primary 的编号串行化。失败模式：primary 写一半 crash → 一个写可能以不同顺序落在部分 replica 上。Fix：lease expiry、新 primary、GFS consistency model（defined / undefined region）。

## 9. GFS Consistency Model：为 Append 调过的弱一致性（Module 15.4）

GFS 给 mutation（变更）后文件 region 定义自定义模型：region 是 **consistent（一一致）** 如果所有 replica 一致；**defined（有定义）** 如果 client 看到 mutation 写的东西（外加可能的并发 append）。无并发干扰的成功 mutation → defined 且 consistent。并发写 / append 可能留下 consistent-but-undefined（一致但无定义：所有 replica 一致，但内容可能混了写）或 inconsistent（不一致：replica 分歧，如 primary 失败后）。**Append** 是主要操作：并发 append 是 at-least-once（至少一次）—— 数据可能 duplicated（重复），reader（读者）必须处理重复。

原则：比 linearizability 弱，为 append-heavy workload 调过。Reader 能容忍重复的地方用 GFS 式 append。别假设每个写都 linearizable。例：两个 client 同时 append → 文件里两个 append 都有，可能一个重复。失败模式：app 假设 append 恰好发生一次。Fix：checksum、record marker、duplicate-tolerant reader。

## 这门课怎么教的（Modules 13–15）

Module 13 协议一个一个讲 Module 12 的目标。每课同一弧线：解决什么问题 → 谁信任谁 → 消息流 → 哪一步跳过会坏。TLS 是 confidential + authenticated 通道的 worked example；PKI vs Web of Trust 对比集中 vs 去中心化信任；OAuth 对比 delegation vs 密码共享。图是 handshake / certificate-chain 流。

Module 14 是一课桥接，显式 reframing（重新框定）课程后半部分：同一批概念（partitioning、replication、consensus、consistency）现在要在真实系统里认出来，而不是当新理论引入。定预期：case study 是比较，不是百科。

Module 15 用 GFS 当 running system，HDFS 当紧密比较。先给 workload 假设（大文件、多 append、多读者），每个设计选择都是它推出来的，再按执行顺序走操作（create / read → write / delete → consistency）。用 manager / chunk-server 图和 chain replication 演练。读者记住 workload 时最易懂；难处是 GFS 的弱一致性偏离了前面学的强模型。值得偷的：**先给 workload，再给设计** —— 每个设计选择都要能追回 workload 假设，否则观众会觉得“为什么不直接做强一致”。

## 自测 4 问（带答案要点）
1. TLS handshake 四步？为什么非对称 + 对称分工？
   → 协商版本/cipher → 证书 + 私钥证明 → 协商 shared secret → 对称加密 + MAC；非对称认证/协商，对称批量数据。
2. PKI vs Web of Trust 各是什么信任模型？各的弱点？
   → PKI：经 CA 层级传递；弱点 CA 攻破/错发、吊销；Web of Trust：社会化信任图；弱点 key 发现/吊销难、恶意背书。
3. OAuth 是什么、不是什么？Token 相对密码赢了什么？
   → Delegation，不是 authentication；token 是 scoped / 限时 / 可吊销，密码不是。
4. GFS 为什么用弱一致性？Append 语义是什么？
   → Workload 是 append-heavy，强一致性代价不值；append 是 at-least-once，reader 必须容忍重复。
