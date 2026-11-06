# 排障（二）：磁盘满、inode 满、删了不释放、iostat

- 日期：2026-11-06（A3）
- 考点：面试高频"磁盘满了怎么处理""df 和 du 对不上""有空间却写不进去""怎么判断磁盘是瓶颈"
- 环境：Rocky Linux 10.2（VirtualBox），node1，1 vCPU；用 100M 的 loop 镜像（ext4）做实验，不碰系统盘；sysstat 12.7.6、lsof 4.98
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

用"数据块 / inode / 目录项"模型解释三种磁盘故障，亲手制造并修复；会用 iostat、pidstat 判断磁盘是不是瓶颈。

## 关键概念

| 组成 | 存什么 | 用完 / 出问题 | 看 |
|---|---|---|---|
| 数据块 | 文件内容 | ENOSPC（Error NO SPaCe，`No space left on device`） | `df -h` |
| inode（index node） | 属性和数据块位置，**不含文件名**，每个文件一个 | 块还有空也报 ENOSPC | `df -i` |
| 目录项 | 文件名 → inode 号 | rm 删的是它 | `ls -i` |

- **数据块释放的条件：没有文件名指向它（链接数 0）并且没有进程开着它。**
- df 问文件系统计数器；du 顺着目录树把文件加起来 → 已删除但仍被打开的文件、被挂载点盖住的文件：df 算，du 不算。
- ext4 默认给 root 保留 5%（按 Block count 算）：普通用户写满后 root 还能写，减少碎片；纯数据盘可 `tune2fs -m 1`。xfs 没有这个机制。
- ext4 的 inode 数在格式化时固定（小文件系统每 4K 一个），inode 表也占空间；xfs 动态分配，很少 inode 满。
- inode 满时：新文件建不了，**已有文件还能追加** → 服务"部分正常"，迷惑性强。常见：会话文件、缓存、邮件队列。
- 清空正在写的日志：`: > 文件` / `truncate -s 0 文件`，不要 rm；已经 rm 了：`lsof +L1` 找进程 → 重启进程或 `: > /proc/PID/fd/N`。
- logrotate 默认 mv + 通知程序重新打开（postrotate reload）；程序不支持重新打开 → `copytruncate`。
- `df -h` 每个数向上取整，精确值用 `df -B1K`。
- `mount` 遇到普通文件会自动建 loop 设备，`-o loop` 可省略。
- iostat（sysstat 包）：看 **r_await / w_await**（每个请求的耗时，含排队）和 **aqu-sz**（队列长度）；%util 对 SSD / NVMe / RAID 不可靠（能并行，100% 不等于满）。
- `--hdd-opts direct`（O_DIRECT）绕过页缓存；`dsync`（O_DSYNC）每次写等落盘 → f/s（flush 请求）≈ w/s。
- **wa 高 ≠ 磁盘慢**：cgroup 限速 1M 时 wa 97%，设备 w_await 0.4ms、%util 0%（请求卡在限速层，没到设备）。

## 关键步骤

