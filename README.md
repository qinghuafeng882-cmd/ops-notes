# ops-notes

个人运维知识库：Linux 运维 / RHCSA 学习记录、排障复盘、可复用的 Skill。

> **关于这个仓库怎么来的**：实验环境是两台 Rocky Linux 10 虚拟机（VirtualBox，node1 服务端 / node2 客户端），所有实验和每周考核都由我本人在虚拟机上动手完成。课程由 AI（Claude）按 RHCSA EX200 大纲讲授，笔记由 AI 根据我的终端记录、报错和答题整理成文，所以提交作者大多显示为 Claude。笔记里的「踩坑」都是实验中真实发生过的故障和排查过程。

## 目录

| 目录 | 放什么 | 命名 |
|---|---|---|
| `labs/` | 每次实验一页笔记 | `YYYY-MM-DD-主题.md`，如 `2026-10-12-lvm-extend.md` |
| `troubleshooting/` | 过往排障经历的脱敏复盘 | `主题.md`，如 `l2-loop-remediation.md` |
| `skills/` | 写给 AI 执行的固定流程 | `动作名.md`，如 `lab-wrapup.md` |
| `cheatsheets/` | 每周一页速查表（周测开卷时用） | `weekN-主题.md` |

每个目录下的 `_template.md` 是模板，新建笔记时复制一份。

## 硬规矩

**单位的任何数据都不进这个仓库**：真实主机名、IP、拓扑、配置、截图、账号、内部系统名。
排障复盘只写脱敏后的现象、思路和方法。

## 学习进度（Linux 运维 12 周课程）

| 周 | 日期 | 主题 | 笔记 |
|---|---|---|---|
| W1 | 09-21 | 发行版、man、FHS | [链接](labs/2026-09-21-linux-basics-fhs-man.md) |
| W1 | 09-22 | 文件、inode、链接、find | [链接](labs/2026-09-22-files-inode-links-find.md) |
| W1 | 09-22 | vim | [链接](labs/2026-09-22-vim.md) |
| W1 | 09-26 | 重定向、管道、grep | [链接](labs/2026-09-26-redirect-pipe-grep.md) |
| W1 | 09-27 | sed、awk | [链接](labs/2026-09-27-sed-awk.md) |
| W1 | 09-28 | 第 1 周周测 | [链接](labs/2026-09-28-week1-exam.md) |
| W2 | 10-04 | 用户、组、sudo | [链接](labs/2026-10-04-users-groups-sudo.md) |
| W2 | 10-05 | 权限 | [链接](labs/2026-10-05-permissions.md) |
| W2 | 10-06 | 软件包 | [链接](labs/2026-10-06-packages-dnf-rpm-flatpak.md) |
| W2 | 10-07 | 进程 | [链接](labs/2026-10-07-processes.md) |
| W2 | 10-10 | systemd | [链接](labs/2026-10-10-systemd.md) |
| W2 | 10-11 | 日志 | [链接](labs/2026-10-11-logs-journald-rsyslog-logrotate.md) |
| W2 | 10-12 | 第 2 周周测 | [链接](labs/2026-10-12-week2-exam.md) |
| W3 | 10-13 | 磁盘、分区、文件系统、挂载 | [链接](labs/2026-10-13-disk-partition-fs-mount.md) |
| W3 | 10-16 | LVM | [链接](labs/2026-10-16-lvm.md) |
| W3 | 10-17 | swap、RAID | [链接](labs/2026-10-17-swap-raid.md) |
| W3 | 10-18 | NFS、autofs | [链接](labs/2026-10-18-nfs-autofs.md) |
| W3 | 10-19 | tar 压缩、fstab 紧急救援 | [链接](labs/2026-10-19-tar-and-fstab-rescue.md) |
| W3 | 10-20 | 第 3 周周测（7.8/8） | [链接](labs/2026-10-20-week3-exam.md) |
| W4 | 10-21 | 网络：nmcli、名字解析 | [链接](labs/2026-10-21-networking-nmcli.md) |
| W4 | 10-22 | SSH：密钥登录、sshd 加固、scp/rsync | [链接](labs/2026-10-22-ssh.md) |
| W4 | 10-23 | firewalld | [链接](labs/2026-10-23-firewalld.md) |
| W4 | 10-24 | SELinux（上）：模式、上下文、fcontext | [链接](labs/2026-10-24-selinux-contexts.md) |
| W4 | 10-25 | SELinux（下）：端口、boolean；启动流程与 GRUB | [链接](labs/2026-10-25-selinux-boolean-port-boot.md) |
| W4 | 10-28 | 救援：重置 root 密码、修复无法启动 | [链接](labs/2026-10-28-rescue-root-password.md) |
| W4 | 10-30 | 第 4 周周测 / RHCSA 模拟 ①（闭卷 77%，最终 93%） | [链接](labs/2026-10-30-week4-exam-rhcsa-mock1.md) |
| W5 | 11-02 | shell 脚本（上）：执行方式、引号、参数、退出码、if/test | [链接](labs/2026-11-02-shell-script-basics.md) |
| W5 | 11-03 | shell 脚本（下）：循环、while read、case、函数、巡检脚本 | [链接](labs/2026-11-03-shell-loops-inspect.md) |
| W5 | 11-04 | 计划任务：cron 与 systemd timer | [链接](labs/2026-11-04-cron-systemd-timer.md) |
| W5 | 11-05 | 排障（一）：CPU、内存、负载 | [链接](labs/2026-11-05-troubleshoot-cpu-mem-load.md) |
| W5 | 11-06 | 排障（二）：磁盘满、inode、删了不释放、iostat | [链接](labs/2026-11-06-troubleshoot-disk.md) |
| W5 | 11-07 | 排障（三）：网站打不开——服务、端口、防火墙、SELinux、网络 | [链接](labs/2026-11-07-troubleshoot-service-network.md) |

## 补缺口小课：RAID 与虚拟化

RHCSA 大纲不覆盖、但运维面试常问的两块，课外单独学习，下面是两次动手实验的完整报告。

| 日期 | 实验 | 报告 |
|---|---|---|
| 10-22 | mdadm 搭建 RAID 5 + 热备盘，模拟坏盘、自动重建、换盘 | [链接](labs/2026-10-22-raid5-mdadm-hotspare.md) |
| 11-04 | KVM/libvirt：嵌套虚拟化失败排查（Hyper-V 独占 AMD-V）与 TCG 方案 | [链接](labs/2026-11-04-kvm-libvirt-tcg.md) |

速查表：[W1 命令行与文本处理](cheatsheets/week1-cli-text.md) · [W2 系统管理](cheatsheets/week2-sysadmin.md) · [W3 存储](cheatsheets/week3-storage.md) · [W4 网络与安全](cheatsheets/week4-network-security.md) · [W5 脚本与计划任务](cheatsheets/week5-scripts-scheduling.md)

## 日常用法

1. 做完实验 → 把终端记录交给 AI，按 `skills/lab-wrapup.md` 整理成笔记
2. 笔记里的回忆题按 D+1 / D+3 / D+7 自测，在笔记底部打勾
3. 同一件事做第二遍 → 考虑写成新的 Skill
