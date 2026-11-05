# 排障（一）：CPU、内存、负载

- 日期：2026-11-05（A2）
- 考点：面试高频"负载高怎么查""内存够不够""服务莫名挂掉"；RHCSA —— 找出并终止高 CPU / 内存进程
- 环境：Rocky Linux 10.2（VirtualBox），node1，**1 个 vCPU**、1.7G 内存、4G swap，宿主机 SSD；stress-ng（EPEL）
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

掌握按资源排查的方法（USE），看懂负载、top 的 CPU 行、vmstat、free，亲手制造 CPU 满载、I/O 压力和一次 cgroup 内的 OOM。

## 关键概念

- **USE 方法**（Brendan Gregg）：每种资源看 Utilization 使用率、Saturation 饱和（排队）、Errors 错误。顺序：负载 → CPU / 内存 / 磁盘 / 网络 → 定位进程 → 看日志。
- **负载** = 平均处于 **R**（运行或排队等 CPU）和 **D**（不可中断睡眠，等 I/O）的进程数；除以核数，持续 > 1 就是排队。三个数看趋势：1 分钟先涨先落。
- top CPU 行：us 用户态、sy 内核态、ni、id、**wa = 空闲时间里有进程在等 I/O**、hi/si 中断、**st = 虚拟机的 CPU 被宿主机拿走**。
- **wa 是空闲时间的一部分**：CPU 忙（如 sy 满）时 wa 显示不出来 → wa 低不代表磁盘没问题；虚拟机里 I/O 开销常表现为 sy；判断磁盘要看 iostat。
- vmstat（virtual memory statistics）：r 运行+排队、b 等 I/O、si/so 换入/换出、bi/bo 读/写块、cs 上下文切换；**第一行是开机以来平均值，看第二行起**。
- free：free 小正常（空闲内存拿去做缓存），**看 available**；swap 用了但 si/so ≈ 0 没关系，si/so 持续非 0 才是内存不够。
- **OOM killer**：内存耗尽时内核按 oom_score 杀进程，痕迹只在内核日志；`oom_score_adj` -1000～1000，1000 = 优先杀。
- cgroup（control group）内存上限：`MemoryMax=` 超限只在该 cgroup 内 OOM（`CONSTRAINT_MEMCG`），整机不受影响；Docker 同理。
- ps 的 %CPU 是进程生命周期内的平均值，看"现在"用 top。
- `dd ... oflag=direct`：oflag = output flags，direct 绕过缓存；`--hdd-opts dsync` 每次写入都等落盘。
- 多级进程（主进程 + 工作进程）里被 OOM 杀的常是子进程，主进程可能照常运行、退出码 0、悄悄重启工作进程 → 只有内核日志有痕迹。

## 关键步骤

