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

## SSH

```bash
ssh-keygen                              # 默认 ed25519，私钥 600
ssh-copy-id fqh@node2                   # 公钥追加到对方 ~/.ssh/authorized_keys
ssh fqh@node2 'hostname; id'            # 远程执行
ssh -v fqh@node2                        # 看认证过程（Offering public key）
ssh -o PubkeyAuthentication=no fqh@node2   # 只用密码，测密码登录是否被禁
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub   # 服务端主机指纹
ssh-keygen -R node2                     # 删除 known_hosts 旧记录（IP 也有记录要一起删）
```

| | 主机密钥 | 用户密钥 |
|---|---|---|
| 证明 | 服务器是真的 | 用户是本人 |
| 私钥 | 服务端 `/etc/ssh/ssh_host_*_key` | 客户端 `~/.ssh/id_ed25519` |
| 公钥 | 客户端 `~/.ssh/known_hosts` | 服务端 `~/.ssh/authorized_keys` |
| 出错 | `REMOTE HOST IDENTIFICATION HAS CHANGED`（直接拒绝，不给 yes） | 一直要密码 / `Permission denied (publickey)` |

权限：`~` 别人不可写、`~/.ssh` 700、`authorized_keys` 600、私钥 600。错了 → 服务端日志 `bad ownership or modes`，客户端只是回退到要密码。

### sshd 配置

```bash
sudo tee /etc/ssh/sshd_config.d/10-lab.conf << 'EOF'
PermitRootLogin no
PasswordAuthentication no
EOF
sudo sshd -t && sudo systemctl reload sshd     # reload 不断开已有会话
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauth'
sudo journalctl -u sshd -n 30 --no-pager
```

⚠️ **第一个值生效**（与 systemd 相反）：`Include sshd_config.d/*.conf` 在主配置开头，drop-in 按文件名顺序 → `05-` 胜过 `10-` 胜过主配置。
⚠️ `sshd -T` 读的是磁盘文件，不是运行中的 sshd → 改/删配置后一定 reload。
⚠️ 改 sshd 前留一个会话或控制台；不要改你登录入口那台机器。

RHEL 10 默认 `PermitRootLogin prohibit-password`（`sshd -T` 显示 `without-password`）。

| 现象 | 原因 |
|---|---|
| 确认指纹后 `Connection closed`，再连 `Connection reset by peer` | LoginGraceTime 120 秒超时 → PerSourcePenalties 暂时封这个 IP（日志 `penalty:`），等一会儿自动恢复 |
| `sudo ssh/rsync` 又问指纹、要密码 | 以 root 身份连接，用 `/root/.ssh` |

## 传文件

```bash
scp file fqh@node2:/tmp/                # 上传
scp fqh@node2:/etc/os-release /tmp/     # 下载
scp -r dir fqh@node2:/tmp/
rsync -av src/ fqh@node2:/tmp/dst/      # -a 归档（含 -r），只传变化的部分
```

| 写法 | 结果 |
|---|---|
| `rsync -av a  host:/tmp/d/` | `/tmp/d/a/...`（带目录本身） |
| `rsync -av a/ host:/tmp/d/` | `/tmp/d/...`（只传目录里的内容） |

| 参数 | 作用 |
|---|---|
| `-a` | = `-rlptgoD`：递归、链接、权限、mtime、属组、属主（接收端 root 才保留）、设备 |
| `-v` / `--stats` | 列出文件 / 统计（Literal = 真传的，Matched = 用对方旧块拼的） |
| `-z` | 压缩（慢网络用） |
| `-P` | 进度 + 断点续传 |
| `-n` | 预演 |
| `--delete` | 删除目标多余文件（镜像）→ **必须先 `-n`** |
| `--exclude='*.log'` | 排除 |
| `-c` | 按校验和比较（默认只比大小 + mtime） |

原理：quick check（大小 + mtime）决定传哪些文件 → delta 算法（切块、滚动校验）决定传文件的哪几块。本机复制不用 delta。两端都要装 rsync。

⚠️ 不要用 sudo 整个拷 `/etc/ssh`：会把主机私钥带走。
⚠️ bash `for i in 1 2 3` 用空格；`1,2,3` 是一个词，逗号只在 `{1,2,3}` 里有效。
