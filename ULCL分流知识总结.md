# ULCL 分流 — 核心网知识总结

> 学习日期：2026-06-02
> 参考教程：https://github.com/s5uishida/free5gc_ueransim_ulcl_sample_config

---

## 一、核心概念

### 1.1 什么是 ULCL？

ULCL = Uplink Classifier = 上行分类器 = 上行分流器

大白话：在 I-UPF 上根据**目标地址**决定 UE 的数据走哪条路。

### 1.2 为什么要分流？

不同流量走不同路径，比如：
- 普通上网流量 → 走 PSA-UPF 上公网
- 特定目标（如 DNS 服务器）→ 走 I-UPF 直连，更快
- 内网流量（如 Docker）→ 走 I-UPF 直连内网，不经过公网

### 1.3 三个关键角色

| 节点 | 全称 | 比喻 | 作用 |
|------|------|------|------|
| **I-UPF** | Intermediate UPF（中间 UPF） | 路口交警 | 拆壳 → 查规则表 → 决策走哪条路 |
| **PSA-UPF** | PDU Session Anchor UPF（锚点 UPF） | 高速收费站 | 最终出口，做 NAT，连公网 |
| **SMF** | Session Management Function | 交管局指挥中心 | 把分流规则（uerouting.yaml）通过 N4/PFCP 下发给 I-UPF |

### 1.4 I-UPF 和 PSA-UPF 的本质

**同一个 UPF 程序，两份不同的配置文件，SMF 指挥它们干不同的活。**

就像同一个演员，演了两个不同的角色：
- I-UPF 有 N3 + N9 接口（既接基站，又接另一个 UPF）
- PSA-UPF 只有 N9 接口（不直接接基站，只接收 I-UPF 转来的数据）

---

## 二、分流拓扑

### 2.1 完整路径图

```
                        📱 UE
                      IP: 10.60.0.1
                     作用：发原始 IP 包，源=10.60.0.1
                     不做 NAT，不做隧道，只发裸 IP 包
                           │
                      🌀 空口（NR-Uu）
                     电磁波，无线传输
                     不上隧道，不做 NAT
                           │
                        📡 gNB
                   IP: 192.168.56.102
                     作用：收到 UE 的包，套 GTP-U 外壳
                     外壳写上自己的 IP，从 N3 发给 I-UPF
                     绝对不做 NAT！只做套壳/拆壳
                           │
                   N3 = GTP-U 隧道 🎁
                   为什么：gNB 和 I-UPF 在核心网内部
                   UE 的 10.60.0.1 是私有 IP，中间网络不认识
                   必须套壳才能转发
                   外层源: 192.168.56.102 (gNB)
                   外层目标: 192.168.56.101 (I-UPF)
                   内层源: 10.60.0.1 (不变！)
                   内层目标: 8.8.8.8 或 google.com
                           │
                    ┌──────┴──────┐
                    │   🔄 I-UPF   │
                    │ IP: 192.168.56.101
                    │ 作用：拆壳→查规则表→决策
                    │ 只有 N3 + N9 接口
                    │ 路径A/C 自己做出口+做NAT
                    │ 路径B 只转发，不做NAT
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
         匹配 8.8.8.8  匹配 Docker  不匹配（默认）
              │            │            │
           路径A        路径C         路径B
              │            │            │
           I-UPF出口    I-UPF出口   转给 PSA-UPF
              │            │            │
           N6 直接      N6 直接     N9 GTP-U 🎁
           ❌不上隧道   ❌不上隧道   源和目标都是
              │            │      192.168.56.101
           做 NAT       不做NAT    （同一台VM）
              │            │            │
              ▼            ▼            ▼
          🌐 公网      Docker内网   ⚓ PSA-UPF
        (8.8.8.8)   (172.17.0.1)  IP: 192.168.56.101
                                   只有 N9 + N6 接口
                                   作用：拆 N9 壳 → 做 NAT → N6 出口
                                         │
                                       N6 直接 ❌不上隧道
                                         │
                                          ▼
                                      🌐 公网
                                   (google.com)
```

### 2.2 三条路径区别

| | 路径A | 路径B | 路径C |
|------|------|------|------|
| 目标 | 8.8.8.8（公网） | google.com（公网） | 172.17.0.1（Docker内网） |
| 触发条件 | 匹配 specificPath | 不匹配，走默认 | 匹配 specificPath |
| 出口节点 | I-UPF 直出 | PSA-UPF 转出 | I-UPF 直出 |
| 经过 N9 隧道？ | ❌ 不经过 | ✅ 经过 | ❌ 不经过 |
| 做 NAT？ | ✅ 做 | ✅ 做 | ❌ 不做 |
| 不做 NAT 的原因 | — | — | Docker 内网认识 10.60.0.1 |

### 2.3 三条回包路径

```
路径A 回包：8.8.8.8 → I-UPF（NAT还原）→ gNB → UE
路径B 回包：google.com → PSA-UPF（NAT还原）→ I-UPF → gNB → UE
路径C 回包：172.17.0.1 → I-UPF → gNB → UE（无需NAT还原，因为没做过NAT）
```

**规律：谁做的 NAT，谁负责还原。回包 = 去程的逆序。**

---

## 三、接口速查表

