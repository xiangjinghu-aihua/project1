# free5GC 5G 核心网 — 环境搭建与知识总结

## 一、名词解释

### 1.1 5G 核心网各网元 (Network Functions)

| 缩写 | 英文全称 | 中文 | 大白话解释 |
|------|----------|------|-----------|
| **NRF** | Network Repository Function | 网络存储功能 | 公司内部通讯录。每个部门启动后都在这里登记，别的部门需要找人时来这查。所有 NF 启动后向 NRF 注册自己 |
| **AMF** | Access and Mobility Management Function | 接入与移动性管理 | 大楼前台接待。手机要接入 5G，先找它登记、验身份、分配临时 ID。手机移动时它负责跟踪位置 |
| **SMF** | Session Management Function | 会话管理 | 宽带客服。你要上网时帮你开通一条通道（PDU 会话），分配 IP 地址，选一个合适的 UPF 转发数据 |
| **UPF** | User Plane Function | 用户面功能 | 网络路由器/网关。你所有上网数据（刷视频、发消息）都经它转发到公网。执行 QoS、计费上报、NAT |
| **AUSF** | Authentication Server Function | 鉴权服务 | 门禁系统。手机接入时验证你是不是合法用户，执行 5G AKA 鉴权算法 |
| **UDM** | Unified Data Management | 统一数据管理 | 用户档案室。存储 SIM 卡信息、鉴权密钥、套餐数据 |
| **UDR** | Unified Data Repository | 统一数据存储 | 数据库仓库。实际存放用户签约数据、策略数据的地方 |
| **PCF** | Policy Control Function | 策略控制 | 公司规章制度。下发 QoS 策略、接入控制策略给 AMF/SMF |
| **NSSF** | Network Slice Selection Function | 网络切片选择 | 分诊台。根据手机要的服务类型（普通上网/工业低延迟），挑合适的网络切片 |
| **CHF** | Charging Function | 计费功能 | 收银台。记录流量、通话时长，在线/离线计费 |
| **BSF** | Binding Support Function | 绑定支持 | 中介登记处。记录哪个 UE 绑定了哪个 PCF，用于策略关联 |
| **NEF** | Network Exposure Function | 网络开放功能 | 对外窗口。让第三方应用能调用 5G 核心网的能力（如设备触发、QoS 请求） |
| **N3IWF** | Non-3GPP InterWorking Function | 非3GPP互通 | WiFi 接入网关。让手机通过 WiFi 也能接入 5G 核心网，用 IPSec 隧道加密 |
| **TNGF** | Trusted Non-3GPP Gateway Function | 可信非3GPP网关 | 可信 WiFi 网关。企业自有 WiFi 接入 5GC 的入口，使用 EAP 认证 |

### 1.2 通信接口

| 缩写 | 连接 | 协议 | 含义 |
|------|------|------|------|
| **N2** | gNB ↔ AMF | NGAP/SCTP | 控制信令。基站和 AMF 之间传递注册、切换、寻呼等指令 |
| **N3** | gNB ↔ UPF | GTP-U | 用户数据隧道。你的视频、网页流量走这条路从基站进核心网 |
| **N4** | SMF ↔ UPF | PFCP | SMF 给 UPF 下发转发规则（PDR/FAR：怎么匹配、往哪转发） |
| **N6** | UPF ↔ DN/互联网 | 纯 IP | UPF 把数据送到公网的最后一段 |
| **N9** | UPF ↔ UPF | GTP-U | 两个 UPF 之间的隧道（ULCL/分流场景） |
| **SBI** | NF ↔ NF | HTTP/2 | 控制面各网元之间互相调用的接口 |
| **Nwu** | UE ↔ N3IWF/TNGF | IPSec/IKEv2 | 非3GPP 接入的安全隧道接口 |

### 1.3 UERANSIM / 终端相关

| 缩写 | 全称 | 含义 |
|------|------|------|
| **UE** | User Equipment | 终端设备（手机）。实验中 nr-ue 模拟它 |
| **gNB** | gNodeB | 5G 基站。实验中 nr-gnb 模拟它 |
| **SUPI** | Subscription Permanent Identifier | 永久用户标识。相当于 SIM 卡底层 ID，格式 imsi-208930000000001 |
| **SUCI** | Subscription Concealed Identifier | 加密后的 SUPI。保护隐私，防止空中接口被窃听 |
| **GUTI** | Globally Unique Temporary Identifier | 临时分配的假名。AMF 给 UE 的临时 ID，避免频繁暴露 SUPI |
| **IMSI** | International Mobile Subscriber Identity | 国际移动用户标识。15 位数字，= MCC + MNC + MSIN |
| **MCC** | Mobile Country Code | 国家码。208 = 法国（测试用） |
| **MNC** | Mobile Network Code | 运营商码。93 = 测试网络 |
| **S-NSSAI** | Single Network Slice Selection Assistance Information | 切片标识。SST(切片类型) + SD(切片区分器)，用于选哪张切片 |
| **DNN** | Data Network Name | 数据网络名。相当于 APN，标识要接入哪个外部网络（如 internet） |
| **PDU Session** | Protocol Data Unit Session | 手机和互联网之间的一条数据通道。建立后手机才拿到 IP |

### 1.4 鉴权相关

| 缩写 | 全称 | 含义 |
|------|------|------|
| **K / Key** | Permanent Key | 永久密钥。存在 SIM 卡和 UDM 里的共享秘密，鉴权的基础 |
| **OP** | Operator Code | 运营商原始密钥。和 K 一样存储在 SIM 和网络侧 |
| **OPc** | Operator Code (derived) | 从 OP 和 K 通过 AES 计算出的加密结果 |
| **SQN** | Sequence Number | 序列号。防重放攻击的关键参数 |
| **AMF** (鉴权) | Authentication Management Field | 鉴权管理字段。8000 表示使用 5G AKA |

