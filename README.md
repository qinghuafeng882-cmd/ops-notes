# ops-notes

个人运维知识库：Linux 运维 / RHCSA 学习记录、排障复盘、可复用的 Skill。

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

速查表：[W1 命令行与文本处理](cheatsheets/week1-cli-text.md) · [W2 系统管理](cheatsheets/week2-sysadmin.md) · [W3 存储](cheatsheets/week3-storage.md) · [W4 网络与安全](cheatsheets/week4-network-security.md)

## 日常用法

1. 做完实验 → 把终端记录交给 AI，按 `skills/lab-wrapup.md` 整理成笔记
2. 笔记里的回忆题按 D+1 / D+3 / D+7 自测，在笔记底部打勾
3. 同一件事做第二遍 → 考虑写成新的 Skill
