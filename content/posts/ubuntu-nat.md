+++
title = 'Ubuntu NAT 流量转发教程：SNAT、DNAT 与排错'
date = 2026-10-13T16:00:00+08:00
draft = false
tags = ['Ubuntu', 'Linux', 'NAT', 'SNAT', 'DNAT', 'iptables']
categories = ['网络技术']
+++

NAT（Network Address Translation，网络地址转换）会在数据包经过网关时改写源地址或目标地址。它常用于让多个内网设备共享出口地址，也可将外部连接转发到内网服务。

本文用 Ubuntu 作为双网卡路由器，演示出站 MASQUERADE 和入站 DNAT。示例网段仅供说明，请按自己的接口、网段和防火墙策略替换后再执行。

## 1. 先区分几种 NAT

- **SNAT**：改写数据包的源 IP，常用于固定公网出口地址。
- **MASQUERADE**：SNAT 的动态地址形式，自动使用出口接口当前地址，适合 DHCP 或地址可能变化的出口。
- **DNAT**：改写数据包的目标 IP 或端口，常用于端口转发。

NAT 负责地址转换，不等同于防火墙。NAT 规则改变了地址，并不代表应该允许所有转发流量。

## 2. 示例网络与数据包路径

假设 Ubuntu 有一块内网接口和一块外网接口：

| 设备/接口 | 示例地址 | 用途 |
| --- | --- | --- |
| Ubuntu 内网接口 `enp1s0` | `10.0.0.1/24` | 内网网关 |
| Ubuntu 外网接口 `ens3` | 由上游分配 | 访问互联网 |
| 内网客户端 | `10.0.0.10/24` | 默认网关为 `10.0.0.1` |

出站连接大致经过以下路径：

```text
内网客户端 10.0.0.10
				|
				| 目标是互联网，源地址仍为 10.0.0.10
				v
Ubuntu 内网接口 enp1s0
				|
				| PREROUTING -> 路由判断 -> FORWARD
				| POSTROUTING: MASQUERADE 改写源地址
				v
Ubuntu 外网接口 ens3（使用该接口的出口地址）
				|
				v
		 互联网

回包到达 Ubuntu 后，conntrack 根据连接跟踪记录反向还原地址，
再经 FORWARD 转回 10.0.0.10。
```

要让这条路径成立，客户端必须把 Ubuntu 设为默认网关；Ubuntu 必须开启 IP 转发；FORWARD 防火墙规则必须允许这类流量。

## 3. 确认接口、路由和防火墙

先不要猜接口名，使用以下命令检查机器现状：

```bash
ip -br addr
ip route
ip route get 1.1.1.1
```

`ip route get` 输出中的 `dev` 通常是访问互联网时使用的出口接口。查看当前 iptables 规则：

```bash
sudo iptables -S
sudo iptables -t nat -S
sudo sysctl net.ipv4.ip_forward
```

Ubuntu 通常通过 `iptables-nft` 将 iptables 命令映射到 nftables 后端。若服务器使用 UFW、云厂商安全组或已有自定义 nftables 规则，应先确认规则链和默认策略；不要直接清空规则，也不要在远程 SSH 会话中盲目修改默认策略，以免断开管理连接。

## 4. 配置出站 NAT（内网访问互联网）

### 4.1 开启 IPv4 转发

临时开启，重启后会恢复：

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

需要持久化时，创建 `/etc/sysctl.d/99-ip-forward.conf`：

```conf
net.ipv4.ip_forward = 1
```

加载并确认：

```bash
sudo sysctl --system
sysctl net.ipv4.ip_forward
```

### 4.2 允许已建立连接回包和指定内网出站

以下示例使用变量，先根据 `ip -br addr` 和 `ip route` 设置真实接口名与内网网段：

```bash
LAN_IF=enp1s0
WAN_IF=ens3
LAN_CIDR=10.0.0.0/24

sudo iptables -A FORWARD \
	-i "$WAN_IF" -o "$LAN_IF" \
	-m conntrack --ctstate ESTABLISHED,RELATED \
	-j ACCEPT

sudo iptables -A FORWARD \
	-i "$LAN_IF" -o "$WAN_IF" \
	-s "$LAN_CIDR" \
	-j ACCEPT
```

第二条允许该内网网段向外转发。生产环境应按需要限制目的地址、协议和端口，并核对现有 FORWARD 默认策略。

### 4.3 添加 MASQUERADE

```bash
sudo iptables -t nat -A POSTROUTING \
	-s "$LAN_CIDR" -o "$WAN_IF" \
	-j MASQUERADE
```

如果出口公网地址固定，也可以用 `SNAT` 明确指定地址，例如：

```bash
sudo iptables -t nat -A POSTROUTING \
	-s "$LAN_CIDR" -o "$WAN_IF" \
	-j SNAT --to-source 198.51.100.2
```

`198.51.100.2` 是文档示例地址，不能直接作为真实公网地址使用。地址动态变化时用 MASQUERADE；固定地址时优先用 SNAT。

