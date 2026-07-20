# free5GC 5G 核心网 -- 环境搭建与知识总结

## 一、名词解释

### 1.1 5G 核心网网元 (Network Functions)

| 缩写 | 全称 | 职责 |
|------|------|------|
| NRF | Network Repository Function | 注册中心。所有 NF 启动后向 NRF 注册，别的 NF 通过 NRF 发现对方 |
| AMF | Access and Mobility Management | 接入与移动管理。UE 注册、鉴权、分配 GUTI、位置跟踪 |
| SMF | Session Management | 会话管理。建立 PDU 会话、分配 IP、选 UPF、下发转发规则 |
| UPF | User Plane | 用户面网关。GTP-U 封装/解封装、NAT、QoS 执行、计费上报 |
| AUSF | Authentication Server | 鉴权服务。执行 5G AKA 鉴权算法 |
| UDM | Unified Data Management | 用户数据管理。存 SIM 卡信息、鉴权密钥、套餐数据 |
| UDR | Unified Data Repository | 数据库。实际存放签约数据、策略数据 |
| PCF | Policy Control | 策略控制。下发 QoS 策略、接入控制策略 |
| NSSF | Network Slice Selection | 切片选择。根据 UE 请求的 S-NSSAI 选合适切片 |
| CHF | Charging | 计费。在线/离线计费 |
| BSF | Binding Support | 记录 UE 绑定了哪个 PCF，用于策略关联 |
| NEF | Network Exposure | 对外能力开放。第三方应用通过 NEF 调用 5GC 能力 |
| N3IWF | Non-3GPP InterWorking | 非 3GPP 接入网关。WiFi 通过 IPSec 隧道接入 5GC |
| TNGF | Trusted Non-3GPP Gateway | 可信 WiFi 网关。企业 WiFi 通过 EAP 认证接入 5GC |

### 1.2 通信接口

| 接口 | 连接 | 协议 | 用途 |
|------|------|------|------|
| N2 | gNB - AMF | NGAP/SCTP | 控制信令（注册、切换、寻呼） |
| N3 | gNB - UPF | GTP-U | 用户数据隧道 |
| N4 | SMF - UPF | PFCP | SMF 下发 PDR/FAR 转发规则 |
| N6 | UPF - DN/互联网 | 纯 IP | UPF 到公网 |
| N9 | UPF - UPF | GTP-U | UPF 间隧道（ULCL 分流） |
| SBI | NF - NF | HTTP/2 | 控制面网元间互调 |

### 1.3 UERANSIM / 终端相关术语

| 缩写 | 全称 | 含义 |
|------|------|------|
| UE | User Equipment | 终端。nr-ue 模拟 |
| gNB | gNodeB | 5G 基站。nr-gnb 模拟 |
| SUPI | Subscription Permanent Identifier | 永久用户 ID，格式 imsi-208930000000001 |
| SUCI | Subscription Concealed Identifier | 加密后的 SUPI |
| GUTI | Globally Unique Temporary Identifier | AMF 分配的临时 ID |
| IMSI | International Mobile Subscriber Identity | 15 位 = MCC + MNC + MSIN |
| MCC | Mobile Country Code | 国家码。208=法国（测试用） |
| MNC | Mobile Network Code | 运营商码。93=测试网络 |
| S-NSSAI | Single Network Slice Selection Assistance Information | 切片标识（SST+SD） |
| DNN | Data Network Name | 数据网络名，相当于 APN |
| PDU Session | Protocol Data Unit Session | UE 到互联网的数据通道 |

### 1.4 鉴权相关

| 缩写 | 含义 |
|------|------|
| K / Key | 永久密钥。存在 SIM 卡和 UDM 中 |
| OP | 运营商原始密钥 |
| OPc | 从 OP+K 经 AES 计算出的结果 |
| SQN | 序列号。防重放攻击 |
| AMF (鉴权字段) | 鉴权管理字段。8000=5G AKA |

### 1.5 网络相关

