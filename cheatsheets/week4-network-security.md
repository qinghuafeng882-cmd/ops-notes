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

## firewalld

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone / --get-active-zones
sudo firewall-cmd --list-all                    # 运行时；加 --permanent 看永久
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload                      # 永久规则生效；丢掉只在运行时的规则
sudo firewall-cmd --add-port=8080/tcp && sudo firewall-cmd --runtime-to-permanent   # 先试再存
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.12/32" port port="8080" protocol="tcp" accept'
sudo firewall-cmd --permanent --remove-rich-rule='...（原样照抄）'
sudo firewall-cmd --permanent --zone=trusted --add-source=192.168.56.12
sudo firewall-cmd --get-services                # 可用 service 名
sudo nft list ruleset                           # 只看不改
```

| 写法 | 现在 | reload / 重启后 |
|---|---|---|
| `--add-...` | ✅ | ❌ |
| `--permanent --add-...` | ❌（要 `--reload`） | ✅ |
| `--runtime-to-permanent` | ✅ | ✅ |

zone 匹配：**来源地址 > 网卡 > 默认 zone（public）**。

| zone | 行为 |
|---|---|
| public | 默认；只放行列出的服务，其余拒绝 |
| trusted | 全放行 |
| drop | 全丢弃（客户端超时） |
| block | 全拒绝 |

- 永久配置：`/etc/firewalld/zones/public.xml`（上一版 `.old`）；service 定义 `/usr/lib/firewalld/services/`，自定义放 `/etc/firewalld/services/`。
- service 优先于 port：可读、可含多个端口。
- rich rule 写 `family="ipv4"` 就只管 IPv4。
- ⚠️ 不要删 `ssh` 服务；不要绕过 firewalld 直接改 nft/iptables。

### 客户端报错 → 哪一层

| Linux 客户端看到 | 原因 |
|---|---|
| `No route to host` | 防火墙拒绝 |
| `Connection refused` | 放行了，但没程序监听 |
| 超时 | 被丢弃 / 网络不通 |

新版 curl 都显示 `Could not connect to server` → 用 `timeout 3 bash -c '</dev/tcp/node1/8080' && echo OK`。Windows 浏览器被拒也显示超时。最终在服务端查：`ss -tlnp` → `firewall-cmd --list-all` → SELinux。

## SELinux（上）

```bash
getenforce; sestatus
sudo setenforce 0 / 1                  # Permissive / Enforcing，只改运行时
grep ^SELINUX= /etc/selinux/config     # 永久，重启生效
id -Z; ps -eZ | grep httpd; ls -Z / ls -Zd 目录
sudo semanage fcontext -l -C           # 本机自定义的规则（policycoreutils-python-utils）
sudo semanage fcontext -l | grep '/var/www'
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"   # 加规则（带 / 和引号）
sudo semanage fcontext -d "/srv/web(/.*)?"                          # 删规则
sudo restorecon -Rv /srv/web           # 按规则库重打；无输出 = 已一致
sudo chcon -t httpd_sys_content_t 文件  # 临时，restorecon 会改回
sudo ausearch -m AVC -ts recent        # 拒绝记录
sudo sealert -a /var/log/audit/audit.log   # 需要 setroubleshoot-server
```

| 模式 | 拦截 | 记录 |
|---|---|---|
| Enforcing | ✅ | ✅ |
| Permissive | ❌ | ✅（`permissive=1`，每一步都记） |
| Disabled | ❌ | ❌ |

| 动作 | 标签 |
|---|---|
| 新建 / cp | 继承目标目录 |
| **mv** | **保留原标签**（inode 不变） |
| restorecon | 按规则库 |
| chcon | 直接改，不改规则库 → 不持久 |

AVC = Access Vector Cache。读法：`denied { read }` 动作、`scontext` 谁、`tcontext` 访问什么、`tclass` 对象类别。

| 常见类型 | 用于 |
|---|---|
| httpd_t | httpd 进程 |
| httpd_sys_content_t | 网页内容 |
| user_home_t | 家目录文件 |
| var_t | /srv 等的默认 |

⚠️ 排障可以临时 `setenforce 0` 确认原因，确认后**马上 `setenforce 1`**，修标签而不是关 SELinux。
⚠️ 改 httpd 配置先 `httpd -t`。

## SELinux（下）：端口、boolean

```bash
sudo ausearch -m AVC -ts recent | audit2why     # 翻译原因（建议不一定最好）
sudo semanage port -l | grep -w http_port_t
sudo semanage port -a -t http_port_t -p tcp 8888   # 已登记的端口用 -m
sudo semanage port -d -t http_port_t -p tcp 8888
getsebool -a | grep httpd
sudo semanage boolean -l | grep httpd_enable_homedirs   # (当前值, 持久值) + 说明
sudo setsebool -P httpd_enable_homedirs on      # 不加 -P 重启后恢复
```

| 拒绝类型 | AVC 特征 | 修法 |
|---|---|---|
| 文件标签 | tclass=file/dir | `restorecon` / `semanage fcontext -a` + `restorecon` |
| 端口标签 | tcp_socket `{ name_bind }` | `semanage port -a` |
| boolean | 标签都对仍被拒 | `setsebool -P` |

⚠️ audit2why 可能建议 `nis_enabled`、`httpd_unified` 这种大范围开关 → 选范围最小的修法。
⚠️ 换端口顺序：`semanage port` → `firewall-cmd` → `Listen` → `httpd -t` → restart。失败重试太多次 → `systemctl reset-failed httpd`。

个人主页三道关：`chmod 711 ~` + `public_html` 755 → `userdir.conf` 里 `UserDir public_html` → `httpd_enable_homedirs`。

## 启动流程

| 阶段 | 配置 | 坏了 |
|---|---|---|
| BIOS / UEFI | 固件 | 找不到引导设备 |
| GRUB2 | `/etc/default/grub`、`/boot/loader/entries/` | `grub>` |
| 内核 + initramfs（dracut） | `rd.*` 参数 | dracut 紧急 shell |
| systemd | fstab、单元文件 | 紧急模式 |
| 默认 target | `systemctl get-default` | — |

```bash
systemd-analyze; systemd-analyze blame | head
cat /proc/cmdline                               # 本次开机的参数
sudo grubby --info=DEFAULT / --default-kernel
sudo grubby --update-kernel=ALL --args="x=1"    # 改 BLS 条目，重启生效；DEFAULT 只改默认内核
sudo grubby --update-kernel=ALL --remove-args="quiet"
sudo vi /etc/default/grub && sudo grub2-mkconfig -o /boot/grub2/grub.cfg   # 等待时间等
sudo grub2-editenv - list
sudo grub2-editenv - unset menu_auto_hide       # 菜单被自动隐藏时（应急：按住 Shift / 按 Esc）
sudo lsinitrd | grep -i lvm                     # initramfs 内容（要 sudo）
sudo dracut -f                                  # 重新生成当前内核的 initramfs
```

- `rd.lvm.lv=rlm/root`：给 initramfs 的参数，先激活根 LV。`root=... ro` 先只读挂根。
- 内核不认识的参数不报错，转交给 init → 拼错只是不起作用。
- 条目：正常内核 + `0-rescue`（通用救援内核）。

## 救援与重置 root 密码

GRUB：选条目 → `e` → linux 行末尾加参数 → `Ctrl+X`（只影响这一次开机）。

| 停在哪 | 参数 | 根分区 | 要 root 密码（RHEL 10） | 用途 |
|---|---|---|---|---|
| rescue.target | `systemd.unit=rescue.target` | 读写，fstab 都挂 | ✅ | 不启动普通服务做维护 |
| emergency.target | `systemd.unit=emergency.target` | 只读，只挂根 | ✅ | fstab 写坏 |
| dracut shell | `rd.break` | 只读挂在 `/sysroot` | ✅ | 根挂载问题 |
| init=/bin/bash | `init=/bin/bash` | `/` 只读 | ❌ | **忘记 root 密码** |
| rescue 内核 | 菜单选 `0-rescue` | 正常 | 正常登录 | initramfs 坏 / 缺驱动 |
| 临时 permissive | `enforcing=0` | 正常 | 正常登录 | 确认是否 SELinux 导致 |

```bash
# 忘记 root 密码（RHEL 10）
init=/bin/bash                    # GRUB e 加在 linux 行末尾
mount -o remount,rw /
passwd
touch /.autorelabel               # 先斜杠后点！不是 ./autorelabel
exec /sbin/init                   # 不能 exit（bash 是 PID 1）

