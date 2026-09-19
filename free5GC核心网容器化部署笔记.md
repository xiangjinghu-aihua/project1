# free5GC 核心网容器化部署笔记（docker-compose 深度理解）

> 用官方 free5gc-compose 方案部署开源 5G 核心网。本文重点：**逐条读懂 docker-compose.yaml 每一行在干什么**、Docker 基本用法、以及容器网络的几个核心问题。

## 一、部署流程速览

| 步骤 | 命令 | 作用 |
|------|------|------|
|  装 Docker | `sudo apt-get install docker-ce docker-compose-plugin` 等 | 装引擎 + Compose v2 插件 |
|  取部署工程 | `git clone https://github.com/free5gc/free5gc-compose.git` | 部署外壳（编排文件+配置+证书） |
|  拉镜像 | `docker compose pull` | 下载全部网元镜像（free5gc v4.2.3 + mongo:4.4） |
| 启动 | `docker compose up -d` | 建网络、建卷、按依赖顺序起 17 个容器 |
|  验证 | `docker compose ps` / `docker logs amf \| tail -20` | 容器 Up + 日志里看到 gNB 的 NG Setup 握手 |
|  停止 | `docker compose down`（加 `-v` 连数据库卷一起删） | 一键整体停止清理 |

部署结果：17 个服务中 16 个容器稳定运行（tngf 启动后自行退出——是 Wi-Fi 接入网关），容器内模拟基站通过 N2 接口成功接入 AMF。

## 二、docker-compose.yaml

### 2.1 文件三大块

整个文件只有三个顶层键：

```yaml
services:   # 定义 17 个容器（一个网元一个，外加数据库、模拟基站、模拟终端）
networks:   # 定义容器间网络 privnet
volumes:    # 定义持久化数据卷 dbdata
```

### 2.2 镜像版本语法

```yaml
image: free5gc/amf:${FREE5GC_IMAGE_TAG:-v4.2.3}
```

`${变量:-默认值}` = 「如果设了环境变量 FREE5GC_IMAGE_TAG 就用它，否则用 v4.2.3」。所以想换版本不用改文件：`FREE5GC_IMAGE_TAG=v4.2.2 docker compose up`。

### 2.3 逐服务解读：为什么这么写

**① 数据库 db（mongodb）**

```yaml
image: mongo:4.4
command: mongod --port 27017
expose: ["27017"]                 # 只对容器网络内开放（不映射到宿主机）
volumes: [dbdata:/data/db]        # 数据写到持久卷，容器删了数据还在
networks: { aliases: [db] }       # 注册 DNS 别名 "db"
```

- **expose 是"文档性质"的**：同一网络内的容器本来就能访问对方任意端口；它不发布到宿主机，所以宿主机上自带的 MongoDB 8.0 不会和它冲突。
- 数据卷：MongoDB 存用户签约数据（你的 UE 信息），必须持久化。

**② 注册中心 nrf 与各控制面网元**

```yaml
environment: { DB_URI: mongodb://db/free5gc }   # 用容器名 "db" 当数据库地址
networks: { aliases: [nrf.free5gc.org] }        # 注册 DNS 别名
depends_on: [db]                                # 等 db 先启动
```

- `mongodb://db/free5gc`：**容器名直接当主机名用**，这就是 Docker 自定义网络的内嵌 DNS。
- `aliases`：给容器注册额外域名。free5GC 各网元配置文件里写死的是域名（如 `nrfUri: http://nrf.free5gc.org:8000`），compose 用别名把这些域名"翻译"成容器 IP——**换 IP 不用改配置**，这是容器化的核心好处。
- `depends_on`：只保证**启动顺序**（先 db 再 nrf），**不保证就绪**——网元启动时 NRF 没就绪就自己重试。

**③ amf：唯一固定 IP 的控制面网元**

```yaml
networks:
  privnet:
    ipv4_address: 10.100.200.16      # 固定 IP
    aliases: [amf.free5gc.org]
```

AMF 是基站的唯一入口（N2 接口），给固定 IP 是为了外部接入时地址确定可控。其余控制面网元都用动态 IP + 域名，足够。

**④ upf：最特殊的一个**

```yaml
command: bash -c "./upf-iptables.sh && ./upf -c ./config/upfcfg.yaml"
volumes:
  - ./config/upfcfg.yaml:/free5gc/config/upfcfg.yaml
  - ./config/upf-iptables.sh:/free5gc/upf-iptables.sh
cap_add: [NET_ADMIN]
```

- `bash -c "A && B"`：先执行 A（配置 NAT 的 iptables 脚本），**成功之后（&&）**才执行 B（启动 UPF 主程序）。顺序不能反——NAT 必须先配好。
- `cap_add: NET_ADMIN`：授予"网络管理"能力（详见 4.4）。
- 配置挂载语法 `宿主机路径:容器内路径`：配置文件留在宿主机，改配置不用重打镜像。

**⑤ ueransim 与 n3iwue：为什么挂 /dev/net/tun**

```yaml
cap_add: [NET_ADMIN]
devices: ["/dev/net/tun"]     # 把内核的 TUN 设备节点"借"给容器
```