| 缩写 | 含义 |
|------|------|
| NAT | 网络地址转换。内网 IP 转公网 IP |
| MASQUERADE | iptables 动态 NAT。自动换源 IP 为出口网卡 IP |
| GTP-U | 用户面隧道协议。UE IP 包封装进 UDP 经 N3 传输 |
| PFCP | N4 协议。SMF 通过它下发转发规则 |
| PDR | 包检测规则。告诉 UPF 怎么识别某个 UE 的包 |
| FAR | 转发动作规则。告诉 UPF 匹配后往哪发 |
| SCTP | N2 传输层协议。比 TCP 更适合信令 |
| Host-Only | VirtualBox 虚拟机间内部通信网卡 |
| IPSec | N3IWF 用的加密隧道协议 |
| IKEv2 | IPSec 密钥协商协议 |
| EAP | 可扩展认证协议。TNGF 用 EAP-TLS/EAP-AKA |
| DNAI | 数据网络接入标识。标识 UPF 的 N6 出口位置 |
| ULCL | Uplink Classifier。上行分流 UPF，按规则导向不同出口 |

---

## 二、网元关系与信令/数据流

### 2.1 整体架构

5GC 核心网分为控制面和用户面。控制面包括 AMF（接入/移动管理）、SMF（会话管理）、NRF（注册中心）、UDM/UDR（用户数据）、AUSF（鉴权）、PCF（策略）、NSSF（切片选择）、CHF（计费）等。用户面只有 UPF，负责 GTP-U 数据转发和 NAT。UPF 通过 N3 口接 gNB、N6 口接互联网、N4 口接受 SMF 下发规则。UERANSIM 模拟 gNB 和 UE，gNB 通过 N2 口跟 AMF 交互信令。

### 2.2 UE 注册流程（控制面信令）

UE -> gNB -> AMF -> UDM/AUSF -> NRF

1. UE 发 Registration Request
2. gNB 封装为 Initial UE Message 发给 AMF
3. AMF 向 UDM 请求鉴权向量
4. AMF 发起 Authentication Request（AUSF 执行 5G AKA）
5. UE 回应 Authentication Response
6. AMF 验证通过，发起 NAS SMC 协商安全
7. AMF 发 Registration Accept（含 GUTI）
8. AMF 向 NRF 注册 UE 上下文

### 2.3 PDU 会话建立流程

UE -> AMF -> SMF -> PCF/UDR -> UPF

1. UE 发 PDU Session Establishment Request
2. AMF 转发给 SMF
3. SMF 向 PCF 查询策略
4. SMF 选 UPF、分配 IP
5. SMF 通过 N4 口向 UPF 下发 PDR/FAR 规则
6. SMF 通过 AMF 向 gNB 发 N2 消息（含 N3 隧道信息）
7. UE 收到 PDU Session Establishment Accept
8. uesimtun0 接口出现，UE 获得 IP

### 2.4 用户数据传输路径（ping 的旅程）

UE -> gNB -> UPF -> 互联网

1. UE 通过 uesimtun0 发原始 IP 包（源 10.60.0.x）
2. gNB 做 GTP-U 封装（外层 UDP port 2152）
3. UPF 去封装，PDR 识别 -> FAR 转发
4. UPF 做 NAT MASQUERADE（源 IP 变为 10.0.2.15）
5. 出公网
6. 回包：UPF 查 NAT 表还原 -> 套 GTP-U 壳 -> gNB 拆壳 -> UE

---

## 三、本地部署全过程（基于 free5GC 官方指南）

> 官方指南：https://free5gc.org/guide/
> 对应 Advanced 部分 "Build free5GC from Scratch"

### 环境概览

| 虚拟机 | 角色 | Host-Only IP | NAT IP |
|--------|------|-------------|--------|
| free5gc | 5G 核心网 | 192.168.56.101 | 10.0.2.15 |
| ueransim | 仿真基站+终端 | 192.168.56.102 | 10.0.2.15 |
| n3iwue (可选) | 非3GPP 终端 | 192.168.56.103 | 10.0.2.15 |

