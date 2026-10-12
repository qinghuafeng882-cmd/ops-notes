# 速查表 W2：用户、权限、软件包、进程、systemd、日志

> 只放最常用的写法和易错点。详细原理和踩坑见 `labs/` 对应笔记。

## 用户与组

```bash
groupadd -g 4000 webadmins
useradd -u 3001 -G webadmins alex          # -G 附加组，-g 主组
useradd -s /sbin/nologin tom               # 不能交互登录
echo 'alex:RedHat2026' | chpasswd          # 设密码（不要用 useradd -p 明文）
usermod -aG wheel alex                     # 追加附加组（漏 -a 会清空原有附加组）
gpasswd -a alex dev / gpasswd -d alex dev  # 加入 / 移出一个组
usermod -L / -U alex                       # 锁定 / 解锁
userdel -r alex                            # 连家目录一起删
id alex                                    # 验证附加组用 id
```

| 文件 | 内容 |
|---|---|
| `/etc/passwd` | 用户名:x:UID:GID:注释:家目录:Shell |
| `/etc/shadow` | 密码哈希（`$6$`/`$y$` 开头；`!` = 锁定）+ 有效期 |
| `/etc/group` | 组名:x:GID:附加成员 |
| `/etc/login.defs` | 新用户默认值（只在 useradd 时读取，不影响已有用户） |

## 密码有效期

```bash
chage -l alex
chage -M 60 -d 0 alex        # 最长 60 天 + 下次登录强制改（先设密码再执行）
chage -m 7 -W 7 -E 2026-12-31 alex
```

`login.defs` 里是 Tab 分隔：`grep '^PASS_MAX_DAYS' /etc/login.defs | cat -A`。

## sudo

```bash
visudo -f /etc/sudoers.d/webadmins
# %webadmins ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart httpd
chmod 0440 /etc/sudoers.d/webadmins
visudo -c
sudo -l -U alex
```

⚠️ 组名加 `%`；主机字段后面是 `=` 不是 `:`；命令写绝对路径（`which systemctl`）；文件名**不能含 `.`**、不能以 `~` 结尾，否则被忽略。

## 切换身份

`su - alex`（带完整环境）、`sudo -u alex 命令`（以某用户执行一条命令，测权限常用）。

## 权限

| 数字 | 权限 | 用途 |
|---|---|---|
| 755 / 644 | rwxr-xr-x / rw-r--r-- | 目录、程序 / 普通文件 |
| 700 / 600 | 私人目录 / 私钥、sshd_config | |
| 440 | r--r----- | sudoers |

- 判断顺序：属主 → 属组 → 其他，**只匹配一次**。
- 目录：r 列文件名，w 增删改名，x 进入。**删文件看目录的 w+x。**
- 访问文件需要路径每层目录都有 x → `namei -l`。

```bash
chmod 750 dir; chmod u+x,g-w,o= file
chmod -R u=rwX,g=rX,o= /data     # 大写 X：只给目录加 x
chown -R nginx:nginx /var/www    # 只有 root 能改属主
umask                            # 文件 666、目录 777 再屏蔽
```

| 特殊权限 | 数值 | 作用 | 例子 |
|---|---|---|---|
| SUID | 4 | 以文件属主身份运行 | `/usr/bin/passwd` |
| SGID | 2 | 目录：新文件继承属组 | 协作目录 |
| Sticky | 1 | 目录：只能删自己的文件 | `/tmp` |

协作目录：`chgrp webadmins /srv/webdata; chmod 2770`（再加 Sticky 用 `3770`）。

## 软件包

| 用途 | 命令 |
|---|---|
| 包里的配置文件 | `rpm -qc openssh-server` |
| 文件属于哪个包 | `rpm -qf /usr/bin/htpasswd` |
| 缺命令该装哪个包 | `dnf provides '*/bin/dig'` |
| 校验包是否被改动 | `sudo rpm -V 包名`（无输出 = 正常；普通用户执行会出现 `?`） |
| 补丁记录 | `rpm -q --changelog openssh` |
| 历史 / 回滚 | `dnf history`；`dnf history undo ID` |
| 恢复被改坏的配置文件 | `rm 文件 && dnf reinstall 包` |
| 只下载 rpm | `dnf download tree zip` |

⚠️ **删除操作不要加 `-y`**，先看 Transaction Summary（删 zip 曾连带删掉 192 个包）。`rpm -q*` 跟包名，不是命令名。

### 仓库

```bash
sudo createrepo_c /opt/w2repo          # 下载完所有包之后再生成元数据
sudo tee /etc/yum.repos.d/w2repo.repo << 'EOF'
[w2repo]
name=w2repo
baseurl=file:///opt/w2repo
enabled=1
gpgcheck=0
EOF
sudo dnf install zip --disablerepo='*' --enablerepo=w2repo   # 依赖也必须在这个仓库里
sudo dnf reinstall zip --disablerepo='*' --enablerepo=w2repo # 已装时用 reinstall 验证仓库
dnf info installed zip | grep -i repo                        # From repo
sudo dnf config-manager --set-disabled w2repo
dnf repolist all
dnf clean all                                                # 改了仓库后
```

⚠️ 文件必须 `.repo` 结尾；`gpgcheck` 拼错会被静默忽略。

### Flatpak