TUN 设备用于创建虚拟网卡（UE 的 uesimtun0 就是靠它）。不挂载会报 can't open /dev/net/tun。

n3iwue 的特殊 command：

```yaml
command: bash -c "ip route del default && ip route add default via 10.100.200.1 dev eth0 metric 203 && sleep infinity"
```

三步：删掉原默认路由 → 把默认路由改指到核心网网关 10.100.200.1（模拟非3GPP终端的流量必须走 N3IWF 的 IPSec 隧道）→ `sleep infinity`（无限期睡觉，占住进程让容器不退出——它没有主程序）。

**⑥ tngf：为什么唯独它用 host 网络**

```yaml
network_mode: host     # 共享宿主机网络栈，不加入 privnet
```

TNGF 需要跑 IKE/IPsec（端口 500/4500、原始套接字），容器 NAT 环境下 IPsec 有 NAT 穿越问题，所以直接共享宿主机网络。这也是后面外接真实 UERANSIM 时 AMF/UPF 的改造思路。

**⑦ webui：唯一映射到宿主机的服务**

```yaml
ports: ["5000:5000", "2122:2122", "2121:2121"]
```

`宿主机端口:容器端口`。只有人（你）要从宿主机浏览器访问的东西才需要映射——所以只有 WebUI 用 `ports`，其他网元只有 `expose`（楼内分机）。**这也正是默认配置下外部 gNB 连不进 AMF 的原因**：AMF 的 38412 没有映射到宿主机。

### 2.4 networks 段：自定义网络 privnet

```yaml
networks:
  privnet:
    ipam:
      config:
        - subnet: 10.100.200.0/24    # 规划子网
    driver_opts:
      com.docker.network.bridge.name: br-free5gc   # 宿主机上的网桥命名
```

固定子网的意义：free5GC 的某些配置写死了这个网段（如 n3iwue 的网关 10.100.200.1、固定 IP 网元），所以不能让它随机分配。

### 2.5 volumes 段

```yaml
volumes:
  dbdata: {}
```

声明一个命名卷。不写 `{}` 时 Docker 也会按默认驱动创建——声明的作用是让它受 compose 生命周期管理（`down` 时保留、`down -v` 时删除）。

## 三、Docker 基本使用笔记

### 3.1 镜像（打包好的模板）

| 命令 | 作用 |
|------|------|
| `docker pull 镜像名:标签` | 下载镜像 |
| `docker images` | 列出本地镜像 |
| `docker rmi 镜像名` | 删除镜像 |
| `docker run 镜像` | 从镜像创建并运行一个容器 |

### 3.2 容器（镜像运行起来的实例）

| 命令 | 作用 |
|------|------|
| `docker run -d -p 8080:80 --name web nginx` | 后台运行 + 端口映射 + 命名 |
| `docker ps` / `docker ps -a` | 列出运行中 / 全部容器 |
| `docker logs 容器名` | 看日志 |
| `docker exec -it 容器名 bash` | 进入容器执行命令 |
| `docker stop / start / restart 容器名` | 停止/启动/重启 |
| `docker rm 容器名` | 删除容器 |

`docker run` 常用参数：

| 参数 | 含义 |
|------|------|
| `-d` | 后台运行（detached） |
| `-it` | 交互模式（-i 保持输入，-t 分配终端） |
| `-p 宿主机端口:容器端口` | 端口映射 |
| `-v 宿主机路径:容器路径` | 挂载文件/目录 |
| `-e 变量=值` | 注入环境变量 |
| `--name` | 给容器命名 |
| `--cap-add NET_ADMIN` | 授予特定能力 |
| `--privileged` | 授予全部能力（粗放，尽量少用） |
| `--device /dev/net/tun` | 挂载宿主设备节点 |

### 3.3 Docker Compose（批量编排）

| 命令 | 作用 |
|------|------|
| `docker compose up -d` | 创建网络/卷并按依赖启动全部服务 |
| `docker compose up -d amf` | 只启动指定服务（及其依赖） |
| `docker compose down` / `down -v` | 停止删除容器（+网络） / 连卷一起删 |
| `docker compose ps` | 看服务状态（默认只显示运行中，加 `-a` 全显） |
| `docker compose logs 服务名` | 看服务日志 |
| `docker compose pull` | 拉取所有服务的镜像 |

#

### 4.1 docker-compose 替你做了哪些"手动起容器很繁琐"的事？
  核心网部署的需求: 网元之间要互相找得到（配置里全是域名）
   compose : 自定义网络 privnet + 内嵌 DNS 别名
   对应文件里的位置: networks: 段 + 每个服务的 aliases:

核心网部署的需求: 启动要有先后（db 先于 nrf，nrf 先于 amf）
compose : depends_on 依赖图
对应文件里的位置: 每个服务的 depends_on:

核心网部署的需求: 每个网元要自己的配置文件（而且改配置不想重打镜像）
compose 的武器: volumes 挂载
对应文件里的位置: 每个服务的 volumes:

核心网部署的需求: 整体一键启停、可重复、环境一致
compose 的武器: up / down 幂等操作
对应文件里的位置: 你敲过的命令

