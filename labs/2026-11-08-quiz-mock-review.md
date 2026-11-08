# 小测与模拟面试复盘（A1–A4）

- 日期：2026-11-08
- 考点：RHCSA EX200 综合复习 —— cron、CPU/内存/负载、磁盘、网站打不开；面试排障表达
- 环境：无实验（口头小测与模拟面试，未执行命令）
- 状态：完成

## 目标

检查 A1–A4 的理解深度，练习面试作答的结构：先分流，再说命令，每个结论后面说"还有什么可能"。

## 关键步骤

无实验命令。

## 验证

无。

## 踩坑

无。

## 卡点 / 没想明白的

**小测（5 题，A1–A4）批改要点**

- 第 1 题（cron）：五个字段是"与"的关系。`*/15 8-18 * * 1-5` 会让 18:15 到 18:45 也执行，因为 18 点整个小时都匹配。边界要自己定：`*/15 8-17` 对应 08:00 到 17:45。
- 第 2 题（CPU/wa）：wa 是 id（空闲时间）的子集，sy 占满时 id 和 wa 都接近 0。解释 sy 高时要落到实验本身：单 vCPU、同步写、进程多为 R 状态。
- 第 3 题（磁盘/NFS）：wa 高而本地盘空闲，只能说明请求卡在设备之前，不能直接断定是 NFS。cgroup 限速也会造成同样现象。挂载选项 `hard` 或 `soft` 决定现象是卡住还是报错（hard 无限重试，进程进入 D 状态）。
- 第 4 题（403）：排除 SELinux 后，还要查目录索引（没有 index 文件且禁止 Indexes 时返回 403）、访问控制规则（`Require`），以及运行 httpd 的用户（Rocky 默认是 apache，未验证）。`sudo -u apache cat 文件` 可以直接验证权限。
- 第 5 题（firewalld）：`--permanent` 只改永久配置，不影响运行时规则。`--reload` 用永久配置重建运行时规则，所以之后端口才真正关闭。结果依赖 zone 的默认动作（A4 实验里是 REJECT）。

**作答习惯**

- 找到一个合理解释就停，没有列出其他可能。这是 5 题里反复出现的问题。

**命令回忆缺口**

- `nfsstat -m` / `nfsstat -rc`：查看 NFS 挂载选项和 RPC 重传（未在实验中用过）。
- `showmount -e 服务端`：查看服务端导出的目录（未验证 NFSv4 下的结果）。
- `httpd -T` 的作用：置信度低，未查 `man httpd`，不要依赖它。
- 配置拒绝和权限拒绝的 error_log 区分：`client denied by server configuration`（置信度高，未在实验中见过）和 `file permissions deny server access`（A3 笔记里的 AH00132）。

## 面试题（模拟，5 道，已作答批改）

1. （排障）一个内部 Web 服务访问不了，怎么排查？
<details><summary>批改要点</summary>

- 分流框架对：`curl -v` 看报错，按 refused、No route、timed out 分流。
- 缺口：先确认范围（一台客户端还是全部）和服务状态（`systemctl status`）；refused 可能是防火墙 `tcp-reset`，不等于没有监听；用 `ss -lntp 'sport = :80'` 精确过滤，避免 grep 子串匹配；`ip neigh` 在客户端执行；同网段查目标 IP，跨网段查网关（先 `ip route get`）；ARP 属于二层（链路层）；timed out 时在服务端抓包，看 SYN 是否到达；firewall-cmd 同时对比运行时和永久规则；curl 加 `-m` 限时。
</details>

2. （排障）`df` 显示有空间，但写文件报 `No space left on device`，怎么排查？
<details><summary>批改要点</summary>

- 方向对：先怀疑 inode，用 `df -i` 检查。
- 缺口：先用 `df -h 路径` 确认写入路径所在的挂载点；用"新建失败、追加成功"区分 inode 满和块满；inode 满后用 `du --inodes -x --max-depth=1` 定位，用 `find ... -mmin +60 -delete` 清理（不用 `rm *`），再查根因；df 正常时还要查配额（报错是 `Disk quota exceeded`，与 ENOSPC 不同）、tmpfs 的 size 限制、ext4 保留块。
</details>

3. （排障）服务每隔几天自己挂掉，应用日志无报错，怎么查？
<details><summary>批改要点</summary>

- 方向对：被 OOM killer 杀掉。
- 缺口：先用 `systemctl status` 看退出状态（`status=9/KILL`）；`journalctl -k --since "7 days ago"` 加时间范围；读懂 `constraint`（整机或 cgroup）、被杀进程的 PID，与主进程 PID 对照（子进程被杀时主进程可能还在）；周期性故障要查同一时间的定时任务（`systemctl list-timers`、cron）。
</details>

4. （操作/排障）脚本手动执行正常，放进 crontab 不执行，怎么查？
<details><summary>批改要点</summary>

- 对：绝对路径（PATH 极简）、`\%` 转义、脚本不加载 `.bashrc` 里的环境变量。
- 缺口：先确认命令有没有执行（`/var/log/cron` 或 `journalctl -t CROND`，或在行末加 `>> 日志 2>&1`）；修复后怎么验证（等一个周期，看日志）。
</details>

5. （概念）`uptime` 显示负载 8.0、7.5、3.2，服务器 4 核，怎么判断？
<details><summary>批改要点</summary>

- 对：负载除以核数，8.0 ÷ 4 = 2，持续超过 1 说明有进程排队；1 分钟值大于 15 分钟值，说明负载正在上升。
- 必须补：负载统计的是 R（可运行）和 D（不可中断睡眠，等 I/O）两类进程的数量。负载高不一定是 CPU 忙，也可能是大量进程在等磁盘。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