### 1.5 网络相关

| 缩写 | 全称 | 含义 |
|------|------|------|
| **NAT** | Network Address Translation | 网络地址转换。把内部 IP（10.60.0.x）转成公网 IP，让 UE 能上网 |
| **MASQUERADE** | iptables MASQUERADE | Linux 动态 NAT。自动把源 IP 换成出口网卡的 IP |
| **GTP-U** | GPRS Tunneling Protocol - User Plane | 用户面隧道协议。把 UE 的 IP 包封装进 UDP，通过 N3 传输 |
| **PFCP** | Packet Forwarding Control Protocol | N4 接口协议。SMF 用它告诉 UPF "这个 UE 的数据该怎么转发" |
| **PDR** | Packet Detection Rule | 包检测规则。告诉 UPF 怎么识别属于某个 UE 的数据包 |
| **FAR** | Forwarding Action Rule | 转发动作规则。告诉 UPF 匹配后往哪个接口送 |
| **SCTP** | Stream Control Transmission Protocol | N2 接口传输层协议。比 TCP 更适合信令传输，支持多流 |
| **iptables** | — | Linux 防火墙工具。实验中用它配 NAT |
| **Host-Only** | VirtualBox Host-Only Network | 虚拟机之间内部通信的虚拟网卡 |
| **IPSec** | Internet Protocol Security | N3IWF 使用的加密隧道协议，保护非3GPP 接入的安全 |
| **IKEv2** | Internet Key Exchange v2 | IPSec 的密钥协商协议 |
| **EAP** | Extensible Authentication Protocol | 可扩展认证协议，TNGF 使用 EAP-TLS/EAP-AKA 进行 WiFi 认证 |
| **RADIUS** | Remote Authentication Dial-In User Service | TNGF 中用于 WiFi AP 认证的协议 |
| **DNAI** | Data Network Access Identifier | 数据网络接入标识，用于流量导向中标识 UPF 的 N6 出口位置 |
| **ULCL** | Uplink Classifier | 上行分流器 UPF，根据流规则把不同流量导向不同出口 |

---

## 二、网元关系与信令/数据流

### 2.1 整体架构总览

```
                                    ┌───────────┐
                                    │  互联网    │
                                    │ 8.8.8.8   │
                                    └─────┬─────┘
                                          │ N6 (纯IP, NAT)
                                          │
┌─────────────────────────────────────────┼──────────────────────────────────────────┐
│                      free5GC 核心网 VM (192.168.56.101)                             │
│                                         │                                           │
│  ╔══════════════════════════════════════════════════════════════════════╗         │
│  ║                         控制面 (Control Plane)                       ║         │
│  ║                                                                      ║         │
│  ║    ┌──────────┐       ┌──────────┐       ┌──────────┐               ║         │
│  ║    │  NSSF    │       │   NRF    │       │   BSF    │               ║         │
│  ║    │ (切片)   │       │ (注册中心)│       │ (PCF绑定)│               ║         │
│  ║    └──────────┘       └────┬─────┘       └──────────┘               ║         │
│  ║                            │                                         ║         │
│  ║           ┌────────────────┼──────────────────┐                     ║         │
│  ║           │                │                  │                     ║         │
│  ║    ┌──────▼──────┐  ┌──────▼──────┐    ┌──────▼──────┐              ║         │
│  ║    │     AMF     │  │     PCF     │    │    CHF      │              ║         │
│  ║    │ (接入/移动) │  │ (策略控制)  │    │   (计费)    │              ║         │
│  ║    └──────┬──────┘  └─────────────┘    └─────────────┘              ║         │
│  ║           │                                                         ║         │
│  ║    ┌──────┼──────────────────┐                                      ║         │
│  ║    │      │                  │                                      ║         │
│  ║    ▼      ▼                  ▼                                      ║         │
│  ║  ┌──────┐ ┌──────┐      ┌──────┐       ┌──────────┐                ║         │
│  ║  │AUSF  │ │ UDM  │      │ SMF  │       │   NEF    │                ║         │
│  ║  │(鉴权)│ │(用户)│      │(会话)│       │ (开放)   │                ║         │
│  ║  └──────┘ └──┬───┘      └──┬───┘       └──────────┘                ║         │
│  ║              │             │                                        ║         │
│  ║         ┌────▼────┐        │ N4 (PFCP 下发PDR/FAR规则)              ║         │
│  ║         │   UDR   │        │                                        ║         │
│  ║         │(数据库) │        │                                        ║         │
│  ║         └─────────┘        │                                        ║         │
│  ╚════════════════════════════╪═══════════════════════════════════════╝         │
│                               │                                                 │
│  ╔════════════════════════════╪═══════════════════════════════════════╗         │
│  ║                      用户面 (User Plane)                          ║         │
│  ║                                                                   ║         │
│  ║                    ┌──────────────┐                               ║         │
│  ║                    │     UPF      │                               ║         │
│  ║                    │ (数据转发    │                               ║         │
│  ║                    │  GTP-U解封   │                               ║         │
│  ║                    │  NAT / 计费  │                               ║         │
│  ║                    └──────┬───────┘                               ║         │
│  ╚══════════════════════════╪════════════════════════════════════════╝         │
│                             │ N3 (GTP-U 隧道)                                   │
│                     ┌───────┴───────┐                                           │
│                     │    N3IWF      │ ← WiFi/非3GPP 接入 (Nwu IPSec)            │
│                     │  (非3GPP网关) │                                           │
│                     └───────────────┘                                           │
└─────────────────────────────┼────────────────────────────────────────────────────┘
                              │
                              │ Host-Only 网络 (192.168.56.x)
                              │
┌─────────────────────────────┼────────────────────────────────────────────────────┐
│                 UERANSIM VM (192.168.56.102)                                      │
│                             │                                                     │
│                      ┌──────▼──────┐                                              │
│                      │     gNB     │  5G 基站                                    │
│                      │ (nr-gnb)    │                                              │
│                      └──────┬──────┘                                              │
│                             │                                                     │
│                      ┌──────▼──────┐                                              │
│                      │     UE      │  终端                                       │
│                      │  (nr-ue)    │                                              │
│                      │ 10.60.0.x   │  ← uesimtun0 隧道接口                        │
│                      └─────────────┘                                              │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 UE 注册流程（控制面信令）

```
  UE              gNB              AMF             AUSF/UDM          NRF
  │                │                │                 │                │
  │ ① Registration │                │                 │                │
  │   Request      │                │                 │                │
  ├───────────────►│  ② Initial UE  │                 │                │
  │                │    Message     │                 │                │
  │                ├───────────────►│                 │                │
  │                │                │ ③ 查UDM获取     │                │
  │                │                │   鉴权向量       │                │
  │                │                ├────────────────►│                │
  │                │                │◄────────────────┤                │
  │                │                │                 │                │
  │ ④ Authentication Request       │                 │                │
  │◄───────────────┤◄───────────────┤                 │                │
  │                │                │                 │                │
  │ ⑤ Authentication Response      │                 │                │
  ├───────────────►├───────────────►│                 │                │
  │                │                │ ⑥ 验证RES=HXRES*│                │
  │                │                │                 │                │
  │ ⑦ NAS SMC                      │                 │                │
  │◄───────────────┤◄───────────────┤                 │                │
  │                │                │                 │                │
  │ ⑧ Registration Accept (含GUTI) │ ⑨ 注册AMF信息到NRF              │
  │◄───────────────┤◄───────────────┤─────────────────────────────────►│
  │                │                │                                  │
  └────────────────┴────────────────┴──────────────────────────────────┘
