# Networking + Security（Modules 11–12）

> 这两章是“先铺路，再上锁”。Module 11 给分层地图 + end-to-end argument（端到端论证）；Module 12 按安全目标组织（who / 谁能读 / 被改过吗），再给密码学工具箱。读法：先分清“哪一层该干什么”，再看“哪个工具服务哪个目标”。

## 1. Networking：每层加服务，也加 Header 和失败模式（Module 11）

**Networking（组网）**：在节点之间搬消息。词汇：
- **Service（服务）**：一层给上一层提供的东西。
- **Protocol（协议）**：同层 peer（对等方）实现服务用的规则。
- **Interface（接口）**：相邻层之间怎么说话。

原则：分层隐藏复杂度，但每层都加自己的 header（头部）、delay（延迟）、failure mode（失败模式）。用分层把问题定位到一层。别把“网络”当成一根可靠管道。例：app 慢可能是 TCP retransmission（重传），不是 app 逻辑。失败模式：在错的层 debug。Fix：逐层查行为（loss、ordering、addressing）。

**Physical layer（物理层）**：把 raw bit（原始比特）当信号传过介质（铜、光纤、无线电）。关心：编码、带宽、信号损失/噪声、物理速度上限（传播延迟 ≈ 光速在介质里）。原则：这层决定 bit rate（比特率）和 latency（延迟）物理上能到多少。用它 sanity-check（合理性检查）延迟断言。别指望软件打败传播延迟。例：跨洲往返有几十毫秒的 floor（下限）。失败模式：假设“更快硬件”去掉延迟。Fix：把工作放近点，或减少往返。

**Link layer — services（链路层服务）**：在直接相连的节点之间搬 frame（帧，一跳）。服务：framing（定界包）、addressing（MAC 地址）、error detection / correction（检错/纠错：checksum、CRC）、flow control（流控）、medium access（介质访问：谁能发）。原则：链路层让单跳相当可靠。用它理解本地投递 vs 端到端投递。别假设链路层检查保证端到端正确。例：带 CRC 的 Ethernet 帧抓住一跳的损坏。失败模式：一跳之后帧丢了，消息还是丢。Fix：更高层重传。

**Link-layer protocols（链路层协议）**：两大家族：point-to-point / switched Ethernet（帧、MAC 地址、switch 按地址转发）和 wireless（Wi-Fi，共享介质、collision / hidden-terminal 问题、CSMA/CA 先听后说 + ack）。原则：共享介质要访问规则；交换介质要转发表。按有线 vs 无线分别设预期。别假设无线像 Ethernet。例：两个听不见对方的 Wi-Fi 发送方同时发，在接收方 collision。失败模式：忽略 hidden terminal。Fix：RTS / CTS、ack、重传。

**Network layer（网络层）**：跨多跳把 packet（包）从源 host 路由到目标 host（IP）。关心：addressing、forwarding table（转发表）、routing protocol（算路径的协议）、fragmentation（分片）、best-effort delivery（尽力投递：不保证顺序、丢失、去重）。原则：IP 给全局可达性，不给可靠性。把它当 addressing / routing 底座。别靠 IP 做顺序或投递。例：两个包到同一目的地走不同路径、乱序到达。失败模式：假设包按序到达。Fix：transport 层 sequencing（TCP）或 app 层排序。

**Transport layer（传输层）**：host-to-host（实为 process-to-process，靠 port）。UDP：最小、无序、不可靠 datagram（数据报）—— 快、无连接。TCP：面向连接、可靠 byte stream（字节流），带 ordering、retransmission、flow control、congestion control（拥塞控制）—— 更慢、有 head-of-line blocking（队头阻塞）。原则：在 reliability（可靠性）vs latency / control（延迟/控制）之间选。

TCP 用于正确有序流；UDP 用于 loss 可容忍或 app 要自己 reliability（QUIC、媒体）。别假设 TCP 永远最好。例：视频通话宁要 UDP + app 层修复，不要 TCP stall（停顿）。失败模式：TCP head-of-line blocking 让所有流卡在一个丢段后面。Fix：多连接、QUIC、或 UDP + app 重传。

**Application layer（应用层）**：app 自己说的协议（HTTP、DNS、SMTP 等）：消息格式、request / response 或 push 模式、naming（命名）、session 语义。原则：transport 之上的一切都是 app 的选择 —— 包括 reliability、ordering、security。用这层定义系统的 wire contract（线上的契约）。别把 app 语义推给 TCP / IP。例：HTTP request / response 跑在 TCP 上；DNS 查询跑在 UDP 上。失败模式：假设 HTTP 保证 exactly-once 投递。Fix：app 层 idempotency + retry。

**Taking a step back：End-to-end argument（端到端论证）**：每层加服务和 header，问题可以活在任何层。端到端论证：只有 endpoint（端点）才能做对的功能（reliability、security、ordering）应该活在 endpoint，不完全委托给网络。原则：别过度信任网络。分层做分离，但端到端验证。别因为某一层 retry 就假设“网络可靠”。例：TCP retry 不让分布式事务 atomic。失败模式：靠网络可靠性保证 app 正确性。Fix：端到端检查（checksum、ack、idempotency）。

## 2. Security：按目标组织，先威胁建模，再选机制（Module 12）

**Security（安全）** 在分布式里 = 在消息穿不可信网络、节点可能被攻破的情况下保护数据和操作。Attacker model（攻击者模型）：eavesdrop（窃听）、tamper（篡改）、replay（重放）、impersonate（伪装）、deny service（拒绝服务）。原则：假设网络是敌对的；没有 perimeter（边界）能救你。选机制前先 threat modelling（威胁建模）。别等协议定好了再 bolt on（事后贴上）安全。例：信任局域网里任何节点的服务。失败模式：内部人或被攻破节点伪造请求。Fix：从一开始 authentication + encryption + least privilege。