每台 VM 两张网卡：NAT 网卡(enp0s3)上外网，Host-Only 网卡(enp0s8)VM 间通信。

---

### 第 1 步：创建 Ubuntu 虚拟机（VirtualBox）

下载 VirtualBox 7.0.12+ 和 Ubuntu Server 20.04 LTS ISO。

选 20.04 的原因：free5GC 的 gtp5g 内核模块在 5.4.x 内核上测试最充分。22.04/24.04 的 6.x 内核需要额外配置。

创建 VM：2 CPU、2048MB 内存、两张网卡（网卡1 NAT、网卡2 Host-Only）。安装时选 Minimal Installation、不选 LVM、勾选 Install SSH Server。用户名和密码设短一点方便输入。

登录后：
```bash
sudo apt install net-tools
ifconfig   # 确认 enp0s3: 10.0.2.15, enp0s8: 192.168.56.x
ping google.com
sudo apt update && sudo apt upgrade
```

从宿主机 SSH：`ssh 192.168.56.101 -l <用户名>`

---

### 第 2 步：克隆并配置 free5GC 和 ueransim VM

关闭基础 VM（`sudo shutdown -P now`），VirtualBox 克隆两份：
- `free5gc`：全克隆，新 MAC 地址
- `ueransim`：链接克隆

分别修改主机名：`/etc/hostname` 和 `/etc/hosts` 中改对应名字，重启。

设置静态 IP（free5gc VM）：
```bash
cd /etc/netplan
sudo nano 00-installer-config.yaml
```

```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses: [192.168.56.101/24]
  version: 2
```

```bash
sudo netplan try && sudo netplan apply
```

ueransim VM 同样操作，静态 IP 设为 192.168.56.102。验证互通：互相 ping。

---

### 第 3 步：构建 free5GC（free5gc VM）

#### 3.1 检查环境
```bash
uname -r          # 期望 5.4.0-xx-generic
lscpu | grep avx  # MongoDB 5.0+ 需要 AVX
```

#### 3.2 安装 Go 1.26.2
```bash
wget https://dl.google.com/go/go1.26.2.linux-amd64.tar.gz
sudo tar -C /usr/local -zxvf go1.26.2.linux-amd64.tar.gz
mkdir -p ~/go/{bin,pkg,src}
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin:$GOROOT/bin' >> ~/.bashrc
echo 'export GO111MODULE=auto' >> ~/.bashrc
source ~/.bashrc
go version
sudo ln -sf /usr/local/go/bin/go /usr/local/bin/go  # 非交互 shell 也需要
```

#### 3.3 安装依赖
```bash
sudo apt update
sudo apt install -y wget git gcc g++ cmake autoconf libtool \
    pkg-config libmnl-dev libyaml-dev gnupg curl
```

#### 3.4 安装 MongoDB 8.0
```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
    sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
# 根据 Ubuntu 版本选源（用 cat /etc/lsb-release 查看）：
# Ubuntu 20.04:
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/8.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.2.list
# Ubuntu 22.04: 把 focal 换成 jammy
# Ubuntu 24.04: 把 focal 换成 noble
sudo apt update && sudo apt install -y mongodb-org
sudo systemctl start mongod && sudo systemctl enable mongod
```

不支持 AVX 则降级到 MongoDB 4.4 或用 `sudo apt install mongodb`。

