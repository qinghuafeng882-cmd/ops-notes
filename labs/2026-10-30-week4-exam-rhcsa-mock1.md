# 第 4 周周测 / RHCSA 模拟测试 ①

- 日期：2026-10-30
- 考点：RHCSA EX200 —— 重置 root 密码、网络、名字解析、SSH 密钥、sshd 加固、firewalld、SELinux（文件上下文 / 端口 / boolean）、内核参数、LVM（W3 复习）
- 环境：Rocky Linux 10.2（VirtualBox），node1 + node2；考前快照 `before-w4exam`
- 状态：完成。**闭卷**（可用 man / --help，卡住 10 分钟后可看笔记）
- 成绩：闭卷 71/92（77%），最终 86/92（93%）；第 4 题单独计分（完成）。及格线 70%
- 来源：根据课程对话和终端记录整理

## 批改结果

| 题 | 内容 | 闭卷 | 最终 | 说明 |
|---|---|---|---|---|
| 1 | node2 重置 root 密码 | 8 | 8 | init=/bin/bash；relabel 那次开机 32 秒 |
| 2 | node2 追加 IPv4/IPv6、主机名 | 4 | 8 | `modify` 靠 `--help`、`hostnamectl` 靠 man；先覆盖后追加导致主地址变成 .22，自查修正 |
| 3 | /etc/hosts 的 web2 | 8 | 8 | |
| 4 | ops 免密登录 node2（node2 禁止密码登录） | — | 完成 | 单独计分；ssh-copy-id 走不通、身份切错两次 |
| 5 | node1 禁止 root 远程登录 | 6 | 8 | 以为默认值就是禁止 root |
| 6 | https + 只允许 .12 访问 9000 | 8 | 8 | `MISSING_FAMILY` 自己改对；用 `--interface` 正反验证 |
| 7 | Alias /exam + SELinux 持久 | 3 | 8 | httpd 配置、semanage 语法、正则引号三处卡住 |
| 8 | httpd 监听 8090 | 8 | 8 | 顺序完全正确，httpd 没停过 |
| 9 | boolean + 所有内核加参数 | 6 | 8 | grubby 靠 man 找到 |
| 10 | examlv 200M ext4 按 UUID 挂载 | 5 | 7 | lvcreate 语法；忘建挂载点（findmnt --verify 拦下）；三步顺序反了；没备份 fstab |
| A | 8090 访问失败三层排查 | 3.5 | 3.5 | 缺每层的现象 |
| B | rd.break 失败、全员登录失败 | 4 | 4 | 缺 unlabeled_t / unix_chkpwd 机制 |
| C | 四个"改了不生效" | 4.5 | 4.5 | |
| D | 公钥手动安装的三要素 | 3 | 3 | 漏了 SELinux 标签 ssh_home_t |

## 关键步骤（正确做法）

```bash
# 1 重置 root（RHEL 10）：GRUB e → linux 行加 init=/bin/bash → Ctrl+X
mount -o remount,rw /; passwd; touch /.autorelabel; exec /sbin/init

# 2 追加地址（+ 追加；不带 + 是覆盖；多个地址逗号分隔，写在前面的是主地址）
sudo nmcli con modify enp0s8 +ipv4.addresses 192.168.56.22/24 +ipv6.addresses fd00:56::22/64
sudo nmcli con up enp0s8
ip route get 192.168.56.11                        # src 是主地址
sudo hostnamectl hostname node2.lab.example

# 4 node2 禁止密码登录时手动装公钥
su - ops; ssh-keygen; exit                          # node1，ops 生成密钥
sudo cat /home/ops/.ssh/id_ed25519.pub | ssh fqh@node2 'cat > /tmp/ops.pub'   # 以 fqh 身份走已有通道
sudo mkdir -p /home/ops/.ssh; sudo cp /tmp/ops.pub /home/ops/.ssh/authorized_keys    # node2
sudo chown -R ops:ops /home/ops/.ssh; sudo chmod 700 /home/ops/.ssh; sudo chmod 600 /home/ops/.ssh/authorized_keys
sudo restorecon -Rv /home/ops/.ssh
ssh -o BatchMode=yes ops@node2 hostname          # node1，以 ops 身份；BatchMode 证明是密钥登录

# 5
echo "PermitRootLogin no" | sudo tee /etc/ssh/sshd_config.d/10-w4test.conf
sudo sshd -t && sudo systemctl reload sshd

# 6
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.12" port port="9000" protocol="tcp" accept'
sudo firewall-cmd --reload
curl -m5 --interface 192.168.56.22 http://node1:9000/   # node2 换源地址测"该挡的挡"

# 7
sudo tee /etc/httpd/conf.d/exam.conf << 'EOF'
Alias /exam /srv/exam
<Directory /srv/exam>
  Require all granted
</Directory>
EOF
sudo httpd -t && sudo systemctl reload httpd
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/exam(/.*)?"
sudo restorecon -Rv /srv/exam

# 8
sudo semanage port -l | grep -w 8090; sudo semanage port -a -t http_port_t -p tcp 8090
sudo firewall-cmd --add-port=8090/tcp && sudo firewall-cmd --runtime-to-permanent
echo "Listen 8090" | sudo tee /etc/httpd/conf.d/port8090.conf; sudo httpd -t && sudo systemctl reload httpd

# 9
sudo setsebool -P httpd_can_network_connect on
sudo grubby --update-kernel=ALL --args="w4exam=1"

# 10
sudo lvcreate -n examlv -L 200M datavg; sudo mkfs.ext4 /dev/datavg/examlv
sudo mkdir -p /examdata; sudo cp -a /etc/fstab /etc/fstab.bak.$(date +%F-%H%M%S)
echo "UUID=$(sudo blkid -s UUID -o value /dev/datavg/examlv) /examdata ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo findmnt --verify; sudo systemctl daemon-reload; sudo mount -a; df -hT /examdata
```

