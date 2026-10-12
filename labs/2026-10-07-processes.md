# 进程管理：ps、top、信号、作业、nice、/proc

- 日期：2026-10-07（W2-D4）
- 考点：RHCSA EX200 —— 找出并终止占用资源高的进程；调整进程调度优先级
- 环境：Rocky Linux 10（VirtualBox）
- 状态：完成
- 来源：根据课程对话整理

## 目标

会定位 CPU/内存占用高的进程，正确终止进程，调整优先级。

## 关键概念

- 进程树根是 PID 1（systemd）。
- 状态：R 运行/排队，S 睡眠，**D 等 I/O（kill -9 也杀不掉，查存储）**，**Z 僵尸（已死，kill 无效，处理父进程）**，T 暂停。
- load average：1/5/15 分钟平均的 R+D 进程数，**和核心数（`nproc`）比**；第一个数大于最后一个数 = 正在上升。
- top 第 3 行：`wa` 高 + `us` 低 = 磁盘瓶颈；`st` 高 = 宿主机资源紧张。
- 内存看 `available`，不看 `free`。
- 信号：1 HUP（常用于重载配置）、2 INT（Ctrl-C）、9 KILL（不可捕获）、15 TERM（默认）、19 STOP、20 TSTP（Ctrl-Z）。**先 TERM，等几秒，不行再 KILL。**
- `pkill` 是部分匹配，先 `pgrep -a` 看会匹配到谁。
- nice -20～19，越小优先级越高；`nice` 启动新程序，`renice` 改运行中的进程（跟 PID）；普通用户只能调高 nice 值且不能调回。
- `/proc/PID/fd/` 可恢复"已删除但仍被打开"的文件；只在启动时读一次的配置文件不适用。

## 关键步骤

```bash
ps -eo pid,user,%mem,rss,cmd --sort=-rss | head -6   # 第一行是表头，所以 head -6 才是 5 个进程
pgrep -a sshd
nice -n 19 /opt/backup.sh                             # 低优先级启动
renice -n 10 -p 3456                                  # 改运行中进程
sleep 1000 &; jobs; fg %1; # Ctrl-Z; bg %1; kill %1
(sleep 1 & exec sleep 300) &                          # 制造僵尸进程
cp /proc/$TPID/fd/<编号> ~/recovered.txt              # 从 /proc 恢复误删文件
sudo pkill -u harry                                   # 强制某用户下线
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 实验 1～7 | 按讲义 | — | ✅（自述已完成） |
| 普通用户把 nice 从 15 改回 0 | `renice -n 0 -p PID` | 拒绝 | ✅ Permission denied |

## 踩坑

- load `8.50, 6.20, 3.10`（4 核）答成"基本满负载、下降趋势" → 8.5 是核心数 2 倍多 = 严重过载；越近越高 = 正在**上升**。
- 低优先级启动写成 `renice -n 15 /opt/backup.sh` → 启动用 `nice`，`renice` 只接 PID。
- 前 5 个进程用 `head -5` → 少一个，表头占一行。
- /proc 恢复配置文件：要先 `grep` 找到 `(deleted)` 的那个 fd、恢复到原路径并恢复属主权限、加 sudo。
- 热身：`sed -i."bak..."-E` 后缀和 `-E` 之间没空格 → 被拼成一个参数，备份名多出 `-E`，替换也没执行。
- 热身：`tee -a` 这次写对了（之前三次覆盖/追加出错）。

## 卡点 / 没想明白的

- 无

## 回忆题

1. （概念）为什么终止进程要先 `kill` 再 `kill -9`？
<details><summary>答案</summary>

TERM 可被捕获，进程能保存数据、清理锁和临时文件；KILL 直接终止，可能丢数据、留锁文件。
</details>

2. （操作）以最低优先级启动 `/opt/report.sh`；把 PID 3456 的 nice 改为 10。
<details><summary>答案</summary>

`nice -n 19 /opt/report.sh`；`renice -n 10 -p 3456`
</details>

3. （排障）负载很高但 `us` 只有 5%、`wa` 60%，问题在哪？
<details><summary>答案</summary>

磁盘 I/O 瓶颈（大量进程在等 I/O，D 状态），查存储而不是杀进程。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