```

### 2.3 PDU 会话建立流程

```
  UE              AMF              SMF              PCF/UDR           UPF
  │                │                │                 │                │
  │ ① PDU Session Establishment Request           │                 │
  ├───────────────►├───────────────►│                 │                │
  │                │                │ ② 查PCF策略     │                │
  │                │                ├────────────────►│                │
  │                │                │◄────────────────┤                │
  │                │                │ ③ 选UPF         │                │
  │                │                │                 │                │
  │                │                │ ④ N4 Session Establishment     │
  │                │                │   (下发PDR/FAR/URR规则)         │
  │                │                ├─────────────────────────────────►
  │                │                │◄─────────────────────────────────┤
  │                │                │                 │                │
  │                │                │ ⑤ N1N2 Message  │                │
  │                │◄───────────────┤  Transfer       │                │
  │                │                │                 │                │
  │ ⑥ AN-specific  │                │                 │                │
  │   Resource     │                │                 │                │
  │   Setup (包含  │                │                 │                │
  │   N3 隧道信息)  │                │                 │                │
  │◄───────────────┤                │                 │                │
  │                │                │                 │                │
  │ ⑦ PDU Session Establishment Accept              │                │
  │◄───────────────┤◄───────────────┤                 │                │
  │                │                │                 │                │
  │   uesimtun0 接口出现, UE 获得 IP 10.60.0.x       │                │
  └────────────────┴────────────────┴─────────────────┴────────────────┘
```

### 2.4 用户数据传输路径（ping 的旅程）

```
  UE (10.60.0.x)            gNB (192.168.56.102)        UPF (192.168.56.101)    互联网 (8.8.8.8)
       │                           │                            │                       │
       │ ① ICMP echo request       │                            │                       │
       │   源:10.60.0.x            │                            │                       │
       │   目的:8.8.8.8            │                            │                       │
       ├──────────────────────────►│                            │                       │
       │  (走 uesimtun0 接口)      │                            │                       │
       │                           │ ② GTP-U 隧道封装            │                       │
       │                           │   外层: UDP port 2152      │                       │
       │                           │   内层: 原始 ICMP 包       │                       │
       │                           ├───────────────────────────►│                       │
       │                           │                            │ ③ 去 GTP-U 封装        │
       │                           │                            │   PDR 识别 → FAR 转发   │
       │                           │                            │                        │
       │                           │                            │ ④ NAT MASQUERADE      │
       │                           │                            │   源IP→10.0.2.15       │
       │                           │                            ├───────────────────────►│
       │                           │                            │   ⑤ 8.8.8.8 回复       │
       │                           │                            │◄───────────────────────┤
       │                           │                            │   ⑥ 查 FAR → GTP-U封装  │
       │                           │ ⑦ GTP-U 隧道 (回复)        │   源:192.168.56.101     │
       │                           │◄───────────────────────────┤   目的:192.168.56.102    │
       │ ⑧ ICMP echo reply        │                            │                        │
       │◄──────────────────────────┤                            │                        │
       │                           │                            │                        │
       └───────────────────────────┴────────────────────────────┴────────────────────────┘