```bash
sudo fallocate -l 100M /var/tmp/a3.img; sudo mkfs.ext4 -q /var/tmp/a3.img
sudo mkdir -p /mnt/a3; sudo mount -o loop /var/tmp/a3.img /mnt/a3; losetup -a
sudo tune2fs -l /var/tmp/a3.img | grep -E '^Block count|^Reserved block count|^Block size|^Inode count'
sudo chown fqh: /mnt/a3; dd if=/dev/zero of=/mnt/a3/fill1 bs=1M; sudo dd if=/dev/zero of=/mnt/a3/fill2 bs=1M
sudo du -xh --max-depth=1 / 2>/dev/null | sort -h | tail -6
sudo find / -xdev -type f -size +50M -exec ls -lh {} + 2>/dev/null

sudo mkfs.ext4 -q -N 2000 /var/tmp/a3.img          # inode 满
touch /mnt/a3/sess/s{1..3000} 2>&1 | tail -2; df -i /mnt/a3
du --inodes -x --max-depth=1 /mnt/a3 | sort -n; find /mnt/a3/sess -type f -delete

dd if=/dev/zero of=/mnt/a3/app.log bs=1M count=60   # 删了不释放
sleep 3600 >> /mnt/a3/app.log &; rm /mnt/a3/app.log
lsof +L1; ls -l /proc/PID/fd; : > /proc/PID/fd/1

iostat -xzN 1 5; pidstat -d 1 5                     # 窗口 B；窗口 A：stress-ng --hdd 1 --hdd-opts dsync -t 60s
sudo systemd-run --scope -p IOWriteBandwidthMax="/dev/loop0 1M" stress-ng --hdd 1 --hdd-opts direct --hdd-bytes 30M --temp-path /mnt/a3 -t 40s
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 保留块比例 | `tune2fs -l` | 5% | ✅ 5120 / 102400 |
| 普通用户写满 | `dd`、`df -h` | ENOSPC，Size − Used ≠ 0 | ✅ 81MiB 后报错；Size 89M、Used 82M |
| root 还能写 | `sudo dd` | ≈ 5M | ✅ 5.0 MiB |
| 硬链接 | `ls -li locale-archive*` | inode 号相同 | ✅ 都是 8422678，链接数 2 |
| inode 满 | `df -h`、`df -i` | 块 1%、inode 100% | ✅ Use% 1%，IUse% 100%（1976） |
| 新建 vs 追加 | `echo >`、`echo >>` | 新建失败、追加成功 | ✅ |
| 删了不释放 | `df -h`、`du -sh` | df 不变、du 变小 | ✅ df 61M，du 54K |
| 找占用进程 | `lsof +L1` | NLINK 0、deleted | ✅ sleep，FD 1w，NODE 13 |
| 截断释放 | `: > /proc/PID/fd/1` | df 回落 | ✅ 61M → 55K |
| 磁盘压力 | `iostat -xzN` | 判断是否瓶颈 | rlm-root w/s ~700、16MB/s、w_await 1.0ms、aqu-sz 0.7、%util 72%；CPU sy 81～91% → **磁盘不是瓶颈** |
| 进程 I/O | `pidstat -d` | 找到写入者 | ✅ stress-ng-hdd ~14MB/s |
| 限速后 wa | `vmstat`、`iostat` | wa 升高 | ✅ wa 96～98、b 1、bo ≈ 960；loop0 w_await 0.4ms、%util 0 |
| 清理 | `losetup -a` | 无输出 | ✅ |

## 踩坑

- `mkfs.xfs -q -N 2000` → `extra arguments`：xfs 的 `-N` 是"只显示不格式化"，2000 成了多余参数 → 同一字母在不同 mkfs 里含义不同；inode 实验用 ext4。
- 100M 镜像 Size 89M → 元数据（inode 表约 6M、日志、位图）；改 `-N 2000` 后 Size 94M（inode 表变小）。
- IUsed 比 du 多 9 → ext4 保留 inode（1～10 号，只有 2 号根目录能被 du 数到）。
- `find` 列出两个 220M locale-archive → 硬链接，实际只占 220M。
- `stress-ng -hdd` → 单横线被拆成 `-h`，打印帮助；`dsysn` / `dsysc` 拼错，报错里列出了正确选项。
- 面试答案里写 `: > 日志 | truncate -s 0 日志` 表示"或" → `|` 在 shell 里是管道，文字里写"或"。
- pidstat 的 iodelay 为 0 → 需开启 `kernel.task_delayacct`（5.14 起默认关闭）。
- 第 4 步巡检：删除 A2 留下的 `~/ddtest`（1.5G）；`/var/crash` 里有 10-22 的 kdump：`BUG: ENTRY_TRAMPOLINE stack guard page was hit` / `Oops: stack guard page`，内核入口栈溢出（推测：缺页异常递归，虚拟化环境偶发），只此一次，已删除。生产环境应保留 vmcore 交厂商分析。

## 卡点 / 没想明白的

- `ls -i`、`df -B1K`、`mount -o`、`du --inodes`、iostat / pidstat / losetup / stress-ng 的参数（见关键概念和速查表）。
- direct 和 dsync 的区别（见关键概念）。
- iostat 输出太宽 → `iostat -dxsy 1 3`、指定设备、或 awk 只取几列。
- 面试题：题 4 把 await 阈值说成"大于 1ms"（要看介质：SSD ≈ 1ms 以下、机械盘十几 ms，和平时的基线比）；题 5 只说了"要看 iostat"，缺少"wa 的定义 + 两种反例 + 换盘前要先证明设备饱和"。

## 回忆题

1. （排障）`df -h` 还有 40% 空间却报 `No space left on device`，怎么确认、怎么定位、怎么清理？
<details><summary>答案</summary>

`df -i` 看 IUse%；`du --inodes -x --max-depth=1 目录 | sort -n` 一层层找文件最多的目录；`find 目录 -type f -mmin +60 -delete` 清理（不用 `rm *`，参数太多会 `Argument list too long`）；根因：程序不清理会话 / 缓存 → 加定时清理。
</details>

2. （排障）df 满、du 小，不能重启服务，怎么释放？
<details><summary>答案</summary>

`lsof +L1` 找到 NLINK 为 0 的文件和 PID、FD → `ls -l /proc/PID/fd` 确认 → `: > /proc/PID/fd/N` 截断；以后清空日志用 `: > 文件`，检查 logrotate（postrotate reload 或 copytruncate）。
</details>

3. （概念）iowait 很高就是磁盘慢吗？
<details><summary>答案</summary>

不一定。wa = CPU 空闲且有进程在等 I/O。磁盘空闲时也可能很高（cgroup 限速、NFS 无响应：实验里 wa 97%、设备 %util 0），磁盘很忙时也可能很低（CPU 忙，没有空闲可记）。看 iostat 的 await、aqu-sz，用 `pidstat -d` 找进程，确认设备饱和再谈换盘。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