#### 3.5 安装 Node.js（WebConsole）
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
corepack enable
```

#### 3.6 编译 free5GC
```bash
cd ~
git clone --recursive -b v4.2.2 -j $(nproc) https://github.com/free5gc/free5gc.git
cd free5gc
make              # 编译所有控制面网元
make webconsole   # 编译 WebConsole
```

#### 3.7 安装 gtp5g 内核模块
```bash
cd ~/free5gc
git clone -b v0.9.14 https://github.com/free5gc/gtp5g.git
cd gtp5g
make && sudo make install
sudo modprobe gtp5g
lsmod | grep gtp5g   # 验证
```

---

### 第 4 步：集成测试

free5GC 自带 11 个集成测试：

| 测试 | 内容 |
|------|------|
| TestRegistration | UE 初始注册、鉴权、安全模式 |
| TestGUTIRegistration | 用 GUTI 重注册，无需重新鉴权 |
| TestServiceRequest | UE 空闲态回到连接态 |
| TestXnHandover | gNB 间 Xn 切换 |
| TestDeregistration | UE 主动去注册 |
| TestPDUSessionReleaseRequest | UE 请求释放 PDU 会话 |
| TestPaging | 下行数据触发寻呼 |
| TestN2Handover | AMF 参与的 gNB 间切换 |
| TestNon3GPP | N3IWF/IPSec 隧道（无 N3IWF 环境预期失败） |
| TestReSynchronization | SQN 重同步 |
| TestULCL | 多 UPF 上行分流 |

```bash
cd ~/free5gc
make upf
chmod +x ./test.sh
./test.sh TestRegistration    # 单个测试
./test_ulcl.sh TestRequestTwoPDUSessions  # ULCL 用专用脚本
```

#### WebConsole 添加用户

```bash
cd ~/free5gc/webconsole
./bin/webconsole
```

浏览器访问 http://192.168.56.101:5000，admin / free5gc 登录。
Subscribers -> New Subscriber，OP Type 选 OP，记下 SUPI、K、OP。
提交后 Ctrl-C 关 WebConsole。

---

### 第 5 步：安装 UERANSIM（ueransim VM）

```bash
sudo apt update && sudo apt upgrade
sudo apt install -y make g++ libsctp-dev lksctp-tools iproute2
sudo snap install cmake --classic

cd ~
git clone https://github.com/aligungr/UERANSIM
cd UERANSIM
git checkout 85a0fbf  # free5GC v3.4.x 适配版本
make
```

#### 配置 free5GC 侧（free5gc VM）

修改三个配置文件，把 127.0.0.1/127.0.0.8 改为 192.168.56.101：

- `amfcfg.yaml`：ngapIpList 改为 192.168.56.101
- `smfcfg.yaml`：UPF endpoints 改为 192.168.56.101
- `upfcfg.yaml`：gtpu addr 改为 192.168.56.101

一键 sed：
```bash
sed -i 's/- 127.0.0.18/- 192.168.56.101/' ~/free5gc/config/amfcfg.yaml
sed -i 's/- 127.0.0.8/- 192.168.56.101/' ~/free5gc/config/smfcfg.yaml
sed -i 's/addr: 127.0.0.8   # GTP-U/addr: 192.168.56.101   # GTP-U/' ~/free5gc/config/upfcfg.yaml
```

#### 配置 UERANSIM 侧（ueransim VM）

`free5gc-gnb.yaml`：ngapIp 和 gtpIp 设 192.168.56.102，amfConfigs address 设 192.168.56.101。

`free5gc-ue.yaml`：supi、key、op、opType 与 WebConsole 一致。

#### 网络转发（free5gc VM，每次启动前执行）

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1400
sudo systemctl stop ufw
# 或一键：sudo ~/free5gc/reload_host_config.sh enp0s3
```

`net.ipv4.ip_forward=1` 可写入 `/etc/sysctl.conf` 永久生效。

#### 启动测试

需要 4 个终端：

| 终端 | VM | 命令 | 作用 |
|------|-----|------|------|
| 1 | free5gc | `cd ~/free5gc && ./run.sh` | 启动所有核心网网元 |
| 2 | ueransim | `cd ~/UERANSIM && build/nr-gnb -c config/free5gc-gnb.yaml` | 启动仿真基站 |
| 3 | ueransim | `sudo build/nr-ue -c config/free5gc-ue.yaml` | 启动仿真 UE |
| 4 | ueransim | 测试 | 验证联通 |