```

---

## 三、本地部署全过程（基于 free5GC 官方 Advanced 指南）

> 官方指南地址：https://free5gc.org/guide/
> 本教程对应 Advanced 部分 "Build free5GC from Scratch"，从零开始构建 5G 核心网。
> 官方推荐普通用户使用 Docker Compose 方式，但从源码构建能让你理解每个组件的作用。

### 环境概览

| 虚拟机 | 角色 | IP (Host-Only) | IP (NAT) | 用途 |
|--------|------|----------------|----------|------|
| **free5gc** | 5G 核心网 | 192.168.56.101 | 10.0.2.15 | 运行所有 5G 核心网网元 |
| **ueransim** | 仿真基站+终端 | 192.168.56.102 | 10.0.2.15 | 模拟 5G 基站和手机 |
| **n3iwue** (可选) | 非3GPP终端 | 192.168.56.103 | 10.0.2.15 | 模拟通过 WiFi 接入 5G 的终端 |

每台 VM 需要两张网卡：
- **NAT 网卡 (enp0s3)**：用于 VM 访问互联网
- **Host-Only 网卡 (enp0s8)**：用于 VM 之间互相通信

---

### 第 1 步：创建 Ubuntu 虚拟机（VirtualBox）

#### 1.1 安装 VirtualBox

从 https://www.virtualbox.org/ 下载并安装 VirtualBox（当前 7.0.12+）。

#### 1.2 下载 Ubuntu Server

从 https://ubuntu.com/download/server 下载 **Ubuntu Server 20.04 LTS** ISO 文件（如 `ubuntu-20.04.6-live-server-amd64.iso`）。

> 为什么选择 20.04？因为 free5GC 的 gtp5g 内核模块在 5.4.x 内核上测试最充分。如果使用 22.04/24.04，内核 6.x 上可能需要额外配置（详见踩坑记录）。

#### 1.3 创建 Ubuntu 基础 VM

1. VirtualBox 点击 **New** → **Expert Mode**
2. 名称：`ubuntu`，类型：Linux，版本：Ubuntu (64-bit)
3. 分配 **2 CPU**、**2048MB 内存**
4. 添加 **两张网卡**：
   - 网卡1：NAT（默认，用于上网，IP 为 10.0.2.15）
   - 网卡2：Host-Only Adapter（虚拟机间通信）

#### 1.4 安装 Ubuntu

- 选择 **Minimal Installation**（最小化安装）
- **不要选 LVM**（方便后续扩展磁盘）
- **选择 Install SSH Server**（后续远程操作）
- 设短用户名和密码（方便输入，如 `ubuntu` / `free5gc`）

#### 1.5 登录并配置网络

```bash
# 安装 ifconfig 工具
sudo apt install net-tools

# 检查网络接口
ifconfig
# enp0s3: NAT 网卡，IP 10.0.2.15
# enp0s8: Host-Only 网卡，IP 192.168.56.101（DHCP分配）

# 测试上网
ping google.com

# 更新系统
sudo apt update && sudo apt upgrade
```

#### 1.6 用 SSH 连接 VM

从宿主机 SSH 到 VM：
```bash
ssh 192.168.56.101 -l ubuntu
```

首次连接会提示确认，输入 `yes`。以后所有操作都可以通过 SSH 进行，方便复制粘贴命令。

---

### 第 2 步：创建并配置 free5GC 虚拟机

#### 2.1 克隆 VM

1. 关闭基础 VM：`sudo shutdown -P now`
2. VirtualBox 选中基础 VM → **Snapshots** → **Clone**
3. 命名 `free5gc`，选择 **创建新 MAC 地址**，使用 **链接克隆**（节省磁盘空间）
4. 同样方式再克隆一台 `ueransim`（后续第5步用）

#### 2.2 修改主机名

```bash
# 在 free5gc VM 上
sudo nano /etc/hostname
# 把 ubuntu 改成 free5gc

sudo nano /etc/hosts
# 把 127.0.1.1 ubuntu 改成 127.0.1.1 free5gc
```

重启后生效。

#### 2.3 设置静态 IP

free5GC 需要固定的 IP 地址给 gNB 连接，改用静态 IP 更可靠：

```bash
cd /etc/netplan
sudo nano 00-installer-config.yaml
```

修改为：
```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true          # NAT 网卡保持 DHCP
    enp0s8:
      dhcp4: no            # Host-Only 使用静态 IP
      addresses: [192.168.56.101/24]
  version: 2
```

应用配置：
```bash
sudo netplan try   # 验证配置
sudo netplan apply # 应用配置
```

用 `ifconfig` 确认 `enp0s8` 已变为 `192.168.56.101`。

**同样操作对 ueransim VM：**
- 主机名改为 `ueransim`
- 静态 IP 设为 `192.168.56.102`

**验证互通：**
```bash
# 从 ueransim ping free5gc
ping 192.168.56.101

# 从 free5gc ping ueransim
ping 192.168.56.102
```

---

### 第 3 步：构建并安装 free5GC

此步骤在 **free5gc VM** 上进行。

#### 3.1 前置要求检查

```bash
# 检查内核版本（必须是 5.0.0-23-generic 或 5.4.x）
uname -r
# 期望输出：5.4.0-xx-generic

# 检查 CPU 是否支持 AVX（MongoDB 5.0+ 需要）
lscpu | grep avx
```

#### 3.2 安装 Go 语言环境

free5GC v4.2.2 需要 Go 1.26.2：

```bash
wget https://dl.google.com/go/go1.26.2.linux-amd64.tar.gz
sudo tar -C /usr/local -zxvf go1.26.2.linux-amd64.tar.gz
mkdir -p ~/go/{bin,pkg,src}

# 设置环境变量
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin:$GOROOT/bin' >> ~/.bashrc
echo 'export GO111MODULE=auto' >> ~/.bashrc
source ~/.bashrc

# 验证
go version
```

**重要：** 为 make 等非交互 shell 创建系统级符号链接：
```bash
sudo ln -sf /usr/local/go/bin/go /usr/local/bin/go
```

#### 3.3 安装控制面依赖

```bash
sudo apt update
sudo apt install -y wget git gcc g++ cmake autoconf libtool \
    pkg-config libmnl-dev libyaml-dev gnupg curl
```

#### 3.4 安装 MongoDB

```bash
# 导入 MongoDB 公钥
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
    sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

# 根据 Ubuntu 版本选择源（用 cat /etc/lsb-release 查看）
# Ubuntu 20.04 (Focal):
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/8.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.2.list
# Ubuntu 22.04 (Jammy):
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.2.list
# Ubuntu 24.04 (Noble):
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.2.list

sudo apt update && sudo apt install -y mongodb-org