| 接口 | 谁和谁通信 | 协议 | 要不要隧道 | 要不要 NAT |
|------|------|------|------|------|
| 空口（NR-Uu） | UE ↔ gNB | 无线 | ❌ 电磁波 | ❌ |
| N2 | gNB ↔ AMF | NGAP/SCTP | — | ❌ 信令不走数据 |
| N3 | gNB ↔ UPF | GTP-U | ✅ 要 | ❌ |
| N4 | SMF ↔ UPF | PFCP | — | ❌ 信令不走数据 |
| N6 | UPF ↔ 互联网 | 纯 IP | ❌ 不要 | ✅ 做 NAT |
| N9 | UPF ↔ UPF | GTP-U | ✅ 要 | ❌ |

---

## 四、NAT 与隧道的关系

### 4.1 什么时候需要隧道？

**判断标准：内层 IP（10.60.0.1）接收方认不认识？**

- 不认识 → 必须套 GTP-U 壳 → N3、N9 需要隧道
- 认识 → 不需要隧道 → N6 直接走，Docker 内网直接走

### 4.2 什么时候需要 NAT？

**判断标准：接收方在不在公网上？**

- 在公网 → 10.60.0.1 是私有 IP，公网不认识 → 必须做 NAT
- 在内网 → 内网认识 10.60.0.1 → 不用 NAT

### 4.3 NAT 怎么还原（回包怎么找回来）？

**NAT 不是瞎换，是有映射表的！**

```
NAT 映射表（UPF 的小本本）：
┌──────────────────────────────────┐
│ 内网侧              外网侧        │
│ 10.60.0.1:12345  ↔  公网IP:56789 │
│ 10.60.0.1:12346  ↔  公网IP:56790 │
└──────────────────────────────────┘
```

去程：换源 IP → 记一笔账
回程：查映射表 → 还原目标 IP → 送回原主人

### 4.4 为什么不能全用 NAT 替代隧道？

因为 NAT **只管换 IP**，分不清不同用户：

```
三个 UE 同时访问 8.8.8.8，都做了 NAT：
  UE-1 → 公网IP:56789 → 8.8.8.8
  UE-2 → 公网IP:56790 → 8.8.8.8
  UE-3 → 公网IP:56791 → 8.8.8.8

回包到了基站，全写的是公网IP → 该给谁？没法区分！

隧道的作用：
  每个 UE 有独立的 TEID 标签 → 回包上带 TEID → 基站一看就知道给谁
```

| | 纯 NAT | GTP-U 隧道 |
|------|------|------|
| 区分不同用户 | ❌ 分不清 | ✅ TEID 区分 |
| 按用户计费 | ❌ 做不到 | ✅ 按隧道统计 |
| QoS 控制 | ❌ 做不到 | ✅ 每个隧道独立配 |
| 用户移动切换 | ❌ 断线 | ✅ 无缝切换 |

---

## 五、PSA-UPF 为什么叫"锚点"？

### Anchor = 锚

UE 移动时，前面的基站和 I-UPF 可以换，但 PSA-UPF 不能换：

```
原来：UE → gNB1 → I-UPF1 → ⚓ PSA-UPF → 互联网
                                  ↑
移动后：UE → gNB2 → I-UPF2 → ⚓ PSA-UPF → 互联网
                                  ↑
                           锚点没变！
```

不能换的原因：
1. UE 的 IP（10.60.0.1）是 PSA-UPF 分配的
2. NAT 映射表在 PSA-UPF 上，换了就丢
3. 互联网只认识 PSA-UPF 的公网 IP

---

## 六、N9 隧道在同一台 VM 上也要走

即使 I-UPF 和 PSA-UPF 都在同一台机器（同一个 IP），N9 隧道也必须走：

```
N9 隧道包：
  外层源: 192.168.56.101  ← I-UPF
  外层目标: 192.168.56.101  ← PSA-UPF（同一个 IP！）
  内层源: 10.60.0.1
  内层目标: google.com
```

**IP 一样怎么区分？靠 TEID。** 两个 UPF 进程监听不同的 TEID 标签。

---

## 七、知识自测（5 道题）

### 第 1 题（★☆☆☆☆）

**以下哪个说法是正确的？**

A. I-UPF 和 PSA-UPF 是两种完全不同的程序

B. I-UPF 和 PSA-UPF 是同一个 UPF 程序，只是 SMF 给它们分配了不同的角色 ✅

C. PSA-UPF 负责分流决策

D. I-UPF 负责最终 NAT 出口

<details>
<summary>答案</summary>

**B 正确。**

- A 错：同一个程序，两份配置
- C 错：分流决策的是 I-UPF（查 uerouting.yaml 规则表）
- D 错：不一定。路径A 是 I-UPF 做出口，路径B 是 PSA-UPF 做出口

</details>

### 第 2 题（★★☆☆☆）

UE 的 IP 是 `10.60.0.1`，访问 `8.8.8.8`。

**数据包经过 N3 接口时，外层源 IP 是多少？**

A. `10.60.0.1`

B. `8.8.8.8`

C. `192.168.56.102` ✅

D. `192.168.56.101`

<details>
<summary>答案</summary>

**C 正确。**

N3 隧道的外壳是 gNB 套的，源 IP 写的是 gNB 自己的 IP（192.168.56.102）。