验证：
```bash
ifconfig uesimtun0                    # 确认隧道接口出现
ping -I uesimtun0 8.8.8.8            # 通过 5GC 上网
sudo ./build/nr-cli imsi-208930000000003 --exec "deregister normal"  # 去注册
```

ping 通说明端到端数据面已打通。

---

### 第 6 步：N3IWUE（非3GPP 接入，WiFi -> 5GC）

从基础 VM 克隆第三台 `n3iwue`，静态 IP 192.168.56.103。N3IWF 让手机通过不安全的 WiFi 经 IPSec/IKEv2 加密隧道接入 5GC。

```bash
# n3iwue VM
cd ~
git clone https://github.com/free5gc/n3iwue.git
cd n3iwue
sudo apt update && sudo apt upgrade
sudo apt install -y make libsctp-dev lksctp-tools iproute2

# 安装 Go 1.21.6
wget https://dl.google.com/go/go1.21.6.linux-amd64.tar.gz
sudo tar -C /usr/local -zxvf go1.21.6.linux-amd64.tar.gz
mkdir -p ~/go/{bin,pkg,src}
# 配置环境变量（同第3步）
source ~/.bashrc
make
```

WebConsole 添加 UE 后，配置 free5GC 侧 `n3iwfcfg.yaml` 的 IKEBindAddress 为 192.168.56.101。配置 N3IWUE 侧 `n3ue.yaml` 的 N3IWF IP 和本机 Host-Only IP。

测试：
```bash
# free5gc VM
cd ~/free5gc && ./run.sh -n3iwf
# n3iwue VM
cd ~/n3iwue && ./run.sh
```

成功后 `uesimtun0` 出现，可 ping 外网。

---

### 第 7 步：TNGFUE（可信非3GPP 接入，企业 WiFi）

TNGF vs N3IWF：N3IWF 接不安全 WiFi（IPSec），TNGF 接受信任 WiFi（EAP 认证）。

free5GC 侧 `tngfcfg.yaml`：IKEBindAddress 和 RadiusBindAddress 设 free5GC IP，RadiusSecret 设 free5gctngf。

TNGFUE 设备需 WiFi 网卡。安装 hostapd 配置企业 WiFi AP（WPA2-EAP + RADIUS 指向 free5GC）。TNGFUE 通过 wpa_supplicant 连接，`sec.conf` 里 K、SQN、AMF、OPC 与 WebConsole 一致。

编译启动：
```bash
cd ~/tngfue/wpa_supplicant && make
# free5gc VM: ./run.sh -tngf
# TNGFUE 设备: cd ~/tngfue && ./run.sh
```

成功后 `greTun0` 出现，`ping -I greTun0 8.8.8.8`。

---

### 第 8 步：简单应用测试

**抓包验证：**
```bash
sudo tcpdump -n -i any host 60.60.0.1 or 192.168.56.101
# 另一个终端: ping -I uesimtun0 8.8.8.8
# 应看到: 内层 ICMP + 外层 UDP port 2152 (GTP-U)
```

**高级技巧：** 关 NAT 网卡，uesimtun0 设默认网关：
```bash
sudo ifconfig enp0s3 down
sudo ip r add default dev uesimtun0
# 直接 ping 8.8.8.8 走 5GC
```

**其他测试：** wget 下载文件、SSH 远程连接。如需要图形界面，安装 Lubuntu Desktop 后用浏览器通过 uesimtun0 访问网页。

---

### 第 9 步（高级）：流量导向（Traffic Influence）

多 UPF 场景（边缘 UPF + 中心 UPF）可动态控制流量走向，用于边缘计算（MEC）：UPF 分流到 MEC 服务器，实现低延迟边缘计算。

拓扑：UE <-> gNB <-> I-UPF <-> PSA-UPF <-> 互联网，同时 I-UPF 可分流到本地 MEC 服务器。