# 启动 MongoDB
sudo systemctl start mongod
sudo systemctl enable mongod
```

> 注意：如果 CPU 不支持 AVX，需要降级到 MongoDB 4.4 或使用 Ubuntu 仓库的 `sudo apt install mongodb`（v3.6.8）。

#### 3.5 安装 Node.js（WebConsole 需要）

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
corepack enable   # 启用 yarn
```

#### 3.6 克隆并编译 free5GC

```bash
cd ~
git clone --recursive -b v4.2.2 -j $(nproc) https://github.com/free5gc/free5gc.git
cd free5gc

# 编译所有网络功能（控制面网元）
make

# 编译 WebConsole
make webconsole
```

#### 3.7 安装 gtp5g 内核模块（UPF 用户面转发核心）

```bash
cd ~/free5gc
git clone -b v0.9.14 https://github.com/free5gc/gtp5g.git
cd gtp5g
make
sudo make install

# 加载内核模块
sudo modprobe gtp5g
lsmod | grep gtp5g  # 验证模块已加载
```

---

### 第 4 步：测试 free5GC（集成测试）

#### 4.1 运行集成测试

free5GC 自带 11 个集成测试，验证各网元基本功能：

```bash
cd ~/free5gc
make upf           # 确保 UPF 已编译
chmod +x ./test.sh
```

| 测试编号 | 测试名称 | 测试内容 |
|----------|----------|----------|
| a | **TestRegistration** | UE 注册流程：验证初始注册、鉴权、安全模式控制 |
| b | **TestGUTIRegistration** | GUTI 注册：用临时 ID 重新注册，无需重新鉴权 |
| c | **TestServiceRequest** | 服务请求：UE 从空闲态回到连接态 |
| d | **TestXnHandover** | Xn 切换：两个 gNB 之间的切换（Xn 接口） |
| e | **TestDeregistration** | 去注册流程：UE 主动断开会话并发起去注册 |
| f | **TestPDUSessionReleaseRequest** | PDU 会话释放：UE 请求释放数据通道 |
| g | **TestPaging** | 寻呼：网络侧有下行数据时呼叫空闲态 UE |
| h | **TestN2Handover** | N2 切换：AMF 参与的两个 gNB 间切换 |
| i | **TestNon3GPP** | 非3GPP 接入：测试 N3IWF/IPSec 隧道建立 |
| j | **TestReSynchronization** | SQN 重同步：SQN 不同步时的自动修复 |
| k | **TestULCL** | 上行分流：多 UPF 分流（ULCL 场景） |

运行方式：
```bash
./test.sh TestRegistration       # 单个测试
./test.sh TestGUTIRegistration
# ... 依次运行
./test_ulcl.sh TestRequestTwoPDUSessions  # ULCL 测试用专用脚本
```

> 说明：TestNon3GPP 在没有 N3IWF 的 VM 环境中会失败，这是预期行为。其他 10 个测试都应通过。

#### 4.2 通过 WebConsole 添加用户

```bash
cd ~/free5gc/webconsole
./bin/webconsole
```

浏览器访问 `http://192.168.56.101:5000`，登录 `admin` / `free5gc`：

1. 左侧点击 **Subscribers** → **New Subscriber**
2. **Operator Code Type** 从 `OPc` 改为 **`OP`**
3. 记下以下三个关键值（后续 UERANSIM 配置要用）：
   - **SUPI (IMSI)**：`imsi-208930000000003`
   - **Key (K)**：`8baf473f2f8fd09487cccbd7097c6862`
   - **OP**：`8e27b6af0e692e750f32667a3b14605d`
4. 滚动到底部点击 **Submit**
5. 提交成功后 `Ctrl-C` 关闭 WebConsole

---

### 第 5 步：安装 UERANSIM（UE/RAN 仿真器）

此步骤在 **ueransim VM (192.168.56.102)** 上进行。

> UERANSIM 是一个开源的 5G UE/RAN 仿真器，可以模拟 5G 基站（gNB）和终端（UE），用于测试 5G 核心网，无需真实基站和手机硬件。

#### 5.1 安装依赖并编译

```bash
sudo apt update && sudo apt upgrade

# 安装编译工具
sudo apt install -y make g++ libsctp-dev lksctp-tools iproute2
sudo snap install cmake --classic

# 克隆 UERANSIM
cd ~
git clone https://github.com/aligungr/UERANSIM
cd UERANSIM

# 根据 free5GC 版本选择对应的 UERANSIM commit：
# free5GC v3.3.0 及以下：git checkout 3a96298
# free5GC v3.4.0 及以上：git checkout e4c492d
# free5GC v3.4.x (EAP-AKA-PRIME 修复)：git checkout 85a0fbf
git checkout 85a0fbf

# 编译
make
```

#### 5.2 配置 free5GC 侧（free5gc VM）

需要在 free5gc VM 上修改三个配置文件，让网元监听外部 IP：

**① ~/free5gc/config/amfcfg.yaml**
```yaml
# 改前：
ngapIpList:
  - 127.0.0.1
# 改后：
ngapIpList:
  - 192.168.56.101
```

**② ~/free5gc/config/smfcfg.yaml**
```yaml
# 在 userplaneInformation → upNodes → UPF → interfaces → endpoints 中
# 改前：
endpoints:
  - 127.0.0.8
# 改后：
endpoints:
  - 192.168.56.101
```

**③ ~/free5gc/config/upfcfg.yaml**
```yaml
# 改前：
gtpu:
  ifList:
    - addr: 127.0.0.8
      type: N3
# 改后：
gtpu:
  ifList:
    - addr: 192.168.56.101
      type: N3
```