```bash
sudo flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak search calculator
sudo flatpak install -y flathub org.gnome.Calculator
flatpak list
sudo flatpak uninstall -y org.gnome.Calculator
sudo flatpak uninstall -y --unused
```

## 进程

```bash
ps -eo pid,user,%cpu,%mem,rss,cmd --sort=-%cpu | head -6   # 第一行是表头
pgrep -au lisa                     # 某用户的进程（pgrep 只接受一个模式）
kill PID → 等几秒 → kill -9 PID    # 先 TERM 再 KILL
sudo pkill -u lisa                 # 操作别人的进程要 sudo
nice -n 19 /opt/backup.sh          # 以低优先级启动
renice -n 10 -p PID                # 改运行中进程（跟 PID）
jobs; fg %1; Ctrl-Z; bg %1; kill %1
nohup cmd > log 2>&1 &
```

| 看什么 | 判断 |
|---|---|
| load average | 和 `nproc` 比；第一个数 > 最后一个数 = 上升 |
| `wa` 高、`us` 低 | 磁盘瓶颈 |
| 内存 | 看 `available` |
| D 状态 | 等 I/O，kill 不掉，查存储 |
| Z 僵尸 | kill 无效，处理父进程 |

信号：1 HUP 重载、2 INT、9 KILL、15 TERM、19 STOP、20 TSTP。

## systemd

```bash
systemctl enable --now httpd          # 现在启动 + 开机自启
systemctl disable --now httpd
systemctl reload httpd                # 不中断服务
systemctl status httpd                # 看 Active 行和最后几行日志
systemctl is-active / is-enabled httpd
systemctl --failed
systemctl mask / unmask httpd         # 链接到 /dev/null，彻底禁止
systemctl cat httpd                   # 原文件 + 所有 drop-in
systemctl show -p MainPID httpd
systemctl daemon-reload               # 新建或修改单元文件后
systemctl get-default / set-default multi-user.target
```

### 自定义服务

```ini
# /etc/systemd/system/w2demo.service
[Unit]
Description=W2 Demo
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/w2demo.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

脚本要 `chmod 755`，直接在 `/usr/local/bin` 创建（不要从家目录 mv，SELinux 标签不对）。

| Restart= | TERM 终止 | KILL 杀死 |
|---|---|---|
| on-failure | 不重启 | 重启 |
| always | 重启 | 重启 |

### drop-in（修改已有服务）

```bash
sudo mkdir -p /etc/systemd/system/httpd.service.d
printf '[Service]\nRestart=on-failure\n' | sudo tee /etc/systemd/system/httpd.service.d/override.conf
sudo systemctl daemon-reload && systemctl cat httpd
```

`systemctl edit httpd`：内容写在**顶部两段注释之间**，写在下面会被丢弃。

### 服务起不来

`status=203/EXEC` = 执行不了 → 查执行权限、第一行 `#!/bin/bash`、SELinux 标签。

## 日志

```bash
journalctl -u sshd -n 50 --no-pager
journalctl -u sshd --since today -p warning
journalctl -b -p err                  # 本次开机 err 及以上
journalctl -b -1                      # 上次开机（需持久化）
journalctl -f                         # 实时跟踪
journalctl --list-boots / --disk-usage
logger -p local3.warning "msg"
```

优先级：0 emerg … 3 err、4 warning、5 notice、6 info、7 debug。`-p err` 包含更严重的级别。

### 持久化 journal（RHEL 10）

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\n' | sudo tee /etc/systemd/journald.conf.d/persistent.conf
sudo systemctl restart systemd-journald && sudo journalctl --flush
```

Rocky 10 上 `/etc/systemd/journald.conf` 不存在，默认在 `/usr/lib/systemd/journald.conf`。

### rsyslog

```bash
echo 'local3.*    /var/log/w2app.log' | sudo tee /etc/rsyslog.d/w2app.conf
sudo rsyslogd -N1 && sudo systemctl restart rsyslog
```

`/var/log/messages` 综合日志，`/var/log/secure` 登录与 sudo（权限 600）。

### logrotate

```
# /etc/logrotate.d/w2app
/var/log/w2app.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    postrotate
        /usr/bin/systemctl kill -s HUP rsyslog.service >/dev/null 2>&1 || true
    endscript
}
```

```bash
sudo logrotate -d /etc/logrotate.d/w2app   # 模拟（新日志第一次会显示不需要轮转）
sudo logrotate -f /etc/logrotate.d/w2app   # 强制（单独跑不读全局 dateext）
```

### SSH 暴力破解统计

```bash
sudo grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

### 排障顺序

`systemctl status` → `journalctl -u 服务 -n 50` → 服务自己的日志 → 配置检查（`httpd -t`、`nginx -t`、`sshd -t`、`visudo -c`、`rsyslogd -N1`）→ SELinux（`/var/log/audit/audit.log`）

## 配置目录的文件命名规则

| 目录 | 要求 |
|---|---|
| `/etc/systemd/system/xxx.service.d/` | `*.conf` |
| `/etc/systemd/journald.conf.d/` | `*.conf` |
| `/etc/rsyslog.d/` | `*.conf` |
| `/etc/yum.repos.d/` | `*.repo` |
| `/etc/sudoers.d/` | 不能含 `.`，不能以 `~` 结尾 |

写错了都**不报错，直接被忽略**。改完一定验证。