# rd.break 版本（要 root 密码）
mount -o remount,rw /sysroot; chroot /sysroot; passwd; touch /.autorelabel; exit; exit

# dracut shell 里手动激活根 LV
lvm lvs; lvm lvchange -ay rlm/root; exit

# 开机后确认
sudo journalctl --list-boots | tail -4          # relabel 会多一次短开机
sudo journalctl -b -1 | grep -i autorelabel
ls -Z /etc/shadow                               # shadow_t
```

⚠️ 漏了 relabel → shadow 是 `unlabeled_t` → 所有用户"密码错误"（`unix_chkpwd` 被拦）→ `enforcing=0` 登录 → `restorecon -v /etc/shadow`。
⚠️ `journalctl -b 0` 是当前开机，重启后编号整体前移；稳定引用用 boot ID。
⚠️ 排障用证据：`--list-boots`、journal、`/root/.bash_history`，不靠盯屏幕和记忆。

| 动作 | SELinux 标签 |
|---|---|
| 新建 / cp | 继承目标目录 |
| mv / **cp -a** | 保留原标签 |

## 周测补充（W4 模拟测试 ①）

```bash
man -k 关键词                     # 搜不到先 sudo mandb
nmcli connection --help           # 子命令：add 新建 / modify 修改 / edit 交互
sudo nmcli con modify enp0s8 ipv4.addresses "192.168.56.12/24,192.168.56.22/24"   # 第一个是主地址
ip route get 192.168.56.11        # 看源地址 src
sudo hostnamectl hostname node2.lab.example
ssh -o BatchMode=yes ops@node2 hostname   # 需要密码就直接失败 → 证明是密钥登录
curl -m5 --interface 192.168.56.22 http://node1:9000/   # 指定源地址测防火墙
ls -Zd /var/www/html              # 拿正确的目录当参照抄类型名
```

| 坑 | 处理 |
|---|---|
| `useradd: group ops exists` | 默认建同名私有组；组已有人用 → `-g ops`，不要删 |
| `useradd -r` | 系统账户，不建家目录，人用的账户不要加 |
| ssh-copy-id 在对方禁止密码时 | 用不了 → 借已有通道传公钥，sudo 安装：chown + 700/600 + restorecon |
| `sudo cmd \| ssh ...` | sudo 只作用于前面的 cmd，ssh 仍是当前用户 |
| `PermitRootLogin no` 后日志写 Failed password | 本机 `su - root` 同密码做对照 |
| `"/srv/exam(/.*)?"` 不加引号 | bash 先解析括号报语法错 |
| `sudo cat 文件 << EOF` | 不会写入，用 tee |
| 两台 VM 同时重启卡住 | 一台起来再起另一台 |

httpd 新路径：`/etc/httpd/conf.d/x.conf` 写 `Alias /exam /srv/exam` + `<Directory /srv/exam> Require all granted </Directory>` → `httpd -t` → reload → semanage fcontext + restorecon。

验证设计三问：最终结果是什么？什么命令能直接看到？有没有看起来对其实不对？证据：效果 > 运行状态 > 配置；"只允许"正反都测。