一键 sed 命令：
```bash
sed -i 's/- 127.0.0.18/- 192.168.56.101/' ~/free5gc/config/amfcfg.yaml
sed -i 's/- 127.0.0.8/- 192.168.56.101/' ~/free5gc/config/smfcfg.yaml
sed -i 's/addr: 127.0.0.8   # GTP-U/addr: 192.168.56.101   # GTP-U/' ~/free5gc/config/upfcfg.yaml
# 内核 6.x 还需要启用 natifname（见踩坑记录）
```

#### 5.3 配置 UERANSIM 侧（ueransim VM）

**① ~/UERANSIM/config/free5gc-gnb.yaml**
```yaml
ngapIp: 192.168.56.102     # gNB 的 N2 地址
gtpIp: 192.168.56.102      # gNB 的 N3 地址
amfConfigs:
  - address: 192.168.56.101 # AMF 地址
```

**② ~/UERANSIM/config/free5gc-ue.yaml**
```yaml
supi: 'imsi-208930000000003'  # 与 WebConsole 一致
mcc: '208'
mnc: '93'
key: '8baf473f2f8fd09487cccbd7097c6862'  # 与 WebConsole 一致
op: '8e27b6af0e692e750f32667a3b14605d'    # 与 WebConsole 一致
opType: 'OP'                               # 必须与 WebConsole 的 OP Type 一致
sessions:
  - type: 'IPv4'
    apn: 'internet'
    slice:
      sst: 0x01
      sd: 0x010203
```

#### 5.4 配置网络转发规则（free5gc VM）

每次启动核心网前都要执行：
```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1400
sudo systemctl stop ufw    # 关闭防火墙
```

也可以用 free5GC 自带的脚本：
```bash
sudo ./~/free5gc/reload_host_config.sh enp0s3
```

> 提示：可将 `net.ipv4.ip_forward=1` 写入 `/etc/sysctl.conf` 使其永久生效。

#### 5.5 启动并测试

需要 **4 个终端**：

| 终端 | 位置 | 命令 | 作用 |
|------|------|------|------|
| 终端1 | free5gc VM | `cd ~/free5gc && ./run.sh` | 启动 5G 核心网所有网元 |
| 终端2 | ueransim VM | `cd ~/UERANSIM && build/nr-gnb -c config/free5gc-gnb.yaml` | 启动仿真 5G 基站 |
| 终端3 | ueransim VM | `cd ~/UERANSIM && sudo build/nr-ue -c config/free5gc-ue.yaml` | 启动仿真终端（UE） |
| 终端4 | ueransim VM | 测试命令 | 验证联通性 |

在终端4验证：
```bash
# 检查 uesimtun0 接口是否创建
ifconfig uesimtun0

# ping 测试（通过 5G 核心网上网）
ping -I uesimtun0 8.8.8.8

# 去注册测试
cd ~/UERANSIM
sudo ./build/nr-cli imsi-208930000000003 --exec "deregister normal"
```

**成功标志：** `ping -I uesimtun0 8.8.8.8` 收到回复，说明 5G 核心网端到端数据面已打通！

---

### 第 6 步：安装 N3IWUE（非3GPP 接入 — WiFi 接入 5GC）

> N3IWF（Non-3GPP InterWorking Function）：让手机通过不安全的 WiFi 网络也能安全接入 5G 核心网。
> 通过 IPSec/IKEv2 建立加密隧道，实现类似 VoWiFi 的功能。

#### 6.1 创建 N3IWUE VM

从基础 VM 克隆第三台 VM：
- 名称：`n3iwue`
- 主机名改为 `n3iwue`
- 静态 IP：`192.168.56.103`
- 确保能 ping 通 `192.168.56.101`

#### 6.2 安装 N3IWUE

```bash
# 在 n3iwue VM 上
cd ~
git clone https://github.com/free5gc/n3iwue.git
cd n3iwue

sudo apt update && sudo apt upgrade
sudo apt install -y make libsctp-dev lksctp-tools iproute2

# 安装 Go（N3IWUE 使用 Go 1.21.6）
wget https://dl.google.com/go/go1.21.6.linux-amd64.tar.gz
sudo tar -C /usr/local -zxvf go1.21.6.linux-amd64.tar.gz
mkdir -p ~/go/{bin,pkg,src}
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin:$GOROOT/bin' >> ~/.bashrc
echo 'export GO111MODULE=auto' >> ~/.bashrc
source ~/.bashrc

go version   # 确认 Go 安装成功
make         # 编译 N3IWUE
```

#### 6.3 通过 WebConsole 添加 UE

在 free5gc VM 上启动 WebConsole 并为 N3IWUE 创建一个新的 Subscriber：
- 记下 SUPI (IMSI)、K、OP，确保与 N3IWUE 配置一致
- 特别要注意 SQN 值也要一致（N3IWUE 的 `n3ue.yaml` 中需要配置 SQN）

#### 6.4 配置 N3IWF（free5gc VM 侧）

编辑 `~/free5gc/config/n3iwfcfg.yaml`：
```yaml
# 改前：
IKEBindAddress: 172.16.2.100
# 改后：
IKEBindAddress: 192.168.56.101
```

#### 6.5 配置 N3IWUE

编辑 `~/n3iwue/config/n3ue.yaml`：
```yaml
N3IWFInformation:
  IPSecIfaceAddr: 192.168.56.101    # N3IWF 的 IP

N3UEInformation:
  IPSecIfaceName: enp0s8            # N3IWUE VM 的 Host-Only 网卡名
  IPSecIfaceAddr: 192.168.56.103    # N3IWUE VM 的 Host-Only IP

# 确保 SUPI、K、OP、SQN 与 WebConsole 中一致
```

#### 6.6 测试

```bash
# 终端1 (free5gc VM)：启动核心网 + N3IWF
cd ~/free5gc
./run.sh -n3iwf

# 终端2 (n3iwue VM)：启动 N3IWUE
cd ~/n3iwue
./run.sh
```