**Authentication（认证）**：回答“这是谁？”—— 用户、服务、节点。机制：password（弱，要 hashing / salting、MFA）、token、certificate（证书）、mutual authentication（双向认证：双方都证明身份）。原则：每个能行动的 principal（主体）都要认证，不只人类用户。能用强、防钓鱼 factor（因素）就用。别把 authentication 和 authorisation（授权：谁能干什么）混淆。例：服务给另一个服务出示 client certificate。失败模式：replay 抓到的 credential（凭证）。Fix：nonce / timestamp、短命 token、channel binding。

**Confidentiality（机密性）**：只有授权方能读数据，传输中和静止时都算。机制：encryption（加密）+ key（密钥）；key distribution（密钥分发）是难点。原则：没有 key management 的 encryption 不是 confidentiality。敏感字段和通道用 encryption，key 要 scoped（限定范围）+ rotated（轮换）。别自己发明 crypto（密码学），别把 key 存在数据旁边。例：TLS 加密流量；disk encryption 保护存的副本。失败模式：key 漏进 log / config。Fix：key-management service + rotation + 最小 key scope。

**Integrity（完整性）**：数据没被改过（意外或恶意）且来自声称的 sender（发送方）。机制：checksum（只防意外错误）、MAC / message authentication code（消息认证码，共享密钥，证明 sender + 完整性）、digital signature（数字签名，公钥，另带 non-repudiation，不可否认性）。原则：检测篡改，别只希望它不发生。攻击者可能篡改时用 MAC 或签名，不用 bare hash（裸哈希）。例：签名过的配置文件 —— 任何编辑都破坏签名。失败模式：用 unkeyed hash 对付主动攻击者（攻击者重算它）。Fix：MAC / 签名。

**Cryptography primer（密码学入门）**：基础：randomness（随机性：key / nonce 必须不可预测）、hash function（哈希函数：单向、collision-resistant 抗碰撞的摘要）、symmetric encryption（对称加密：同一 key 加密/解密，快，需要共享秘密）。原则：安全 rests（立）于 key 和随机性，不在于算法保密。用 vetted（经审查的）算法和库。别发明 cipher（密码），别跨用途复用 nonce / key。例：每连接一个随机 session key。失败模式：可预测的“随机”数泄漏 key。Fix：cryptographic RNG（密码学随机数发生器）+ proper key generation。

**Symmetric / asymmetric encryption + digital signature（对称/非对称加密 + 数字签名）**：Symmetric：一个共享 key，快，需要安全 key 分发。Asymmetric（非对称）：key pair（密钥对，public 加密/验证，private 解密/签名）；基于 intractable problem（难解问题，RSA 是 factoring 因数分解，Diffie-Hellman 是 discrete log 离散对数）。Trapdoor function（陷门函数）：一个方向容易，没有秘密难。Diffie-Hellman 让两方在公开通道上协商共享秘密。Digital signature：private key 签名，public key 验证 —— 完整性 + 真实性，不要 confidentiality（和 MAC 不同，不需要共享秘密）。

原则：用非对称 crypto 分发 key 和签名，用对称 crypto 做 bulk data（批量数据）。用 hybrid scheme（混合方案：非对称 key exchange + 对称加密）。别直接用 RSA 加密大数据，别不加考虑把一个 key 同时用于加密和签名。例：TLS handshake 用非对称 crypto 协商对称 session key。失败模式：信任被攻击者换掉的 public key。Fix：certificate / PKI（下一章）。

## 这门课怎么教的（Modules 11–12）

Module 11 是紧凑的自底向上分层 tour，框架是“每层加了什么、留给上层什么”。先给 service vs protocol vs interface 词汇，再按 physical → link → network → transport → application 走，每层复用同一双 host 消息例子。最后退一步给 end-to-end argument：多数分布式关切活在 edge（边缘），不在网络里。视觉是分层图 + header / payload 草图；最好懂处是把每层绑到一个具体分布式后果（为什么 TCP in-order 投递会 hurt，为什么 UDP 仍需要 app 层 reliability）。

Module 12 按目标组织，不按算法：先讲分布式里安全是什么意思（攻击者在网络上，没有可信边界），再每目标一课 —— authentication（谁）、confidentiality（谁能读）、integrity（被改过吗）—— 再给实现它们所需的 crypto 工具箱（randomness / hash、对称 vs 非对称）。全程 Alice / Bob / Mallory 式场景。Crypto 原语重叠处较难；课程反复用“这个原语服务哪个目标？”重新锚定。值得偷的：**按目标讲安全，按目标选工具** —— 讲 crypto 时先问“它服务哪个目标”，观众不会在原语里迷路。

## 自测 4 问（带答案要点）
1. Service / protocol / interface 各是什么？分层的好处和代价？
   → 服务 = 一层给上层的；协议 = 同层 peer 实现服务的规则；接口 = 相邻层怎么说话；分层定位问题，但每层加 header / delay / failure mode。
2. TCP vs UDP 怎么选？Head-of-line blocking 是什么？
   → TCP 正确有序流；UDP loss 可容忍或 app 要自己 reliability；队头阻塞 = 一个丢段卡住后面所有流。
3. End-to-end argument 说什么？举一例。
   → 只有 endpoint 能做对的功能应活在 endpoint；TCP retry 不让分布式事务 atomic。
4. Authentication / confidentiality / integrity 各对应什么机制？为什么 bare hash 不够？
   → 认证：密码/token/证书/双向；机密：加密 + key 管理；完整：MAC / 签名；bare hash 攻击者能重算，要 keyed MAC / 签名。