- A `10.60.0.1` → 内层源，不是外层
- B `8.8.8.8` → 内层目标，不是外层
- D `192.168.56.101` → I-UPF 的 IP，是外层目标

</details>

### 第 3 题（★★★☆☆）

UE 访问 `google.com`，流量走路径B（I-UPF → PSA-UPF → 互联网）。

**I-UPF 的操作顺序是？**

A. 拆 N3 壳 → 查规则 → 做 NAT → 发往互联网

B. 查规则 → 拆 N3 壳 → 套 N9 壳 → 做 NAT

C. 拆 N3 壳 → 查规则 → 套 N9 壳 → 发给 PSA-UPF ✅

D. 查规则 → 套 N9 壳 → 拆 N3 壳 → 发给 PSA-UPF

<details>
<summary>答案</summary>

**C 正确。**

路径B 中 I-UPF 只转发，不做 NAT，不直接发互联网——那是 PSA-UPF 的活。

A 描述的是路径A（I-UPF 直连出口）的流程。

</details>

### 第 4 题（★★★★☆）

**填上回包路径的空白，并回答 NAT 还原在哪做：**

```
回包 A（8.8.8.8 回来）：
  8.8.8.8 → I-UPF → gNB → UE
  NAT还原在：I-UPF

回包 B（google.com 回来）：
  google.com → PSA-UPF → I-UPF → gNB → UE
  NAT还原在：PSA-UPF
```

<details>
<summary>答案</summary>

**规律：回包 = 去程的逆序。谁做的 NAT，谁负责还原。**

- 路径A：I-UPF 做的 NAT → I-UPF 还原
- 路径B：PSA-UPF 做的 NAT → PSA-UPF 还原

</details>

### 第 5 题（★★★★★）

I-UPF 和 PSA-UPF 都在同一台 VM 上（192.168.56.101）。UE 访问 google.com，走路径B。

**N9 接口上，外层源 IP 和外层目标 IP 分别是多少？**

A. 源 `192.168.56.102` → 目标 `192.168.56.101`

B. 源 `192.168.56.101` → 目标 `192.168.56.101` ✅

C. 源 `10.60.0.1` → 目标 `google.com`

D. 源 `192.168.56.101` → 目标 `192.168.56.102`

<details>
<summary>答案</summary>

**B 正确。**

N9 = I-UPF → PSA-UPF，壳是 I-UPF 套的，源 = I-UPF 的 IP，目标 = PSA-UPF 的 IP。同一台 VM 所以源和目标都是 192.168.56.101。区分靠 TEID。

</details>

---

## 八、ULCL 架构的设计合理性

### 8.1 为什么拆成「锚点 + 分流器」两个角色？

5G 之前（4G），所有流量必须经过 PGW（锚点）。哪怕访问隔壁的服务器，流量也得绕到几百公里外的锚点。5G 想解决的是：**在离用户近的地方就分流。**

但一个问题出现了——锚点和分流器的需求天然矛盾：

| 锚点需要 | 分流器需要 |
|------|------|
| 不动（UE 移动换不了） | 动（离 UE 越近越好） |
| 在数据中心（稳定可靠） | 在基站旁边（边缘机房） |
| 一个 UE 一个 | 多个（跟着位置换） |

**一个人没法既动又不动 → 必须拆成两个角色。**

拆法探索：
- ❌ 基站直接分流：基站只懂无线，不能把业务逻辑全塞进来
- ❌ 一个 UPF 既锚点又分流：UE 一移动就得换 UPF → 换锚点 → 换 IP → 全断
- ✅ 分开：PSA-UPF 不动管锚点，I-UPF 就近可换管分流

### 8.2 锚点的真正本质

NAT 不是 UPF 的功能——UPF 只做转发，NAT 是 Linux iptables 干的。分 IP 是 SMF 干的。

**锚点的核心价值：为 DN（数据网络）提供一个固定的 N6 出入口。**

```
UE 不管怎么移动：
  DN 回包永远送到 PSA-UPF 的 N6 口（固定地址）
  DN 不需要知道 I-UPF 在哪、gNB 在哪
  移动性被隔离在核心网内部，外面无感知
```

---

## 九、UPF 转发机制：PDR + FAR

### 9.1 UPF 自己不思考，只按规则执行

SMF 通过 N4/PFCP 下发两类规则：

**PDR（Packet Detection Rule，包检测规则）→ "认出这个包"**

```
PDR: 如果内层源 IP=10.60.0.1，来自 N3 口，TEID=0xA001
     → 这是 UE-1 的包 → 执行对应的 FAR
```

**FAR（Forwarding Action Rule，转发动作规则）→ "往哪送"**

```
FAR-特殊: 目标=8.8.8.8/32   → 做 NAT → N6 口出（路径A）
FAR-内网: 目标=172.17.0.0/16 → 不做 NAT → N6 口入 Docker（路径C）
FAR-默认: 其他所有目标        → 套 N9 壳 → 发给 PSA-UPF（路径B）
```

### 9.2 ULCL 不是一条规则，是一整套方案

```
ULCL 方案 = 拓扑设计（links）+ 角色分配（谁I谁PSA）+ 分流规则（specificPath）

SMF 根据这套方案，为每条 PDU 会话生成具体的 PDR/FAR，下发到各 UPF
```

---

## 十、TEID 详解

### 10.1 TEID 是什么