成功后可在 N3IWUE VM 上通过 `uesimtun0` 接口 ping 通外网，证明非3GPP 接入链路建立成功。

---

### 第 7 步：安装 TNGFUE（可信非3GPP 接入 — 企业 WiFi）

> TNGF（Trusted Non-3GPP Gateway Function）：用于"可信"WiFi 网络（如企业内网）。
> 与 N3IWF 的区别：N3IWF 连不安全的 WiFi（用 IPSec），TNGF 连受信任的 WiFi（用 EAP 认证）。

#### 7.1 free5GC 侧配置

编辑 `~/free5gc/config/tngfcfg.yaml`：
```yaml
IKEBindAddress: <free5GC_IP>     # 如 192.168.56.101
RadiusBindAddress: <free5GC_IP>  # 如 192.168.56.101
RadiusSecret: free5gctngf        # RADIUS 共享密钥
```

#### 7.2 WiFi 接入点 (AP) 配置（使用 hostapd）

在 TNGFUE 的设备上（需要有 WiFi 网卡，不一定是 VM）：
```bash
sudo apt install hostapd

sudo nano /etc/hostapd/hostapd.conf
```

配置文件内容：
```
interface=wlan0
driver=nl80211
ssid=free5gc-ap
hw_mode=g
channel=6
ieee8021x=1
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-EAP
rsn_pairwise=CCMP
# RADIUS 配置指向 free5GC
auth_server_addr=192.168.56.101
auth_server_port=1812
auth_server_shared_secret=free5gctngf
nas_identifier=myhostapd
```

启动 hostapd：
```bash
sudo systemctl unmask hostapd
sudo systemctl enable --now hostapd
```

#### 7.3 安装配置 TNGFUE

```bash
git clone https://github.com/free5gc/tngfue.git
cd tngfue

# 自动化安装
./prepare.sh
```

需要配置的文件：

**wpa_supplicant.conf**（WiFi 连接信息）：
```
ctrl_interface=udp
update_config=1
network={
    ssid="free5gc-ap"
    key_mgmt=WPA-EAP
    eap=VENDOR-TEST IKEV2
    identity="tngfue"
    password="free5gctngf"
}
```

**sec.conf**（安全参数，与 WebConsole 保持一致）：
```
wifiifname: <WiFi网卡名>
K: 8baf473f2f8fd09487cccbd7097c6862
imsi_identity: 208930000000007
SQN: 16f3b3f70fe1
AMF: 8000
OPC: 8e27b6af0e692e750f32667a3b14605d
```

#### 7.4 编译并测试

```bash
cd ~/tngfue/wpa_supplicant
make
```

```bash
# 终端1 (free5gc VM)：启动核心网 + TNGF
cd ~/free5gc
./run.sh -tngf

# 终端2 (TNGFUE 设备)：
cd ~/tngfue
./run.sh
```

成功后接口 `greTun0` 出现，可通过它 ping 外网：
```bash
ping -I greTun0 8.8.8.8
```

---

### 第 8 步：free5GC 简单应用测试

这一节演示如何通过 5G 核心网运行实际应用。

#### 8.1 ping + tcpdump（包抓取分析）

```bash
# 在 ueransim VM 上
# 启动 free5GC + gNB + UE 后

# 验证路由表
route -n

# 抓包观察 GTP-U 隧道
sudo tcpdump -n -i any host 60.60.0.1 or 192.168.56.101

# 另一个终端 ping
ping -I uesimtun0 8.8.8.8
```

抓包示例输出：
```
60.60.0.1 > 8.8.8.8: ICMP echo request
192.168.56.102.2152 > 192.168.56.101.2152: UDP (GTP-U 封装)
192.168.56.101.2152 > 192.168.56.102.2152: UDP (GTP-U 响应)
8.8.8.8 > 60.60.0.1: ICMP echo reply
```

**高级路由技巧：** 关闭 NAT 网卡(enp0s3)后，将 uesimtun0 设为默认网关：
```bash
sudo ifconfig enp0s3 down
sudo ip r add default dev uesimtun0
# 现在直接 ping 8.8.8.8 也会走 5G 核心网
```

#### 8.2 wget / curl（文件下载）

```bash
# 测试通过 5GC 下载文件
wget https://golang.org/dl/go1.15.8.darwin-amd64.pkg
```

#### 8.3 SSH 远程连接

```bash
# 通过 5GC 访问远程 SSH 服务
ssh bbsu@ptt.cc
```

#### 8.4 YouTube / 桌面应用

如果要测试图形化应用，可以安装 Lubuntu Desktop（轻量版 Ubuntu 桌面）：
- 分配 2 CPU + 2048MB 内存
- 安装 UERANSIM 后，浏览器通过 uesimtun0 访问 YouTube

---

### 第 9 步（高级）：流量导向（Traffic Influence）

> 当有多个 UPF 时（如边缘 UPF + 中心 UPF），可以动态控制流量走哪个 UPF。
> 适用场景：边缘计算（MEC），让低延迟流量走本地 UPF 处理。

#### 9.1 拓扑结构

```
UE ←→ gNB ←→ I-UPF ←→ PSA-UPF ←→ 互联网
                  ↑
                  └──→ MEC 应用服务器（低延迟）
```

#### 9.2 通过 UDR 影响流量导向

将导向数据 PUT 到 UDR：
```bash
curl -X PUT -H "Content-Type: application/json" --data @./ti_data.json \
    http://<udr-ip>:8000/nudr-dr/v1/application-data/influenceData/1
```

ti_data.json 示例：
```json
{
    "dnn": "internet",
    "snssai": { "sst": 1, "sd": "010203" },
    "interGroupId": "AnyUE",
    "trafficFilters": [{
        "flowId": 1,
        "flowDescriptions": [
            "permit out ip from <server-cidr> to 10.60.0.0/16"
        ]
    }],
    "trafficRoutes": [{ "dnai": "mec" }]
}
```