手动 `docker run` 起 17 个容器，每次都要自己：

| 繁琐事 | compose 怎么替你干 |
|--------|-------------------|
| 创建容器网络 | 按 `networks:` 段自动建 privnet 网桥，`down` 时自动删 |
| 分配 IP / 固定 IP | 自动分配，`ipv4_address` 按需固定 |
| 配置 DNS 别名 | 自动把 `aliases` 注册进内嵌 DNS |
| 控制启动顺序 | 按 `depends_on` 依赖图排序启动、逆序停止 |
| 挂载每个配置文件 | 按 `volumes:` 列表逐个挂载 |
| 注入环境变量 | 按 `environment:` 逐个注入 |
| 端口映射 | 按 `ports:` 自动配置 |
| 批量启停 | `up` 一键全起、`down` 一键全停——手动方式要逐个 `docker stop` 17 次 |
| 可重复性 | 重复 `up` 幂等：已存在的网络/容器自动跳过，不会报错 |

一句话：**compose 把「运维脚本」写成了声明式的 YAML 清单**——你声明"要什么"，它去执行"怎么做"。

### 4.2 网元容器之间如何互相发现和通信？

分两层：

**第一层：Docker 内嵌 DNS（网络层）**。自定义 bridge 网络自带一个 DNS 服务器（127.0.0.11），容器名和 `aliases` 自动注册。所以容器里 ping `nrf.free5gc.org` 能自动解析成 nrf 容器的 IP。free5GC 配置里全写域名（`nrfUri: http://nrf.free5gc.org:8000`），compose 用别名把域名和容器绑定——**网元换 IP 不用改一行配置**。

**第二层：NRF 注册中心（应用层）**。即使 IP 能解析，5G 规范还要求：每个网元启动后去 NRF「报到」（登记自己能提供的服务），需要调别的网元时先去 NRF「查通讯录」。你在 AMF 日志里看到的 `OAuth2 setting receive from NRF: true` 就是 AMF 向 NRF 报到成功的记录。

### 4.3 自定义 bridge 网络和默认网络有什么不同？

| | 默认 bridge（docker0） | 自定义 bridge（本项目的 privnet） |
|---|---|---|
| DNS | ❌ 没有内嵌 DNS，容器名不能互相解析（只能靠已废弃的 --link） | ✅ 内嵌 DNS，容器名/alias 直接当主机名用 |
| 网络隔离 | 所有容器默认挤在一起 | 一个网络一个"局域网"，跨网络不通 |
| 子网 | 固定 172.17.0.0/16 | 自己指定（10.100.200.0/24） |
| 固定 IP | 不支持 | 支持 `ipv4_address` |
| 网桥命名 | 只能叫 docker0 | 可命名（br-free5gc），好辨识 |

free5GC 必须用自定义网络：既要 DNS（配置全靠域名），又要固定 IP 和固定网段。

### 4.4 为什么 UPF 需要 NET_ADMIN / privileged / 设备挂载？去掉会怎样？

**背景知识**：容器隔离靠 Linux namespace——每个容器有自己的网络命名空间（自己的网卡、路由表、iptables 规则），但**内核是共享的**（所以宿主机装的 gtp5g 内核模块，容器里能用）。修改网络命名空间属于特权操作，默认容器内的进程（即使是 root）**没有**这个权限——Linux 把 root 的权限拆成几十块叫 capabilities，`CAP_NET_ADMIN` 是「网络管理」那一块。

UPF 启动时要干两件特权事：

1. **创建 gtp5g 隧道接口**（GTP-U 数据通道，内核网络对象）→ 需要 NET_ADMIN
2. **配置 iptables NAT**（upf-iptables.sh，让 UE 能出公网）→ 需要 NET_ADMIN

**去掉这些配置会发生什么**（从轻到重）：

| 去掉什么 | 现象 |
|----------|------|
| `cap_add: NET_ADMIN`（UPF） | 创建 gtp5g 隧道接口失败 / iptables 报权限拒绝 → **控制面正常（UE 能注册），但数据面全断**——会话建了 ping 不通 |
| `devices: /dev/net/tun`（UERANSIM） | UE 无法创建 TUN 虚拟网卡 → 报 can't open /dev/net/tun，模拟终端没有网卡可用 |
| `network_mode: host`（tngf） | IPsec 在容器 NAT 下握手失败，Wi-Fi 接入路线不通 |

**为什么官方教程说 privileged 而 compose 文件用 NET_ADMIN**：官方教程原文"containers run privileged with root rights"（因为要建隧道接口）。privileged 等于把所有 capabilities 全开（约等于容器内 root ≈ 宿主机 root），简单粗暴但危险。compose 文件实际采用更精细的 `cap_add: NET_ADMIN`——只给网络管理权限，**最小权限原则**：需要什么给什么。

## 参考资料

- free5GC 官方指南：https://free5gc.org/guide/0-compose/
- free5gc-compose 仓库：https://github.com/free5gc/free5gc-compose
- 相关笔记：《free5GC-5G核心网知识总结.md》（网元与接口名词）、《ULCL分流知识总结.md》