TEID = Tunnel Endpoint Identifier = 隧道端点标识 = 贴在外壳上的编号标签

```
一条 N9 隧道上跑着几百个 UE 的包：
  外层源 IP 和目标 IP 全一样 → 怎么区分？

  靠 TEID：
    UE-1 → TEID=0xB001
    UE-2 → TEID=0xB002
    UE-3 → TEID=0xB003
```

### 10.2 TEID 的粒度

粒度 = 精细到什么程度。

**TEID 的粒度 = 每 PDU 会话 × 每隧道**

```
一个 UE 有两条 PDU 会话（上网 + 企业专网）：

  N3 隧道上：上网会话 TEID=0xA001，企业会话 TEID=0xA003
  N9 隧道上：上网会话 TEID=0xB001，企业会话 TEID=0xB003

每条会话在每条隧道上各有一个独立的 TEID，会话存活期间不变
会话释放 → TEID 回收 → 可以给下一个 UE 用
```

### 10.3 N3 和 N9 都是 GTP-U 隧道

N3 和 N9 协议完全相同（GTP-U），都套壳、都有 TEID。唯一区别是两端是谁：
- N3：gNB ↔ UPF
- N9：UPF ↔ UPF

---

## 十一、隧道的本质——封装

### 11.1 为什么要在原报文上套新头部？

因为中间设备不认识内层 IP，但你又不能改原始报文。

替代方案为什么不行：
- ❌ 每跳都做 NAT：多跳 NAT 表耦合，回包还原路径太长，不能区分 UE
- ❌ 公网加 10.60.0.1 路由：上亿 UE，公网路由表直接崩

### 11.2 封装 = 内外分离

```
外层（给中间设备看）：               内层（给收发双方看）：
  源: 192.168.56.102                 源: 10.60.0.1
  目标: 192.168.56.101               目标: 8.8.8.8
  TEID: 0xA001                       原封不动，全程不变

外层解决路由可达 + 多用户复用
内层保证原始报文完整性
```

### 11.3 中间设备

数据从 UE 到公网不是一步到位。中间设备 = gNB、I-UPF、交换机、路由器。每个只负责一段，用隧道保证接力不丢不混。分层设计：每层只解决自己擅长的问题。

---

## 十二、I-UPF 与 PSA-UPF 是逻辑角色

### 12.1 同一个 UPF 程序，角色由 SMF 决定

free5GC 只有一份 UPF 二进制。启动两个实例，各用各的配置：

```bash
./upf -c upfcfg.yaml &     # 实例1
./upf -c upf2cfg.yaml &    # 实例2
```

### 12.2 配置声明能力，links 决定角色

```
UPF 的配置声明"我有哪些接口"（N3/N9/N6）
SMF 的 links 决定"你在拓扑的哪个位置"
  → 放中间 = I-UPF
  → 放末端 = PSA-UPF

同一台 UPF：
  会话A 的 links 把它放中间 → 它是 I-UPF
  会话B 的 links 把它放末端 → 它是 PSA-UPF
```

---

## 十三、PDU 会话建立与 SMF 决策全流程

### 13.1 PDU 会话是什么

PDU 会话 = UE 和互联网之间的一条数据通道。建好之后 UE 才拿到 IP，有了 IP 才能上网。

### 13.2 SMF 建立 ULCL 会话的决策顺序

```
① 选 PSA-UPF（先定锚点——IP 池绑定在这）
② 分配 UE IP（SMF 从 PSA-UPF 的池子里挑 10.60.0.1）
③ 选 I-UPF（确定分流器，建 gNB→I-UPF→PSA 链路）
④ 生成 PDR/FAR（根据 uerouting.yaml 生成转发规则）
⑤ 下发规则（通过 N4/PFCP 发给两个 UPF）
```

**IP 池挂在 PSA-UPF 名下，不是 PSA-UPF 自己分 IP，是 SMF 分。PSA-UPF 只是被告知"这个 IP 的 NAT 你来管"。**

### 13.3 各节点作用总览

**控制面（只传信令，不传用户数据）：**

| 节点 | 作用 | 比喻 |
|------|------|------|
| NRF | 通讯录，所有 NF 启动后在此登记，别人找人先查它 | 公司前台 |
| AMF | 鉴权管理 + 转发 PDU 会话请求给 SMF | 门禁前台 |
| SMF | 选 UPF + 分 IP + 生成 PDR/FAR + 下发规则（N4） | 交管局总指挥 |
| UDM/UDR | 存用户签约数据和密钥 | 档案室 |
| AUSF | 执行 5G AKA 鉴权 | 身份证验证机 |
| PCF | 下发 QoS 策略 | 规章制度 |

**用户面（只传数据，不参与信令）：**

| 节点 | 作用 | 比喻 |
|------|------|------|
| UE | 发原始 IP 包 | 寄信人 |
| gNB | 套 N3 壳/拆壳，不做 NAT | 邮局收件窗口 |
| I-UPF | 拆壳 → 查 PDR/FAR → 分流决策 | 路口交警 |
| PSA-UPF | 拆 N9 壳 → N6 出口（NAT 是 Linux 干的） | 收费站/锚点 |

---

## 十四、网络切片（Slicing）

**切片 = 同一套物理 5G 网络上切出多个虚拟专网，各有独立的 QoS/带宽/延迟。**