## 验证（重启后）

| 题 | 证据 | 结果 |
|---|---|---|
| 1 | 控制台 root 登录；`--list-boots` 有 32 秒短开机；shadow_t | ✅ |
| 2 | `ip -br addr` 4 个地址；src .12；`node2.lab.example`；node1 ping 两个新地址 | ✅ |
| 3 | `getent hosts web2`、`ping web2` | ✅ |
| 4 | `ssh -o BatchMode=yes ops@node2` 有输出；node2 `passwordauthentication no` | ✅ |
| 5 | `permitrootlogin no`；fqh 能登录、root 被拒；本机 `su - root` 同一密码可登录（对照） | ✅ |
| 6 | `--list-all`；`.12` 成功、`.22` 4 ms 失败 | ✅ |
| 7 | `curl /exam/`；`semanage fcontext -l -C`；chcon 改坏后 restorecon 恢复为 httpd_sys_content_t | ✅ |
| 8 | `curl :8090`；`ss -tlnp`；`http_port_t` | ✅ |
| 9 | `(on, on)`；`/proc/cmdline` 有 w4exam=1；`grubby --info=ALL` 两个条目都有 | ✅ |
| 10 | `df -hT /examdata` ext4 182M；fstab 以 `UUID=` 开头 | ✅ |

## 踩坑

- `nmcli con add enp0s8 ...` → `invalid <setting>.<property>`；`con edit enp0s8 ...` → `Unexpected argument` → 修改已有连接是 `modify`（`nmcli connection --help` 查到）。
- 不带 `+` 的 `modify ipv4.addresses` 是**覆盖**；先覆盖再 `+` 加回 → 地址顺序变了，**第一个是主地址**，`ip route get` 显示 src 变成 .22（.12 标 secondary）→ 会让第 6 题按源地址放行失效 → 逗号列表重排。
- `man -k hostname` → `nothing appropriate` → man 索引没建 → `sudo mandb`。
- `useradd ops` → `group ops exists` → 默认建同名私有组（USERGROUPS_ENAB），旧 ops 组（GID 1001）有 4 个成员 → 不动它，`-g ops`。`-r` 是系统账户（不建家目录），不能用。`-u 3001` 笔误撞了 alex。
- ssh-copy-id 需要先能登录对方（密码或已有密钥）；node2 禁止密码 → 用不了。以 fqh 执行时送的是 fqh 的公钥；以 ops 执行时 ssh 也是 ops 身份 → 都失败。**动手前看提示符：我是谁、在哪台机器。**
- 第一次 sudo 时出现 "We trust you have received the usual lecture" → 说明是这个用户第一次用 sudo（当时在 ops 里）。
- 用 sudo 建的文件属于 root → authorized_keys 必须 chown；`.ssh` 由策略按名字给 `ssh_home_t`，cp 继承，restorecon 是保险（mv 会带来 user_tmp_t）。
- 默认 `PermitRootLogin prohibit-password` 允许 root 用密钥登录，不等于禁止 root。
- `PermitRootLogin no` 时 sshd 给 PAM 假密码（推测，源码层面），日志也写 `Failed password` → 用 `su - root` 同一密码做对照证明是策略拒绝。
- `ssh -T` ≠ `sshd -T`；`ssh_config.d`（客户端）≠ `sshd_config.d`；改 sshd 用 reload 比 restart 稳。
- rich rule 有地址必须写 `family`（`MISSING_FAMILY`）。
- `sudo cat 文件 << EOF` 不会写入（cat 去读文件）→ 用 tee。
- httpd 配置原理补课：`IncludeOptional conf.d/*.conf`；`Alias` 把 URL 路径映射到目录；主配置 `<Directory />` 默认 `Require all denied`，新目录要 `Require all granted`。
- `semanage fcontext -R ...` → 缺动作 `-a`；类型拼成 `http_sys_content_t`；`/srv/exam(/.*)?` 不加引号 → `-bash: syntax error near unexpected token '('`（shell 先解析括号）。找类型名：`ls -Zd /var/www/html` 照着抄。
- grubby man 没有 EXAMPLES（"一般都有"说法不对）→ 读 SYNOPSIS + DESCRIPTION（`DEFAULT or ALL`）+ OPTIONS。
- `lvcreate examlv -L ...` → 没选项名的参数被当成 VG → `-n examlv`；题目是 200M 不是 300M。
- 写完 fstab 忘建 `/examdata` → `findmnt --verify` 报 `[E] unreachable on boot required target` → 三步检查救了一次；顺序应为 verify → daemon-reload → mount -a。ext4 显示 182M（日志、inode 表、超级块备份）。
- 9000 本来就是 `http_port_t`。
- 两台虚拟机同时重启，node2 卡在内核 3.8 秒（IMA/EVM 初始化行，无报错）→ 强制重启后正常；卡住那次没写进持久 journal（早期日志只在内存）→ 推测宿主资源不足，以后一台起来再起另一台。
- 拼写：`crul`、`BatchMatch`、`/etc/cmdline`、`httpd_conf`、`rpm -rf`。