查询/删除导向数据：
```bash
curl -X GET http://<udr-ip>:8000/nudr-dr/v1/application-data/influenceData?dnns=internet
curl -X DELETE http://<udr-ip>:8000/nudr-dr/v1/application-data/influenceData/1
```

#### 9.3 通过 NEF + AF 影响流量导向

AF（应用功能）通过 NEF 请求流量导向（3GPP TS 23.502）：

**针对所有 UE：**
```bash
curl -X POST -H "Content-Type: application/json" --data @./af_ti_anyUE.json \
    http://<nef-ip>:8000/3gpp-traffic-influence/v1/af001/subscriptions
```

**针对单个 UE（通过 IPv4 地址）：**
```bash
curl -X POST -H "Content-Type: application/json" --data @./af_ti_singleUE.json \
    http://<nef-ip>:8000/3gpp-traffic-influence/v1/af001/subscriptions
```

---

## 四、版本对照

| 组件 | 版本 | 说明 |
|------|------|------|
| free5GC | v4.2.2 | 5G 核心网稳定版 |
| Go | 1.26.2 | free5GC 编译语言 |
| gtp5g | v0.9.14 | 内核 GTP-U 模块 |
| MongoDB | 8.0 | 用户数据存储 |
| Node.js | 20.x | WebConsole 前端运行时 |
| UERANSIM | commit 85a0fbf | UE/RAN 仿真器（适配 free5GC v3.4.x） |
| N3IWUE | v1.0.1+ | 非3GPP 接入仿真（需要 flow rule 修复版） |
| Ubuntu | 20.04 LTS | 推荐版本（内核 5.4.x，gtp5g 兼容性最好） |
| Linux Kernel | 5.4.x（推荐）/ 6.8（实测可用） | gtp5g 仅测试过 5.0.0-23-generic 和 5.4.x |
| VirtualBox | 7.0.12+ | 虚拟机软件 |

---

## 五、踩坑记录

### 1. `go: not found` — make 编译失败
- **原因**：`make` 使用 `/bin/sh`（非交互式 shell），不加载 `.bashrc` 中的 PATH
- **解决**：`sudo ln -sf /usr/local/go/bin/go /usr/local/bin/go`

### 2. N3 GTP-U 隧道建立但 ping 不通
- **原因**：内核 6.8 + gtp5g v0.9.14 的兼容性问题，需显式设置 `natifname` 且需添加 UE IP 到 upfgtp 接口的路由
- **解决**：
  ```bash
  # 在 upfcfg.yaml 中启用 natifname
  sed -i 's/# natifname: eth0/natifname: enp0s3/' ~/free5gc/config/upfcfg.yaml
  # 添加 UE IP 到 upfgtp（路由查找需要）
  sudo ip addr add 10.60.0.1/32 dev upfgtp
  ```

### 3. VM 重启后网络规则丢失
- **原因**：iptables 和 sysctl 配置在重启后不持久化
- **解决**：
  ```bash
  # 每次重启后重执行：
  sudo sysctl -w net.ipv4.ip_forward=1
  sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
  sudo iptables -I FORWARD 1 -j ACCEPT
  # 或用 free5GC 自带脚本：
  sudo ./free5gc/reload_host_config.sh enp0s3
  # 永久生效：echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
  ```

### 4. OPc vs OP 类型不匹配
- **原因**：WebConsole 默认 `OPc`，UE 配置用 `OP`，鉴权失败
- **解决**：确保 WebConsole 和 free5gc-ue.yaml 的 `opType` 一致（都设 `OP` 或都设 `OPC`）

### 5. MongoDB 需要 CPU 支持 AVX
- **原因**：MongoDB 5.0+ 需要 CPU 支持 AVX 指令集
- **检查**：`lscpu | grep avx`
- **解决**：不支持 AVX 的 CPU 降级到 MongoDB 4.4 或使用 `sudo apt install mongodb`

### 6. sed 替换 IP 时误改了 pfcp 部分
- **原因**：`sed 's/addr: 127.0.0.8/addr: 192.168.56.101/'` 同时匹配了 gtpu 和 pfcp 两个段落中的地址
- **解决**：用范围限定语法 `sed -i '/^pfcp:/,/^gtpu:/{s/addr: 192.168.56.101/addr: 127.0.0.8/}'` 撤销 pfcp 部分的改动

### 7. TestNon3GPP 测试失败
- **原因**：N3IWF/IPSec 隧道建立需要额外的网络配置，标准 VM 环境不支持
- **影响**：不影响核心 5G 功能，10/11 测试通过即可

### 8. N3IWUE kill 后重连 SQN 不同步
- **原因**：N3IWUE 在鉴权进行中被 kill，SQN 未正常同步
- **解决**：使用 N3IWUE v1.0.1+ 版本，或手动重置 WebConsole 中该 UE 的 SQN

---

## 六、总结：四个终端的作用

| 终端 | VM | 命令 | 角色 |
|------|-----|------|------|
| 1 | free5gc | `./run.sh` | **核心网大脑** — 启动所有网元（AMF、SMF、UPF、NRF 等），相当于电信机房里的一排服务器同时开机 |
| 2 | ueransim | `nr-gnb -c ...` | **仿真基站** — 模拟 5G 基站的信号发射和 N2/N3 连接，没有它手机（UE）找不到网络 |
| 3 | ueransim | `sudo nr-ue -c ...` | **仿真手机** — 模拟手机注册、鉴权、建立数据通道（PDU 会话），建隧道接口 uesimtun0 |
| 4 | ueransim | `ping ...` | **验证测试** — 在 uesimtun0 出现后验证网络连通性，确认从手机到公网的完整链路 |

---

*文档生成时间：2026年5月 | 基于 free5GC v4.2.2 + UERANSIM 实测*