```
S-NSSAI = SST + SD（切片的身份证号）
  SST=1: eMBB（手机上网，大带宽）
  SST=2: URLLC（自动驾驶，超低延迟）
  SST=3: MIoT（抄水表，海量连接）
  SD: 同一 SST 下再细分
```

你实验里用的 SST=1 SD=010203 就是一个普通上网切片。

---

## 十五、实验手动改配置 vs 生产自动化

| | 运营商 | 你的实验 |
|------|------|------|
| 配置方式 | OSS 界面操作 → MANO 自动下发 | 手改 yaml → 重启进程 |
| 设备发现 | NRF 自动注册发现 | 手动写 nodeID |
| 修改生效 | 热更新不掉线 | 重启 |

**实验手动是为了看清原理。运营商用全自动化系统，不需要人手写 yaml。**

YAML = 纯文本配置格式，用缩进表示层级，用 `-` 表示列表，用 `#` 写注释。

---

## 十六、I-UPF vs PSA-UPF 宕机分析

| | I-UPF 宕机 | PSA-UPF 宕机 |
|------|------|------|
| 路径A（I-UPF 直连） | ❌ 断 | ✅ 可能仍可用 |
| 路径B（经 PSA） | ❌ 断 | ❌ 断 |
| 路径C（Docker 内网） | ❌ 断 | ✅ 可能仍可用 |
| 新建 PDU 会话 | ❌ | ❌（SMF 找不到锚点分 IP） |
| 原因 | 所有数据必经入口 | PSA 只负责路径B + 新会话 IP |

---

## 十七、知识自测（第二轮，5 题）

### 第 1 题（★★☆☆☆）PSA-UPF 需要 N3 接口吗？

A. 需要，所有 UPF 都要有 N3
B. 需要，PSA 也要能收基站数据
C. 不需要，PSA 只通过 N9 收 I-UPF 转发的数据 ✅
D. 取决于 UE 离哪个 UPF 近

<details>
<summary>答案</summary>
C 正确。PSA-UPF 只有 N9 + N6，不直接接基站。gNB 的 N3 只连 I-UPF。
</details>

### 第 2 题（★★★☆☆）SMF 建 ULCL 会话的正确顺序？

A. 选 PSA → 选 I-UPF → 下发 PDR/FAR → 分配 UE IP
B. 分配 UE IP → 选 PSA → 选 I-UPF → 下发 PDR/FAR
C. 选 PSA → 分配 UE IP → 选 I-UPF → 下发 PDR/FAR ✅
D. 下发 PDR/FAR → 选 PSA → 分配 UE IP → 选 I-UPF

<details>
<summary>答案</summary>
C 正确。必须先定 PSA-UPF（IP 池在它名下）→ 从池子里分 IP → 再选 I-UPF 建分流链路 → 最后下发规则。
</details>

### 第 3 题（★★★★☆）N9 隧道上，UE-1 和 UE-2 的包，外层源/目标 IP 相同吗？靠什么区分？

<details>
<summary>答案</summary>
外层源 IP 和目标 IP 完全相同（都是 I-UPF → PSA-UPF）。区分靠 TEID。
TEID 在外层壳上，UPF 扫一眼就知道是谁，不需要拆壳看内层。
</details>

### 第 4 题（★★★★☆）回包路径B，IP 在各跳是什么？

```
google.com 回包到达 PSA-UPF：
① PSA 收到裸 IP：源=google.com，目标=PSA公网IP:56789
② PSA 查 NAT 映射表 → 还原为 10.60.0.1
③ PSA 套 N9 壳：外层源=192.168.56.101，外层目标=192.168.56.101，TEID=0xB001
  内层：源=google.com，目标=10.60.0.1
④ I-UPF 拆 N9 壳，套 N3 壳：外层源=192.168.56.101，外层目标=192.168.56.102，TEID=0xA001
  内层不变
⑤ gNB 拆 N3 壳，看到内层：源=google.com，目标=10.60.0.1
⑥ gNB 通过空口发给 UE
```

</details>

### 第 5 题（★★★★★）I-UPF 宕机 vs PSA-UPF 宕机，会怎样？

<details>
<summary>答案</summary>

| | I-UPF 宕机 | PSA-UPF 宕机 |
|------|------|------|
| 所有数据流 | ❌ 全断（I-UPF 是必经入口） | 部分可用（路径A/C 不经过 PSA） |
| 新建会话 | ❌ | ❌（SMF 找不到锚点） |
| NAT 映射表 | 丢了 | I-UPF 自己的表还在 |

I-UPF 是数据面单点故障；PSA-UPF 是控制面（新会话）+ 部分数据面（路径B）的单点。

</details>

---

## 十八、名词汇总