**通过 UDR 影响流量导向：**
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
        "flowDescriptions": ["permit out ip from <server-cidr> to 10.60.0.0/16"]
    }],
    "trafficRoutes": [{ "dnai": "mec" }]
}
```

查询/删除：
```bash
curl -X GET http://<udr-ip>:8000/nudr-dr/v1/application-data/influenceData?dnns=internet
curl -X DELETE http://<udr-ip>:8000/nudr-dr/v1/application-data/influenceData/1
```

**通过 NEF + AF 影响流量导向：**
```bash
# 针对所有 UE
curl -X POST -H "Content-Type: application/json" --data @./af_ti_anyUE.json \
    http://<nef-ip>:8000/3gpp-traffic-influence/v1/af001/subscriptions
# 针对单个 UE
curl -X POST -H "Content-Type: application/json" --data @./af_ti_singleUE.json \
    http://<nef-ip>:8000/3gpp-traffic-influence/v1/af001/subscriptions
```

---

## 四、版本对照

| 组件 | 版本 | 说明 |
|------|------|------|
| free5GC | v4.2.2 | 5GC 稳定版 |
| Go | 1.26.2 | 编译语言 |
| gtp5g | v0.9.14 | 内核 GTP-U 模块 |
| MongoDB | 8.0 | 数据库 |
| Node.js | 20.x | WebConsole 前端 |
| UERANSIM | commit 85a0fbf | 适配 free5GC v3.4.x |
| Ubuntu | 20.04 LTS | 推荐版本（内核 5.4.x） |
| Kernel | 5.4.x 推荐 / 6.8 可用 | gtp5g 官方测试过 5.0.0-23 和 5.4.x |
| VirtualBox | 7.0.12+ | 虚拟机 |

---

## 五、踩坑记录

1. **go: not found** -- make 用 /bin/sh 不加载 .bashrc。解决：`sudo ln -sf /usr/local/go/bin/go /usr/local/bin/go`

2. **N3 GTP-U 隧道建了但 ping 不通** -- 内核 6.8 + gtp5g 兼容问题。解决：
   ```bash
   sed -i 's/# natifname: eth0/natifname: enp0s3/' ~/free5gc/config/upfcfg.yaml
   sudo ip addr add 10.60.0.1/32 dev upfgtp
   ```

3. **VM 重启后网络规则丢失** -- iptables 不持久。每次重启执行 `reload_host_config.sh`，或把 `net.ipv4.ip_forward=1` 写入 `/etc/sysctl.conf`

4. **OPc vs OP 类型不匹配** -- WebConsole 默认 OPc，UE 配 OP 则鉴权失败。两边 opType 必须一致

5. **MongoDB 需要 AVX** -- MongoDB 5.0+ 需要 CPU 支持 AVX。`lscpu | grep avx` 检查，不支持则降级到 4.4

6. **sed 替换 IP 误改 pfcp 段** -- `addr: 127.0.0.8` 同时匹配 gtpu 和 pfcp。用范围限定语法撤销 pfcp 部分

7. **TestNon3GPP 失败** -- 预期行为，需要额外 N3IWF 环境。不影响核心功能

8. **N3IWUE kill 后重连 SQN 不同步** -- 鉴权进行中被 kill 导致。用 v1.0.1+ 或手动重置 WebConsole 中 SQN

---

## 六、四个终端的作用

| 终端 | VM | 命令 | 角色 |
|------|-----|------|------|
| 1 | free5gc | `./run.sh` | 核心网大脑 -- 启动 AMF/SMF/UPF/NRF 等所有网元 |
| 2 | ueransim | `nr-gnb -c ...` | 仿真基站 -- N2/N3 连接，手机找不到网络就没它 |
| 3 | ueransim | `sudo nr-ue -c ...` | 仿真手机 -- 注册、鉴权、建 PDU 会话、创建 uesimtun0 |
| 4 | ueransim | `ping ...` | 验证 -- uesimtun0 出现后验证端到端联通 |

---

*2026年5月 | free5GC v4.2.2 + UERANSIM 实测*
