# 速查表 W4：网络、SSH、防火墙、SELinux、启动

> 只放最常用的写法和易错点。详细原理和踩坑见 `labs/` 对应笔记。

## nmcli

```bash
nmcli device                           # 网卡 ↔ 当前连接
nmcli con show                         # 所有连接（profile）
nmcli con show enp0s8 | grep -E 'ipv4|ipv6'
sudo nmcli con mod enp0s8 +ipv4.addresses 192.168.56.111/24   # + 追加，- 删除，不带 = 覆盖
sudo nmcli con mod enp0s8 ipv4.method manual ipv4.addresses 192.168.56.11/24 ipv4.gateway 192.168.56.1 ipv4.dns "223.5.5.5 119.29.29.29"
sudo nmcli con mod enp0s8 ipv6.method manual ipv6.addresses fd00:56::11/64
sudo nmcli con mod enp0s3 ipv4.ignore-auto-dns yes             # DHCP 连接用自己的 DNS
sudo nmcli con up enp0s8               # mod 之后必须 up 才生效
sudo nmcli con reload                  # 手动改了 .nmconnection 后
sudo nmcli con add con-name lab type ethernet ifname enp0s8 ipv4.method manual ipv4.addresses 192.168.56.20/24
sudo nmcli con mod enp0s8 connection.autoconnect yes           # 开机自动连接
```

⚠️ 先查连接名（不一定等于网卡名）；命令里**漏写连接名** → `unknown connection 'ipv6.method'`。

| 操作 | 现在生效 | 重启后 |
|---|---|---|
| `nmcli con mod` | ❌（要 `con up`） | ✅ |
| `nmcli con up` | ✅ | ✅ |
| `ip addr add` | ✅ | ❌（`con up` 也会清掉） |
| 手改 `.nmconnection` | ❌（要 `con reload` + `con up`） | ✅ |

| `method` | 含义 |
|---|---|
| auto | DHCP |
| manual | 静态，必须带 addresses |
| disabled | 不配 |

- 配置文件：`/etc/NetworkManager/system-connections/*.nmconnection`，权限 600，别人可读写会被 NM 忽略。
- RHEL 10 **不支持 ifcfg**（`network-scripts/ifcfg-*` 不读），teaming 删除 → bond。

## 查看

| 看什么 | 命令 |
|---|---|
| 地址 | `ip -br addr` |
| 路由 / 网关 | `ip route`（`default via`） |
| 监听端口 | `ss -tlnp` |
| DNS | `cat /etc/resolv.conf`（NM 生成，不要手改） |

## 主机名与解析

```bash
hostnamectl                             # 查看
sudo hostnamectl set-hostname node1.lab.example
grep ^hosts /etc/nsswitch.conf          # files dns myhostname
getent hosts web1                       # 和程序走同一路径
dig web1 / nslookup web1                # 只查 DNS，不看 /etc/hosts（bind-utils）
```