| 缩写 | 全称 | 中文 |
|------|------|------|
| ULCL | Uplink Classifier | 上行分类器/上行分流 |
| I-UPF | Intermediate UPF | 中间 UPF |
| PSA-UPF | PDU Session Anchor UPF | PDU 会话锚点 UPF |
| PDR | Packet Detection Rule | 包检测规则 |
| FAR | Forwarding Action Rule | 转发动作规则 |
| PFCP | Packet Forwarding Control Protocol | 包转发控制协议（N4 接口） |
| GTP-U | GPRS Tunneling Protocol - User Plane | GPRS 隧道协议-用户面 |
| TEID | Tunnel Endpoint Identifier | 隧道端点标识 |
| NAT | Network Address Translation | 网络地址转换 |
| DN | Data Network | 数据网络 |
| PDU Session | Protocol Data Unit Session | 协议数据单元会话 |
| DNN | Data Network Name | 数据网络名称 |
| S-NSSAI | Single NSSAI | 单一网络切片选择辅助信息 |
| SST | Slice/Service Type | 切片/服务类型 |
| SD | Slice Differentiator | 切片区分器 |
| SMF | Session Management Function | 会话管理功能 |
| UPF | User Plane Function | 用户面功能 |
| AMF | Access and Mobility Management Function | 接入与移动性管理功能 |
| NRF | Network Repository Function | 网络存储功能 |
| UDM | Unified Data Management | 统一数据管理 |
| AUSF | Authentication Server Function | 鉴权服务功能 |
| PCF | Policy Control Function | 策略控制功能 |
| gNB | Next Generation Node B | 下一代基站 |
| UE | User Equipment | 用户设备 |
| OSS | Operations Support System | 运营支撑系统 |
| MANO | Management and Orchestration | 管理与编排 |
| YAML | YAML Ain't Markup Language | YAML 配置格式 |
| N3 | — | gNB ↔ UPF 接口 |
| N4 | — | SMF ↔ UPF 接口 |
| N6 | — | UPF ↔ DN/互联网接口 |
| N9 | — | UPF ↔ UPF 接口 |

---

## 十九、ULCL 分流实验记录

> 实验日期：2026-07-19
> 参考教程：https://github.com/s5uishida/free5gc_ueransim_ulcl_sample_config
> 环境：free5GC v4.2.2 + UERANSIM v3.2.6，Ubuntu 24.04，Go 1.26.2，gtp5g v0.9.14

### 19.1 实验环境（教程 Section 1：Overview）

教程用 5 台 VM，我的环境只有 2 台，映射关系如下：

| 教程 | 我的环境 |
|------|------|
| VM1: C-Plane, 192.168.0.141 | free5gc VM, 192.168.56.101 |
| VM2: I-UPF, 192.168.0.142 | ← 合并在同一台 VM |
| VM3: PSA-UPF, 192.168.0.143 | ← 合并在同一台 VM |
| VM4: gNodeB, 192.168.0.131 | ueransim VM, 192.168.56.102 |
| VM5: UE, 192.168.0.132 | ← 合并在同一台 VM |

教程的 PLMN = 001/01，我保持原来的 208/93 不动（WebConsole 里已注册的用户和 UERANSIM 配置都用的这个）。

UE 信息：`imsi-208930000000001`，切片 SST=1 SD=010203，DNN=internet，OP 类型，K=8baf473f2f8fd09487cccbd7097c6862。

设计目标：去 8.8.8.8 的流量走 I-UPF 直连出口，其余流量走默认路径 I-UPF→PSA-UPF→互联网。

### 19.2 配置文件改动（教程 Section 2：Changes in configuration files）

教程分了四组配置：C-Plane、I-UPF、PSA-UPF、UERANSIM。我的 C-Plane 和 U-Plane 都在同一台 VM，PLMN 保持不变，所以 AMF/AUSF/NRF/NSSF 这四个 C-Plane 配置文件不用动（教程改它们只是为了换 PLMN 和 IP，我的本来就对）。

UERANSIM 的 gNB 和 UE 配置也不用动，之前已经配好了。

真正需要改的是这四个：SMF、UPF、uerouting、run.sh。

---

#### 19.2.1 smfcfg.yaml（对应教程 Section 2-A：C-Plane 的 smfcfg.yaml）

教程的核心改动：把原来一个 UPF 节点拆成 I-UPF 和 PSA-UPF 两个，加 links 拓扑，末尾加 `ulcl: true`。

**PLMN**：保持 208/93，不改。

**PFCP**：保持 127.0.0.1，不改（两个 UPF 都在同一台机器，SMF 走 loopback 跟它们通信没问题）。

**upNodes**：这是改动最大的地方。原来的单 UPF：

```yaml
UPF:
  type: UPF
  nodeID: 127.0.0.8
  addr: 127.0.0.8
  interfaces:
    - interfaceType: N3
      endpoints: [192.168.56.101]
```

教程要求 I-UPF 有 N3+N9，PSA-UPF 只有 N9，PSA-UPF 带 IP 池。但如果照搬教程给两个节点不同的 nodeID，SMF 会分别发起两次 PFCP 关联，而第二个 UPF 进程因为 gtp5g 冲突根本启不来（见 19.4.2）。

尝试过程：先给 I-UPF 用 192.168.56.101，PSA-UPF 用 127.0.0.8，并为 PSA-UPF 单独写了 upf2cfg.yaml，还加了不同的 GTP-U 设备名（ifname: upfgtp2）。第二个 UPF 进程始终报 `open Gtp5g: open link: create: file exists`，PFCP 永远连不上，SMF 不断打 ALERT for UPF[127.0.0.8]。

最终的做法：两个节点用**相同的 nodeID**，都指向 192.168.56.101。SMF 对这个 nodeID 只建一个 PFCP 会话，但在这个会话里把 I-UPF 和 PSA-UPF 两套 PDR/FAR 全发过去。UPF 按规则 ID 各干各的，N9 走本机内核 loopback，靠 TEID 区分。

