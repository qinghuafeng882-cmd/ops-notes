# Linux 基础：发行版、Shell、man 手册、FHS 目录结构

- 日期：2026-09-21（W1-D1）
- 考点：RHCSA EX200 —— 使用基础工具；查阅系统文档（man、/usr/share/doc）
- 环境：Rocky Linux 10（VirtualBox），主机名 node1
- 状态：完成
- 来源：根据课程对话整理，无完整终端记录

## 目标

搞清楚系统的层次和目录布局，学会只靠系统自带帮助查资料（RHCSA 考场不能联网）。

## 关键概念

- Linux 只是内核；发行版 = 内核 + GNU 工具 + 包管理器 + 软件。
- 红帽系上下游：Fedora → CentOS Stream（RHEL 上游）→ RHEL → Rocky / Alma（用 RHEL 源码重新编译，**二进制兼容**）。
- man 章节：1 用户命令，5 **配置文件格式**，8 管理员命令。查配置文件写法就加 `5`。
- `man -k` 只搜 NAME 行，**用单个关键词**，不要拼短语。
- usr-merge：`/bin /sbin /lib /lib64` 是指向 `/usr/...` 的符号链接。
- 手册页也是 RPM 包装的：谁的软件谁带手册；`passwd(5)`、`hier(7)` 这类通用文档来自 `man-pages` 包，最小化安装默认没有。

## 关键步骤

```bash
# 认清系统
cat /etc/os-release; uname -r; echo $SHELL

# 修复 man 环境
sudo dnf install -y man-pages man-db
sudo mandb                       # 建 man -k 的索引
man -k password                  # 单个关键词

# 反查手册页属于哪个包
man -w 5 fstab                   # 手册页文件路径
rpm -qf $(man -w 5 fstab)        # 属于哪个包
dnf provides '*/hier.7*'         # 缺的文件该装哪个包

# 看磁盘布局
df -h
lsblk
swapon --show; free -h
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| `/tmp` 重启后是否保留 | 写文件到 `/tmp`、`/var/tmp` 后重启再 `ls` | 两个都在（`/tmp` 不是 tmpfs） | ✅ 两个都在 |
| man 环境修复 | `man 7 hier` | 能打开 | ✅（自述已跑通） |

## 踩坑

- `man 5 passwd` 报 No manual entry → `man-pages` 包没装 → `dnf install man-pages`。
- `man 5 fstab` 能打开，误以为 man-pages 已装 → `fstab(5)` 随挂载工具的包自带，不能用它判断 → 用 `rpm -q man-pages` 确认。
- `man -k "change password"` 返回 nothing appropriate → 没建索引 + 短语匹配不上 NAME 行 → `sudo mandb`，改用单个关键词。
- 以为 `/tmp` 重启就清空 → Rocky 上 `/tmp` 在根分区磁盘上，由 `systemd-tmpfiles-clean.timer` 按年龄清理（`/tmp` 10 天、`/var/tmp` 30 天，规则在 `/usr/lib/tmpfiles.d/tmp.conf`）→ 重启实验验证。

## 本机磁盘布局（lsblk 读图）

- `sda1` 1M 无挂载点：GPT + BIOS 引导的 BIOS boot 分区。
- `/boot` 是普通分区 `sda2`，根在 LVM 上（GRUB 早期阶段读 LVM 不方便）。
- 根逻辑卷 `rlm-root` 25G 横跨 `sda3`(18G) 和 `sdb`(10G) —— LVM 打破物理盘边界。
- `lab--vg-data--lv`：设备映射名里原名的 `-` 被转义成 `--`，即 VG `lab-vg`、LV `data-lv`。
- swap 显示为 `/dev/dm-1`，和 `/dev/mapper/rlm-swap` 是同一设备（后者是符号链接）。
- `sdd` 整盘直接格式化挂载，没有分区表（能用但不规范）。

## 卡点 / 没想明白的

- 无

## 回忆题

1. （概念）Rocky Linux 和 RHEL 是什么关系？CentOS Stream 在 RHEL 上游还是下游？
<details><summary>答案</summary>

Rocky 用 RHEL 公开源码重新编译，二进制兼容；CentOS Stream 现在是 RHEL 的上游。
</details>

2. （操作）忘了改时区的命令，只记得和 timezone 有关，怎么找？`man 5 passwd` 查不到又该怎么办？
<details><summary>答案</summary>

`man -k timezone`（找到 timedatectl）；查不到先 `rpm -q man-pages`，没装就 `dnf install man-pages && mandb`。
</details>

3. （排障）同事说 Rocky 上 `/tmp` 的文件重启就没了，你怎么验证？
<details><summary>答案</summary>

`mount | grep -w /tmp` 看是不是 tmpfs；写一个测试文件后重启查看；清理规则看 `/usr/lib/tmpfiles.d/tmp.conf` 和 `systemctl list-timers | grep tmpfiles`。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
