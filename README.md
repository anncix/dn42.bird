# DN42 Bird2 配置模板

适用于 [DN42](https://dn42.dev/) 网络的 [BIRD 2](https://bird.network.cz/) 路由守护进程配置模板，包含 OSPF 内部路由、iBGP 全互联以及 eBGP Peer 接入。

## 特性

- **双栈支持**：IPv4 + IPv6 同时运行
- **OSPF 内部路由**：自动发现 AS 内节点拓扑（OSPFv2 + OSPFv3）
- **iBGP Full Mesh**：AS 内部路由同步，支持多节点部署
- **eBGP 接入**：基于 DN42 官方模板，支持 Community 标记
- **ROA / RPKI 验证**：自动校验路由起源，防止劫持
- **安全过滤**：严格阻止外部 BGP 路由注入 OSPF

## 文件结构

```
.
├── bird.conf          # 主配置文件
├── ospf/
│   └── 0.0.0.0.conf   # OSPF 主干区域配置
├── peers/
│   └── README.md      # Peer 配置示例与说明
└── README.md          # 本文档
```

## 快速开始

### 1. 安装 BIRD 2

```bash
# Debian / Ubuntu (推荐 >= 2.0.8，需扩展下一跳支持)
sudo apt update && sudo apt install bird2

# 或使用官方源安装最新版
sudo apt install -y apt-transport-https ca-certificates wget lsb-release
sudo wget -O /usr/share/keyrings/cznic-labs-pkg.gpg https://pkg.labs.nic.cz/gpg
echo "deb [signed-by=/usr/share/keyrings/cznic-labs-pkg.gpg] https://pkg.labs.nic.cz/bird2 $(lsb_release -sc) main" | \
  sudo tee /etc/apt/sources.list.d/cznic-labs-bird2.list
sudo apt update && sudo apt install bird2 -y
```

### 2. 克隆本仓库到 `/etc/bird`

```bash
cd /etc
sudo mv bird bird.bak
sudo git clone https://github.com/anncix/dn42.bird.git bird
sudo mkdir -p bird/peers
```

### 3. 替换变量

编辑 [`bird.conf`](bird.conf) 文件头部，替换所有占位符：

| 占位符 | 示例值 | 说明 |
|--------|--------|------|
| `<OWNAS>` | `4242421234` | 你的 DN42 ASN |
| `<OWNIP>` | `172.20.1.1` | 你的 IPv4 路由器地址 |
| `<OWNIPv6>` | `fdb5:32:ad4a::1` | 你的 IPv6 路由器地址 |
| `<OWNNET>` | `172.20.1.0/27` | 你的 IPv4 网段 |
| `<OWNNETv6>` | `fdb5:32:ad4a::/48` | 你的 IPv6 网段 |
| `DN42_REGION` | `52` | 区域 Community（可选） |

### 4. 创建 Dummy 接口

```bash
sudo ip link add dn42-dummy type dummy
sudo ip link set dev dn42-dummy up
sudo ip addr add dev dn42-dummy <OWNIP>/32
sudo ip addr add dev dn42-dummy <OWNIPv6>/128
```

### 5. 配置 OSPF 内部接口

如果你使用 WireGuard 进行内部互联，确保隧道接口名称与 [`ospf/0.0.0.0.conf`](ospf/0.0.0.0.conf) 中的匹配：

```bird
interface "wg-*" {      # 或 "dn42_*"
    type ptp;
};
```

**重要**：WireGuard 的 `AllowedIPs` 必须包含 `ff02::5`（OSPFv3 组播地址）。

### 6. 添加 iBGP Peer（多节点时必需）

在 [`bird.conf`](bird.conf) 中找到 `iBGP Peer 实例` 部分，为每个内部节点添加：

```bird
protocol bgp ibgp_node2_v4 from ibgp_peers {
    neighbor 172.20.x.2 as OWNAS;
}
protocol bgp ibgp_node2_v6 from ibgp_peers {
    neighbor fdxx:xxxx:xxxx::2 as OWNAS;
}
```

### 7. 添加 eBGP Peer

为每个外部 Peer 在 `/etc/bird/peers/` 下创建配置文件：

```bash
sudo tee /etc/bird/peers/peer_example.conf << 'EOF'
protocol bgp example from dnpeers {
    neighbor 172.23.x.x as 424242xxxx;
}
protocol bgp example_v6 from dnpeers {
    neighbor fdxx:xxxx:xxxx::x as 424242xxxx;
}
EOF
```

根据实际链路质量调整 Community 值（详见 [`peers/README.md`](peers/README.md)）。

### 8. 验证并启动

```bash
# 语法检查
sudo bird -p -c /etc/bird/bird.conf

# 重载配置
sudo birdc configure

# 查看状态
sudo birdc show protocols
sudo birdc show ospf neighbor igp_ospf_v4
sudo birdc show route table master4
```

## 关键注意事项

1. **永远不要将外部 BGP 路由注入 OSPF**  
   本配置已在 OSPF export filter 中做了严格限制，请勿随意放宽，否则可能 [引爆整个 DN42](https://lantian.pub/article/modify-website/how-to-kill-the-dn42-network.lantian/)。

2. **iBGP 必须开启 `next hop self`**  
   否则内部节点无法正确到达外部 Peer 的下一跳地址。

3. **WireGuard 隧道必须配置 `type ptp`**  
   点对点链路不设置此类型会导致 OSPF DR/BDR 选举异常。

4. **Bird >= 2.16 推荐**  
   支持通过 IPv6 Link-Local 地址传递 IPv4 OSPF 数据，可省去额外的 IPv4 隧道地址配置。

## 常用命令

```bash
# 查看所有协议状态
birdc show protocols

# 查看 OSPF 邻居
birdc show ospf neighbor igp_ospf_v4
birdc show ospf neighbor igp_ospf_v6

# 查看 BGP 路由
birdc show route table master4 protocol <peer_name>
birdc show route table master6 protocol <peer_name>

# 查看特定前缀的详细信息
birdc show route for 172.20.0.0/14 all

# 查看路由的 Community
birdc show route where (64511, 24) ~ bgp_community

# 实时日志
birdc show log
```

## 参考链接

- [DN42 Wiki - Bird2](https://wiki.dn42.us/howto/Bird2)
- [DN42 Wiki - Bird Communities](https://dn42.dev/howto/Bird-communities)
- [DN42 实验网络介绍及注册教程](https://lantian.pub/article/modify-website/dn42-experimental-network-2020.lantian/)
- [MuskaNet DN42 配置模板](https://github.com/MuskaNet/DN42-bird-configuration)
- [DN42探究日记 - OSPF + iBGP](https://www.iyoroy.cn/archives/103/)

## 许可证

MIT