```yaml
userplaneInformation:
  upNodes:
    gNB1:
      type: AN
    I-UPF:
      type: UPF
      nodeID: 192.168.56.101
      addr: 192.168.56.101
      sNssaiUpfInfos:
        - sNssai: { sst: 1, sd: 010203 }
          dnnUpfInfoList:
            - dnn: internet
        - sNssai: { sst: 1, sd: 112233 }
          dnnUpfInfoList:
            - dnn: internet
      interfaces:
        - interfaceType: N3
          endpoints: [192.168.56.101]
          networkInstances: [internet]
        - interfaceType: N9
          endpoints: [192.168.56.101]
          networkInstances: [internet]
    PSA-UPF:
      type: UPF
      nodeID: 192.168.56.101          # 跟 I-UPF 一样
      addr: 192.168.56.101
      sNssaiUpfInfos:
        - sNssai: { sst: 1, sd: 010203 }
          dnnUpfInfoList:
            - dnn: internet
              pools:
                - cidr: 10.60.0.0/16  # IP 池挂在 PSA-UPF
              staticPools:
                - cidr: 10.60.100.0/24
        - sNssai: { sst: 1, sd: 112233 }
          dnnUpfInfoList:
            - dnn: internet
              pools:
                - cidr: 10.61.0.0/16
              staticPools:
                - cidr: 10.61.100.0/24
      interfaces:
        - interfaceType: N9          # PSA-UPF 只有 N9，没有 N3
          endpoints: [192.168.56.101]
          networkInstances: [internet]
  links:
    - A: gNB1
      B: I-UPF
    - A: I-UPF
      B: PSA-UPF
ulcl: true
```

节点名（I-UPF / PSA-UPF）必须跟 uerouting.yaml 里的一致，不然拓扑对不上。

---

#### 19.2.2 upfcfg.yaml（对应教程 Section 2-B：I-UPF 的 upfcfg.yaml）

教程里 I-UPF 和 PSA-UPF 各有一个 upfcfg.yaml，跑在不同 VM 上。在我这合并成一个——这一个 UPF 进程同时处理 N3 和 N9。

原来只有 N3，没有 natifname（做了 NAT 的 DNN 是 10.61.0.0/16 那个）。改动：加 N9 接口，主 DNN（10.60.0.0/16）加 natifname，PFCP 从 127.0.0.8 改成真实 IP。

```yaml
pfcp:
  addr: 192.168.56.101
  nodeID: 192.168.56.101

gtpu:
  forwarder: gtp5g
  ifList:
    - addr: 192.168.56.101
      type: N3
    - addr: 192.168.56.101     # 新增 N9
      type: N9

dnnList:
  - dnn: internet
    cidr: 10.60.0.0/16
    natifname: enp0s3           # 新增：N6 出口做 NAT
```

---

#### 19.2.3 uerouting.yaml（对应教程 Section 2-A：uerouting.yaml）

这是 SMF 启动时通过 `-u` 参数加载的分流规则文件。教程里叫 uerouting.yaml，是 ULCL 专用配置。

直接用 free5GC 自带的 multiUPF 目录里的模板改的，节点名换成 SMF 里定义的名字：

```yaml
ueRoutingInfo:
  UE1:
    members:
    - imsi-208930000000001
    topology:
      - A: gNB1
        B: I-UPF
      - A: I-UPF
        B: PSA-UPF
    specificPath:
      - dest: 8.8.8.8/32
        path: [I-UPF]           # 匹配这条的流量只到 I-UPF 就出口
```

topology 跟 SMF 的 links 一致，决定默认路径。specificPath 是分流规则——匹配目标 IP 的流量按 path 走，不匹配的走默认。

---

#### 19.2.4 UERANSIM 配置（对应教程 Section 2-D）

教程改了 PLMN（208→001）和 IP（127.0.0.1→真实 IP），我之前已经配好 208/93 和真实 IP 了，不需要改。

gNB 配置确认：ngapIp/gtpIp=192.168.56.102，AMF=192.168.56.101:38412，切片 SST=0x1 SD=0x010203。

UE 配置确认：supi=imsi-208930000000001，gnbSearchList=127.0.0.1（同 VM 上搜 gNB），切片同 gNB。

### 19.3 网络转发（教程 Section 3：Network settings）

教程要求每台 UPF VM 上开 ip_forward 和配置 iptables NAT。I-UPF 和 PSA-UPF 各做各的 MASQUERADE。

我的环境 I-UPF 和 PSA-UPF 在同一台 VM，只需要配一次：

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -I FORWARD 1 -j ACCEPT
```

每次 VM 重启后 iptables 会丢，需要重跑。free5GC 提供了 `~/free5gc/reload_host_config.sh enp0s3` 一键执行，但这次实验发现里面少了一条 FORWARD 规则，手动补上了。

**踩坑**：这一步忘了做的话，UE 能注册、能拿到 IP，但 ping 任何地址都是 100% 丢包。

### 19.4 启动（教程 Section 5：Run free5GC 5GC and UERANSIM UE / RAN）

#### 19.4.1 修改 run.sh（对应教程 Step 2：Start C-Plane）

教程的启动脚本里 SMF 那行是：

```bash
./bin/smf -c config/smfcfg.yaml -u config/uerouting.yaml &
```

关键在 `-u config/uerouting.yaml`。原来的 run.sh 把 smf 放在 NF_LIST 循环里统一启动，只传了 `-c`，没传 `-u`。

改法：NF_LIST 里去掉了 smf 关键字，在循环结束之后单独启动 SMF：

```bash
NF_LIST="nrf amf udr pcf udm nssf ausf chf nef"   # 原为 "nrf amf smf udr ..."