## 卡点 / 没想明白的（闭卷暴露的薄弱点）

- **命令语法记不住**：nmcli modify、hostnamectl、semanage fcontext（-a、类型名、引号）、grubby、lvcreate -n、httpd Alias 配置。原理都知道，卡在拼写和参数位置。
- 不会设计验证命令 → 三个问题：最终结果是什么 / 什么命令能直接看到 / 有没有"看起来对其实不对"；证据等级：效果 > 运行状态 > 配置文件；"只允许"类要求正反都测。
- 分析题 A 没写每层的现象；D 漏了 SELinux 标签。

## 回忆题

1. （概念）为什么 `nmcli con modify enp0s8 ipv4.addresses X` 之后再 `+` 加回原地址，第 6 题的按源地址放行会出问题？
<details><summary>答案</summary>

不带 + 是覆盖，原地址被删；再加回来排在后面变成 secondary。同网段多个地址时第一个是主地址，主动发起连接用主地址做源地址，于是 node2 用 .22 去连 node1，被只放行 .12 的规则挡住。用 `ip route get` 看 src，用逗号列表 `"192.168.56.12/24,192.168.56.22/24"` 排序。
</details>

2. （操作）node2 禁止密码登录，配置 node1 的 ops 免密登录 node2 的 ops。
<details><summary>答案</summary>

node1：`su - ops; ssh-keygen; exit`；以 fqh：`sudo cat /home/ops/.ssh/id_ed25519.pub | ssh fqh@node2 'cat > /tmp/ops.pub'`。node2：`mkdir -p /home/ops/.ssh`、`cp` 到 `authorized_keys`、`chown -R ops:ops`、`chmod 700/600`、`restorecon -Rv`。验证：node1 以 ops `ssh -o BatchMode=yes ops@node2 hostname`。三要素：属主、权限、ssh_home_t 标签；失败看 node2 `journalctl -u sshd` 和 `ausearch -m AVC`。
</details>

3. （排障）node2 `curl http://node1:8090/` 失败，三层各看什么现象、用什么命令？
<details><summary>答案</summary>

① 服务没监听：`Connection refused`（用 `/dev/tcp` 才看得出）→ `ss -tlnp | grep 8090`；没监听再看 `systemctl status httpd`、`journalctl -u httpd`。② SELinux 端口：httpd 起不来或不监听 8090，AVC `{ name_bind }` → `semanage port -l | grep -w 8090`、`ausearch -m AVC`，修 `semanage port -a -t http_port_t -p tcp 8090`。③ 防火墙：`No route to host` → `firewall-cmd --list-all`，修 `--permanent --add-port=8090/tcp` + `--reload`。
</details>

4. （概念）nmcli con mod、firewall --permanent、semanage fcontext -a、grubby --args 改完都"没变化"，共同规律是什么？各自怎么生效？
<details><summary>答案</summary>

配置（文件 / 规则库）和运行状态是两回事，改配置后要单独一步应用：`nmcli con up`、`firewall-cmd --reload`、`restorecon -Rv`、重启（`/proc/cmdline` 是本次开机的参数）。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