```bash
nproc; uptime; free -h
stress-ng --cpu 2 --timeout 90s &       # CPU 满载
vmstat 1 5                              # r、us
top                                     # P 按 CPU，M 按内存，1 每个核
ps -eo pid,stat,%cpu,cmd --sort=-%cpu | head -4
stress-ng --hdd 2 --hdd-opts dsync --timeout 60s    # 窗口 A
vmstat 1 10; ps -eo pid,stat,%cpu,cmd | awk '$2 ~ /D/'   # 窗口 B
sudo systemd-run --scope -p MemoryMax=200M -p MemorySwapMax=0 stress-ng --vm 1 --vm-bytes 400M --timeout 20s
journalctl -k --since "5 min ago" | grep -i -E 'oom|killed'
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| CPU 满载时 r | `vmstat 1 5` | ≈ 2（1 核 2 进程） | ✅ 2～3，us 85～92，id 0 |
| 单核平分 | top | 每个 stress-ng ≈ 50% | ✅ 48% |
| 负载趋势 | `uptime` 前 / 中 / 后 | 1 分钟先涨先落 | ✅ 0.12 → 1.55 → 0.58,1.00,0.56 |
| vmstat 第一行 | `vmstat` | 与后面不同 | ✅ us 1 / id 96 |
| I/O 压力 | `vmstat`、`ps` D | （预期 wa 高、b > 0） | ⚠️ 实际 **sy 84～94**、wa 0～3、b 基本 0、bo ≈ 20MB/s、负载 2.8，偶尔抓到 D |
| cgroup OOM | 内核日志 | Killed process，CONSTRAINT_MEMCG | ✅ anon-rss 183MB ≈ 上限 200M |
| 整机不受影响 | `free -h` | available 正常 | ✅ 1.2Gi |
| swap 用了但不换页 | `vmstat` | si/so = 0 | ✅ swap 524K，si/so 0 |

## 踩坑

- `dd bs=1M count=1500 oflag=direct` 2.5 秒就写完（635MB/s，宿主机 SSD）→ 来不及观察 → 改用持续 60 秒的 `stress-ng --hdd`，开两个终端：一个制造压力、一个观察。
- I/O 实验结果与"高 wa、多 D"的典型表现不同 → 单核虚拟机里内核处理同步写（XFS 日志、虚拟磁盘模拟）占满 CPU 显示为 sy，虚拟磁盘响应快，进程大多是 R → wa 无空闲可记。课上的对照表适用于"磁盘慢且 CPU 有空闲"的典型情况。想复现高 wa 可用 cgroup 限速磁盘（待做）。
- `ps` 自己显示 66.6% CPU → 生命周期平均值的假象。
- OOM 后 `systemd-run` 退出码 0：被杀的是 stress-ng 的子进程（1019642），主进程（1019640）报告 passed（推测：stress-ng 主动处理 OOM）。
- `stress-ng ... $`（应为 `&`）；`wait` 被后台输出打断成 `wa`。

## 卡点 / 没想明白的

- vmstat、oflag 的全称（见关键概念）；`-p` = `--property`；`--vm N` 启动 N 个内存压力进程，`--vm-bytes` 每个申请多少。
- 面试题 1 缺排查顺序和"定位到进程后看日志"；题 2 把"4 核负载 4"说成不忙（其实是刚好满负荷）；题 5 的 grep 写成 `'oom | killed'`（带空格匹配不到 `oom-killer`）。

## 面试回答模板："负载很高怎么查"

1. `uptime` + `nproc`：负载 ÷ 核数，看三个数是在升还是降。
2. `top`（或 `vmstat 1`）看 CPU 行：us 高 → 应用计算；sy 高 → 内核忙（I/O、网络、频繁切换）；wa 高 / vmstat b 多 → 等磁盘；st 高 → 虚拟机被宿主机抢 CPU。
3. 定位进程：top 按 P / M，`ps --sort=-%cpu`，D 状态进程 `ps ... | awk '$2 ~ /D/'`。
4. 内存：`free -h` 看 available，`vmstat` 看 si/so；`journalctl -k | grep -i oom`。
5. 磁盘：`iostat -x 1`（下一课），wa 低不等于磁盘没问题。
6. 找到进程后：看它的日志、`journalctl -u 服务`、最近有没有变更，再决定扩容、限流、优化还是重启。

## 回忆题

1. （概念）负载很高但 CPU 几乎空闲，可能是什么？怎么确认？
<details><summary>答案</summary>

大量进程处于 D 状态等 I/O（磁盘慢，或 NFS 服务端无响应）。`vmstat` 看 b 列、`ps -eo pid,stat,cmd | awk '$2 ~ /D/'` 找 D 进程、`iostat -x` 看磁盘、`mount | grep nfs` 查网络存储。
</details>

2. （排障）free 只剩 100MB，同事说内存不够要加内存，你怎么判断？
<details><summary>答案</summary>

看 available 而不是 free（cache 可回收）；`vmstat 1` 看 si/so 是否持续非 0；查内核日志有没有 OOM。available 充足、没有换页 → 不缺内存。
</details>

3. （排障）一个服务隔几天自己挂掉 / 工作进程重启，应用日志里没有报错，怀疑什么？怎么确认、怎么处理？
<details><summary>答案</summary>

被 OOM killer 杀了：`journalctl -k | grep -i -E 'oom|killed'` 或 `dmesg -T | grep -i oom` 看被杀进程和 constraint（整机 / cgroup）；`systemctl status` 看重启次数。处理：查内存泄漏、调整服务内存上限或 JVM 堆大小、加内存。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