# ... NF_LIST 循环 ...

# ULCL: SMF 单独启动，加载 uerouting
./bin/smf -c ./config/smfcfg.yaml -u ./config/uerouting.yaml -l ${LOG_PATH}${LOG_NAME} &
```

UPF 部分保持原样，只启一个实例（原来的 `upf2cfg.yaml` 不再使用）。

#### 19.4.2 两 UPF 方案的尝试和放弃

教程分别在 VM2 和 VM3 上各启一个 UPF。我先尝试照搬：写了一个独立的 upf2cfg.yaml（PSA-UPF，PFCP 用 127.0.0.8，GTP-U 也用 127.0.0.8 避免跟 I-UPF 的 192.168.56.101:2152 冲突），run.sh 里加了一行启动。

实际结果：第二个 UPF 进程启动时报 `open Gtp5g: open link: create: file exists`，PFCP 服务根本起不来，SMF 那边持续报 ALERT。给两个 UPF 设了不同的 GTP-U 设备名（ifname: upfgtp / upfgtp2）也不管用，因为 gtp5g 内核模块只允许一个进程打开 netlink socket。

所以最终的方案就是上面说的"单 UPF 双角色"——靠配置而非多进程来区分 I-UPF 和 PSA-UPF。这不是教程的标准做法，但面对 2 VM 的现实约束，这是能跑通的方案。

#### 19.4.3 启动 UERANSIM（对应教程 Step 3 & 4）

```bash
# 终端1：gNB
cd ~/UERANSIM && build/nr-gnb -c config/free5gc-gnb.yaml

# 终端2：UE
cd ~/UERANSIM && sudo build/nr-ue -c config/free5gc-ue.yaml
```

UE 需要 sudo 因为它要创建 uesimtun0 虚拟网卡。多次测试时注意先用 `sudo pkill -9 nr-ue; sudo pkill -9 nr-gnb` 清理残留进程，否则 2152 和 4997 端口被占着，新进程起不来。

#### 19.4.4 gtp5g 规则残留问题

PDU Session Establishment 阶段报了 `CreateFAR[0x31] failed: file exists`。原因是前一次跑的 gtp5g 规则没清干净。

解决：停 free5GC，卸载然后重载 gtp5g 内核模块，再启动。

```bash
sudo rmmod gtp5g && sudo modprobe gtp5g
```

#### 19.4.5 完整启动顺序

总结一下正确的启动顺序：

```
1. free5gc VM: sudo sysctl/iptables（开转发 + NAT）
2. free5gc VM: sudo rmmod gtp5g && sudo modprobe gtp5g（清残留）
3. free5gc VM: cd ~/free5gc && ./run.sh
4. ueransim VM: sudo pkill -9 nr-ue; sudo pkill -9 nr-gnb（清残留）
5. ueransim VM: 终端1 ./nr-gnb, 终端2 sudo ./nr-ue
```

### 19.5 验证（教程 Section 5 Step 5：Ping test）

UE 注册成功后的输出：

```
Initial Registration is successful
PDU Session establishment is successful PSI[1]
TUN interface[uesimtun0, 10.60.0.1] is up
```

SMF 日志确认 ULCL 分流规则已下发：

```
ue routing config Info: Version[1.0.7] Description[ULCL Routing Information for UE]
Has default path
Has pre-config ULCL paths
Add PSAAndULCL
[SMF] Establish ULCL msg has been send
```

ping 测试，两条路径都对：

```
$ ping 8.8.8.8 -I uesimtun0
3 packets transmitted, 3 received, 0% loss           ← I-UPF 直连出口

$ ping google.com -I uesimtun0
3 packets transmitted, 3 received, 0% loss           ← PSA-UPF 默认路径
```

### 19.6 小结

要点：

1. 教程是 5 VM 部署，2 VM 环境下需要把 I-UPF 和 PSA-UPF 合并到一个 UPF 进程里，通过给 SMF 的两个 UPF 节点配相同 nodeID 来实现。gtp5g 内核模块不允许同一台机器上跑两个 UPF 进程，这是这次实验最关键的发现。

2. SMF 启动必须带 `-u config/uerouting.yaml`，否则 ulcl: true 不生效——因为 SMF 不知道往哪个 UPF 发什么规则。

3. VM 重启后有两样东西会丢：iptables NAT 规则和 gtp5g 上一次的 PDR/FAR 残留。前者导致 ping 不通，后者导致 PDU 会话建不起来。每次完整跑实验前先清一遍。

4. 这个方案的功能是完整的——SMF 正确下发了 ULCL 分流规则，两条路径都能通。局限是 N9 隧道走了本机 loopback 而非真正的跨机 GTP-U，如果要验证跨机的场景还得加 VM 或用 Docker 网络命名空间。不过对于理解 ULCL 的配置流程和验证分流逻辑来说够用了。