### 4.4 配置内网客户端并测试

客户端需要满足：

1. IP 地址属于 `10.0.0.0/24`。
2. 默认网关指向 `10.0.0.1`。
3. DNS 服务器可用。

从客户端依次检查网关、IP 连通性和域名解析：

```bash
ip route
ping -c 3 10.0.0.1
ping -c 3 1.1.1.1
getent hosts example.com
```

能 ping 通 IP 但域名解析失败，通常应检查 DNS，而不是继续增加 NAT 规则。

## 5. 配置入站 DNAT（端口转发）

假设外部访问 Ubuntu 的 TCP 8080，需要转到内网服务器 `10.0.0.10:80`。将以下变量替换为实际接口：

```bash
WAN_IF=ens3
LAN_IF=enp1s0
SERVER_IP=10.0.0.10

sudo iptables -t nat -A PREROUTING \
	-i "$WAN_IF" -p tcp --dport 8080 \
	-j DNAT --to-destination "$SERVER_IP":80

sudo iptables -A FORWARD \
	-i "$WAN_IF" -o "$LAN_IF" \
	-p tcp -d "$SERVER_IP" --dport 80 \
	-m conntrack --ctstate NEW,ESTABLISHED,RELATED \
	-j ACCEPT

sudo iptables -A FORWARD \
	-i "$LAN_IF" -o "$WAN_IF" \
	-p tcp -s "$SERVER_IP" --sport 80 \
	-m conntrack --ctstate ESTABLISHED,RELATED \
	-j ACCEPT
```

同时确认：

- 云安全组或上游防火墙允许外部 TCP 8080。
- Ubuntu 上游路由能把回包送回 Ubuntu。
- 内网服务器的默认网关指向 Ubuntu，或配置了到外部客户端网段的回程路由。
- 服务本身正在监听内网地址的 TCP 80 端口。

若回程流量不经过 Ubuntu，DNAT 请求可能到达服务器，但响应无法正确返回。不要为了“让它工作”就随意对所有流量做 SNAT；先检查回程路由和 conntrack 状态。

## 6. 保存规则并验证持久化

用 `iptables` 添加的规则通常不会自动跨重启保存。Ubuntu 可使用 `iptables-persistent`：

```bash
sudo apt update
sudo apt install iptables-persistent
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

保存前先检查规则，避免把临时调试规则或不需要的开放规则一并持久化：

```bash
sudo iptables -S
sudo iptables -t nat -S
```

如果项目统一使用原生 nftables，应通过 `/etc/nftables.conf` 管理规则，不要同时维护两套互相冲突的配置。对远程服务器修改防火墙前，准备好控制台或带外恢复途径。

## 7. 排错顺序

### 查看转发和 NAT 规则计数

```bash
sudo iptables -vnL FORWARD
sudo iptables -t nat -vnL POSTROUTING
sudo iptables -t nat -vnL PREROUTING
```

重复发起测试流量后观察对应规则的 packet/byte 计数是否增长。计数为零通常说明接口、来源网段、目的端口或流量路径与规则不匹配。

### 抓包确认数据走向

```bash
sudo tcpdump -ni enp1s0 host 10.0.0.10
sudo tcpdump -ni ens3 host 1.1.1.1
```

按实际接口和目标地址修改命令。若内网接口能看到请求、外网接口看不到，检查 IP 转发和 FORWARD 链；外网能看到请求但看不到回包，检查上游、路由和安全组。

### 检查 conntrack

```bash
sudo conntrack -L
```

如系统未安装该命令，可安装 `conntrack` 软件包。连接跟踪表可帮助确认内核是否记录了 NAT 会话；不要在生产环境随意清空整张连接跟踪表。

### 常见原因

- 内网客户端默认网关不是 Ubuntu。
- `net.ipv4.ip_forward` 仍为 `0`。
- 接口名或来源网段填错。
- FORWARD 链默认丢弃，但缺少对应允许规则。
- 只有 POSTROUTING NAT，却忘了配置 FORWARD。
- 云安全组、UFW 或上游路由器拦截流量。
- DNAT 目标服务器的回程路由不经过 Ubuntu。
- IP 连通正常但 DNS 配置错误。

## 8. 删除示例规则

不要使用 `iptables -F` 清空整台服务器的防火墙。应先查看规则并删除本次添加的精确规则；例如：

```bash
sudo iptables -t nat -D POSTROUTING \
	-s 10.0.0.0/24 -o ens3 -j MASQUERADE
```

删除前将接口和网段替换成添加规则时使用的值，并检查 FORWARD 及 DNAT 规则是否也需要移除。

## 9. 小结

出站共享公网地址通常需要：正确的默认路由、启用 IPv4 转发、允许所需 FORWARD 流量，以及在 POSTROUTING 使用 MASQUERADE/SNAT。端口转发通常需要 PREROUTING DNAT、受限的 FORWARD 放行规则和正确的回程路由。逐段验证数据包路径，比不断叠加规则更容易定位问题。
